//@version=5
strategy("Professional Sniper v18 - Refactored Core", shorttitle="PS v18", overlay=true,
     initial_capital=10000, default_qty_type=strategy.fixed,
     commission_type=strategy.commission.percent, commission_value=0.05,
     slippage=1, calc_on_every_tick=false, process_orders_on_close=true,
     pyramiding=0, max_labels_count=100)

// ============================================================================
// Professional Sniper v18
// Refactored core: Classic breakout -> retest -> reclaim plus risk sizing.
// This version is intentionally conservative and avoids illegal/unsupported
// Pine constructs that caused the previous build errors.
// ============================================================================

g1 = "1) Regime and HTF"
allowLong = input.bool(true, "Allow Long", group=g1)
allowShort = input.bool(false, "Allow Short", group=g1)
adxLen = input.int(14, "ADX Length", minval=2, group=g1)
adxMin = input.float(20.0, "Minimum ADX", minval=1.0, group=g1)
regimeLength = input.int(50, "Regime Window", minval=5, group=g1)
regimeMin = input.float(55.0, "Minimum Trending Bars %", minval=0.0, maxval=100.0, group=g1)
useHTF = input.bool(true, "Use Confirmed HTF Filter", group=g1)
htfTimeframe = input.timeframe("60", "HTF", group=g1)
htfEmaLength = input.int(50, "HTF EMA Length", minval=2, group=g1)

g2 = "2) Classic Breakout / Retest"
structureLength = input.int(20, "Structure Lookback", minval=5, group=g2)
breakoutAtrBuffer = input.float(0.15, "Breakout ATR Buffer", minval=0.0, step=0.05, group=g2)
breakoutBodyAtr = input.float(0.35, "Minimum Breakout Body ATR", minval=0.0, step=0.05, group=g2)
breakoutCloseLocation = input.float(0.65, "Breakout Close Location", minval=0.5, maxval=0.95, step=0.05, group=g2)
retestBars = input.int(8, "Maximum Retest Bars", minval=1, maxval=50, group=g2)
retestToleranceAtr = input.float(0.25, "Retest Tolerance ATR", minval=0.01, step=0.01, group=g2)
reclaimAtr = input.float(0.05, "Reclaim ATR", minval=0.0, step=0.01, group=g2)
allowSweep = input.bool(true, "Allow Liquidity Sweep Retest", group=g2)
requireRejection = input.bool(true, "Require Retest Rejection", group=g2)

g3 = "3) Risk"
stopPercent = input.float(1.5, "Stop Loss %", minval=0.1, group=g3)
targetPercent = input.float(4.0, "Target %", minval=0.1, group=g3)
tp1Percent = input.float(2.0, "TP1 %", minval=0.1, group=g3)
minStopPercent = input.float(0.3, "Minimum Stop Distance %", minval=0.05, group=g3)
riskPercent = input.float(1.0, "Risk % of Equity", minval=0.1, group=g3)
maxPositionPercent = input.float(50.0, "Maximum Position % of Equity", minval=1.0, maxval=100.0, group=g3)
usePartial = input.bool(true, "Use Partial Exit", group=g3)
partialPercent = input.float(50.0, "TP1 Partial %", minval=10.0, maxval=90.0, group=g3)
breakevenBuffer = input.float(0.05, "Breakeven Buffer %", minval=0.0, group=g3)
maxBarsInTrade = input.int(40, "Maximum Bars in Trade", minval=1, group=g3)

g4 = "4) Quality"
useQuality = input.bool(false, "Use Quality Gate", group=g4)
minimumQuality = input.float(6.0, "Minimum Quality / 10", minval=0.0, maxval=10.0, group=g4)
volumeLength = input.int(20, "Volume Average Length", minval=2, group=g4)
volumeReference = input.float(1.5, "Reference Volume Ratio", minval=0.1, group=g4)

g5 = "5) Execution and Diagnostics"
entryCooldownBars = input.int(6, "Cooldown After Exit", minval=0, group=g5)
useExtensionGuard = input.bool(false, "Use Extension Guard", group=g5)
maxExtensionAtr = input.float(2.0, "Maximum Extension ATR", minval=0.5, group=g5)
showDiagnostics = input.bool(true, "Show Diagnostics", group=g5)

// ---------------------------------------------------------------------------
// Core calculations
// ---------------------------------------------------------------------------
atr = ta.atr(14)
barRange = high - low
body = math.abs(close - open)
volumeAverage = ta.sma(volume, volumeLength)
volumeRatio = volumeAverage > 0 ? volume / volumeAverage : na
bodyRatio = barRange > 0 ? body / barRange : 0.0

