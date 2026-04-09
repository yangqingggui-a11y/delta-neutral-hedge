# 策略名称
Delta-Neutral 对冲策略 V1.0

# 执行节奏
每 10 分钟触发一次

# Step 1 · 行情数据采集

调用 market_get_ticker 获取 BTC-USDT-SWAP 最新价格。
调用 market_get_orderbook 获取 BTC-USDT-SWAP 盘口深度（depth=1），取 bid1 和 ask1。
调用 market_get_candles 获取 BTC-USDT-SWAP 的 1m K线，计算最近 60 根的波动率（ATR）。

# Step 2 · 账户状态检查

调用 account_get_balance 获取 USDT 可用余额。
调用 account_get_positions 获取当前 BTC-USDT-SWAP 持仓。

检查条件：
1. 如果已有 BTC-USDT-SWAP 的 long 和 short 持仓 → 跳到 Step 4（监控阶段）
2. 如果可用余额 < 140 USDT → 本次跳过，等待下一轮
3. 如果只有单侧持仓（异常状态）→ 平掉单侧，等待下一轮

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

## 3.5 记录开仓价

双侧成交后，记录：
- fill_long = 多头成交价
- fill_short = 空头成交价
- mid = (fill_long + fill_short) / 2

计算止盈止损价：
- tp_long = mid × 1.002（多头止盈 +0.20%）
- sl_long = mid × 0.995（多头止损 -0.50%）
- tp_short = mid × 0.998（空头止盈 -0.20%）
- sl_short = mid × 1.005（空头止损 +0.50%）

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

## 持仓监控

OCO 单由交易所自动监控和执行，不需要轮询价格。
每 10 分钟检查一次持仓状态（account_get_positions）：
- 两侧都还在 → 继续等待
- 一侧已平（OCO触发）→ 等待另一侧 OCO 触发
- 两侧都平 → 记录结果，等待 5 秒冷却，回到 Step 1

## 结果记录

- DOUBLE_TP（双边止盈）：最优结果，两边都赚
- LONG_TP_SHORT_SL：多头止盈 + 空头止损
- SHORT_TP_LONG_SL：空头止盈 + 多头止损

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
