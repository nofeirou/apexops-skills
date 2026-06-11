# 采集器数据解读参考

> 此文件为参考文档，仅在分析巡检结果时按需读取。不要每次对话都加载。

## 8 种采集器数据解读

### 1. vSphere 采集器 (vCenter)

**协议**: `vsphere`
**覆盖**: `vmware_vcenter`
**采集方式**: REST API + govc + pyVmomi

#### 平台信息

| Section | 关键 Key | 含义 | 异常判断 |
|---------|----------|------|----------|
| `platform` | `version`, `build`, `deployment_type` | vCenter 版本/构建号/部署类型 | — |
| `version_detail` | `version`, `build`, `api_version`, `full_name` | vSphere 详细版本信息 | 版本过旧（6.x）建议升级 |

#### 健康与证书

| Section | 关键 Key | 含义 | 异常判断 | 建议动作 |
|---------|----------|------|----------|----------|
| `health` | `system`, `storage`, `mem`, `swap`, `load`, `db`, `applmgmt`, `pkgs` | 8 个子系统健康状态 | 非 "HEALTHY"/"green" 即为异常 | `/apexops-action 诊断 <vcenter> --area=health` |
| `certificate` | `days_remaining`, `issuer`, `subject_alt_name` | TLS 证书剩余天数 | ⚠️ < 90天, 🔴 < 30天, 🚨 < 7天 | 更新 vCenter 证书 |
| `ntp` | `servers`, `time_sync` | NTP 服务器/同步状态 | 时间不同步 🔴, 服务器数 ≤1 ⚠️ | 配置至少 2 个 NTP 源 |
| `dns` | `hostname`, `server_count` | DNS 配置 | 服务器数为 0 🔴 | 配置 DNS 服务器 |
| `access` | `ssh`, `shell`, `dcui`, `consolecli` | 访问接口状态 | — | — |

#### 备份与服务

| Section | 关键 Key | 含义 | 异常判断 | 建议动作 |
|---------|----------|------|----------|----------|
| `backup_jobs` | 备份任务列表 | VAMI 备份历史 | 无任何记录 🚨 | 立即建立备份计划 |
| `backup_schedules` | 备份计划列表 | VAMI 备份计划 | 无计划 ⚠️ | 创建定期备份计划 |
| `services` | `started`, `stopped`, `details` | 核心服务启停数 | stopped > 0 🔴 | 检查停止服务的日志 |
| `update` | 补丁信息 | 当前补丁状态 | — | — |

#### 主机 (hosts)

| 关键 Key | 含义 | 异常判断 | 建议动作 |
|----------|------|---------------------|----------|
| `connection_state` | 连接状态 | 非 "connected" 🚨 | `/apexops-action 诊断 <host> --area=network` |
| `power_state` | 电源状态 | 非 "poweredOn" 🔴 | 检查主机是否被关机 |
| `maintenance_mode` | 维护模式 | 非维护窗口期启用 ⚠️ | 确认维护窗口 |
| `cpu_usage_pct` | CPU 使用率 | ⚠️ >80%, 🔴 >90%, 🚨 >95% | `/apexops-action 诊断 <host> --area=cpu` |
| `memory_usage_pct` | 内存使用率 | ⚠️ >80%, 🔴 >90%, 🚨 >95% | `/apexops-action 诊断 <host> --area=memory` |
| `uptime_days` | 运行天数 | 🔴 < 1天 (近期重启), ⚠️ < 7天 | 确认是否计划内重启 |
| `hardware_health.sensors` | 温度/电压/风扇传感器 | 状态非 "green" 🔴 | 检查硬件健康状态 |
| `vendor`, `model` | 服务器厂商/型号 | — | — |

#### 虚拟机 (vms)

