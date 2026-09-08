# smc-ict-indicator-
//@version=6
indicator("RSI Div + Momentum + CVD - Short Entry (1H & 4H)", overlay=false, max_labels_count=500, max_lines_count=500)

// ───────────────────────────────
// INPUTS
// ───────────────────────────────
rsiLength     = input.int(14, "RSI Länge", minval=1)
pivotLeft     = input.int(5, "Pivot Left Bars", minval=1)
pivotRight    = input.int(5, "Pivot Right Bars", minval=1)
rsiOverbought = input.int(60, "RSI Mindestwert am 1. Hoch (Filter)", minval=0, maxval=100)

bbLength      = input.int(20, "Squeeze BB/KC Länge", minval=1)
bbMult        = input.float(2.0, "BB Multiplikator")
kcMult        = input.float(1.5, "KC Multiplikator")
momLength     = input.int(20, "Momentum Länge", minval=1)

cvdResetDaily = input.bool(false, "CVD täglich zurücksetzen")
showLabels    = input.bool(true, "Signal-Label anzeigen")
showLines     = input.bool(true, "Divergenz-Linien zeichnen")

tf1 = input.timeframe("60", "Timeframe 1")
tf2 = input.timeframe("240", "Timeframe 2")

// ───────────────────────────────
// FUNKTION: berechnet alle Signale für EINEN Timeframe
// (wird per request.security auf tf1 = 1H und tf2 = 4H ausgewertet)
// ───────────────────────────────
f_signals() =>
    rsiValue = ta.rsi(close, rsiLength)
    priceHighPivot = ta.pivothigh(high, pivotLeft, pivotRight)

    var float lastPivotPrice  = na
    var float lastPivotRSI    = na
    var int   lastPivotBarIdx = na
    var float prevPivotPrice  = na
    var float prevPivotRSI    = na

    if not na(priceHighPivot)
        prevPivotPrice := lastPivotPrice
        prevPivotRSI   := lastPivotRSI
        lastPivotPrice := priceHighPivot
        lastPivotRSI   := rsiValue[pivotRight]
        lastPivotBarIdx := bar_index - pivotRight

    bearishDivergence = not na(priceHighPivot) and not na(prevPivotPrice) and
         lastPivotPrice > prevPivotPrice and lastPivotRSI < prevPivotRSI and
         prevPivotRSI > rsiOverbought

    basis = ta.sma(close, bbLength)
    dev   = bbMult * ta.stdev(close, bbLength)
    upperBB = basis + dev
    lowerBB = basis - dev
    ma      = ta.sma(close, bbLength)
    rangeK  = ta.sma(high - low, bbLength)
    upperKC = ma + rangeK * kcMult
    lowerKC = ma - rangeK * kcMult
    squeezeOn = (lowerBB > lowerKC) and (upperBB < upperKC)

    highestHigh = ta.highest(high, momLength)
    lowestLow   = ta.lowest(low, momLength)
    avgHL       = math.avg(highestHigh, lowestLow)
    avgClose    = math.avg(avgHL, ta.sma(close, momLength))
    momVal      = ta.linreg(close - avgClose, momLength, 0)
    momRising   = momVal > momVal[1]
    momentumTurningDown = momRising[1] and not momRising

    barDelta = close > open ? volume : close < open ? -volume : 0.0
    var float cvd = 0.0
    newSession = cvdResetDaily and ta.change(time("D"))
    cvd := newSession ? barDelta : cvd + barDelta

    var float lastPivotCVD = na
    var float prevPivotCVD = na
    if not na(priceHighPivot)
        prevPivotCVD := lastPivotCVD
        lastPivotCVD := cvd[pivotRight]

    cvdDivergence = not na(priceHighPivot) and not na(prevPivotCVD) and
         lastPivotPrice > prevPivotPrice and lastPivotCVD < prevPivotCVD

    shortConfirmed = bearishDivergence and momentumTurningDown
    shortStrong    = shortConfirmed and cvdDivergence

    [bearishDivergence, momentumTurningDown, cvdDivergence, shortConfirmed, shortStrong, momVal, momRising, cvd, squeezeOn, lastPivotPrice, lastPivotBarIdx, prevPivotPrice]

