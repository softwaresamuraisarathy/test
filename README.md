// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © parthaveltrader

//@version=5
indicator("Price cross two sma", overlay = true)

SMA1 = input.int(5, minval=1, title="fast sma")
SMA2 = input.int(20, minval=1, title="slow sma")
showLong = input.bool(defval = true)
showShort = input.bool(defval = true)
fastSma = ta.sma(close, SMA1)
slowSma = ta.sma(close, SMA2)

green = close > open
red = open > close
longCondition = false
shortCondition = false
if (showLong)
    longCondition := green and close >= fastSma and close >= slowSma and fastSma <= slowSma and open <= slowSma
if (showShort)    
    shortCondition := red and close <= fastSma and close <= slowSma and fastSma >= slowSma and open >= slowSma
plotshape(longCondition, style=shape.diamond, color=color.green, location=location.belowbar, size = size.tiny, text = "", textcolor = color.yellow)
plotshape(shortCondition, style=shape.diamond, color=color.red, location=location.abovebar, size = size.tiny, text = "", textcolor = color.yellow)
