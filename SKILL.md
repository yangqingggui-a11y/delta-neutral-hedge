# 策略名称
Delta-Neutral 对冲策略 V1.1

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
  "long": {"ordId": "", "avgPx": 0, "sz": 2, "algoId": "", "state": "open|tp|sl|orphan"},
  "short": {"ordId": "", "avgPx": 0, "sz": 2, "algoId": "", "state": "open|tp|sl|orphan"},
  "result": "",
  "pnl": 0
}
```

每次触发时先读取该文件获取上一轮状态。文件不存在则初始化为空数组 `[]`。

# Step 1 · 行情数据采集

调用 market_get_ticker 获取 BTC-USDT-SWAP 最新价格。
调用 market_get_orderbook 获取 BTC-USDT-SWAP 盘口深度（depth=1），取 bid1 和 ask1。
调用 market_get_candles 获取 BTC-USDT-SWAP 的 1m K线，计算最近 60 根的波动率（ATR）。

# Step 2 · 账户状态检查

调用 account_get_balance 获取 USDT 可用余额。
调用 account_get_positions 获取当前 BTC-USDT-SWAP 持仓。

同时调用 swap_get_algo_orders 获取当前挂着的 OCO/conditional 委托单。

## 2.1 正常状态判断

1. 如果 long pos > 0 且 short pos > 0 → 对冲完整，跳到 Step 4（监控阶段）
2. 如果 long pos = 0 且 short pos = 0 且无挂单 → 空仓，跳到 Step 3（开仓）
3. 如果可用余额 < 140 USDT → 本次跳过，等待下一轮

## 2.2 异常状态：孤儿单处理

如果只有单侧持仓（long > 0 但 short = 0，或反之）：

1. 读取 hedge_log.json，找到最后一条 status="open" 或 "partial" 的记录
2. 用记录中的 ordId 和 avgPx 与当前持仓对比：
   - 如果当前持仓的 avgPx 匹配记录 → 这是对冲的一侧，另一侧已被 OCO 平仓
     → 检查对应的 algoId 是否还活着（swap_get_algo_orders）
     → 如果 algo 还在 → 正常等待，另一侧会被 OCO 自动平
     → 如果 algo 已触发/取消 → 标记为 "partial"，用限价单平掉剩余侧
   - 如果当前持仓的 avgPx 不匹配任何记录 → 未知敞口（可能是手动开的或其他策略）
     → 不操作，跳过本轮，记录警告日志

3. 如果存在未成交的挂单（swap_get_orders state=live）但无对应持仓：
   - 匹配 hedge_log 中的 ordId → 上一轮开仓未完成，撤单
   - 不匹配任何记录 → 未知挂单，不操作，记录警告

## 2.3 状态同步

每次检查后更新 hedge_log.json：
- 如果两侧都平了，将最后一条记录 status 改为 "closed"，记录 result 和 pnl
- 如果一侧平了，标记 status="partial"，记录哪侧已平

# Step 3 · 开仓执行（仅当无持仓时）

## 3.1 设置杠杆

调用 account_set_leverage：
- instId = BTC-USDT-SWAP
- lever = 20
- mgnMode = isolated
- posSide = long

再次调用 account_set_leverage：
- instId = BTC-USDT-SWAP
- lever = 20
- mgnMode = isolated
- posSide = short

## 3.2 计算张数

notional = 70 × 20 = 1400 USDT
qty = floor(1400 / (mid_price × 0.01))
最小 qty = 1

## 3.3 双向同时开仓

调用 swap_place_order 开多：
- instId = BTC-USDT-SWAP
- side = buy
- posSide = long
- ordType = post_only（确保 maker）
- px = bid1（盘口最优买价）
- sz = qty
- tag = "agentTradeKit"

调用 swap_place_order 开空：
- instId = BTC-USDT-SWAP
- side = sell
- posSide = short
- ordType = post_only（确保 maker）
- px = ask1 + 0.1（比卖一价高0.1，避免post_only被拒）
- sz = qty
- tag = "agentTradeKit"
- 注意：如果 post_only 被拒（cancelSource=31），改用 ask1 + 1.0 重试

## 3.4 等待成交

每 5 秒检查一次订单状态（调用 swap_get_order）。
超时 120 秒仍未双侧成交 → 撤销未成交侧（swap_cancel_order），平掉已成交侧。

## 3.5 记录开仓并计算 TP/SL

双侧成交后，写入 hedge_log.json：
```json
{
  "cycle": <上一轮cycle+1>,
  "status": "open",
  "open_time": "<当前UTC时间>",
  "long": {"ordId": "<多头订单号>", "avgPx": <多头成交价>, "sz": <张数>, "algoId": "", "state": "open"},
  "short": {"ordId": "<空头订单号>", "avgPx": <空头成交价>, "sz": <张数>, "algoId": "", "state": "open"},
  "result": "",
  "pnl": 0
}
```

计算止盈止损价（基于各自成交价）：
- tp_long = long.avgPx × 1.002（多头止盈 +0.20%）
- sl_long = long.avgPx × 0.995（多头止损 -0.50%）
- tp_short = short.avgPx × 0.998（空头止盈 -0.20%）
- sl_short = short.avgPx × 1.005（空头止损 +0.50%）

# Step 4 · 挂 TP/SL 委托单（OCO限价）

开仓成交后，立即调用 swap_place_algo_order 为两侧挂 OCO 止盈止损单。
OCO 单特性：TP 先触发则自动取消 SL，反之亦然。

## 多头 TP/SL

调用 swap_place_algo_order：
- instId = BTC-USDT-SWAP
- tdMode = isolated
- side = sell（平多头）
- posSide = long
- ordType = oco
- sz = 持仓张数
- tpTriggerPx = tp_long（开仓价 × 1.002）
- tpOrdPx = tp_long（限价，确保 maker）
- slTriggerPx = sl_long（开仓价 × 0.995）
- slOrdPx = sl_long（限价）
- reduceOnly = true

## 空头 TP/SL

调用 swap_place_algo_order：
- instId = BTC-USDT-SWAP
- tdMode = isolated
- side = buy（平空头）
- posSide = short
- ordType = oco
- sz = 持仓张数
- tpTriggerPx = tp_short（开仓价 × 0.998）
- tpOrdPx = tp_short（限价，确保 maker）
- slTriggerPx = sl_short（开仓价 × 1.005）
- slOrdPx = sl_short（限价）
- reduceOnly = true

## 记录 algoId

OCO 单挂出后，更新 hedge_log.json 当前记录：
- long.algoId = <多头 OCO algoId>
- short.algoId = <空头 OCO algoId>

## 持仓监控

OCO 单由交易所自动监控和执行，不需要轮询价格。
每 10 分钟检查一次（account_get_positions + swap_get_algo_orders）：

1. 两侧持仓都在 + 两个 OCO 都 pending → 正常，继续等待
2. 一侧持仓已平（pos=0）→ 该侧 OCO 已触发
   → 更新 hedge_log：该侧 state="tp" 或 "sl"（通过 swap_get_algo_orders history 查询结果）
   → 更新 status="partial"
   → 等待另一侧 OCO 触发
3. 两侧都平（pos=0）→ 两个 OCO 都已触发
   → 更新 hedge_log：status="closed"，记录 result 和 pnl
   → 等待 5 秒冷却，回到 Step 1
4. 持仓在但对应的 OCO 消失（被手动取消等）→ 重新挂 OCO

## 结果与 PnL 记录

根据两侧 state 确定结果：
- long.state="tp" + short.state="tp" → DOUBLE_TP
- long.state="tp" + short.state="sl" → LONG_TP_SHORT_SL
- long.state="sl" + short.state="tp" → SHORT_TP_LONG_SL

PnL 计算：调用 account_get_positions_history 获取已平仓位的 realizedPnl，写入 hedge_log。

# Step 5 · AI 综合判断（核心）

基于以上数据，你作为交易 AI 需要在每轮开仓前判断：

1. 当前 ATR 是否在合理区间？
   - ATR 过低（< 价格的 0.05%）→ 波动不足，DTP 概率低，跳过本轮
   - ATR 过高（> 价格的 0.5%）→ 波动过大，SL 风险高，减半仓位（qty ÷ 2）
   - ATR 正常 → 正常开仓

2. 盘口价差是否合理？
   - spread = (ask1 - bid1) / mid
   - spread > 0.05% → 价差过大，开仓成本高，跳过
   - spread <= 0.05% → 正常开仓

3. 最近 5 轮结果分析：
   - 连续 5 轮 TPSL（无 DTP）→ 市场可能进入单边行情，暂停 30 分钟
   - DTP 率 > 40% → 策略运行良好，正常执行

综合以上三点，给出决策：开仓 / 减仓 / 跳过，并说明理由。

# 风控规则

// 单轮最大亏损约 $4.20（一侧 TP + 一侧 SL）
// 缓冲资金 $360 不得用于开仓保证金
// 可用余额低于 $140 时停止开新仓
// 当日累计亏损超过 $50（账户净值 10%）→ 停止交易至次日
// 所有下单必须带 tag = "agentTradeKit"