| 关键 Key | 含义 | 异常判断 | 建议动作 |
|----------|------|---------------------|----------|
| `power_state` | 电源状态 | 大量 "poweredOff" ⚠️ | 确认是否为闲置 VM |
| `cpu_usage_pct` | CPU 使用率 | ⚠️ >80%, 🔴 >90%, 🚨 >95% | `/apexops-action 诊断 <vm> --area=cpu` |
| `memory_usage_pct` | 内存使用率 | ⚠️ >80%, 🔴 >90%, 🚨 >95% | `/apexops-action 诊断 <vm> --area=memory` |
| `vm_tools_running` | VMware Tools 运行 | `false` ⚠️ | 安装/升级 VMware Tools |
| `folder_path` | vCenter 目录路径 | — | — |
| `hostname` / `ip_address` | 客机主机名/IP | 空则 VMware Tools 异常 | 检查 VMware Tools |
| `disks` | 磁盘列表(容量/剩余) | 单盘使用率 >85% ⚠️ | 扩容客机磁盘 |
| `nics` | 网卡列表(MAC/IP/连接) | `connected: false` 🔴 | 检查虚拟网络配置 |
| `snapshot_count` | 快照数量 | > 3 ⚠️, > 5 🔴 | 清理过期快照 |

#### 数据存储 (datastores)

| 关键 Key | 含义 | 异常判断 | 建议动作 |
|----------|------|---------------------|----------|
| `used_pct` | 使用率 | ⚠️ >75%, 🔴 >85%, 🚨 >95% | 清理或扩容 |
| `accessible` | 可访问性 | `false` 🚨 | 检查存储连接 |
| `vm_count` | 承载 VM 数 | — | — |
| `type` | 类型 (VMFS/NFS) | — | — |

#### 集群 (clusters)

| 关键 Key | 含义 | 异常判断 | 建议动作 |
|----------|------|----------|----------|
| `drs_enabled` | DRS 启用 | 生产集群禁用 🔴 | 启用 DRS 自动负载均衡 |
| `ha_enabled` | HA 启用 | 生产集群禁用 🚨 | 启用 HA 高可用 |
| `evc_mode` | EVC 模式 | "Disabled" 且是混合 CPU ⚠️ | 启用 EVC |
| `memory_usage_pct` | 集群内存使用率 | ⚠️ >80%, 🔴 >90% | 增加主机或回收资源 |
| `effective_memory_gb` | 有效内存 | 接近 total_memory ⚠️ | 检查超分配 |

#### 快照 (snapshots)

| 关键 Key | 含义 | 异常判断 | 建议动作 |
|----------|------|----------|----------|
| `snapshot_count` | 快照数量 | > 3 ⚠️, > 5 🔴 | 清理过期快照 |
| `size_mb` | 快照总大小 | > 10GB ⚠️, > 50GB 🔴 | 快照过大影响性能 |
| `age_days` (per snapshot) | 快照年龄 | ⚠️ >3天, 🔴 >7天, 🚨 >30天 | 确认快照是否仍需保留 |

#### 告警与事件 (alarms + events)

| Section | 关键 Key | 含义 | 异常判断 | 建议动作 |
|---------|----------|------|----------|----------|
| `alarms` | `status`, `alarm`, `entity`, `age_hours` | 活动告警 | 红色 🚨 立即, 黄色 ⚠️ 关注 | 见 alarm 中的 `suggested_actions` |
| `alarms[].suggested_actions` | AI 路由建议 | 直接可用的诊断命令 | — | 执行建议的 `/apexops-action` 命令 |
| `alarms[].acknowledged` | 是否已确认 | 红色告警未确认 >24h 🚨 | 确认或处理告警 |
| `events` | `severity`, `type`, `message` | 最近 3 天严重/警告事件 | 同一实体重复 >10 次 🔴 | 交叉比对 alarms，判断是否为已知问题 |

**告警/事件交叉分析规则**：
- `alarms` 多 `events` 少 → 可能是瞬时告警，持续 1h+ 则需关注
- `alarms` 少 `events` 多 → 阈值校准问题，建议调整告警策略
- 同一实体同时出现在 alarms 和 events → 已确认的持续问题，优先处理

#### 性能 TOP20 (performance_top20)

