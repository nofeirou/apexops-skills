---
name: apexops-check
description: ApexOps 只读采集与诊断。当用户需要巡检设备、检查健康状态、诊断异常、分析运行数据、生成巡检报告时触发。调用 ApexOps API 全量采集 24 类资产运行指标，AI 解读数据并生成结构化报告。触发关键词：巡检、检查、健康、状态、报告、诊断、分析、排查、inspect。
version: 1.0.7
user-invocable: true
metadata: {"openclaw": {"requires": {"env": ["APEXOPS_URL", "APEXOPS_TOKEN"]}, "primaryEnv": "APEXOPS_TOKEN", "emoji": "🔍", "os": ["darwin", "linux"]}}
---

# ApexOps 巡检

## 安全规则（硬性约束）

1. **只读操作**：仅调用 GET 查询接口和 `POST /api/skills/inspect`（采集数据）。所有操作均为只读。
2. **禁止创建/修改/删除资产**：绝不调用资产的 CRUD API。资产由管理员在 Web UI 管理。
3. **禁止猜测密码**：所有凭证由 ApexOps 统一管理，Agent 不接触密码。
4. **禁止操作凭证/Token**：不调用 credentials 和 auth/token 相关接口。

## ApexOps API

所有操作通过 HTTP API 调用 ApexOps。

> ⚠️ **SSL 证书**：ApexOps 使用内网自签证书，所有 curl 必须加 `-k`（跳过证书验证）。Python 请求需设 `verify=False`，exec 工具需设 `insecure: true`。

### 轻量查询（不触发采集）

用户只是查看/列出资产信息时，**直接调 GET 查询，不走 inspect**。这些接口毫秒级返回，无副作用。

```bash
APEXOPS_URL="${APEXOPS_URL}"
APEXOPS_TOKEN="${APEXOPS_TOKEN}"

# 列出所有资产（分页，默认 limit=50）
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" "$APEXOPS_URL/api/assets"
# → {"assets":[{"id":1,"name":"web-01",...},...], "total":29, "limit":50, "offset":0}
# 数据在 .assets 数组中，总数在 .total

# 按组过滤
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" "$APEXOPS_URL/api/assets?group=生产"

# 按类型过滤 + 分页
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" "$APEXOPS_URL/api/assets?system_version=linux&limit=200"

# 查看资产类型列表
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" "$APEXOPS_URL/api/asset-types"

# 查看分组列表
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" "$APEXOPS_URL/api/groups"
```

**触发规则**：用户说「列出资产」「有哪些设备」「查看服务器列表」「看看资产」等 → 调 `GET /api/assets`，不调 inspect。

### 巡检采集（异步）

**inspect 为异步模式**，POST 后轮询 GET 直到完成。仅在用户明确要求巡检/检查健康/采集数据时使用。

```bash
# 1. 发起巡检 — 优先用 query 传用户原话（ApexOps 自动匹配资产名+系统版本）
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$APEXOPS_URL/api/skills/inspect" -d '{"query":"巡检域控服务器"}'
# → {"task_id":"a1b2c3d4","status":"running","hint":"GET .../inspect/a1b2c3d4 to poll"}

# 按系统类型巡检（明确类型时用 system_version）
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$APEXOPS_URL/api/skills/inspect" -d '{"system_version":"windows"}'

# 2. 轮询进度（每 5-10 秒一次，直到 status 变为 done 或 error）
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" \
  "$APEXOPS_URL/api/skills/inspect/a1b2c3d4"
# → {"status":"running","total":6,"completed":3,...}
# → {"status":"done","results":[...],"count":6,"report":{...}}

# 3. 按名称巡检（支持模糊匹配，如 "AD" 匹配 "域控服务器-AD01"）
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$APEXOPS_URL/api/skills/inspect" -d '{"asset_name":"域控服务器-01,域控服务器-02"}'

# 模糊匹配："AD服务器" → 匹配所有含"AD"的资产；"域控" → 匹配所有含"域控"的资产
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$APEXOPS_URL/api/skills/inspect" -d '{"asset_name":"域控"}'

# 4. 全量巡检（不传 system_version）
curl -sk -H "Authorization: Bearer $APEXOPS_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST "$APEXOPS_URL/api/skills/inspect" -d '{}'
```

