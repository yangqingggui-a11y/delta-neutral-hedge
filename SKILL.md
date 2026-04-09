# 策略名称
Delta-Neutral 对冲策略 V1.2

# 执行节奏
每 10 分钟触发一次

# 交易记录

所有开仓和平仓事件必须写入本地日志文件 `~/.okmem/trading/hedge_log.json`。
每条记录格式：

```json
{
  "cycle": 1,
  "status": "open|partial|closed",
  "open_time": "2026-04-09T18:26:00Z",
  "close_time": "",
  "long": {"ordId": "", "avgPx": 0, "sz": 0, "algoId": "", "state": "open|tp|sl|orphan"},
  "short": {"ordId": "", "avgPx": 0, "sz": 0, "algoId": "", "state": "open|tp|sl|orphan"},
  "result": "",
  "pnl": 0
}
```

每次触发时先读取该文件获取上一轮状态。文件不存在则初始化为空数组 `[]`。

# Step 1 · 行情数据采集

调用 market_get_ticker 获取 BTC-USDT-SWAP 最新价格。
调用 market_get_orderbook 获取 BTC-USDT-SWAP 盘口深度（depth=1），取 bid1 和 ask1。
调用 market_get_candles 获取 BTC-USDT-SWAP 的 1m K线（limit=60），计算 ATR(14)。

# Step 2 · 账户与持仓检查

并行调用：
- account_get_balance（ccy=USDT）获取可用余额
- swap_get_positions（instId=BTC-USDT-SWAP）获取持仓
- swap_get_algo_orders（instId=BTC-USDT-SWAP, ordType=oco, status=pending）获取活跃 OCO 单
- swap_get_orders（instId=BTC-USDT-SWAP）获取活跃挂单

仅关注 mgnMode=isolated 且 pos > 0 的持仓，忽略 cross 模式的空仓位。

## 2.1 状态判断

| 多头持仓 | 空头持仓 | 状态 | 动作 |
|---------|---------|------|------|
| pos > 0 | pos > 0 | 对冲完整 | → Step 4 监控 |
| pos = 0 | pos = 0 | 空仓 | → Step 3 开仓 |
| pos > 0 | pos = 0 | 孤儿单 | → Step 2.2 |
| pos = 0 | pos > 0 | 孤儿单 | → Step 2.2 |

余额检查：可用余额 < 140 USDT → 跳过本轮

## 2.2 孤儿单处理

读取 hedge_log.json 最后一条 status="open" 或 "partial" 的记录：

1. 当前持仓的 avgPx 匹配记录中某侧 → 对冲的一侧已被 OCO 平仓
   - 检查剩余侧的 algoId 是否还在（swap_get_algo_orders pending）
   - algo 还在 → 正常等待，OCO 会自动平
   - algo 不在 → 为剩余侧重新挂 OCO（Step 4 逻辑）
   - 更新 hedge_log status="partial"

2. 当前持仓不匹配任何记录 → 未知敞口
   - 不操作，记录警告，跳过本轮

3. 有活跃挂单但无持仓 → 上一轮开仓未完成
   - 匹配 hedge_log 中的 ordId → 撤单（swap_cancel_order）
   - 不匹配 → 未知挂单，不操作，记录警告

# Step 3 · 开仓（仅当空仓时）

## 3.1 AI 判断（开仓前必须执行）

1. ATR 检查：
   - ATR < 价格 × 0.05% → 波动不足，跳过
   - ATR > 价格 × 0.5% → 波动过大，保证金减半（$35/侧）
   - 正常 → $70/侧

2. 盘口价差：
   - spread = (ask1 - bid1) / mid > 0.05% → 跳过
   - spread ≤ 0.05% → 正常

3. 最近 5 轮结果（读 hedge_log）：
   - 连续 5 轮无 DTP → 暂停 30 分钟
   - DTP 率 > 40% → 正常

综合判断：开仓 / 减仓 / 跳过，记录理由。

## 3.2 设置杠杆

调用 swap_set_leverage（两次）：
- instId=BTC-USDT-SWAP, lever=20, mgnMode=isolated, posSide=long
- instId=BTC-USDT-SWAP, lever=20, mgnMode=isolated, posSide=short

## 3.3 开仓（使用 tgtCcy=margin 按保证金下单）

计算 TP/SL 价格（基于 mid = (bid1+ask1)/2）：
- tp_long = mid × 1.002
- sl_long = mid × 0.995
- tp_short = mid × 0.998
- sl_short = mid × 1.005

开多头（附带 TP/SL，一步到位）：
调用 swap_place_order：
- instId = BTC-USDT-SWAP
- tdMode = isolated
- side = buy
- posSide = long
- ordType = limit
- px = mid - 5（低于中间价$5，确保 maker）
- sz = 70（保证金 $70）
- tgtCcy = margin（系统自动按杠杆计算张数：$70 × 20 = $1400 名义）
- tpTriggerPx = tp_long
- tpOrdPx = tp_long（限价 maker）
- slTriggerPx = sl_long
- slOrdPx = sl_long（限价）