[diPlus, diMinus, adx] = ta.dmi(adxLen, adxLen)
isTrending = adx >= adxMin
trendPercent = ta.sma(isTrending ? 1.0 : 0.0, regimeLength) * 100.0
regimeReady = not na(trendPercent) and trendPercent >= regimeMin

htfEma = request.security(syminfo.tickerid, htfTimeframe, ta.ema(close, htfEmaLength), gaps=barmerge.gaps_off, lookahead=barmerge.lookahead_off)
htfReady = not na(htfEma)
longBias = not useHTF or (htfReady and close > htfEma)
shortBias = not useHTF or (htfReady and close < htfEma)

historyReady = bar_index >= math.max(structureLength, math.max(regimeLength, volumeLength))
dataReady = historyReady and not na(atr) and atr > syminfo.mintick and not na(volumeAverage) and htfReady

extensionLong = atr > 0 and htfReady ? (close - htfEma) / atr : na
extensionShort = atr > 0 and htfReady ? (htfEma - close) / atr : na
extensionLongOk = not useExtensionGuard or na(extensionLong) or (extensionLong >= 0 and extensionLong <= maxExtensionAtr)
extensionShortOk = not useExtensionGuard or na(extensionShort) or (extensionShort >= 0 and extensionShort <= maxExtensionAtr)

qualityLong = math.min(2.5, nz(volumeRatio, 0.0) / volumeReference * 2.5) +
     math.min(2.0, bodyRatio * 2.0) +
     (longBias ? 2.0 : 0.0) +
     math.min(1.5, math.max(0.0, trendPercent - regimeMin) / math.max(1.0, 100.0 - regimeMin) * 1.5) +
     (diPlus > diMinus ? 1.5 : 0.0) +
     (barstate.isconfirmed ? 0.5 : 0.0)

qualityShort = math.min(2.5, nz(volumeRatio, 0.0) / volumeReference * 2.5) +
     math.min(2.0, bodyRatio * 2.0) +
     (shortBias ? 2.0 : 0.0) +
     math.min(1.5, math.max(0.0, trendPercent - regimeMin) / math.max(1.0, 100.0 - regimeMin) * 1.5) +
     (diMinus > diPlus ? 1.5 : 0.0) +
     (barstate.isconfirmed ? 0.5 : 0.0)

// ---------------------------------------------------------------------------
// Classic state machine
// ---------------------------------------------------------------------------
IDLE = 0
BROKEN = 1
RETESTED = 2

var int longState = IDLE
var int shortState = IDLE
var float longLevel = na
var float shortLevel = na
var int longBreakBar = na
var int shortBreakBar = na
var int longRetestBar = na
var int shortRetestBar = na
var bool longRejected = false
var bool shortRejected = false

resistance = ta.highest(high[1], structureLength)
support = ta.lowest(low[1], structureLength)
closeLocationLong = barRange > 0 ? (close - low) / barRange : 0.5
closeLocationShort = barRange > 0 ? (high - close) / barRange : 0.5

bullBreak = dataReady and close > resistance and close >= resistance + atr * breakoutAtrBuffer and body >= atr * breakoutBodyAtr and closeLocationLong >= breakoutCloseLocation
bearBreak = dataReady and close < support and close <= support - atr * breakoutAtrBuffer and body >= atr * breakoutBodyAtr and closeLocationShort >= breakoutCloseLocation

if strategy.position_size == 0 and longState == IDLE and bullBreak and allowLong and longBias and regimeReady
    longState := BROKEN
    longLevel := resistance
    longBreakBar := bar_index
    longRetestBar := na
    longRejected := false
    shortState := IDLE
    shortLevel := na
    shortBreakBar := na
    shortRetestBar := na
    shortRejected := false

if strategy.position_size == 0 and shortState == IDLE and bearBreak and allowShort and shortBias and regimeReady
    shortState := BROKEN
    shortLevel := support
    shortBreakBar := bar_index
    shortRetestBar := na
    shortRejected := false
    longState := IDLE
    longLevel := na
    longBreakBar := na
    longRetestBar := na
    longRejected := false

if longState != IDLE and bar_index - nz(longBreakBar, bar_index) > retestBars
    longState := IDLE
    longLevel := na
    longBreakBar := na
    longRetestBar := na
    longRejected := false

if shortState != IDLE and bar_index - nz(shortBreakBar, bar_index) > retestBars
    shortState := IDLE
    shortLevel := na
    shortBreakBar := na
    shortRetestBar := na
    shortRejected := false