**异步巡检流程**：
```
1. POST /api/skills/inspect → 获取 task_id（立即返回，不阻塞）
2. 首次 GET 轮询 → 从响应中获取 total（资产总数），按公式计算轮询超时：
   MAX_WAIT = max(600, total × 90)   ← WinRM 每台 30-120s，取 90s 保守预算
   例：total=7 → MAX_WAIT=max(600,630)=630s；total=3 → MAX_WAIT=600s
   若未传 system_version（全量）total 未知 → 默认 MAX_WAIT=1800
3. 每 5-10 秒轮询一次 GET /api/skills/inspect/{task_id}
   - status=running 时告知用户进度（如 "3/7 已完成..."）
   - 若轮询脚本因 MAX_WAIT 到期而退出，直接 curl 查一次当前状态
   - 若仍在 running，新开一轮轮询（MAX_WAIT 重算）
4. status=done → 解读数据，提交分析结论到 `/api/skills/report/analysis`
5. status=error → 报告 error 信息，建议重试或缩小范围
```

**超时说明**：WinRM/SSH 等慢协议采集耗时较长（每资产 30-120s），异步模式下 ApexOps 无请求超时限制，Agent 只需持续轮询即可。切勿在 POST 上设置短超时（如 curl 默认 30s），也不要因单次轮询返回 running 就认为失败。

v2606.117+ WinRM 有 130s socket 级硬超时，超时后该资产会标记为 `timeout` 并计入 error_count 继续后续资产。

**「全面」≠「全量」**：「全面巡检 VMware」意思是深度巡检 VMware，不是巡检所有设备。

