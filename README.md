# Requirements

## Data

- File: CSV with columns: `date`, `time`, `open`, `high`, `low`, `close`, `volume`.
- Sort ascending by timestamp; skip malformed rows (e.g., non-numeric prices/volumes, invalid dates/times, `high < low`, negative values).
- Parse `date`/`time` into a unified timestamp (e.g., `YYYY-MM-DD HH:MM:SS`).
- Timezone: use the CSV's timezone; assume UTC if unspecified. All outputs and calculations use this timezone consistently.
- Validate data:
  - No duplicate timestamps.
  - Prices within reasonable ranges (strategy/backtest can define thresholds).
  - Gaps/weekends handled as-is (no interpolation).

## Platform & Account

- Platform: MT4; Instrument: `XAUUSD` (Gold vs USD).
- Contract size: `1 lot = 100 oz`; lot step `0.01`; min lot `0.01`; max lots `1000.00`.
- Starting balance: `$1,000`; account leverage: `1:100` (max notional = `equity * 100`).
- Account currency: `USD`.
- Hedging enabled: multiple positions (long/short, same/opposite directions) allowed; each entry is an independent ticket.

## Costs & Execution

- Spread: `0.5` (price terms, USD/oz per side).
- Slippage: `0.25` (price terms, per side).
- Signals on bar close → execution at next bar open.
- No look-ahead bias.
- Entry fills (next bar open):
  - BUY: `fill_price = next_open + 0.5 + 0.25`
  - SELL: `fill_price = next_open - 0.5 - 0.25`
- Exit fills (stop/TP/trailing/liquidation):
  - Closing BUY: `fill_price = next_open - 0.5 - 0.25`
  - Closing SELL: `fill_price = next_open + 0.5 + 0.25`
- Round-trip cost per oz = `1.50` (deducted from realized PnL).
- Full fills only (no partial fills).

## Position Sizing (MT4 Lots)

- Strategies CAN and SHOULD use up to full `100x` leverage to maximize returns:
  - `notional_target = equity * leverage_multiplier * position_pct`
  - `leverage_multiplier <= 100` (typically `100` for aggressive strategies).

- Sizing + pre-entry margin check (at bar close):
  - `target_lots = notional_target / (close_price * 100)`
  - `rounded_lots = round(target_lots / 0.01) * 0.01`
  - `projected_margin = (rounded_lots * 100 * close_price) / 100`
  - If `projected_margin > free_margin` **OR** `rounded_lots < 0.01` → skip trade.
  - `final_lots = min(rounded_lots, 1000.00)`

- After fill → per-ticket required margin:
  - `required_margin_ticket = (final_lots * 100 * fill_price) / 100`
  - This is fixed for the lifetime of the ticket.

- Pyramiding/adds:
  - Treated as new independent tickets.
  - Same sizing and margin rules/checks as above.

## Backtest Engine Rules

### Equity & PnL (Marked-to-Market Each Bar Close)

- `equity = balance + Σ(unrealized_pnl_all_open_tickets)`
- Long unrealized PnL:  
  `unrealized_long = lots * 100 * (close - entry_fill)`
- Short unrealized PnL:  
  `unrealized_short = lots * 100 * (entry_fill - close)`
- On exit:
  - `realized_pnl_gross = lots * 100 * (exit_fill - entry_fill) * direction`
    - `direction = +1` for long, `-1` for short.
  - `costs = round_trip_cost_per_oz * (lots * 100)` as applicable.
  - `realized_pnl_net = realized_pnl_gross - costs`
  - `balance += realized_pnl_net`

### Margin (MT4-Accurate)

- Per-ticket required margin (fixed at entry, using `fill_price`):
  - `required_margin_ticket = (lots * 100 * entry_fill) / 100`
- `total_used_margin = Σ(required_margin_ticket)` over all open tickets.
- `free_margin = equity - total_used_margin`
- `margin_level = (equity / total_used_margin) * 100%`  
  (if no open positions → `margin_level = +∞`).

### Broker Stop-Out (50%)

- At each bar close:
  - If `margin_level <= 50%` → trigger broker stop-out.
  - At the next bar open, close tickets one by one (largest negative `unrealized_pnl` first) at modeled exit prices (with spread + slippage) until:
    - `margin_level > 50%`, or
    - No positions remain.

### Equity Drawdown Kill-Switch (80% Max DD)

- `starting_equity = 1000` (initial equity).
- At each bar close:
  - If `equity <= 0.20 * starting_equity` (i.e., equity ≤ `$200`, an 80% drawdown from start):
    - Halt opening any new trades or adds.
    - Schedule full liquidation of all open positions (largest losers first) at the next bar open with costs (spread + slippage).
    - After full liquidation, halt all trading for the remainder of the backtest.

### Other Engine Rules

- Account blown:
  - If `equity <= 0` at any time (before or after liquidation), mark account as blown and halt all trading.
- Stops/TP/Trailing:
  - Evaluated on bar close (using the bar's `high/low/close` as needed by strategy).
  - Exits execute at next bar open at modeled exit prices with spread + slippage.
- Multiple concurrent trades and pyramiding allowed as per hedging rules.
- Margin checks occur at bar close; any forced liquidations (stop-out or kill-switch) execute at the next bar open.

## Strategy Process

- Goal: create the most profitable and innovative XAUUSD strategy (competition vs Claude/GPT/Grok).
- Strategies must be:
  - Rule-based and fully implementable.
  - Not duplicates of existing strategies in this folder.
- Target: `PL% >= 1000%` (i.e., `final_equity >= $11,000`) **or** document the best achieved strategy if target not reached.

## Outputs per Strategy

- Summary file: `<strategy-name><PL%-integer>.md`
  - Includes:
    - Rationale.
    - Full rules and parameters.
    - Metrics table:

| Metric        | Value | Definition                                                                                 |
|--------------|-------|---------------------------------------------------------------------------------------------|
| PL%          | X%    | `((final_equity - 1000) / 1000) * 100`                                                     |
| Final Equity | $X    | `final_equity`                                                                             |
| Max DD       | X%    | Maximum peak-to-trough equity drop                                                         |
| Sharpe       | X     | `(mean_bar_returns / std_bar_returns) * sqrt(bars_per_year)`, risk-free rate = 0          |
| Trades       | X     | Total number of closed trades                                                              |
| Win Rate     | X%    | `% of winning trades`                                                                      |
| Profit Factor| X     | `gross_profit / abs(gross_loss)`                                                           |
| Avg Exposure | X%    | Average % of bars with a non-zero position                                                 |
| Total Costs  | $X    | Sum of all spread + slippage costs                                                         |

- Trades CSV: `<strategy-name><PL%-integer>.csv`
  - Columns:
    - `time_in`
    - `time_out`
    - `side`
    - `lots`
    - `entry_fill`
    - `exit_fill`
    - `gross_pnl`
    - `costs`
    - `net_pnl`
    - `mfe` (max favorable excursion)
    - `mae` (max adverse excursion)
    - `equity_at_entry`
    - `equity_at_exit`
    - `reason` (rule or event that triggered entry/exit)

## Assumptions & Exclusions

- No swaps/overnight financing, commissions, or partial fills.
- Bar-based backtest (timeframe specified when running, e.g., M5, H1, etc.).
- All core parameters (spread, slippage, leverage, kill-switch threshold, etc.) should be configurable, with defaults as specified above.
- Full event logging recommended (entries, exits, liquidations, margin checks, kill-switch triggers) for debugging and analysis.

