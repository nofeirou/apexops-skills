---
name: apexops-action
description: ApexOps 修复执行。当巡检报告确认了具体问题、用户明确要求执行修复/重启/清理等写入操作时触发。调用 Gateway API 对目标资产执行预定义修复剧本。注意：只读诊断/排查已归入 apexops-check，本 skill 仅处理有副作用的写入操作。触发关键词：修复、重启、清理、扩容、恢复、执行、操作、remediate、action、处理。
version: 1.0.4
user-invocable: true
metadata: {"openclaw": {"requires": {"env": ["APEXOPS_URL", "APEXOPS_TOKEN"]}, "primaryEnv": "APEXOPS_TOKEN", "emoji": "🔧", "os": ["darwin", "linux"]}}
---

# ApexOps 诊断与修复

## 安全规则（最高优先级）

**这些规则是硬性约束，任何情况下不可违反。**

### 禁止操作

1. **禁止创建/修改/删除资产**：绝不调用 `POST|PUT|DELETE /api/assets`。资产由管理员在 Web UI 手动注册。Agent 只能操作已存在的资产。
2. **禁止创建/修改/删除凭证**：绝不调用 `POST|PUT|DELETE /api/credentials`。所有凭证由 Gateway 的 Credential Vault (AES-256-GCM) 统一管理。
3. **禁止创建/修改/删除 API Token**：绝不调用 `/api/auth/token` 的增删改接口。
4. **禁止猜测密码**：绝不尝试猜测、推断、暴力破解任何密码。凭证由 Gateway 加密存储并自动注入执行器。如果连接失败（认证错误），报告给用户，**不要尝试其他密码**。反复尝试会导致账户锁定。
5. **禁止在无资产时自行注册**：如果找不到需要的资产，告诉用户去 Web UI 添加，绝不自行为之。

### VMware 操作边界

**所有 ESXi 主机和 VM 操作必须通过 vCenter (govc) 执行。**

| 能力 | 方式 | 示例 |
|------|------|------|
| ✅ 能做 | govc playbook (vsphere 协议) | 服务管理、NTP、主机信息、VM、快照、维护模式 |
| ❌ 不能做 | govc 不支持 | ESXi 本地磁盘清理、主机 shell 命令、主机本地文件操作 |

如果操作不在 govc 能力范围内（如主机磁盘满需要清理本地文件），**直接告诉用户需要手动处理**，说明原因和替代方案。不要尝试 SSH 到 ESXi 主机。

### 操作规范

6. **高风险操作必须二次确认**：`risk: high` 或 `critical` 的操作，执行前向用户明确说明操作内容、影响范围、不可逆性
7. **不可逆操作警告**：`vm snapshot-delete`、`power_off_vm`、`destroy_vm`、`kill_process` 等操作，必须告知"此操作不可逆，执行后无法回滚"
8. **限制操作范围**：一次只修复一个资产、一个问题。批量操作逐项确认
9. **记录操作结果**：每次操作后报告 exit_code、stdout、stderr
10. **修复前确认**：总是先执行诊断确认问题，再建议修复方案
11. **critical 操作不可跳过确认**：即使紧急故障，`risk: critical` 级别操作也须用户确认

## ApexOps API

```bash
APEXOPS_URL="${APEXOPS_URL}"
APEXOPS_TOKEN="${APEXOPS_TOKEN}"

# 诊断（单资产，所有区域）
curl -s -H "Authorization: Bearer $APEXOPS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$APEXOPS_URL/api/skills/diagnose" \
  -d '{"asset_name":"linux-web-01"}'

# 诊断（指定区域: cpu/memory/disk/network/process）
curl -s -H "Authorization: Bearer $APEXOPS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$APEXOPS_URL/api/skills/diagnose" \
  -d '{"asset_name":"linux-web-01","area":"disk"}'

# 修复
curl -s -H "Authorization: Bearer $APEXOPS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$APEXOPS_URL/api/skills/remediate" \
  -d '{"asset_name":"linux-web-01","playbook":"restart_service","variables":{"service":"nginx"}}'

# 查询可用 playbook
curl -s -H "Authorization: Bearer $APEXOPS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$APEXOPS_URL/api/skills/remediate" \
  -d '{"asset_name":"linux-web-01","playbook":"__list__"}'
```