**核心规则**：
1. **优先使用 `query` 参数**：把用户的资产描述原样放进去，ApexOps 自动匹配。你不需要自己判断类型或 `system_version`
2. 用户指定了具体资产名 → 传 `asset_name`
3. 无法确定时 → 不传任何参数做全量巡检
4. **绝不要一次请求里做多次 inspect 调用**
```

## 巡检流程

```
1. 提取用户原话 → 放入 `query` 字段，ApexOps 自动匹配
2. POST /api/skills/inspect → ApexOps 匹配资产 → 获取采集数据（JSON 原始数据）
3. AI 解读 JSON 数据，逐项判断异常
4. AI 生成完整 HTML 报告，推送到 `/api/skills/report`
5. 告知用户报告链接
```

## 巡检响应格式

**inspect 完成后返回摘要 JSON，不含全量原始数据。** Agent 应从摘要快速判断健康状态，必要时通过 `json_url` 获取完整数据（已剪枝），或通过 Todos API 获取结构化发现（Findings 已合并入 Todo）。

```json
{
  "status": "done",
  "results": [
    {
      "asset_name": "vcenter-01",
      "system_version": "vmware_vcenter",
      "host": "10.0.0.1:443",
      "protocol": "vsphere",
      "summary": {
        "sections": ["asset", "health", "hosts", "datastores", "vms", "alarms"],
        "section_count": 6,
        "health": "ok",
        "metrics": {"health_preview": "..."}
      }
    }
  ],
  "count": 12,
  "timestamp": "20260603_1430",
  "json_url": "https://apexops:9100/data/inspect/inspect_20260603_1430.json",
  "findings_extracted": 5,
  "todos_created": 3
}
```

- **`summary`**: 每资产的轻量摘要（健康状态 + 采集区块列表 + 关键指标预览），不包含原始采集数据
- **`json_url`**: 剪枝后的完整数据文件本地路径（/app/ 容器内），Agent 可读取用于深度分析
- **`findings_extracted`** / **`todos_created`**: 自动提取的 Findings 和 Todo 数量
- 通过 `GET /api/todos?source_task={mission_id}` 获取该任务自动生成的待办事项（原 Findings）
- 严禁在 Python 脚本中对 list 类型调用 `.get()` / `.items()` — 始终先检查类型

> ⚠️ **Python 脚本规范**：处理 API 响应时，始终做类型检查。`results` 是 list，`summary` 是 dict。对 list 使用索引 `[0]`，对 dict 使用 `.get()`。不确定类型时用 `isinstance(obj, dict)` 和 `isinstance(obj, list)` 判断。

### 新版采集器结构化格式（v2606.079+）

Brocade SAN 和网络交换机采集器已升级为结构化输出，通过 `json_url` 文件获取完整数据，其中包含分类 sections 和自动生成的 `findings`：

```json
{
  "data": {
    "system": {"hostname": "...", "model": "...", "fos_version": "...", ...},
    "health": {"temperature": {"max_c": 45.0, "sensors": [...]}, "power": {...}, "fans": {...}},
    "ports": {"summary": {"total": 48, "online": 44, ...}, "errors": {"0": {"crc": 0, ...}}},
    "optics": {"items": [...], "rx_low_warning": [...]},
    "fabric": {"ns_devices": [...], "devices": {"hosts": 12, "storage": 4}},
    "findings": [
      {"severity": "critical", "category": "san_port", "title": "端口 12 CRC 错误严重 (10234)", "evidence": {...}, "suggestion": "检查光纤/模块/衰减"}
    ]
  }
}
```

**findings 是采集器自动诊断的结果**，直接按 severity 排序呈现给用户即可。Brocade SAN 有 9 类自动诊断（CRC/Loss/Power/温度/SFP/RASLOG），网络交换机有 10 类（CPU/内存/温度/电源/风扇/端口down/错误/STP/端口安全/日志）。

### IPMI 采集器中文化（v2606.087+）

IPMI 采集器已内置中文化支持，生成报告时遵循以下规则：

1. **事件表格**：优先用 `description_cn` + `severity_cn` 展示，英文原文放脚注
2. **健康状态**：用 `health.overall_cn`（正常/警告/严重/已关机）
3. **组件状态**：用 `status_cn`（正常/警告/故障/缺失），不用英文 status
4. **硬件配置**：`hardware` section 含详细的 CPU/内存/电源/网卡/GPU/存储控制器/背板型号，报告中必须逐项列出（制造商 + 型号 + 料号），缺失的标注「未获取到」
5. **传感器读数**：温度带 °C，电压带 V，功率带 W，风扇带 RPM

## 巡检分析方法论（必读）

## 巡检分析方法论（必读）

在解读任何采集数据之前，遵循以下分析流程：

### 分析顺序（不要平铺所有数据）

```
1. 告警分析  → 先看 alarms/events，了解 vCenter 已知的问题
2. 健康概览  → 检查 health/certificate/services 等平台级状态
3. 容量评估  → datastores/hosts/clusters 使用率，按严重程度排序
4. 安全审计  → host_security/compliance_checks，标记不合规项
5. 细节深挖  → vms/snapshots/performance 逐项排查
```

**核心原则**：告警告诉你 vCenter 认为什么错了，事件告诉你实际发生了什么。两者不同步时（告警静默但事件暴增），通常是阈值校准问题。

### 三级阈值体系

所有指标使用统一的三级评判标准：

| 级别 | 标签 | 含义 | 响应要求 |
|------|------|------|----------|
| ⚠️ **警告** | `warning` | 接近阈值，需关注 | 纳入趋势观察，下次巡检对比 |
| 🔴 **严重** | `severe` | 已超标，影响业务风险 | 建议 24h 内排查，可执行诊断 |
| 🚨 **紧急** | `critical` | 严重超标，可能影响可用性 | 立即建议用户执行诊断/修复 |

### 简化版根因分析协议

发现异常时，不要在报告中只写"XX 指标超标"。遵循以下 4 条标准：

1. **可验证**：你能引用哪个具体指标、哪个阈值？避免模糊描述如"网络很慢"
2. **因果链**：根因 → 传播路径 → 放大因素 → 业务影响。例：数据存储满（根因）→ VM I/O 阻塞（传播）→ 客机文件系统只读（放大）→ 应用超时（影响）
3. **可复现**：同样的条件在其他地方也会产生同样的症状吗？
4. **可消除**：消除根因后症状会消失吗？

**反模式**：
- ❌ "主机负载高" — 只说现象，没解释原因
- ❌ "数据存储满了" — 只说状态，没因果链
- ❌ "建议重启试试" — 跳过诊断直接给方案
- ❌ "可能是网络问题" — 不可验证

### 建议动作格式

报告中每个异常项需要提供建议动作：

```
[资产名] 问题描述
  → 建议: /apexops-action 诊断 <资产名> --area=<区域>
  → 或: 在 vSphere Client 中执行 <具体操作步骤>