开空头（附带 TP/SL）：
调用 swap_place_order：
- instId = BTC-USDT-SWAP
- tdMode = isolated
- side = sell
- posSide = short
- ordType = limit
- px = mid + 5（高于中间价$5，确保 maker）
- sz = 70（保证金 $70）
- tgtCcy = margin
- tpTriggerPx = tp_short
- tpOrdPx = tp_short（限价 maker）
- slTriggerPx = sl_short
- slOrdPx = sl_short（限价）

说明：多空各偏移中间价$5，总敞口$10（占$72K仅0.014%，可忽略）。
使用 limit 而非 post_only，避免盘口过紧时空头侧被拒。

## 3.4 等待成交

每 5 秒检查订单状态（swap_get_order）。
超时 120 秒未双侧成交：
- 撤销未成交侧（swap_cancel_order）
- 已成交侧用 swap_close 市价平仓（紧急平仓命令）
- 等待下一轮

## 3.5 写入交易记录

双侧成交后，写入 hedge_log.json：
```json
{
  "cycle": <上一轮cycle+1>,
  "status": "open",
  "open_time": "<当前UTC时间>",
  "long": {"ordId": "<多头ordId>", "avgPx": <成交价>, "sz": <张数>, "algoId": "<附带的algoId>", "state": "open"},
  "short": {"ordId": "<空头ordId>", "avgPx": <成交价>, "sz": <张数>, "algoId": "<附带的algoId>", "state": "open"},
  "result": "",
  "pnl": 0
}
```

# Step 4 · 持仓监控与自动循环

TP/SL 由开仓时附带的条件单在交易所侧静默执行，我们无法实时感知触发事件。
每 10 分钟通过检查持仓状态来发现结果。

## 4.1 检查流程

调用 swap_get_positions（instId=BTC-USDT-SWAP），只看 isolated 且 pos > 0 的仓位：

| 多头 pos | 空头 pos | 含义 | 动作 |
|---------|---------|------|------|
| > 0 | > 0 | 两侧都在，还没触发 | 等待，检查条件单是否还在（4.2） |
| 0 | 0 | **两侧都已平 → 本轮结束** | 判断结果（4.3），开新一轮（回 Step 1） |
| > 0 | 0 | 空头已被平（TP 或 SL 触发） | 等待多头 OCO 触发 |
| 0 | > 0 | 多头已被平（TP 或 SL 触发） | 等待空头 OCO 触发 |

## 4.2 条件单健康检查

如果仍有持仓，检查对应的条件单是否还活着：
调用 swap_get_algo_orders（status=pending, instId=BTC-USDT-SWAP）

- 条件单在 → 正常等待
- 条件单消失但持仓还在 → 异常，根据 hedge_log 中的 avgPx 重新计算 TP/SL，补挂 OCO：
  调用 swap_place_algo_order（ordType=oco, reduceOnly=true, 限价）

## 4.3 本轮结束判定

当两侧都平仓后（pos=0 且 pos=0），本轮结束。

结果只有两种：
- **DOUBLE_TP**：价格先往一个方向走触发一侧 TP，再回调触发另一侧 TP
- **TP+SL**：价格单边走，一侧 TP 另一侧 SL

通过 swap_get_algo_orders(status=history) 查询两个 algoId 的触发结果确认。

## 4.4 记录并开新轮

1. 调用 account_get_positions_history 获取 realizedPnl、fee、rebate
2. 更新 hedge_log.json：status="closed"，记录 result 和 pnl
3. 检查是否满足开新仓条件：
   - 可用余额 >= $140 → 回到 Step 1 开新一轮
   - 当日累计亏损 > $50 → 停止至次日
   - 连续 5 轮无 DTP → 暂停 30 分钟
4. 满足条件 → 立即回到 Step 1，不需要等下一个 10 分钟周期

# 紧急操作

如遇异常需要立刻清仓：
调用 swap_close：
- instId = BTC-USDT-SWAP
- mgnMode = isolated
- posSide = long（或 short）
此命令市价平掉整个仓位，不需要指定张数。

# 风控规则

- 单轮最大亏损约 $4.20（一侧 TP 0.20% + 另一侧 SL 0.50%）
- 缓冲资金 $360 不得用于开仓保证金
- 可用余额低于 $140 时停止开新仓
- 当日累计亏损超过 $50（读 hedge_log 当日 closed 记录汇总）→ 停止交易至次日
- 网格策略仓位（cross 模式）绝对不碰
- 审计日志自动记录在 ~/.okx/logs/trade-YYYY-MM-DD.log
