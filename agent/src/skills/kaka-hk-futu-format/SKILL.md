---
name: kaka-hk-futu-format
description: [本地自维护 kaka] 港股富途（Futu）代码格式与命中身份锁的固定规避方案 — 分析港股富途持仓/个股时必须遵守。行情用 .HK 后缀代码，富途持仓用 HK. 前缀代码，搜索/概况工具失败时按本规则的夹具序列执行。
category: workflow
origin: local-kaka
maintainer: kaka
---

# 港股富途代码格式与命中身份锁的固定规避方案

> **本技能由本地用户 kaka 自维护，非上游 Vibe-Trading 官方技能。**
> 标识：`origin: local-kaka`。同步上游 main 分支时注意保留，避免被覆盖。

> **用途**：当用户要求分析**富途模拟盘/实盘持仓**或**港股个股**时，始终应用本技能。它解决 vibe-trading 内部"代码格式分歧"导致的 `identity_conflict` / 空数据问题。

## 核心铁律（必须遵守）

### 1. 代码格式按工具分，绝不混用

vibe-trading 里港股有两种格式，**用途不同，绝不混用**：

| 工具 | 代码格式 | 示例 |
|------|---------|------|
| `get_market_data`（行情 fallback 链：腾讯/东财/...）| **后缀** `<code>.HK` | `01288.HK`、`06693.HK` |
| `trading_quote` / `trading_history`（富途连接）| **前缀** `HK.<code>` | `HK.01288` |
| `trading_positions` / `trading_account`（富途持仓/账户）| 富途返回即 `HK.<code>` | 读取即可，无需转换 |

- **绝不能**把 `HK.01288` 当成行情代码传给 `get_market_data`（会导致 `_unresolved`）；
- **绝不能**把 `01288.HK` 传给富途 `trading_quote`（返回空 quote）。

### 2. 身份锁（identity gate）规避序列 —— 正确顺序（实测有效）

`identity_required` / `identity_conflict` 的触发原因：**在取价格工具（`get_market_data` / `get_financial_statements` / `get_sector_info` / `trading_quote` / `trading_history`）执行前，身份锁必须是 `locked` 状态**。这个锁**只能由 `search_symbol` 锁定**，而且 lock 必须发生在取价工具之前、**同一批或前一批完成**。取价工具本身不能用来锁定身份（会报 `identity_required`）。

**关键：`search_symbol` 不能带前导 0 的 `.HK`（如 `06693.HK`），也不能用裸数字（`06693` 会返回 `066938.TW` 错误候选）。要用能唯一锁定目标港股的查询：英文全名 或 去掉前导 0 的代码。**

**已验证能锁定 `06693.HK` 的查询：**
- `search_symbol("Chifeng Gold")` → 候选 `06693.HK`(market=hk) ✅
- `search_symbol("6693.HK")` → 候选 `06693.HK`(market=hk) ✅
- `search_symbol("06693.HK")` → count=0（Eastmoney 挂 + Yahoo 找不到）❌
- `search_symbol("06693")` → 返回 `066938.TW`（台股，错误）❌

**固定正确的调用序列（按此顺序，缺一不可）：**

1. **先锁定身份**：用 `search_symbol(英文名或去前导0代码)` 锁定目标港股，记下返回的 `<code>.HK`（后缀格式）。示例：`search_symbol("Chifeng Gold")` → `06693.HK`。
2. **后取行情**：用 `get_market_data`，代码用**刚锁定的 `.HK` 后缀格式**（`06693.HK`）。此时身份已锁，可正常执行。
3. **再取财务/行业**：`get_financial_statements` / `get_sector_info` 同样用 `.HK` 后缀（身份已锁即可通过）。
4. **富途持仓**：`trading_positions`（连接用 `futu-paper-sdk` 或 `futu-live-sdk-readonly`）；持仓返回 `HK.<code>`，只是读取展示，**不用来当行情代码**。
5. **组合风险（可选）**：`portfolio_risk_xray` **必须传 `symbols` 参数**（列表，用 `.HK` 后缀），否则报 `symbols must be a non-empty list`。例：`portfolio_risk_xray(symbols=["06693.HK"])`。

### 3. 禁止事项

- ❌ 不要用 `search_symbol("06693.HK")` / `search_symbol("01288")` / 中文名 / 带前导 0 的代码——锁不住身份。
- ❌ **绝不要**用 `get_market_data` 去"锁定身份"——它只能读，不能锁；身份必须先用 `search_symbol` 锁。
- ❌ 不要用富途前缀 `HK.xxxxx` 去调行情链 `get_market_data`（会 `_unresolved`）。
- ❌ 不要在同一批次把 `.HK` 后缀和 `HK.` 前缀混用，会触发 `identity_conflict`。
- ❌ `portfolio_risk_xray` 不要空参调用。

### 4. 若工具仍失败

- 东财解析失败（`Expecting value`）或 Yahoo `403/429` 时，**接受现状**：只用能通过的工具（`get_market_data` 行情 + `trading_positions` 持仓），在结论里标注"基本面/新闻工具当前源不可用"，**不要编造**数字。
- 所有数字必须来自工具输出；无法通过工具验证的价格不要写进结论。

## 用户常用问法模板

> **分析我的富途模拟持仓农业银行(01288.HK)和中远海控(01919.HK)。用 tool：1) trading_positions 读富途模拟账户持仓盈亏；2) get_market_data 取 01288.HK 和 01919.HK 近期K线（用 .HK 后缀）；3) 若要组合风险，用 portfolio_risk_xray 且传 symbols=['01288.HK','01919.HK']。基于工具数据给出分析和结论，所有数字必须来自工具输出。**
