  Execution Parameters:
   * Capital: $10,000 initial balance.
   * Position: Min 0.01 lots, Max 100x leverage (verify margin before entry).
   * Limits: Max 10 concurrent open trades.
   * Target: Minimum Take Profit must be a $5 price movement.
   * Cost: Factor in a fixed Bid-Ask spread of $0.5 (CRITICAL).
   * Risk Protocol: On every candle close, calculate Equity. If Equity drops below 30% of initial capital, close all positions and cease trading
     immediately.
   * STRICTLY NO CURVE FITTNG and CHECKING NEXT CANDLES.

  Expectation:
	Every week,  win $1000 and end of the week close all trades and move $1000 to second account if won
	
  Required Outputs:
   1. Summary: Save strategy logic and performance metrics to <strategy_name._<PL%>.md
   2. Ledger: Save full trade history to <strategy_name>.csv (Columns: Ticket, Type, Open/Close Times & Prices, Profit, Lot Size, Status, main account balance, second account balance)."