| 关键 Key | 含义 | 异常判断 | 建议动作 |
|----------|------|----------|----------|
| `cpu_ready_avg` | CPU Ready 时间 (ms) | ⚠️ >5%, 🔴 >10%, 🚨 >20% | 减少 vCPU 或检查超分配 |
| `cpu_usage_avg` | CPU 使用率 (%) | ⚠️ >80%, 🔴 >90% | 检查 VM 内进程 |
| `mem_vmmemctl_avg` | Memory Balloon (KB) | > 0 ⚠️ | ESXi 正在回收内存，检查主机内存压力 |
| `mem_swapinRate_avg` | Memory Swap (KB/s) | > 0 🚨 | 严重内存不足，立即扩容 |

> CPU Ready >5% 意味着 VM 在等待物理 CPU。一个 90% CPU 但 Ready=0 的 VM 是健康的；一个 30% CPU 但 Ready=15% 的 VM 是饥饿的。

#### 安全与合规 (host_security + compliance_checks)

| Section | 关键 Key | 含义 | 异常判断 | 建议动作 |
|---------|----------|------|----------|----------|
| `host_security` | `lockdown` | Lockdown Mode | "Disabled" ⚠️ | 生产环境启用 Normal 模式 |
| `host_security` | `ssh`, `shell` | SSH/Shell 状态 | 生产环境 "on" 🔴 | 关闭 SSH 和 ESXi Shell |
| `host_security` | `ntp` | NTP 服务 | "off" 🔴 | 启用 ntpd |
| `host_security[].syslog` | Syslog 服务 | "off" ⚠️ | 配置远程 Syslog |
| `host_security[].services` | 服务运行/策略详情 | 自动启动但未运行 🔴 | 检查服务日志 |
| `host_security[].password_policy` | 密码策略配置 | 空或弱策略 🔴 | 按等保要求配置密码复杂度 |
| `compliance_checks` | 等保 2.0 L3 自动化检查 | `fail` 🔴, `warn` ⚠️ | 按 suggestion 字段逐项整改 |

**等保 2.0 三级检查覆盖**：

| 等保条款 | 检查项 | 自动化 |
|----------|--------|:------:|
| 8.1.3.1 身份鉴别 | vCenter 密码策略 | 提示手动验证 |
| 8.1.3.2 访问控制 | Lockdown Mode | ✅ 自动 |
| 8.1.3.3 安全审计 | Syslog 配置 | ✅ 自动 |
| 8.1.3.5 时间同步 | NTP 配置 | ✅ 自动 |
| 8.1.4.1 安全审计 | 审计日志保存 | 提示手动验证 |
| 8.1.4.2 最小安装 | SSH/Shell 状态 | ✅ 自动 |

#### 资源超分配

| 关键 Key | 含义 | 异常判断 | 建议动作 |
|----------|------|----------|----------|
| `cpu_ratio` | vCPU:pCPU 比率 | ⚠️ >2:1, 🔴 >3:1, 🚨 >5:1 | 增加主机或减少 vCPU 分配 |
| `mem_ratio` | vRAM:pRAM 比率 | ⚠️ >1:1, 🔴 >1.5:1, 🚨 >2:1 | 增加内存或减少分配 |

#### 其他

| Section | 关键 Key | 含义 | 异常判断 |
|---------|----------|------|----------|
| `networks` | 网络列表 | — | — |
| `resource_pool_count` | 资源池数 | — | — |
| `_thresholds` | 三级阈值参考值 | AI 分析时的**参考**，非强制规则；最终判断需结合业务上下文 | — |
| `collected_at` | 采集时间戳 | — | — |

#### 平台级自动诊断规则（v2606.102+ 新增字段）

以下规则基于新增的采集字段，Agent 必须在报告中逐项检查：