// ───────────────────────────────
// AUSWERTUNG AUF TF1 (1H) UND TF2 (4H)
// ───────────────────────────────
[div1, mom1, cvdDiv1, conf1, strong1, momVal1, momRising1, cvd1, sq1, pivP1, pivB1, prevP1] = request.security(syminfo.tickerid, tf1, f_signals())
[div2, mom2, cvdDiv2, conf2, strong2, momVal2, momRising2, cvd2, sq2, pivP2, pivB2, prevP2] = request.security(syminfo.tickerid, tf2, f_signals())

// ───────────────────────────────
// PLOTS: Momentum & CVD je Timeframe
// ───────────────────────────────
momColor1 = momRising1 ? color.new(#40E0D0, 20) : color.new(#0B5D5A, 0)
momColor2 = momRising2 ? color.new(#40E0D0, 60) : color.new(#0B5D5A, 40)

plot(momVal1, title="Momentum 1H", style=plot.style_columns, color=momColor1)
plot(momVal2, title="Momentum 4H", style=plot.style_line, color=momColor2, linewidth=2)

plot(0, title="Squeeze Dots 1H", style=plot.style_circles, color=sq1 ? color.gray : color.white, linewidth=3)

cvdColor1 = cvd1 > cvd1[1] ? color.new(color.lime, 0) : color.new(color.red, 0)
plot(cvd1, title="CVD 1H", color=cvdColor1, linewidth=2)
plot(cvd2, title="CVD 4H", color=color.new(color.yellow, 30), linewidth=1)

// ───────────────────────────────
// SIGNALE: nur wenn 1H UND 4H übereinstimmen
// ───────────────────────────────
mtfShortConfirmed = conf1 and conf2
mtfShortStrong    = strong1 and strong2

plotshape(conf1, title="Short 1H", style=shape.triangledown, location=location.top,
     color=color.new(color.red, 40), size=size.tiny)
plotshape(mtfShortConfirmed, title="Short bestätigt 1H+4H", style=shape.triangledown,
     location=location.top, color=color.new(color.red, 0), size=size.small, text="1H+4H")
plotshape(mtfShortStrong, title="Starker Short 1H+4H+CVD", style=shape.triangledown,
     location=location.top, color=color.new(color.orange, 0), size=size.normal, text="CVD")

// ───────────────────────────────
// STATUS-TABELLE oben rechts
// ───────────────────────────────
var table statusTable = table.new(position.top_right, 3, 4, border_width=1)
if barstate.islast
    table.cell(statusTable, 0, 0, "", bgcolor=color.new(color.black, 0))
    table.cell(statusTable, 1, 0, "1H", text_color=color.white, bgcolor=color.new(color.gray, 60))
    table.cell(statusTable, 2, 0, "4H", text_color=color.white, bgcolor=color.new(color.gray, 60))

    table.cell(statusTable, 0, 1, "RSI Div", text_color=color.white)
    table.cell(statusTable, 1, 1, div1 ? "✓" : "-", text_color=div1 ? color.red : color.gray)
    table.cell(statusTable, 2, 1, div2 ? "✓" : "-", text_color=div2 ? color.red : color.gray)

    table.cell(statusTable, 0, 2, "Momentum ↓", text_color=color.white)
    table.cell(statusTable, 1, 2, mom1 ? "✓" : "-", text_color=mom1 ? color.red : color.gray)
    table.cell(statusTable, 2, 2, mom2 ? "✓" : "-", text_color=mom2 ? color.red : color.gray)

    table.cell(statusTable, 0, 3, "CVD Div", text_color=color.white)
    table.cell(statusTable, 1, 3, cvdDiv1 ? "✓" : "-", text_color=cvdDiv1 ? color.red : color.gray)
    table.cell(statusTable, 2, 3, cvdDiv2 ? "✓" : "-", text_color=cvdDiv2 ? color.red : color.gray)

// ───────────────────────────────
// LABELS auf 1H-Divergenzpunkten
// ───────────────────────────────
if div1 and showLabels
    label.new(pivB1, pivP1, "SHORT\nRSI Div", style=label.style_label_down,
         color=color.new(color.red, 0), textcolor=color.white, size=size.small, yloc=yloc.abovebar)

alertcondition(mtfShortConfirmed, title="Short 1H+4H bestätigt", message="RSI Divergenz + Momentum-Dreh auf 1H UND 4H -> Short-Entry")
alertcondition(mtfShortStrong, title="Short 1H+4H+CVD stark", message="Volle Konfluenz auf 1H und 4H inkl. CVD -> starkes Short-Setup")