## 诊断

诊断调 `POST /api/skills/diagnose`，Gateway 按 YAML profile 执行。返回格式：

```json
{
  "asset_name": "linux-web-01",
  "areas": {
    "cpu": {
      "description": "CPU 诊断",
      "commands": [
        {"name": "cpu_top", "stdout": "...", "stderr": "", "exit_code": 0}
      ]
    }
  }
}
```

### 诊断区域

**通用 Linux 7 区**（`linux`/`kvm`/`pve` 等 SSH 资产）：

| 区域 | 用途 | 代表命令 |
|------|------|----------|
| `cpu` | CPU 负载/TOP进程/上下文切换/中断 | `ps aux`, `vmstat`, `/proc/interrupts` |
| `memory` | 内存使用/TOP进程/OOM历史/Slab | `free -h`, `dmesg`, `slabtop` |
| `disk` | 磁盘使用率/inode/IO/大目录 | `df -h`, `iostat`, `du -h` |
| `network` | TCP 连接/监听端口/路由/DNS/接口统计 | `ss -s`, `ss -tlnp`, `ip -s link` |
| `process` | 失败服务/僵尸进程/高资源进程 | `systemctl --failed`, `ps aux` |
| `system` | 内核版本/运行时间/SELinux/防火墙/安全参数 | `uname -a`, `getenforce`, `sysctl` |
| `logs` | dmesg 错误/journal 错误/认证失败/cron 错误 | `dmesg`, `journalctl`, `grep auth.log` |

**Windows 7 区**（`windows` 等 WinRM 资产）：

| 区域 | 用途 |
|------|------|
| `performance` | CPU/内存压力/磁盘队列 |
| `services` | 核心服务/AD服务状态/DCDiag |
| `replication` | AD 复制摘要/队列/FSMO/SYSVOL |
| `disk` | 卷使用率/物理磁盘 |
| `network` | DNS 区域/转发器/缓存/IP/防火墙 |
| `security` | 审计策略/账户锁定/Kerberos/NTLM |
| `logs` | 系统错误/目录服务错误/DFSR 事件 |

**特殊区域**（特定资产类型）：

| 资产类型 | 诊断区域 |
|----------|----------|
| `vmware_vcenter` | `health`, `hosts_detail`, `vms_detail`, `storage`, `cluster`, `alarms`, `performance`, `security` |
| `vmware_esxi` | `cpu`, `memory`, `disk`, `network`, `process`, `services`, `vms`, `sensors` |
| `pve` | `cpu`, `memory`, `disk`, `storage`, `network`, `process`, `backup` |
| `kvm` | `cpu`, `memory`, `disk`, `network`, `vm` |
| `citrix_netscaler` | `performance`, `memory`, `disk`, `network`, `services`, `logs` |
| `citrix_vdi` | `site`, `delivery`, `catalog`, `sessions`, `events` |
| `dell_unity` | `health`, `capacity`, `hardware`, `performance` |
| `nutanix` | `cluster`, `storage`, `performance`, `network`, `vm` |
| `xenserver` | `system`, `vm`, `storage`, `network`, `performance` |
| `network` | `cpu`, `memory`, `storage`, `network`, `system` |
| `brocade_san` | `system`, `fabric`, `ports`, `hardware`, `performance` |
| `veeam` | `backup_jobs`, `storage`, `infrastructure`, `tape`, `system` |

### VMware vCenter 8 区详细说明

| 区域 | 诊断内容 | 对应巡检异常 |
|------|----------|-------------|
| `health` | 平台健康：核心服务状态/证书到期/NTP 同步/DNS/备份 | services stopped, cert expiring, NTP out of sync |
| `hosts_detail` | ESXi 主机：CPU%/Mem%/传感器/服务/Lockdown/超分配 | host CPU/mem high, sensor alert, connection lost |
| `vms_detail` | VM 详情：电源/CPU/Mem/磁盘/网卡/VMware Tools/快照 | VM resources, tools not running |
| `storage` | 存储：Datastore 使用率/VM 分布/存储集群 | datastore >75%, inaccessible |
| `cluster` | 集群：DRS/HA/EVC/总资源/网络 | DRS disabled, HA disabled |
| `alarms` | 告警+事件交叉分析：活动告警 + 3 天事件 | red alarms, events surge |
| `performance` | 性能 TOP20：CPU Ready/Balloon/Swap（最近 1h） | performance degradation |
| `security` | 等保 2.0 L3：Lockdown/NTP/Syslog/SSH/Shell 自动检查 | compliance fail |