| 规则 | 检查字段 | 异常判断 | 建议动作 |
|------|---------|----------|----------|
| 健康最后检查 | `platform.health_lastcheck` | 空或 "N/A" | 检查 health 服务是否正常 |
| 网卡状态 | `platform.network_interfaces[].status` | 有 NIC status≠up | 检查对应物理网卡连接 |
| DNS 主机名 | `platform.dns.dns.hostname` | = "localhost" | 配置 VCSA 正式 FQDN |
| DNS 服务器数 | `platform.dns.dns.servers` | < 2 | 配置 ≥2 个 DNS 避免单点 |
| 时间同步模式 | `platform.timesync` | ≠ "NTP" | 改为 NTP 模式，避免时钟漂移 |
| NTP 服务器数 | `platform.ntp` | < 2 | 建议 ≥2 个 NTP 源 |
| SSH 访问 | `platform.access.ssh` | enabled=true (生产) | 关闭 SSH，排错时临时启用 |
| 备份历史 | `platform.backup.jobs` | 空列表 | VAMI 5480 配置周期备份 |
| 备份计划 | `platform.backup.schedules` | 空或 name count=0 | 同上，至少每周一次 |
| 证书类型 | `platform.certificate.issuer_dn` | 含 "localhost" | 自签证书，生产考虑企业 CA |
| 证书 SAN | `platform.certificate.subject_alternative_name` | 不含 vCenter FQDN | 证书 SAN 缺失，可能引发 SSL 校验失败 |
| 集群 HA | `clusters[].ha_enabled` | = false 且 host≥2 | 启用 HA 保障 VM 故障迁移 |
| 集群 DRS | `clusters[].drs_enabled` | = false 且 host≥2 | 启用 DRS 自动均衡负载 |
| VMware Tools | `vms[].vm_tools_status` | ≠ "guestToolsCurrent" | 升级 VMware Tools 到最新 |
| 补丁状态 | `platform.update.state` | ≠ "UP_TO_DATE" | VAMI 5480 → Update 查看可用补丁 |
| 快照年龄 | `snapshots[].snapshots[].age_days` | > 7 天 | 清理过期快照，防止数据存储填充 |

#### vSphere 深度分析指引

vSphere 采集器的数据区块之间存在强关联，Agent 必须执行交叉关联分析，不只看单个指标。

**关联矩阵**：

| 症状 | 需检查的关联数据 | 分析方法 |
|------|-----------------|----------|
| 告警(alarms) | events(同主机/VM) + performance(cpu.ready) + datastores(容量) | 告警是症状，事件和性能指标是原因。告警状态=red 但 events 无对应事件 → 检查阈值是否过严 |
| CPU 使用率高(hosts) | overcommit(cpu_ratio) + performance(cpu.ready) + clusters(drs) | cpu_ratio > 3:1 且 cpu.ready > 5% → vCPU 争用；DRS 手动模式时检查是否需迁移 VM |
| 内存使用率高(hosts) | overcommit(mem_ratio) + performance(mem.vmmemctl) + performance(mem.swapin) | balloon > 0 → 主机正在回收内存；swapin > 0 → 已触发交换，性能严重下降 |
| 数据存储空间不足(datastores) | snapshots(该 DS 上 VM 的快照) + vms(disks) + events(VmDiskFailedEvent) | used_pct > 90% → 逐 VM 查快照大小和磁盘增长；有 VmDiskFailedEvent → DS 已满影响 VM |
| VMware Tools 异常(vms) | hostname(空) + ip_address(空) + guest_os(空) + events(该 VM 的告警) | tools_running=false 且 hostname 为空 → Tools 未运行，客机内无感知 |
| 主机断连(hosts) | events(HostConnectionLostEvent) + alarms(该主机) | connection_state≠connected 且最近有 HostConnectionLostEvent → 网络或主机崩溃 |
| 证书到期(platform) | services(vmware-sts-idmd) | days_remaining < 30 → 证书更新后 STS 服务需重启 |
| 服务异常(platform.services) | health(对应子系统) + events(服务相关) | stopped 但 health=HEALTHY → 冗余服务停止，可能不影响功能 |
| 安全不合规(compliance) | host_security + events(UserLoginSessionEvent) | Lockdown=Disabled 且最近有 BadUsernameSessionEvent → 存在暴力破解尝试 |

**诊断路径（按场景）**：

1. **"某主机负载高"** → hosts(cpu_usage_pct/memory_usage_pct) → performance(cpu.ready/mem.vmmemctl top VMs) → vms(该主机上 VM 的 CPU/内存分配) → overcommit(cpu_ratio/mem_ratio)
2. **"数据存储快满了"** → datastores(used_pct) → snapshots(size_mb/age_days) → vms(disks 容量) → events(VmDiskFailedEvent)
3. **"VM 性能慢"** → vms(cpu_usage_pct/memory_usage_pct) → performance(该 VM 的 cpu.ready/swapin) → hosts(该主机资源争用) → datastores(VM 所在 DS 延迟)
4. **"告警频繁"** → alarms(列表+status) → events(同实体 3 天内事件) → host_security/compliance(是否因配置触发)