```

## 采集器数据解读参考

各采集器的数据字段含义、异常判断标准、阈值体系详见：
→ `references/data-guide.md`

**仅在分析巡检结果、需要解读具体字段时按需读取此文件，不要每次对话都加载。**

### vSphere 分析强制要求

**vSphere 采集器返回 13 个数据区块（platform/hosts/vms/datastores/clusters/networks/snapshots/alarms/events/host_security/compliance/performance/overcommit），summary 仅含 8-10 个关键指标摘要，远不足以支撑深度分析。**

对 `vmware_vcenter` / `vmware_esxi` 资产，Agent 必须：

1. **读取完整 JSON**：从 `json_url` 获取剪枝后的全量数据，**不可只看 summary**
2. **交叉关联**：按 data-guide.md 中的「关联矩阵」交叉比对 alarms↔events、hosts↔performance、datastores↔snapshots
3. **按诊断路径分析**：按 data-guide.md 中「诊断路径」逐项排查，不从单一指标下结论
4. **报告覆盖 6 大方面**：主机清单、数据存储、告警摘要、安全态势、快照年龄、容量趋势

## 报告输出规范

⚠️ **你必须输出结构化 JSON 报告**（v2606.100+），ApexOps 后端自动渲染为 HTML。不要直接生成 HTML。

### 输出格式

输出一个 JSON 文件，结构如下。**参数/指标用固定字段**（保证显示准确），**分析/建议用 Markdown**（保证分析深度）。

```json
{
  "meta": {
    "title": "2026-06-08 数据中心巡检报告",
    "mission_id": "a1b2c3d4",
    "timestamp": "2026-06-08 14:30",
    "asset_count": 12,
    "health_summary": {"ok": 10, "warning": 1, "critical": 1}
  },
  "overview": "## 整体评估\n\n本次巡检覆盖 **12 台设备**...",
  "top_issues": [
    {
      "severity": "critical",
      "asset": "vcenter-01",
      "title": "数据存储 ds_store_01 使用率 94%",
      "analysis": "### 根因分析\n- **根因**: 虚拟机快照未及时清理...",
      "suggestion": "建议立即清理过期快照。执行: `/apexops-action 诊断 ...`"
    }
  ],
  "assets": [
    {
      "asset_name": "vcenter-01",
      "system_version": "vmware_vcenter",
      "health": "critical",
      "parameters": [
        {"label": "CPU 使用率", "value": "23%", "status": "ok", "note": ""},
        {"label": "活动告警", "value": "12 条", "status": "critical", "note": "1 条严重告警"}
      ],
      "analysis": "### 告警详情\n已确认的严重告警 **1 条**..."
    }
  ],
  "comparisons": [
    {"asset": "域控服务器-01", "metric": "内存使用率", "previous": "78%", "current": "92%", "change": "↑14%"}
  ],
  "recommendations": "## 优先级建议\n\n1. **紧急**: 清理 vCenter 过期快照..."
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `meta.title` | string | 报告标题，必填 |
| `meta.health_summary` | object | `ok`/`warning`/`critical` 计数 |
| `overview` | Markdown | 整体评估，自由撰写 |
| `top_issues` | array | 重点问题，每个含 `severity`(critical/warning)、`asset`、`title`、`analysis`(Markdown)、`suggestion`(Markdown) |
| `assets[].parameters` | array | **固定格式**。每项: `label`(指标名)、`value`(当前值)、`status`(ok/warning/critical)、`note`(可选备注)。只放关键指标，每个资产 ≤ 15 项 |
| `assets[].analysis` | Markdown | 该资产详细分析，自由撰写 |
| `comparisons` | array | 历史对比，可选项 |
| `recommendations` | Markdown | 修复建议汇总，可选项 |

### 每资产必现字段

每个 `asset` 的 `parameters` 必须包含以下基础信息（从 `summary.metrics` 中取值）：

| 参数 label | 来源 metrics key | 说明 |
|-----------|-----------------|------|
| 主机名 | `主机名` | 计算机名，summary 自动提取 |
| IP 地址 | `IP 地址` | 从 assets[].host 获取 |
| 系统运行时间 | `运行时间` 或 `系统运行时间` | 从 summary.metrics 取 |
| 上次启动 | `上次启动` | 上次开机时间 |
| 最近登录用户 | `最近登录用户` | 近期的登录用户名列表 |
| 登录失败 | `24h 登录失败` 或 `认证失败次数` | 登录安全状态 |

> 上述字段 summary.metrics 已自动提取，直接映射到 parameters 即可。若某项缺失（如网络设备无登录用户），跳过即可。

### 硬性约束

1. **`health` 和 `status` 只能取 `"ok"` / `"warning"` / `"critical"`**，不能写 "normal"/"error" 等
2. **`parameters` 中每个对象必须有 `label`、`value`、`status`**
3. **数值类 value 带单位**（如 `"92%"` 而非 `92`），方便渲染器直接展示
4. **Markdown 字段**（overview/analysis/suggestion/recommendations）是你自由表达的空间，可以写标题、列表、表格、代码块、加粗等
5. **Markdown 中不要用 heredoc 语法**，也用不上
6. **中文字符**：所有 label、value、note 中支持中文，Markdown 中自由使用中文。后端渲染器使用 PingFang SC / Microsoft YaHei 字体，UTF-8 编码，中文显示完美

### 布局定制（可选）

可通过 `layout` 字段控制报告的展示方式。不提供时使用默认布局，完全兼容。

```json
{
  "layout": {
    "hero_metrics": [
      {"key": "health_score", "label": "健康评分", "format": "score", "color": "auto"},
      {"key": "critical_total", "label": "Critical", "format": "count", "color": "crit"}
    ],
    "asset_columns": [
      {"key": "asset_name", "label": "资产", "format": "text"},
      {"key": "health", "label": "状态", "format": "badge"}
    ],
    "domain_labels": {"vmware_esxi": "ESXi", "linux": "Linux"},
    "section_labels": {"findings": {"title": "自定义标题", "note": "自定义备注"}},
    "finding_fields": [
      {"key": "asset", "label": "资产"},
      {"key": "category", "label": "类别"}
    ]
  }
}
```

> 大多数场景不需要传 `layout`。仅在需要自定义列顺序/标签/显示内容时使用。

### 推送报告

```bash
python3 -c "
import json, subprocess, os
report = json.loads(open('/tmp/report.json','r').read())
report['mission_id'] = '<本次 mission_id>'
report['type'] = 'inspect'
url = os.environ['APEXOPS_URL'] + '/api/skills/report'
token = os.environ['APEXOPS_TOKEN']
payload = json.dumps(report, ensure_ascii=False)
r = subprocess.run(['curl','-sk','-X','POST',url,'-H','Authorization: Bearer '+token,'-H','Content-Type: application/json','-d',payload],capture_output=True,text=True)
print(r.stdout)
"
```

ApexOps 返回 `{"ok": true, "file": "inspect_xxx.html", "url": "...", "format": "structured"}`。

告知用户：`📄 完整报告: $APEXOPS_URL/reports/inspect_TIMESTAMP.html`

**此步骤不可跳过**。如果发现严重问题，主动建议用户执行诊断。