---

## VMware 巡检→修复路由表（核心）

巡检发现异常时，按此表确定诊断目标和修复方案。**所有 ESXi 主机和 VM 操作均通过 vCenter (govc) 执行，不需要直接 SSH 到 ESXi 主机。**

**关键**：所有 ESXi 主机和 VM 操作都通过 vCenter (vsphere 协议 + govc)，**不需要**单独注册 ESXi 资产或 SSH 直连主机。

| 巡检发现 | Severity | 诊断目标 | 诊断区域 | 诊断关键命令 | 修复 Playbook | 风险 |
|----------|----------|----------|----------|-------------|--------------|------|
| vCenter 核心服务停 (vpxd/vapi) | 🔴 | vCenter | `health` | services_status | `vc_start_service` → service=xxx | low |
| vCenter 全部服务异常 | 🚨 | vCenter | `health` | services_status | `vc_restart_vmon` (停止+启动全部) | **high** |
| vCenter 证书 <30 天 | 🔴 | vCenter | `health` | certificate | `vc_renew_cert_guide` (手动步骤) | low |
| vCenter NTP 不同步 | 🔴 | vCenter | `health` | ntp_dns | `vc_fix_ntp` | low |
| vCenter 无备份 | 🚨 | vCenter | `health` | backup_status | 手动创建 VAMI 备份计划 | — |
| vCenter 磁盘满 | 🔴 | vCenter | `health` | (用`vc_disk_usage`) | `vc_clear_logs` / `vc_clear_core_dumps` | low |
| vCenter VCDB 肥大 | 🔴 | vCenter | — | — | `vc_vacuum_vcdb` | **high** |
| 主机 CPU >90% | 🔴 | vCenter | `hosts_detail` | hosts_overview | 无自动修复; 建议: vMotion 迁移 VM / 检查 VM 内进程 | — |
| 主机 CPU >95% | 🚨 | vCenter | `hosts_detail` | hosts_overview | 紧急: vMotion 迁移 VM（govc vm.migrate） | — |
| 主机内存 >90% | 🔴 | vCenter | `hosts_detail` | hosts_overview | 无自动修复; 建议: Balloon 检查 / vMotion 迁移 VM | — |
| 主机断连 (disconnected) | 🚨 | vCenter | `hosts_detail` | hosts_overview | 检查物理网络; 若需重启 mgmt agents 用 `vc_host_service_control` | — |
| 主机传感器异常 (温度/风扇) | 🔴 | vCenter | `hosts_detail` | hosts_overview | 硬件报修; 建议迁移 VM 到其他主机 | — |
| 主机维护模式 (非窗口期) | ⚠️ | vCenter | `hosts_detail` | hosts_overview | 确认维护窗口; 退出维护模式 | — |
| 主机 Lockdown 禁用 | ⚠️ | vCenter | `security` | compliance_audit | 手动启用 Lockdown Mode (Normal) | low |
| 主机 NTP 关闭/未配置 | 🔴 | vCenter | `hosts_detail` | hosts_ntp_detail | `vc_host_ntp_set` → host=IP, server=ntp.aliyun.com | medium |
| 主机 SSH/Shell 开启 | ⚠️ | vCenter | `hosts_detail` | hosts_services_detail | `vc_host_service_control` → action=stop, service=TSM-SSH | medium |
| 主机 Syslog 未配置 | ⚠️ | vCenter | `security` | compliance_audit | `vc_host_service_control` → action=start, service=vmsyslogd | medium |
| 数据存储 >75% | ⚠️ | vCenter | `storage` | datastore_detail | 无自动修复; 建议: 迁移 VM / 清理快照 / 扩容 | — |
| 数据存储 >85% | 🔴 | vCenter | `storage` | datastore_detail | 紧急: `vc_vm_snapshot_remove_all` 清理过期快照 | **critical** |
| 数据存储 >95% | 🚨 | vCenter | `storage` | datastore_detail | 濒临只读; 立即清理快照或 vMotion 迁移 VM | **critical** |
| 数据存储不可访问 | 🚨 | vCenter | `storage` | datastore_detail | 通过 vCenter 重新扫描存储适配器; 检查 FC/iSCSI | — |
| VM CPU >90% | 🔴 | vCenter | `vms_detail` | vm_detail:VMNAME | 无自动修复; 建议: 增加 vCPU / 检查 VM 内进程 | — |
| VM 内存 >90% | 🔴 | vCenter | `vms_detail` | vm_detail:VMNAME | 无自动修复; 建议: 增加内存 / 检查内存泄漏 | — |
| VM Tools 未运行 | ⚠️ | vCenter | `vms_detail` | vm_tools | 手动安装/升级 VMware Tools | — |
| VM 快照 >3 个 / >7 天 | 🔴 | vCenter | `vms_detail` | vm_names (先找 VM) | `vc_vm_snapshot_remove` → vm=VM名, snap_name=快照名 | **high** |
| VM 快照 >30 天 / >10GB | 🚨 | vCenter | `vms_detail` | vm_names (先找 VM) | `vc_vm_snapshot_remove_all` → vm=VM名 (确认后执行) | **critical** |
| 多个 VM CPU Ready >10% | 🔴 | vCenter | `performance` | perf_top20 | 无自动修复; 根因: CPU 超分配 → 减少 vCPU 或增加主机 | — |
| VM Balloon >0 | ⚠️ | vCenter | `performance` | perf_top20 | 诊断主机内存压力 → 迁移 VM | — |
| VM Swap >0 | 🚨 | vCenter | `performance` | perf_top20 | 严重内存不足; 立即: 增加主机内存或迁移 VM | — |
| CPU 超分配 >2:1 | ⚠️ | vCenter | `hosts_detail` | hosts_overcommit | 无自动修复; 建议: 增加主机或回收闲置 vCPU | — |
| 内存超分配 >1:1 | ⚠️ | vCenter | `hosts_detail` | hosts_overcommit | 无自动修复; 建议: 增加内存或回收闲置分配 | — |
| DRS 禁用 | 🔴 | vCenter | `cluster` | cluster_config | 手动启用 DRS（vSphere Client → 集群 → 配置） | — |
| HA 禁用 | 🚨 | vCenter | `cluster` | cluster_config | 手动启用 HA（vSphere Client → 集群 → 配置） | — |
| 红色告警未确认 | 🔴 | vCenter | `alarms` | active_alarms | 确认告警或执行对应修复; 见 `suggested_actions` | — |
| 事件暴增 (alarms 少 events 多) | ⚠️ | vCenter | `alarms` | recent_events | 阈值校准; 建议调整告警策略 | — |
| VM 快照 >3 个 / >7 天 | 🔴 | vCenter | `vms_detail` | vm_detail:VMNAME | `snapshot_delete` → 指定 VM+快照名 | **high** |
| VM 快照 >30 天 / >10GB | 🚨 | vCenter | `vms_detail` | vm_detail:VMNAME | `snapshot_delete_all` → 确认后删除全部 | **critical** |
| 多个 VM CPU Ready >10% | 🔴 | vCenter | `performance` | perf_top20 | 无自动修复; 根因: CPU 超分配 → 减少 vCPU 或增加主机 | — |
| VM Balloon >0 | ⚠️ | vCenter | `performance` | perf_top20 | 诊断主机内存压力 → 迁移 VM | — |
| VM Swap >0 | 🚨 | vCenter | `performance` | perf_top20 | 严重内存不足; 立即: 增加主机内存或迁移 VM | — |
| CPU 超分配 >2:1 | ⚠️ | vCenter | `hosts_detail` | hosts_overcommit | 无自动修复; 建议: 增加主机或回收闲置 vCPU | — |
| 内存超分配 >1:1 | ⚠️ | vCenter | `hosts_detail` | hosts_overcommit | 无自动修复; 建议: 增加内存或回收闲置分配 | — |
| DRS 禁用 | 🔴 | vCenter | `cluster` | cluster_config | 手动启用 DRS（vSphere Client → 集群 → 配置） | — |
| HA 禁用 | 🚨 | vCenter | `cluster` | cluster_config | 手动启用 HA（vSphere Client → 集群 → 配置） | — |
| 红色告警未确认 | 🔴 | vCenter | `alarms` | active_alarms | 确认告警或执行对应修复; 见 `suggested_actions` | — |
| 事件暴增 (alarms 少 events 多) | ⚠️ | vCenter | `alarms` | recent_events | 阈值校准; 建议调整告警策略 | — |