**report 中必须包含的信息**：

- **主机清单**: 每个 cluster 的主机数+连接状态+维护模式，CPU/内存使用率分布
- **数据存储**: 容量排行（按 used_pct 降序），worst 3 的详细信息
- **告警摘要**: 按 severity 分组计数，列出 top 5 严重告警
- **安全态势**: Lockdown/NTP/Syslog/SSH 不合规项
- **快照年龄**: >7 天的快照数量和总大小
- **容量趋势**: overcommit 比率 + 是否有 balloon/swap

### 2. SSH 采集器

协议 `ssh`，覆盖 15 种资产类型。单连接批量执行所有命令。

**linux (通用 Linux 服务器)**

| Section | 关键 Key | 含义 | 异常判断 |
|---------|----------|------|----------|
| `cpu` | CPU 型号/核数/上下文切换/中断 | 处理器诊断 | 使用率 > 90% 持续 |
| `memory` | 总内存/已用/Swap/OOM历史/Slab | 内存诊断 | 可用 < 10% 或 OOM 事件 |
| `disk` | 各分区使用率/挂载点/inode/IO/大目录 | 磁盘诊断 | 使用率 > 85% 或 inode > 90% |
| `network` | TCP 连接/监听端口/路由/DNS/接口统计 | 网络诊断 | CLOSE_WAIT 堆积或异常监听 |
| `process` | 失败服务/僵尸进程/高资源进程 | 进程诊断 | 有失败服务或僵尸进程 |
| `system` | 内核/运行时间/SELinux/防火墙/安全参数 | 系统基线 | ip_forward=1 或 SELinux disabled |
| `logs` | dmesg 错误/journal 错误/认证失败/cron 错误 | 日志审计 | 频繁认证失败或内核错误 |

**vmware_esxi**: cpu/memory/disk/network/process/services/vms/sensors

**pve (Proxmox)**: cpu/memory/disk/storage/network/process/backup

**kvm**: cpu/memory/disk/network/vm

**nutanix**: cluster/storage/performance/network/vm

**network (交换机/路由器)**: cpu/memory/storage/network/system

**brocade_san**: system/fabric/ports/hardware/performance

**citrix_netscaler**: performance/memory/disk/network/services/logs

**huawei_firewall**: system/interfaces/routing/policies/sessions/version/ha

**xenserver**: system/vm/storage/network/performance

**citrix_vdi**: site/delivery/catalog/sessions/events

**veeam**: backup_jobs/storage/infrastructure/tape/system

**emc_me**: health/capacity/hardware/performance

**dell_powerstore**: cluster/capacity/hardware/performance

**dell_sc**: system/storage/hardware/performance

**dell_unity**: health/capacity/hardware/performance

### 3. WinRM 采集器 (v2.1)

协议 `winrm`，覆盖 `windows`、`veeam`。单会话批量执行 PowerShell，13 种角色自发现。

#### 输出结构 (v2.1+)

```json
{
  "_collector": {"version": "2.1", "protocol": "winrm", "auth": "ntlm",
                 "connect_status": "ok|failed", "duration_ms": 1234,
                 "category_map": {"status": [...], "config": [...]}},
  "<section>": {
    "status": "ok",
    "items": {
      "<key>": {
        "status": "ok|failed",
        "output": "raw",
        "exit_code": 0,
        "error_code": -1,
        "message": "error detail",
        "description": "中文说明",
        "category": "status|config"
      }
    }
  }
}
```

**分层规则**:
- `status` = 运行时状态 (CPU/内存/服务/安全事件/日志错误/进程)
- `config` = 配置元数据 (主机名/硬件/网络/DNS/组策略/计划任务/启动项/补丁)

#### 3.1 基础 Section

