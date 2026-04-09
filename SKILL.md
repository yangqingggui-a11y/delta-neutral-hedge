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
- ordType = limit
- px = bid1（盘口最优买价，确保 maker）
- sz = qty
- tag = "agentTradeKit"

调用 swap_place_order 开空：
- instId = BTC-USDT-SWAP
- side = sell
- posSide = short
- ordType = limit
- px = ask1（盘口最优卖价，确保 maker）
- sz = qty
- tag = "agentTradeKit"

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

# Step 4 · 持仓监控与平仓

每 2 秒调用 market_get_ticker 获取最新价格，判断是否触发止盈或止损。

## 多头判断
- 如果 last >= tp_long → 平多头
- 如果 last <= sl_long → 平多头

## 空头判断
- 如果 last <= tp_short → 平空头
- 如果 last >= sl_short → 平空头

## 平仓执行

平多头时调用 swap_place_order：
- instId = BTC-USDT-SWAP
- side = sell
- posSide = long
- ordType = limit
- px = ask1（挂卖一价，maker）
- sz = 持仓张数
- tag = "agentTradeKit"

平空头时调用 swap_place_order：
- instId = BTC-USDT-SWAP
- side = buy
- posSide = short
- ordType = limit
- px = bid1（挂买一价，maker）
- sz = 持仓张数
- tag = "agentTradeKit"

## 两侧都平仓后

记录本轮结果：
- DOUBLE_TP（双边止盈）：最优结果，两边都赚
- LONG_TP_SHORT_SL：多头止盈+空头止损
- SHORT_TP_LONG_SL：空头止盈+多头止损

等待 5 秒冷却，回到 Step 1 开始下一轮。

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