---

## 自动闭环流程

巡检→修复的标准执行顺序：

```
巡检报告 (apexops-check)
    ↓ AI 提取所有异常，按严重程度排序
    ↓
用户说"处理" / "修复这些"
    ↓
┌─────────────────────────────────────────┐
│ For each 异常 (按严重度: 🚨 → 🔴 → ⚠️)   │
│                                          │
│ 1. 查路由表 → 确定诊断目标 + 诊断区域      │
│ 2. POST /api/skills/diagnose             │
│ 3. 解读诊断输出 → 定位根因                 │
│ 4. 匹配修复 playbook                      │
│ 5. 告知用户: 问题/根因/方案/风险/影响       │
│ 6. 用户确认后: POST /api/skills/remediate │
│ 7. 报告执行结果                           │
│ 8. （关键问题）再次诊断验证修复效果          │
└─────────────────────────────────────────┘
    ↓
生成修复总结报告
```

### 示例对话

```
User: 处理巡检发现的红色告警

Agent:
  🚨 4 个严重问题需要处理，我来逐项处理。

  ── 问题 1/4: vc01 NTP 不同步 ──
  正在诊断 vCenter → health...
  [诊断结果] ntpd 未运行，NTP 服务器为空
  → 根因: NTP 服务未配置
  → 修复方案: vc_fix_ntp (风险: low)
  执行后 NTP 服务重启并同步。

  ── 问题 2/4: esxi02 主机断连 ──
  正在诊断 esxi02 → services (SSH)...
  [诊断结果] vpxa 未运行，hostd running
  → 根因: vpxa 代理僵死
  → 修复方案: restart_management_agents (风险: medium)
  该操作会重启 ESXi 管理代理，不影响 VM 运行，是否执行？
  ...
```