| Section | 关键 Key | 含义 | 异常判断 |
|---------|----------|------|----------|
| `system` | hostname, os_version, uptime_days, model, manufacturer | 系统基本信息 | — |
| `performance` | cpu_usage, cpu_cores, mem_total_gb, mem_free_gb, mem_usage_pct | CPU/内存性能 | CPU > 90%, 内存 > 90% |
| `disk` | volumes (JSON: DeviceID/SizeGB/FreeGB/UsedPct) | 磁盘卷使用 | UsedPct > 85% |
| `services` | auto_not_running, auto_not_running_count | 自动启动但未运行的服务 | count > 0 |
| `updates` | hotfix_count, last_patch_days, pending_count | 补丁状态 | last_patch > 90天, pending > 10 |
| `network` | ipv4, dns_client, listening_ports | 网络配置/监听端口 | — |
| `security` | failed_logons_24h, failed_rdp_24h, ntlm_auths_24h, account_lockouts_24h | 安全事件统计 | 登录失败 > 100, 账户锁定 > 0 |
| `processes` | total_count, top_cpu | 进程概况 | — |

#### 3.2 组策略 (`group_policy`) ★新增

| Key | 含义 | 异常判断 |
|-----|------|----------|
| `gpo_count` | 组策略对象总数 | 非 DC 返回提示 |
| `gpo_list` | GPO 详细列表 (JSON): DisplayName / CreationTime / ModificationTime / GpoStatus / Id | 修改时间 > 6 个月未更新需关注 |
| `last_applied` | 客户端最近应用时间 | 超过 24h 未刷新需关注 |
| `rsop_model` | 本地组策略生效摘要 (RSoP) | — |
| `password_policy` | 域密码策略 (net accounts) | 密码长度 < 8, 不过期 |
| `audit_policy` | 高级审核策略分类 | 关键类别 'No Auditing' |

#### 3.3 事件日志 (`event_logs`) ★新增

| Key | 含义 | 异常判断 |
|-----|------|----------|
| `system_errors_24h` | System 日志 24h 错误数 | > 50 |
| `system_warnings_24h` | System 日志 24h 警告数 | > 200 |
| `system_critical_24h` | System 日志 24h 严重数 | > 0 |
| `system_recent_errors` | System 最近 10 条错误详情 | 磁盘/服务/NIC 错误 |
| `app_errors_24h` | Application 日志 24h 错误数 | > 100 |
| `app_crash_24h` | 应用崩溃事件 (EventID 1000) | > 5 |
| `security_audit_failures_24h` | Security 审核失败数 | > 100 |
| `log_sizes` | 日志文件大小/MaxSize/是否满 | IsLogFull = True 紧急 |

#### 3.4 计划任务 (`scheduled_tasks`) ★新增

| Key | 含义 | 异常判断 |
|-----|------|----------|
| `total_count` | 计划任务总数 | — |
| `enabled_count` / `disabled_count` | 启用/禁用任务数 | — |
| `running_count` | 正在运行任务数 | — |
| `last_failed` | 最近 5 个失败任务 + 退出码 | 持续失败 = 调度异常 |
| `tasks_at_boot` | 开机触发任务 | — |
| `tasks_at_logon` | 登录触发任务 | — |

#### 3.5 启动项 (`startup`) ★新增

| Key | 含义 | 异常判断 |
|-----|------|----------|
| `registry_run` | HKLM Run 键 (系统级启动) | 可疑程序名 |
| `registry_run_hkcu` | HKCU Run 键 (用户级启动) | 可疑程序名 |
| `startup_folder` | 启动文件夹路径 | — |
| `startup_services` | 自动启动服务数 | — |
| `auto_services_details` | 自动启动服务列表 (JSON) | 异常服务名 |

#### 3.6 角色专项

检测到 Windows Feature 或服务后自动追加专项 Section (`roles_<name>`)：