lowerWick = math.min(open, close) - low
upperWick = high - math.max(open, close)
realBody = math.max(body, syminfo.mintick)
bullReject = close > open and lowerWick >= realBody * 0.5
bearReject = close < open and upperWick >= realBody * 0.5
bullEngulf = close > open and close >= open[1] and open <= close[1]
bearEngulf = close < open and close <= open[1] and open >= close[1]

longTolerance = nz(atr) * retestToleranceAtr
shortTolerance = nz(atr) * retestToleranceAtr
longTouch = longState == BROKEN and bar_index > longBreakBar and low <= longLevel + longTolerance and low >= longLevel - longTolerance
shortTouch = shortState == BROKEN and bar_index > shortBreakBar and high >= shortLevel - shortTolerance and high <= shortLevel + shortTolerance
longSweep = longState == BROKEN and allowSweep and bar_index > longBreakBar and low < longLevel and close > longLevel
shortSweep = shortState == BROKEN and allowSweep and bar_index > shortBreakBar and high > shortLevel and close < shortLevel

if longTouch or longSweep
    longState := RETESTED
    longRetestBar := bar_index
    longRejected := not requireRejection or bullReject or bullEngulf

if shortTouch or shortSweep
    shortState := RETESTED
    shortRetestBar := bar_index
    shortRejected := not requireRejection or bearReject or bearEngulf

longReclaim = longState == RETESTED and bar_index > longRetestBar and close > longLevel + atr * reclaimAtr and close > open
shortReclaim = shortState == RETESTED and bar_index > shortRetestBar and close < shortLevel - atr * reclaimAtr and close < open

classicLongSignal = longReclaim and longRejected and longBias and regimeReady and extensionLongOk and (not useQuality or qualityLong >= minimumQuality)
classicShortSignal = shortReclaim and shortRejected and shortBias and regimeReady and extensionShortOk and (not useQuality or qualityShort >= minimumQuality)

// ---------------------------------------------------------------------------
// Risk and sizing
// ---------------------------------------------------------------------------
f_stopLong(price) => price - math.max(price * stopPercent / 100.0, price * minStopPercent / 100.0)
f_stopShort(price) => price + math.max(price * stopPercent / 100.0, price * minStopPercent / 100.0)

f_qty(entryPrice, stopPrice) =>
    riskCash = strategy.equity > 0 ? strategy.equity * riskPercent / 100.0 : 0.0
    riskPerUnit = math.abs(entryPrice - stopPrice)
    rawQty = riskPerUnit > 0 ? riskCash / riskPerUnit : 0.0
    unitValue = entryPrice
    maxQty = unitValue > 0 ? strategy.equity * maxPositionPercent / 100.0 / unitValue : 0.0
    step = 1.0
    math.floor(math.min(rawQty, maxQty) / step) * step

var int lastExitBar = na
var int activeDirection = 0
var int activeEntryBar = na
var float activeEntry = na
var float activeStop = na
var float activeTarget = na
var float activeTp1 = na
var bool partialDone = false
var float tradeProfitAtEntry = na
var float activeQuality = na
var string activeRoute = ""

cooldownOk = na(lastExitBar) or bar_index - lastExitBar >= entryCooldownBars
canEnter = dataReady and strategy.position_size == 0 and cooldownOk and barstate.isconfirmed

if canEnter and classicLongSignal and not classicShortSignal
    plannedEntry = close
    plannedStop = f_stopLong(plannedEntry)
    plannedTarget = plannedEntry * (1 + targetPercent / 100.0)
    qty = f_qty(plannedEntry, plannedStop)
    if qty > 0
        strategy.entry("CLASSIC_LONG", strategy.long, qty=qty)
        activeRoute := "CLASSIC_LONG"
        activeQuality := qualityLong
        longState := IDLE

if canEnter and classicShortSignal and not classicLongSignal
    plannedEntry = close
    plannedStop = f_stopShort(plannedEntry)
    plannedTarget = plannedEntry * (1 - targetPercent / 100.0)
    qty = f_qty(plannedEntry, plannedStop)
    if qty > 0
        strategy.entry("CLASSIC_SHORT", strategy.short, qty=qty)
        activeRoute := "CLASSIC_SHORT"
        activeQuality := qualityShort
        shortState := IDLE

justEntered = strategy.position_size != 0 and strategy.position_size[1] == 0
if justEntered
    activeDirection := strategy.position_size > 0 ? 1 : -1
    activeEntry := strategy.position_avg_price
    activeStop := activeDirection == 1 ? f_stopLong(activeEntry) : f_stopShort(activeEntry)
    activeTarget := activeDirection == 1 ? activeEntry * (1 + targetPercent / 100.0) : activeEntry * (1 - targetPercent / 100.0)
    activeTp1 := activeDirection == 1 ? activeEntry * (1 + tp1Percent / 100.0) : activeEntry * (1 - tp1Percent / 100.0)
    activeEntryBar := bar_index
    partialDone := false
    tradeProfitAtEntry := strategy.netprofit