---

## 修复剧本清单

### VMware vCenter Appliance (vsphere 协议)

**vCenter 自身管理** (VCSA shell):

| Playbook | 风险 | 变量 | 说明 |
|----------|------|------|------|
| `vc_start_service` | low | `service_name` | 启动指定服务 (vmware-vpxd/vsphere-ui/...) |
| `vc_stop_service` | medium | `service_name` | 停止指定服务 |
| `vc_check_services` | low | — | 检查所有服务状态 |
| `vc_restart_vpxd` | **high** | — | 重启 vpxd（vCenter 核心） |
| `vc_restart_vmon` | **high** | — | 重启全部 VCSA 服务 |
| `vc_vacuum_vcdb` | **high** | — | VCDB 数据库 VACUUM |
| `vc_clear_logs` | low | — | 清理 30 天前旧日志 |
| `vc_clear_core_dumps` | low | — | 清理 core dump 文件 |
| `vc_check_ntp` | low | — | 检查 vCenter NTP 状态 |
| `vc_fix_ntp` | low | — | 重启 vCenter NTP 服务 |
| `vc_renew_cert_guide` | low | — | 证书续期手动步骤指引 |
| `vc_disk_usage` | low | — | 检查磁盘使用/大目录 TOP10 |

**ESXi 主机管理** (通过 vCenter govc，不需要直接 SSH 主机):

