# momentum-Bias-Conformation-Overlay-1

A visual-only market-state overlay for TradingView, designed around a 3-minute execution chart with 15-minute higher-timeframe context. Combines local MACD/RSI momentum, session VWAP bias, HTF EMA(21) trend, ADX/DI strength, and a 15m ATR ZLEMA trend-state into a single compact score dashboard and a set of non-repainting trade markers.

**Features**
- 3m MACD histogram + RSI local momentum coloring
- Session VWAP with configurable neutral buffer
- 15m EMA(21) directional bias and ADX/DI strength gate
- Optional 15m MACD/RSI momentum confirmation
- 15m ATR ZLEMA trend-state with confirmed and potential reversal markers
- Compact score dashboard (top-right) summarising all layers
- All 15m data sourced from the last *confirmed* bar — no repainting

```pine
// This Pine Script® code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © luckyrooster
//@version=6
indicator("3m / 15m Alignment Overlay + Confirmed 15m ATR ZLEMA State", shorttitle="3m-15m Align + ATRZ C", overlay=true, max_bars_back=1000, max_labels_count=500, max_boxes_count=200)

//====================================================================
// PURPOSE
//====================================================================
// Visual-only market-state overlay.
// Designed around a 3m execution chart with 15m context projected onto it.
//
// Core layers:
// - 3m MACD histogram + RSI local momentum
// - 3m Session VWAP location
// - 15m EMA(21) directional bias
// - 15m ADX / DI strength
// - Optional 15m MACD / RSI momentum confirmation
// - 15m ATR ZLEMA trend-state + reversal-state
// - Compact score dashboard
//
// HTF timing model:
// All 15m requests use the last confirmed 15m bar only.
// This keeps the overlay stable and non-repainting.
//
// Marker model:
// - Buy      = bullish regime begins
// - Sell     = bearish regime begins
// - StopSell = exit long when bullish regime ends without same-bar short flip
// - StopBuy  = exit short when bearish regime ends without same-bar long flip
// - Bull/Bear Reversal = confirmed 15m ATR ZLEMA reversal-state begins
// - Potential Bull/Bear Reversal = regime first disagrees with 15m ATR ZLEMA state
//
// Marker rendering fix:
// Markers use label.new() instead of plotshape() to avoid TradingView's
// 64-plot script limit while preserving configurable marker sizes.

//====================================================================
// INPUT GROUPS
//====================================================================
const string GROUP_CORE   = "Core Inputs"
const string GROUP_HTF    = "Higher Timeframe Inputs"
const string GROUP_ATRZ   = "15m ATR ZLEMA Reversal-State Inputs"
const string GROUP_VISUAL = "Visual Settings"
const string GROUP_LABELS = "Signal / Marker Settings"
const string GROUP_BOXES  = "15m Candle Box Overlay"
const string GROUP_COLORS = "Colors"

//====================================================================
// CORE INPUTS
//====================================================================
fastLength         = input.int(12, "MACD Fast Length", minval=1, maxval=100, group=GROUP_CORE)
slowLength         = input.int(26, "MACD Slow Length", minval=1, maxval=100, group=GROUP_CORE)
signalLength       = input.int(9, "MACD Signal Length", minval=1, maxval=100, group=GROUP_CORE)
rsiLength          = input.int(14, "RSI Length", minval=1, maxval=100, group=GROUP_CORE)
vwapNeutralAtrLen  = input.int(14, "VWAP Neutral Buffer ATR Length", minval=1, maxval=100, group=GROUP_CORE)
vwapNeutralAtrFrac = input.float(0.10, "VWAP Neutral Buffer ATR Fraction", minval=0.0, maxval=5.0, step=0.01, group=GROUP_CORE)

//====================================================================
// HIGHER TIMEFRAME INPUTS
//====================================================================
htfTf                 = input.timeframe("15", "Higher Timeframe", group=GROUP_HTF)
emaLen                = input.int(21, "HTF EMA Length", minval=1, maxval=100, group=GROUP_HTF)
adxLen                = input.int(14, "ADX DI Length", minval=1, maxval=100, group=GROUP_HTF)
adxSmooth             = input.int(14, "ADX Smoothing", minval=1, maxval=100, group=GROUP_HTF)
adxThreshold          = input.float(20.0, "ADX Threshold", minval=1.0, maxval=100.0, step=0.5, group=GROUP_HTF)
useHTFMomentumConfirm = input.bool(true, "Use 15m MACD / RSI Momentum Confirmation", group=GROUP_HTF)

//====================================================================
// 15m ATR ZLEMA INPUTS
//====================================================================
atrzSourceChoice     = input.string("Close", "ATR ZLEMA Source", options=["Close", "Open", "High", "Low", "HL2", "HLC3", "OHLC4"], group=GROUP_ATRZ)
zlemaLengthInput     = input.int(14, "ZLEMA Length", minval=1, group=GROUP_ATRZ)
atrLengthInput       = input.int(14, "ATR Length", minval=1, group=GROUP_ATRZ)
atrMultiplierInput   = input.float(3.5, "ATR Multiplier", minval=0.1, step=0.1, group=GROUP_ATRZ)
enableNoiseFilter    = input.bool(true, "Enable Noise Filter", group=GROUP_ATRZ)
noiseFilterInput     = input.float(0.3, "Noise Filter Threshold", minval=0.0, step=0.1, group=GROUP_ATRZ)
presetConfig         = input.string("Default", "ATR ZLEMA Preset", options=["Default", "Fast Response", "Smooth Trend"], group=GROUP_ATRZ)

showHTFATRZTrail     = input.bool(true, "Plot 15m ATR ZLEMA Trailing Line", group=GROUP_ATRZ)
showHTFATRZZLEMA     = input.bool(false, "Plot 15m ATR ZLEMA Line", group=GROUP_ATRZ)
showHTFReversalMarks = input.bool(true, "Show Confirmed 15m Reversal Symbols", group=GROUP_ATRZ)

// Apply ATR ZLEMA preset values
int   zlemaLength   = zlemaLengthInput
int   atrLength     = atrLengthInput
float atrMultiplier = atrMultiplierInput
float noiseFilter   = noiseFilterInput

if presetConfig == "Fast Response"
    zlemaLength   := 10
    atrLength     := 10
    atrMultiplier := 2.5
    noiseFilter   := 0.2
else if presetConfig == "Smooth Trend"
    zlemaLength   := 21
    atrLength     := 21
    atrMultiplier := 5.0
    noiseFilter   := 0.5

//====================================================================
// VISUAL INPUTS
//====================================================================
showVWAP      = input.bool(true, "Plot Session VWAP", group=GROUP_VISUAL)
showHTFEMA    = input.bool(true, "Plot 15m EMA(21)", group=GROUP_VISUAL)
showHTFClose  = input.bool(false, "Plot 15m Close (Stepped)", group=GROUP_VISUAL)
showBarColors = input.bool(true, "Color Base Bars by Final Regime", group=GROUP_VISUAL)
showBgZones   = input.bool(true, "Show Background Zones", group=GROUP_VISUAL)
showDashboard = input.bool(true, "Show Compact Dashboard", group=GROUP_VISUAL)

showTradeSignals             = input.bool(true, "Show Buy / Sell Symbols", group=GROUP_LABELS)
showExitSignals              = input.bool(true, "Show StopBuy / StopSell Symbols", group=GROUP_LABELS)
showPotentialReversalSymbols = input.bool(true, "Show Potential Reversal Symbols", group=GROUP_LABELS)
tradeSignalSize              = input.string(size.tiny, "Buy / Sell Symbol Size", options=[size.tiny, size.small, size.normal, size.large, size.huge], group=GROUP_LABELS)
exitSignalSize               = input.string(size.small, "StopBuy / StopSell Symbol Size", options=[size.tiny, size.small, size.normal, size.large, size.huge], group=GROUP_LABELS)
confirmedReversalSize        = input.string(size.tiny, "Confirmed Reversal Symbol Size", options=[size.tiny, size.small, size.normal, size.large, size.huge], group=GROUP_LABELS)
potentialReversalSize        = input.string(size.tiny, "Potential Reversal Symbol Size", options=[size.tiny, size.small, size.normal, size.large, size.huge], group=GROUP_LABELS)

showHTFBoxes  = input.bool(false, "Show 15m Candle Boxes", group=GROUP_BOXES)
htfBoxOpacity = input.int(88, "15m Box Opacity (0-100)", minval=0, maxval=100, group=GROUP_BOXES)
maxBoxCount   = input.int(60, "Max 15m Boxes to Keep", minval=10, maxval=200, group=GROUP_BOXES)

//====================================================================
// COLORS
//====================================================================
bullColor        = input.color(color.new(color.lime, 0), "Bullish Candle State", group=GROUP_COLORS)
bearColor        = input.color(color.new(color.red, 0), "Bearish Candle State", group=GROUP_COLORS)
neutralColor     = input.color(color.new(color.gray, 0), "Neutral Candle State", group=GROUP_COLORS)
bullBgColor      = input.color(color.new(color.lime, 88), "Bullish Background", group=GROUP_COLORS)
bearBgColor      = input.color(color.new(color.red, 88), "Bearish Background", group=GROUP_COLORS)
vwapColor        = input.color(color.new(color.blue, 0), "VWAP Line", group=GROUP_COLORS)
emaColor         = input.color(color.new(color.yellow, 0), "15m EMA(21)", group=GROUP_COLORS)
htfCloseColor    = input.color(color.new(color.orange, 35), "15m Close Overlay", group=GROUP_COLORS)
atrzBullColor    = input.color(color.new(color.aqua, 0), "15m ATR ZLEMA Bullish", group=GROUP_COLORS)
atrzBearColor    = input.color(color.new(color.fuchsia, 0), "15m ATR ZLEMA Bearish", group=GROUP_COLORS)

buySignalColor         = input.color(color.new(color.lime, 0), "Buy Symbol", group=GROUP_COLORS)
sellSignalColor        = input.color(color.new(color.red, 0), "Sell Symbol", group=GROUP_COLORS)
stopBuySignalColor     = input.color(color.new(color.lime, 0), "StopBuy Symbol", group=GROUP_COLORS)
stopSellSignalColor    = input.color(color.new(color.red, 0), "StopSell Symbol", group=GROUP_COLORS)
potentialBullRevColor  = input.color(color.new(color.teal, 0), "Potential Bull Reversal Symbol", group=GROUP_COLORS)
potentialBearRevColor  = input.color(color.new(color.orange, 0), "Potential Bear Reversal Symbol", group=GROUP_COLORS)
confirmBullRevColor    = input.color(color.new(color.aqua, 0), "Confirmed Bull Reversal Symbol", group=GROUP_COLORS)
confirmBearRevColor    = input.color(color.new(color.fuchsia, 0), "Confirmed Bear Reversal Symbol", group=GROUP_COLORS)

//====================================================================
// VALIDATION
//====================================================================
int chartSeconds = timeframe.in_seconds()
int htfSeconds   = timeframe.in_seconds(htfTf)

if htfSeconds <= chartSeconds
    runtime.error("Higher Timeframe must be strictly higher than the chart timeframe.")

//====================================================================
// LOCAL 3m CALCULATIONS
//====================================================================
fastMA      = ta.ema(close, fastLength)
slowMA      = ta.ema(close, slowLength)
macdLine    = fastMA - slowMA
signalLine  = ta.ema(macdLine, signalLength)
hist        = macdLine - signalLine

rsiVal      = ta.rsi(close, rsiLength)
sessionVWAP = ta.vwap(hlc3)

float vwapBuffer = ta.atr(vwapNeutralAtrLen) * vwapNeutralAtrFrac

ltfBullMomentum = hist > 0 and rsiVal > 50
ltfBearMomentum = hist < 0 and rsiVal < 50

aboveVWAP = close > sessionVWAP + vwapBuffer
belowVWAP = close < sessionVWAP - vwapBuffer
nearVWAP  = not aboveVWAP and not belowVWAP

//====================================================================
// HELPER FUNCTIONS
//====================================================================
f_macd_hist(int _fast, int _slow, int _sig) =>
    float _fastMA = ta.ema(close, _fast)
    float _slowMA = ta.ema(close, _slow)
    float _macd   = _fastMA - _slowMA
    float _signal = ta.ema(_macd, _sig)
    _macd - _signal

f_diplus(int _len, int _smooth) =>
    [_p, _m, _a] = ta.dmi(_len, _smooth)
    _p

f_diminus(int _len, int _smooth) =>
    [_p, _m, _a] = ta.dmi(_len, _smooth)
    _m

f_adx(int _len, int _smooth) =>
    [_p, _m, _a] = ta.dmi(_len, _smooth)
    _a

f_pick_source(string _choice) =>
    switch _choice
        "Open"  => open
        "High"  => high
        "Low"   => low
        "HL2"   => hl2
        "HLC3"  => hlc3
        "OHLC4" => ohlc4
        => close

f_atr_zlema_state(float _src, int _zlen, int _alen, float _mult, bool _useNoise, float _noise) =>
    float _atr = ta.rma(ta.tr(true), _alen)
    int _lag   = int(math.floor((_zlen - 1) / 2.0))
    float _safeSrc  = nz(_src, close)
    float _rawZlema = ta.ema(_safeSrc + (_safeSrc - _safeSrc[_lag]), _zlen)

    var float _zlema = na
    if na(_zlema)
        _zlema := _rawZlema
    else if _useNoise
        float _noiseThreshold = _atr * _noise
        float _priceChange    = math.abs(_rawZlema - _zlema)
        if _priceChange > _noiseThreshold
            _zlema := _rawZlema
    else
        _zlema := _rawZlema

    float _atrBand = _atr * _mult

    var float _trail = na
    var int   _trend = 1

    if na(_trail)
        _trail := _zlema - _atrBand

    if _trend == 1
        if _zlema < _trail
            _trend := -1
            _trail := _zlema + _atrBand
        else
            _trail := math.max(_trail, _zlema - _atrBand)
    else
        if _zlema > _trail
            _trend := 1
            _trail := _zlema - _atrBand
        else
            _trail := math.min(_trail, _zlema + _atrBand)

    bool _bullReversal = _trend == 1 and _trend[1] == -1
    bool _bearReversal = _trend == -1 and _trend[1] == 1
    [_trend, _bullReversal, _bearReversal, _trail, _zlema]

f_atr_zlema_state_shifted(float _src, int _zlen, int _alen, float _mult, bool _useNoise, float _noise) =>
    [_trend0, _bull0, _bear0, _trail0, _zlema0] = f_atr_zlema_state(_src, _zlen, _alen, _mult, _useNoise, _noise)
    [_trend0[1], _bull0[1], _bear0[1], _trail0[1], _zlema0[1]]

//====================================================================
// HIGHER TIMEFRAME (15m) DATA REQUESTS - LAST CONFIRMED 15m BAR ONLY
//====================================================================
[htfOpen, htfHigh, htfLow, htfClose, htfEMA21, htfEMA21Prev, htfRSI, htfHist, htfDIPlus, htfDIMin, htfADX] = request.security(
    syminfo.tickerid,
    htfTf,
    [open[1], high[1], low[1], close[1],
     ta.ema(close, emaLen)[1], ta.ema(close, emaLen)[2],
     ta.rsi(close, rsiLength)[1],
     f_macd_hist(fastLength, slowLength, signalLength)[1],
     f_diplus(adxLen, adxSmooth)[1],
     f_diminus(adxLen, adxSmooth)[1],
     f_adx(adxLen, adxSmooth)[1]],
    gaps=barmerge.gaps_off,
    lookahead=barmerge.lookahead_on)

[htfATRZTrend, htfBullReversalRaw, htfBearReversalRaw, htfATRZTrail, htfATRZZlema] = request.security(
    syminfo.tickerid,
    htfTf,
    f_atr_zlema_state_shifted(f_pick_source(atrzSourceChoice), zlemaLength, atrLength, atrMultiplier, enableNoiseFilter, noiseFilter),
    gaps=barmerge.gaps_off,
    lookahead=barmerge.lookahead_on)

//====================================================================
// HIGHER TIMEFRAME STATES
//====================================================================
htfBullTrend    = htfClose > htfEMA21 and htfEMA21 > htfEMA21Prev
htfBearTrend    = htfClose < htfEMA21 and htfEMA21 < htfEMA21Prev

htfBullStrength = htfDIPlus > htfDIMin and htfADX > adxThreshold
htfBearStrength = htfDIMin > htfDIPlus and htfADX > adxThreshold

htfBullMomentum = htfHist > 0 and htfRSI > 50
htfBearMomentum = htfHist < 0 and htfRSI < 50

htfATRZBull     = not na(htfATRZTrend) and htfATRZTrend == 1
htfATRZBear     = not na(htfATRZTrend) and htfATRZTrend == -1

//====================================================================
// FINAL REGIME / ALIGNMENT STATES
//====================================================================
bullAlignmentBase = ltfBullMomentum and aboveVWAP and htfBullTrend and htfBullStrength
bearAlignmentBase = ltfBearMomentum and belowVWAP and htfBearTrend and htfBearStrength

bullAlignment = useHTFMomentumConfirm ? (bullAlignmentBase and htfBullMomentum) : bullAlignmentBase
bearAlignment = useHTFMomentumConfirm ? (bearAlignmentBase and htfBearMomentum) : bearAlignmentBase
neutralState  = not bullAlignment and not bearAlignment

//====================================================================
// STATE TRANSITION EVENTS
//====================================================================
bullRegimeBegins    = bullAlignment and not bullAlignment[1]
bearRegimeBegins    = bearAlignment and not bearAlignment[1]
neutralRegimeBegins = neutralState and not neutralState[1]

newHTFBar = timeframe.change(htfTf)
htfBullReversalBegins = newHTFBar and htfBullReversalRaw
htfBearReversalBegins = newHTFBar and htfBearReversalRaw

//====================================================================
// MARKER EVENTS (confirmed bars only)
//====================================================================
barClosed = barstate.isconfirmed

buySignal  = barClosed and bullRegimeBegins
sellSignal = barClosed and bearRegimeBegins

stopSellSignal = barClosed and bullAlignment[1] and not bullAlignment and not bearRegimeBegins
stopBuySignal  = barClosed and bearAlignment[1] and not bearAlignment and not bullRegimeBegins

confirmBullReversalSignal = barClosed and htfBullReversalBegins
confirmBearReversalSignal = barClosed and htfBearReversalBegins

potentialBullReversalSignal = barClosed and bearAlignment and htfATRZBull and not (bearAlignment[1] and htfATRZBull[1])
potentialBearReversalSignal = barClosed and bullAlignment and htfATRZBear and not (bullAlignment[1] and htfATRZBear[1])

//====================================================================
// SCORE METER CALCULATION
//====================================================================
scoreLTFMomentum = ltfBullMomentum ? 1 : ltfBearMomentum ? -1 : 0
scoreVWAP        = aboveVWAP ? 1 : belowVWAP ? -1 : 0
scoreHTFTrend    = htfBullTrend ? 1 : htfBearTrend ? -1 : 0
scoreHTFStrength = htfBullStrength ? 1 : htfBearStrength ? -1 : 0
scoreHTFMomentum = htfBullMomentum ? 1 : htfBearMomentum ? -1 : 0
scoreHTFATRZ     = htfATRZBull ? 1 : htfATRZBear ? -1 : 0

scoreMax = useHTFMomentumConfirm ? 6 : 5

alignmentScore =
     scoreLTFMomentum +
     scoreVWAP +
     scoreHTFTrend +
     scoreHTFStrength +
     (useHTFMomentumConfirm ? scoreHTFMomentum : 0) +
     scoreHTFATRZ

f_score_state(int _score) =>
    _score >= 4 ? "Strong Bullish" :
     _score > 0 ? "Bullish" :
     _score <= -4 ? "Strong Bearish" :
     _score < 0 ? "Bearish" :
     "Neutral"

f_score_color(int _score) =>
    _score >= 4 ? color.new(color.lime, 0) :
     _score > 0 ? color.new(color.green, 15) :
     _score <= -4 ? color.new(color.red, 0) :
     _score < 0 ? color.new(color.maroon, 10) :
     color.new(color.gray, 20)

f_score_meter(int _score, int _max) =>
    int _absScore = math.min(math.abs(_score), _max)
    string _filled = _score > 0 ? str.repeat("▲", _absScore) :
                     _score < 0 ? str.repeat("▼", _absScore) :
                     ""
    string _empty = str.repeat("·", _max - _absScore)
    _filled + _empty

f_score_text(int _score, int _max) =>
    (_score > 0 ? "+" : "") + str.tostring(_score) + " / " + str.tostring(_max)

//====================================================================
// VISUAL OUTPUT
//====================================================================
plot(showVWAP ? sessionVWAP : na, title="Session VWAP", color=vwapColor, linewidth=2)
plot(showHTFEMA ? htfEMA21 : na, title="15m EMA(21)", color=emaColor, linewidth=2, style=plot.style_stepline)
plot(showHTFClose ? htfClose : na, title="15m Close Overlay", color=htfCloseColor, linewidth=1, style=plot.style_stepline)

atrzPlotColor = htfATRZBull ? atrzBullColor : htfATRZBear ? atrzBearColor : color.new(color.gray, 30)
plot(showHTFATRZTrail ? htfATRZTrail : na, title="15m ATR ZLEMA Trail", color=atrzPlotColor, linewidth=2, style=plot.style_stepline)
plot(showHTFATRZZLEMA ? htfATRZZlema : na, title="15m ATR ZLEMA", color=color.new(atrzPlotColor, 20), linewidth=1, style=plot.style_stepline)

barStateColor = bullAlignment ? bullColor : bearAlignment ? bearColor : neutralColor
barcolor(showBarColors ? barStateColor : na)
bgcolor(showBgZones ? (bullAlignment ? bullBgColor : bearAlignment ? bearBgColor : na) : na)

//====================================================================
// MARKERS
//====================================================================
if showTradeSignals and buySignal
    label.new(bar_index, na, "", yloc=yloc.belowbar, style=label.style_label_up, color=buySignalColor, textcolor=buySignalColor, size=tradeSignalSize)

if showTradeSignals and sellSignal
    label.new(bar_index, na, "", yloc=yloc.abovebar, style=label.style_label_down, color=sellSignalColor, textcolor=sellSignalColor, size=tradeSignalSize)

if showExitSignals and stopBuySignal
    label.new(bar_index, na, "", yloc=yloc.belowbar, style=label.style_xcross, color=stopBuySignalColor, textcolor=stopBuySignalColor, size=exitSignalSize)

if showExitSignals and stopSellSignal
    label.new(bar_index, na, "", yloc=yloc.abovebar, style=label.style_xcross, color=stopSellSignalColor, textcolor=stopSellSignalColor, size=exitSignalSize)

if showHTFReversalMarks and confirmBullReversalSignal
    label.new(bar_index, na, "", yloc=yloc.belowbar, style=label.style_diamond, color=confirmBullRevColor, textcolor=confirmBullRevColor, size=confirmedReversalSize)

if showHTFReversalMarks and confirmBearReversalSignal
    label.new(bar_index, na, "", yloc=yloc.abovebar, style=label.style_diamond, color=confirmBearRevColor, textcolor=confirmBearRevColor, size=confirmedReversalSize)

if showPotentialReversalSymbols and potentialBullReversalSignal
    label.new(bar_index, na, "", yloc=yloc.belowbar, style=label.style_circle, color=potentialBullRevColor, textcolor=potentialBullRevColor, size=potentialReversalSize)

if showPotentialReversalSymbols and potentialBearReversalSignal
    label.new(bar_index, na, "", yloc=yloc.abovebar, style=label.style_circle, color=potentialBearRevColor, textcolor=potentialBearRevColor, size=potentialReversalSize)

//====================================================================
// OPTIONAL 15m CANDLE BOX OVERLAY
//====================================================================
var array<box> htfBoxes = array.new<box>()

if showHTFBoxes and newHTFBar and not na(time(htfTf)[1]) and not na(htfOpen)
    color prevBoxColor  = htfClose >= htfOpen ? color.new(color.lime, htfBoxOpacity) : color.new(color.red, htfBoxOpacity)
    color prevBorderCol = htfClose >= htfOpen ? color.new(color.lime, 70) : color.new(color.red, 70)

    box b = box.new(
         left=time(htfTf)[1],
         top=htfHigh,
         right=time(htfTf),
         bottom=htfLow,
         xloc=xloc.bar_time,
         bgcolor=prevBoxColor,
         border_color=prevBorderCol)

    array.push(htfBoxes, b)

    if array.size(htfBoxes) > maxBoxCount
        box oldBox = array.shift(htfBoxes)
        box.delete(oldBox)

//====================================================================
// COMPACT DASHBOARD
//====================================================================
var table dash = table.new(position.top_right, 2, 6, border_width=1)

f_state_short(bool _bull, bool _bear) =>
    _bull ? "Bull" : _bear ? "Bear" : "Neut"

f_bias_text() =>
    bullAlignment ? "Bullish" : bearAlignment ? "Bearish" : "Neutral"

f_bias_bg() =>
    bullAlignment ? bullColor : bearAlignment ? bearColor : color.new(color.gray, 20)

f_vwap_short() =>
    aboveVWAP ? "Above" : belowVWAP ? "Below" : "Near"

if barstate.islast and showDashboard
    // Row 0 — overall alignment
    table.cell(dash, 0, 0, "Alignment",    text_color=color.white, bgcolor=color.new(color.gray, 50), text_size=size.small)
    table.cell(dash, 1, 0, f_bias_text(),  text_color=color.white, bgcolor=f_bias_bg(),               text_size=size.small)

    // Row 1 — 3m (LTF) momentum
    table.cell(dash, 0, 1, "3m Momentum",  text_color=color.white, bgcolor=color.new(color.gray, 50), text_size=size.small)
    table.cell(dash, 1, 1, f_state_short(ltfBullMomentum, ltfBearMomentum),
         text_color=color.white,
         bgcolor=ltfBullMomentum ? color.new(bullColor, 20) : ltfBearMomentum ? color.new(bearColor, 20) : color.new(color.gray, 20),
         text_size=size.small)

    // Row 2 — VWAP position
    table.cell(dash, 0, 2, "VWAP",         text_color=color.white, bgcolor=color.new(color.gray, 50), text_size=size.small)
    table.cell(dash, 1, 2, f_vwap_short(),
         text_color=color.white,
         bgcolor=aboveVWAP ? color.new(bullColor, 20) : belowVWAP ? color.new(bearColor, 20) : color.new(color.gray, 20),
         text_size=size.small)

    // Row 3 — 15m trend
    table.cell(dash, 0, 3, "15m Trend",    text_color=color.white, bgcolor=color.new(color.gray, 50), text_size=size.small)
    table.cell(dash, 1, 3, f_state_short(htfBullTrend, htfBearTrend),
         text_color=color.white,
         bgcolor=htfBullTrend ? color.new(bullColor, 20) : htfBearTrend ? color.new(bearColor, 20) : color.new(color.gray, 20),
         text_size=size.small)

    // Row 4 — 15m ADX/DI strength
    table.cell(dash, 0, 4, "15m Strength", text_color=color.white, bgcolor=color.new(color.gray, 50), text_size=size.small)
    table.cell(dash, 1, 4, f_state_short(htfBullStrength, htfBearStrength),
         text_color=color.white,
         bgcolor=htfBullStrength ? color.new(bullColor, 20) : htfBearStrength ? color.new(bearColor, 20) : color.new(color.gray, 20),
         text_size=size.small)

    // Row 5 — composite score (meter | numeric)
    table.cell(dash, 0, 5, f_score_meter(alignmentScore, scoreMax),
         text_color=f_score_color(alignmentScore), bgcolor=color.new(color.gray, 50), text_size=size.small)
    table.cell(dash, 1, 5, f_score_text(alignmentScore, scoreMax),
         text_color=f_score_color(alignmentScore), bgcolor=color.new(color.gray, 50), text_size=size.small)
```