| 角色 | Section 名 | 关键指标 |
|------|-----------|----------|
| AD DS | `roles_adds` | dc_count, user_count, locked_users, replication_status |
| DNS | `roles_dns` | zone_count, forwarders, scavenging_enabled, resolution_test |
| DHCP | `roles_dhcp` | scope_count, scope_stats, authorized |
| IIS | `roles_iis` | site_count, sites, app_pools |
| Hyper-V | `roles_hyperv` | vm_count, vms, checkpoints |
| WSUS | `roles_wsus` | last_sync, pending_approval |
| Print Server | `roles_printserver` | printer_count, jobs_queued |
| NPS/RADIUS | `roles_nps` | radius_clients, network_policies |
| RDS | `roles_rds` | session_count |
| CA | `roles_ca` | ca_type, templates |
| File Server | `roles_fileserver` | share_count, open_files, smb_version |
| DFS | `roles_dfs` | namespace_count, replication_groups |
| Failover Cluster | `roles_failovercluster` | node_count, nodes, resource_groups |

### 4. Redfish 采集器

协议 `redfish`，覆盖 `dell_idrac`、`hpe_ilo`、`lenovo_xclarity`。

| Section | 关键 Key | 含义 | 异常判断 |
|---------|----------|------|----------|
| `Systems` | `Manufacturer`, `Model`, `SerialNumber`, `PowerState`, `Health` | 服务器基本信息/健康状态 | Health 非 "OK" |
| `Chassis` | `ChassisType`, `Manufacturer`, `Health` | 机箱信息/健康 | Health 非 "OK" |
| `Thermal` | 各温度传感器读数/状态 | 温度监控 | 温度 > 80°C 或状态异常 |
| `Power` | 各电源输出/容量/状态 | 功耗监控 | 电源状态异常 |
| `Managers` | `FirmwareVersion`, `Health`, `DateTime` | BMC 固件/健康/时间 | 时间偏差 > 5 分钟 |
| `Firmware` | 固件清单 | 设备固件版本 | — |

### 5. SNMP 采集器

协议 `snmp`，覆盖 `network`、`huawei_firewall`。

| Section | 关键 Key | 含义 | 异常判断 |
|---------|----------|------|----------|
| `sysDescr` | — | 系统描述 | — |
| `sysObjectID` | — | 设备 OID | 用于识别厂商型号 |
| `sysUpTime` | — | 运行时间 (Tick) | 近期重启 |
| `sysContact` | — | 管理员联系信息 | 为空 = 配置不完整 |
| `sysName` | — | 设备主机名 | — |
| `interfaces` | 接口数量 | 接口总数 | — |
| `ip_addresses` | IP 地址列表 | — | — |

### 6. IPMI 采集器

协议 `ipmi`，覆盖 BMC（`dell_idrac`、`hpe_ilo`、`huawei_ibmc`、`inspur_ism` 等）。

#### 6.1 资产信息 (`asset`)

| Key | 含义 | 说明 |
|-----|------|------|
| `manufacturer` | 制造商 | 从 MC info 和 FRU 提取 |
| `model` | 服务器型号 | — |
| `serial_number` | 序列号 | — |
| `bmc_version` | BMC 固件版本 | — |
| `bios_version` | BIOS 版本 | — |
| `bmc_ip` | BMC 管理 IP | — |

#### 6.2 健康状态 (`health`)

| Key | 含义 | 异常判断 |
|-----|------|----------|
| `overall` / `overall_cn` | 整体健康 / 中文标签 | `critical`(严重) / `warning`(警告) / `ok`(正常) |
| `system_power` | 系统电源 | 非 "on" 时告警 |
| `faults.power` | 电源故障 | true → 严重 |
| `faults.drive` | 磁盘故障 | true → 严重 |
| `faults.cooling` | 散热故障 | true → 警告 |
| `sensor_summary` | 传感器统计 (total/ok/warning/critical) | critical > 0 → 严重 |

#### 6.3 硬件配置 (`hardware`) ★新增

FRU 扫描全部设备，按类型分类呈现详细硬件规格：