| Playbook | 风险 | 变量 | 说明 |
|----------|------|------|------|
| `vc_host_info` | low | `host` | 查看主机详细信息（CPU/Mem/网卡） |
| `vc_host_list` | low | — | 列出所有 ESXi 主机 |
| `vc_host_ntp_get` | low | `host` | 查看主机 NTP 配置 |
| `vc_host_ntp_set` | medium | `host`, `server` | 配置主机 NTP 并启动 |
| `vc_host_service_list` | low | `host` | 列出主机所有服务状态 |
| `vc_host_service_control` | medium | `host`, `action`, `service` | 启停主机服务（TSM-SSH/TSM/ntpd/vmsyslogd） |

**VM 快照管理** (通过 vCenter govc):

| Playbook | 风险 | 变量 | 说明 |
|----------|------|------|------|
| `vc_vm_find` | low | `vm` | 按名称关键词查找 VM |
| `vc_vm_snapshot_tree` | low | `vm` | 查看 VM 快照树 |
| `vc_vm_snapshot_remove` | **high** | `vm`, `snap_name` | 删除指定快照（不可逆） |
| `vc_vm_snapshot_remove_all` | **critical** | `vm` | 删除 VM 所有快照（不可逆） |

### ESXi 主机 (已废弃 — 所有操作请通过 vCenter govc playbook)

> ESXi 主机不需要注册为资产，也不需要直接 SSH。所有主机/VM 操作使用上方「vCenter govc」部分的 playbook，通过 vCenter 代理执行。

### linux — 通用 Linux 服务器

| Playbook | 风险 | 变量 | 说明 |
|----------|------|------|------|
| `restart_service` | low | `service` | systemctl restart |
| `check_service` | low | `service` | systemctl status |
| `dmesg_tail` | low | — | 查看最近内核日志 |
| `kill_process` | medium | `process` | pkill -9 / kill -9 |
| `truncate_log` | medium | `log_file` | truncate -s 0 日志 |
| `emergency_cleanup` | low | — | 清理 /tmp + 包缓存 + journal |
| `clear_page_cache` | medium | — | echo 3 > drop_caches |
| `set_swappiness` | low | `value` | 临时调整 swappiness |
| `remount_fs_rw` | **high** | — | 重新挂载只读文件系统为读写 |
| `restart_ssh` | **high** | — | systemctl restart sshd |
| `kill_stuck_nfs` | **high** | `mount_point` | fuser -km + umount -l |
| `clean_old_logs` | low | — | 压缩+删除旧日志 |
| `check_failed_units` | low | — | 查看失败 systemd 单元 |

**common 通用 playbook**: `restart_service`, `check_service`, `check_failed_units`, `dmesg_tail`, `kill_process`, `clear_disk_cache`, `kill_zombie_parent`, `restart_ssh`, `rotate_logs`, `truncate_log`, `check_fs_ro`, `emergency_cleanup`

### windows — Windows Server / AD

| Playbook | 风险 | 变量 | 说明 |
|----------|------|------|------|
| `restart_service` | low | `service` | Restart-Service |
| `restart_netlogon` | low | — | 重启 Netlogon（刷新安全通道） |
| `force_replication` | medium | `target_dc` | repadmin /syncall 强制 AD 复制 |
| `force_kcc_recalc` | medium | — | 强制 KCC 重新计算拓扑 |
| `clear_persistent_linger` | **high** | `dc_name`, `nc` | 清理 AD 滞留对象 |
| `clear_dns_cache` | low | — | Clear-DnsServerCache |
| `reload_dns_zones` | medium | — | 从 AD 重新加载 DNS 区域 |
| `restart_dns_service` | medium | — | Restart-Service DNS |
| `enable_dns_scavenging` | low | — | 启用 DNS 老化清理 |
| `resync_w32time` | low | — | w32tm /resync /force |
| `set_ntp_source` | medium | `ntp_servers` | 设置外部 NTP 源 |
| `defrag_ntds` | **high** | — | NTDS 数据库脱机整理 |
| `cleanup_stale_computers` | medium | — | 禁用 90 天未活动的计算机 |
| `metadata_cleanup` | **high** | `dc_name` | 清理 DC 元数据 |
| `kill_process` | medium | `process` | Stop-Process -Force |
| `restart_winrm` | **high** | — | 重启 WinRM（远程管理不可用时） |
| `clean_temp` | low | — | 清理 Windows 临时文件 |
| `flush_dns` | low | — | ipconfig /flushdns |
| `clear_event_logs` | medium | `log_name` | Clear-EventLog |
| `reboot_host` | **high** | — | shutdown /r /t 300（5 分钟后重启） |