if strategy.position_size > 0
    if usePartial and not partialDone and high >= activeTp1
        strategy.close("CLASSIC_LONG", qty_percent=partialPercent, comment="TP1_LONG")
        partialDone := true
        activeStop := activeEntry * (1 + breakevenBuffer / 100.0)
    strategy.exit("EXIT_LONG", from_entry="CLASSIC_LONG", stop=activeStop, limit=partialDone ? na : activeTarget)

if strategy.position_size < 0
    if usePartial and not partialDone and low <= activeTp1
        strategy.close("CLASSIC_SHORT", qty_percent=partialPercent, comment="TP1_SHORT")
        partialDone := true
        activeStop := activeEntry * (1 - breakevenBuffer / 100.0)
    strategy.exit("EXIT_SHORT", from_entry="CLASSIC_SHORT", stop=activeStop, limit=partialDone ? na : activeTarget)

if strategy.position_size != 0 and not na(activeEntryBar) and bar_index - activeEntryBar >= maxBarsInTrade and not partialDone
    strategy.close_all(comment="TIME_CAP")

positionClosed = strategy.position_size == 0 and strategy.position_size[1] != 0

var int completedTrades = 0
var int completedWins = 0
var float completedProfit = 0.0
var float completedLoss = 0.0
var float lastTradeProfit = na

if positionClosed
    lastExitBar := bar_index
    lastTradeProfit := strategy.netprofit - nz(tradeProfitAtEntry, strategy.netprofit)
    completedTrades += 1
    if lastTradeProfit > 0
        completedWins += 1
        completedProfit += lastTradeProfit
    else
        completedLoss += math.abs(lastTradeProfit)
    activeDirection := 0
    activeEntryBar := na
    activeEntry := na
    activeStop := na
    activeTarget := na
    activeTp1 := na
    partialDone := false
    tradeProfitAtEntry := na
    activeRoute := ""
    activeQuality := na

winRate = completedTrades > 0 ? completedWins / completedTrades * 100.0 : na
profitFactor = completedLoss > 0 ? completedProfit / completedLoss : completedProfit > 0 ? 99.0 : 0.0

plot(useHTF ? htfEma : na, \"Confirmed HTF EMA\", color=color.aqua, linewidth=2)
plotshape(showDiagnostics and classicLongSignal and canEnter, \"Classic Long Signal\", shape.triangleup, location.belowbar, color.lime, size=size.tiny)
plotshape(showDiagnostics and classicShortSignal and canEnter, \"Classic Short Signal\", shape.triangledown, location.abovebar, color.red, size=size.tiny)

var table report = table.new(position.top_right, 2, 9, border_width=1)
if barstate.islast
    table.cell(report, 0, 0, \"Professional Sniper v18\", bgcolor=color.rgb(36,27,78), text_color=color.yellow)
    table.cell(report, 1, 0, syminfo.ticker + \" / \" + timeframe.period, bgcolor=color.rgb(36,27,78), text_color=color.yellow)
    table.cell(report, 0, 1, \"Completed trades\")
    table.cell(report, 1, 1, str.tostring(completedTrades))
    table.cell(report, 0, 2, \"Win rate\")
    table.cell(report, 1, 2, na(winRate) ? \"-\" : str.tostring(winRate, \"#.##\") + \"%\")
    table.cell(report, 0, 3, \"Profit factor\")
    table.cell(report, 1, 3, str.tostring(profitFactor, \"#.##\"))
    table.cell(report, 0, 4, \"Regime\")
    table.cell(report, 1, 4, regimeReady ? \"READY\" : \"BLOCKED\")
    table.cell(report, 0, 5, \"HTF\")
    table.cell(report, 1, 5, longBias ? \"BULL\" : shortBias ? \"BEAR\" : \"NEUTRAL\")
    table.cell(report, 0, 6, \"Classic state\")
    table.cell(report, 1, 6, \"L=\" + str.tostring(longState) + \" / S=\" + str.tostring(shortState))
    table.cell(report, 0, 7, \"Quality\")
    table.cell(report, 1, 7, \"L=\" + str.tostring(qualityLong, \"#.##\") + \" / S=\" + str.tostring(qualityShort, \"#.##\"))
    table.cell(report, 0, 8, \"Active route\")
    table.cell(report, 1, 8, activeRoute == \"\" ? \"-\" : activeRoute)