| Section | 字段 | 说明 |
|---------|------|------|
| `summary` | `cpu_sockets` / `memory_slots` / `psu_count` / `nic_count` / `gpu_count` / `storage_controllers` / `disk_slots` / `fan_modules` | 硬件插槽/数量汇总 |
| `cpu` | `name` / `manufacturer` / `model` / `part_number` / `serial` | CPU 型号/制造商/料号 |
| `memory` | `name` / `manufacturer` / `model` / `part_number` / `serial` | 内存条型号/制造商/料号 |
| `psu` | `name` / `manufacturer` / `model` / `part_number` | 电源型号/制造商/料号 |
| `nic` | `name` / `manufacturer` / `model` / `part_number` | 网卡型号/制造商/料号 |
| `gpu` | `name` / `manufacturer` / `model` / `part_number` | GPU 型号/制造商/料号 |
| `storage_controller` | `name` / `manufacturer` / `model` / `part_number` | RAID/SAS 控制器 |
| `backplane` | `name` / `manufacturer` / `model` | 硬盘背板 |
| `riser` | `name` / `manufacturer` / `model` | PCIe 转接卡 |
| `fan_module` | `name` / `manufacturer` / `model` | 风扇模块 |

> ⚠️ **FRU 数据依赖**：硬件型号信息来自 BMC 的 FRU 数据，部分老式 BMC 可能不提供完整信息。报告时明确标注哪些数据来自 FRU，哪些来自传感器推估。

#### 6.4 组件状态 (`components`)

传感器实时监控数据，每条记录含 `status_cn`（正常/警告/故障/缺失）。

| Section | 关键字段 | 异常判断 |
|---------|---------|----------|
| `cpu` | `temperature` / `status_cn` | status 非"正常"或温度异常 |
| `memory` | `slot` / `status_cn` / `ecc_errors` | ECC 错误 > 0 或 status 非"正常" |
| `power` | `status_cn` / `output_watts` / `input_voltage` | status 非"正常" |
| `fan` | `status_cn` / `speed_rpm` | status 非"正常"或转速异常 |
| `disk` | `status_cn` / `predictive_failure` / `offline` | predictive_failure / offline / fault |

#### 6.5 事件日志 (`events`) ★已中文化

每条事件含 `severity_cn`（严重/警告/信息）和 `description_cn`（中文描述）。

| 字段 | 说明 |
|------|------|
| `total` / `critical_count` / `warning_count` | 事件统计 |
| `entries[].time` | 时间戳 |
| `entries[].sensor` | 触发传感器 |
| `entries[].description` | 原始英文描述 |
| `entries[].description_cn` | **中文翻译**：温度严重超标 / 电压异常 / 风扇故障 / 电源故障 / 内存不可恢复错误 / CPU 热降频 / 磁盘故障 / 系统重启 / 网卡链路断开 等 |
| `entries[].severity_cn` | **中文严重级别**：严重 / 警告 / 信息 |

> 🤖 **AI 报告要求**：报告中事件表格优先用 `description_cn` + `severity_cn` 展示，英文 `description` 仅在脚注保留。

#### 6.6 固件 (`firmware`)

| Key | 说明 |
|-----|------|
| `versions[].component` | BMC / BIOS / RAID / NIC |
| `versions[].version` | 固件版本号 |

### 7. REST 采集器

协议 `rest`，支持可配置的 REST API 端点（NITRO、Nutanix API 等）。

通用 Key: `status`, `version`, `health`, `capacity` — 返回结构与具体 API 响应一致。按目标系统 API 文档正常解读。

### 8. Unity 采集器

协议 `unity`，覆盖 `dell_unity`。

| Section | 关键 Key | 含义 | 异常判断 |
|---------|----------|------|----------|
| `system` | `name`, `model`, `version`, `health` | 存储系统健康 | health 非 "OK" |
| `pools` | 存储池列表 | 容量/使用率 | 使用率 > 85% |
| `disks` | 硬盘清单 | 类型/容量/状态 | 有故障盘 |
| `luns` | LUN 列表 | 容量/归属/状态 | — |
| `filesystems` | 文件系统列表 | 容量/使用率 | 使用率 > 85% |
| `nas_servers` | NAS 服务器列表 | 健康/状态 | 非 OK |
| `hosts` | 主机连接配置 | 启动器/WWN | — |
| `eth_ports` | 以太网端口 | 速度/状态 | link down |
| `fc_ports` | FC 端口 | 速度/状态 | link down |
| `alerts` | 活动告警列表 | 严重级别/描述 | 严重告警 |