### network — 交换机/路由器（28 个 playbook）

**接口**: `interface_up`/`interface_down`(high), `interface_desc`(low), `shut_interface`(high), `enable_interface`(high)

**VLAN**: `create_vlan`(low), `delete_vlan`(high), `add_port_vlan`(medium), `remove_port_vlan`(medium)

**STP/L2**: `enable_stp`(medium), `disable_stp`(high), `enable_bpdu_guard`(low), `enable_portfast`(low), `port_security`(medium)

**路由**: `bgp_soft_reset`(medium), `bgp_hard_reset`(high), `clear_ospf`(medium), `add_static_route`(medium), `remove_static_route`(high)

**ACL**: `add_acl_rule`(medium), `remove_acl_rule`(high)

**系统**: `save_config`(low), `clear_counters`(low), `reload`(critical), `dhcp_snooping`(medium), `show_inventory`(low), `erase_startup`(critical)

### 其他资产

**huawei_firewall**: `save_config`(low), `clear_counters`(low), `enable_interface`(medium), `shut_interface`(high), `clear_session`(medium), `reload`(critical)

**brocade_san**: `port_enable`(medium), `port_disable`(high), `switch_enable`(medium), `switch_disable`(critical), `cfg_save`(low), `cfg_enable`(high)

**pve**: `start_vm`(low), `shutdown_vm`(medium), `stop_vm`(high), `start_ct`(low), `restart_pveproxy`(low), `restart_pvedaemon`(medium), `migrate_vm`(high), `scrub_zfs_pool`(low), `clear_zfs_errors`(low), `restart_ceph_services`(high), `clean_pve_logs`(low), `restart_pvestatd`(low)

**kvm**: `start_vm`(low), `shutdown_vm`(medium), `destroy_vm`(high), `reboot_vm`(medium), `restart_libvirtd`(medium), `refresh_pool`(low), `start_pool`(low), `destroy_network`(high), `start_network`(low), `clear_libvirt_logs`(low)

**存储** (emc_me / dell_powerstore / dell_sc / dell_unity): 查看磁盘/池/卷/端口状态(low)，创建/删除卷(medium-high)，扩容池/卷(medium)，端口启用/禁用(high)。具体名称先调 `__list__`。

**nutanix / xenserver / citrix_vdi / citrix_netscaler / veeam**: 均有对应 playbook。先通过 `__list__` 确认可用列表。

## 变量说明

修复 playbook 中的 `{{variable}}` 占位符从 `variables` 参数替换。

```json
// 单变量
{"asset_name": "web-01", "playbook": "restart_service", "variables": {"service": "nginx"}}

// 多变量
{"asset_name": "esxi-01", "playbook": "snapshot_delete", "variables": {"vm_name": "web-01", "snap_name": "pre-patch"}}

// 必填变量缺失时 Gateway 返回 400
```

## 资产类型映射

诊断/修复使用 Gateway 的 `_resolve_family()` 做模糊匹配：

| 资产类型 | 诊断协议 | 修复协议 | 备注 |
|----------|----------|----------|------|
| `vmware_vcenter` | vsphere | vsphere (shell to VCSA) | 8 区深层诊断 |
| `vmware_esxi` | — | — | **废弃，统一用 vCenter govc** |
| `linux` → `linux` | | | |
| `windows` → `windows` | | | |
| `dell_idrac` / `hpe_ilo` / `lenovo_xclarity` | | Redfish / IPMI | |
| `huawei_ibmc` / `inspur_ism` | | IPMI | |
| `network` / `huawei_firewall` / `brocade_san` | | SSH | |

**重点**：vCenter 巡检发现的 ESXi 主机问题，诊断/修复目标资产仍是 **vCenter**（vsphere 协议），通过 govc playbook 操作主机/VM。
