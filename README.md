# Hacdonald's v0.2 · 团队预算配餐 Skill

把团队预算、逐人食量、饮品和名称偏好转成餐单。宿主模型理解需求，Python 做有界组合搜索，麦当劳官方 MCP 提供菜单与整篮核价；**本项目不创建或支付订单**。

> v0.2 本轮已完成代码与离线验证，尚未对当前版本重新执行麦当劳真实接口验收。文中 2026-10-24 金额是历史记录，不能作为当前报价。

项目：[tedgamer-hub/Hacdonald-s](https://github.com/tedgamer-hub/Hacdonald-s) · 快速上手：[中文快速开始](中文快速开始.md) · 请求协议：[请求与变更格式](skills/hacdonalds/references/request-format.md)

## 功能总览

| 能力 | 使用方式 |
|---|---|
| 持久团队需求 | `crew start` 新建，返回 `session_id` 与 `revision` |
| 连续修改 | `crew revise --expected-revision` 按成员 ID 更新、增删成员和预算 |
| 离线核对 | `crew show` 读取当前需求与历史；`--dry-run` 仅预览修改 |
| 中文方案说明 | 默认 JSON 含 `summary`；`--format text` 显示逐人餐品、数量、官方价、余额、偏好、门店和核价时间 |
| 搜索参考缓存 | 同门店、同采集上限的目录可复用十分钟；`--refresh` 手动重采 |
| 禁忌成员预留 | 保留 avoid，确认预留额后只为其余成员规划，结果为 `partial_verified` |
| 人工恢复 | `recovery_options` 区分预算、食量/饮品、禁忌、认证、网络和限流问题；应用层不自动重试 |

## 快速开始（Windows）

前置：Python 3.11+、D 盘、可访问依赖源与官方服务的网络。推荐将源码放在 `D:\AI-Tools\Hacdonalds\project`（其他 D 盘目录也可，含中文、空格时运行命令加引号）。

```powershell
# 1. 初始化 D 盘环境（不安装 Python）
.\setup.cmd

# 2. 在自己的终端隐藏输入 Token，勿发到聊天
.\start.cmd setup

# 3. 本地自检 + 联网工具发现
.\start.cmd           # doctor：只查本机配置与依赖
.\start.cmd check     # 联网发现工具，不代表菜单或价格验收通过

# 4. 选定门店后，把样例的 store_code 换成实际编码，先 dry-run 再正式
.\start.cmd crew start --request "examples/team-request.json" --dry-run --format text
.\start.cmd crew start --request "examples/team-request.json" --format text
```

环境、工具和依赖下载缓存集中在 `D:\AI-Tools\Hacdonalds`；源码位于 D 盘时项目 `.local` 也在 D 盘。`setup.cmd` 经 `scripts/bootstrap.py` 初始化，`bootstrap.py --check-only` 不安装、不联网。

## 核心用法

### 持久团队会话（crew）

新需求先保存，再规划。正式修改即使超时或失败也不会回滚成员要求；旧成功结果保留在历史区，不能代表新需求已成功。**每次修改前先读取最新 revision：**

```powershell
.\start.cmd crew show --session "实际会话ID" --format text
.\start.cmd crew revise --session "实际会话ID" --expected-revision 1 --changes "examples/crew-changes.json" --dry-run --format text
.\start.cmd crew revise --session "实际会话ID" --expected-revision 1 --changes "examples/crew-changes.json" --format text
```

- 用实际 ID 和刚读到的 revision 替换示例；dry-run 不增加版本。
- `crew show` 明确显示"已保存的核价记录，查看未重新核价"，原 quote 时间保留。
- 换门店应新建会话。

### 一次性规划（plan）

无需会话时仍可无状态规划：

```powershell
.\start.cmd plan --request "examples/team-request.json" --format text --output ".local/team-plan.json"
```

### 策略与推荐

三个策略：价格优先 `saver`、均衡搭配 `balanced`、丰盛搭配 `full_power`。完整 `verified` 方案按**均衡 → 价格 → 丰盛**顺序推荐；少于三套不复制凑数。差价按各套官方整篮金额计算，价格优先策略不保证最终实价最低。

### 退出码

完整规划退出 `0`；部分或未完成退出 `2`；`show` 读取成功退出 `0`，需同时看状态。

## 禁忌成员与部分规划

"不吃牛肉/辣"和过敏等要求保存在 `avoid`。名称偏好 `prefer` 不等于配料确认；`exclude_keywords` 只用于明确按名称过滤的要求。

```powershell
# 先 dry-run 查看禁忌样例（门店换成实际选定门店）
.\start.cmd crew start --request "examples/dietary-request.json" --dry-run --format text

# 明确确认"成员2 预留 25 元"后，才提交预留样例
.\start.cmd crew start --request "examples/dietary-request.json" --reservations "examples/dietary-reservations.json" --format text
```

预留规则：

- 映射金额为**正整数分**，只能对应当前有 avoid 的成员。
- `revise` 显式传映射是**完整替换**；省略时继承仍有禁忌成员的预留；显式空对象 `{}` 清除。
- 预留未齐、全员禁忌或预留用尽预算时**不联网规划**。
- `partial_verified` 表示已覆盖成员取得官方小计、禁忌成员待核实；展示覆盖/待处理名单、预留额和其余成员可用预算。预留不是消费，不能说全队已完成或给出全队最终价。

## 查询、并发与缓存

```powershell
# 先读实时 Schema，再查询
.\start.cmd schema stores
.\start.cmd schema menu --args-file 文件.json

# 手动重采（不是后台自动重试）
.\start.cmd plan --request "examples/team-request.json" --refresh --format text
.\start.cmd crew revise --session "实际会话ID" --expected-revision 2 --refresh --format text
```

- 可用查询对象：`stores` / `menu` / `detail` / `quote`；`coupons`、`nutrition` 可单独查询但未进入优化器。其他官方工具仅发现，不执行。
- 默认采集上限 24，`--limit 32` 可扩大；套餐子项、酱料和餐具不进入单品组合。
- 详情与单品参考核价按最多两个任务并发采集；每轮最多 9 个不同候选篮子核价。
- 缓存只保存单品搜索参考；每轮整篮金额仍重新查询并核对商品/数量，过期、门店或结构不符会重新采集。
- 进度写 stderr，默认 JSON 结果写 stdout。
- 只把 `official_quote.amount_fen` 当最终整篮金额；参考价格和食量规则不作营养、饱腹或精确分账结论。
- `session` 是保持一个 MCP 连接的交互调试入口，与持久保存团队需求的 `crew` 不同；接入细节见 [MCP_INTEGRATION.md](MCP_INTEGRATION.md)。

## Token 与隐私

- Token 明文保存在项目 `.local/credentials.json`，**不是系统钥匙串**。
- 外部 `MCD_MCP_TOKEN` 环境变量优先，空值也不会回退；程序不自动读取 `.env`。
- Windows 文件保护取决于 ACL；POSIX 下创建 600 文件、700 目录。
- 不要公开凭证或完整本地业务记录。

## Skill 接入

- 维护入口：[skills/hacdonalds/SKILL.md](skills/hacdonalds/SKILL.md)。
- [.agents 薄入口](.agents/skills/hacdonalds/SKILL.md)只负责仓库发现；另提供 CodeBuddy 目录与 WorkBuddy 项目级薄入口，见 [WorkBuddy 接入说明](WorkBuddy接入说明.md)。
- 宿主是否自动加载取决于客户端；当前尚无 WorkBuddy 客户端实测或真实对话记录。

## 测试与验收

```powershell
# 用初始化后的环境执行离线测试
& "D:\AI-Tools\Hacdonalds\env\Scripts\python.exe" -m unittest discover -s tests -v
```

- 离线假数据和 SDK 内存协议**不能代替**真实服务验收；现场需重新确认门店营业、菜单、商品详情、整篮业务成功与金额。
- `scripts/verify_team.py` 是固定两轮技术验收；改单由代码执行，不单独证明宿主理解连续自然语言，旧门店查询样例也不自动定位到比赛现场。
- 交付文档：[改进计划](改进计划.md)、[实施验证记录](实施验证记录.md)、[独立 Skill 行为验证](Skill行为验证记录.md)。
- 完整性：源码压缩包附带逐文件 `FILE_SHA256.json`，外部 `.zip.sha256` 校验压缩包整体。

## 历史演示（2026-10-24）

仓库记载：2026-10-24 13:57（北京时间），用户终端在沈阳盛京大奥莱餐厅完成 50 次调用，两轮均 `verified`：

| 历史需求 | 价格优先 | 均衡搭配 | 丰盛搭配 |
|---|---:|---:|---:|
| 3 人，总预算 80 元 | 64.00 元 | 78.50 元 | 79.00 元 |
| 成员1 也不要饮料，总预算 75 元 | 64.00 元 | 73.50 元 | 73.50 元 |

这些是**当时门店**的整篮价格，不代表当前报价；v0.2 的持久会话、缓存、部分规划和启动器尚未基于真实接口重新验收。旧版详情见[演示摘要](examples/demo-session.md)与[历史验证记录](VALIDATION.md)，完整记录仅在原本机保存。

## 边界与已知限制

- 不保证全菜单最优（有界搜索）。
- 未支持：套餐组合、特调、优惠券优化、营养约束、精确分账、订单操作。

## 声明

- 报名状态见 [SUBMISSION.md](SUBMISSION.md)；文档不等于已向官方提交报名。
- Hacdonald's 是独立项目，**非麦当劳官方产品**。
- 官方来源：[接入说明](https://github.com/M-China/mcd-mcp-server) · [MCP 平台](https://open.mcd.cn/mcp) · [Python 
