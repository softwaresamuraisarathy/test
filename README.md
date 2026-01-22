import pandas as pd
import numpy as np
import os
import glob

# --- Configuration ---
DATA_DIR = r"C:\gemini\survivor2\data"
INITIAL_BALANCE = 1000.0
SPREAD = 0.50
MAX_LEVERAGE = 100.0
CONTRACT_SIZE = 100 

# --- Top Configurations from Grid Search ---
CONFIGS = [
    {
        'NAME': "Top1_TP2.0",
        'RISK_PCT': 0.15,
        'NR_FACTOR': 0.7,
        'TP_RATIO': 2.0,
        'ADX_THRESH': 20.0,
        'TIMEFRAME': '4h'
    },
    {
        'NAME': "Top2_TP3.0",
        'RISK_PCT': 0.15,
        'NR_FACTOR': 0.7,
        'TP_RATIO': 3.0,
        'ADX_THRESH': 20.0,
        'TIMEFRAME': '4h'
    }
]

class ApexEngine:
    def __init__(self, initial_balance, params):
        self.balance = initial_balance
        self.equity = initial_balance
        self.params = params
        self.trades = []
        self.active_trades = [] 
        self.trade_counter = 0
        self.ruined = False
        self.peak_equity = initial_balance

    def calculate_indicators(self, df):
        # ATR
        df['H-L'] = df['High'] - df['Low']
        df['H-C'] = abs(df['High'] - df['Close'].shift(1))
        df['L-C'] = abs(df['Low'] - df['Close'].shift(1))
        df['TR'] = df[['H-L', 'H-C', 'L-C']].max(axis=1)
        df['ATR'] = df['TR'].rolling(window=14).mean()
        
        # ADX
        df['UpMove'] = df['High'] - df['High'].shift(1)
        df['DownMove'] = df['Low'].shift(1) - df['Low']
        df['PDM'] = np.where((df['UpMove'] > df['DownMove']) & (df['UpMove'] > 0), df['UpMove'], 0)
        df['NDM'] = np.where((df['DownMove'] > df['UpMove']) & (df['DownMove'] > 0), df['DownMove'], 0)
        df['TR_s'] = df['TR'].rolling(window=14).sum()
        df['PDM_s'] = df['PDM'].rolling(window=14).sum()
        df['NDM_s'] = df['NDM'].rolling(window=14).sum()
        df['PDI'] = 100 * (df['PDM_s'] / df['TR_s'])
        df['NDI'] = 100 * (df['NDM_s'] / df['TR_s'])
        df['DX'] = 100 * abs(df['PDI'] - df['NDI']) / (df['PDI'] + df['NDI'])
        df['ADX'] = df['DX'].rolling(window=14).mean()
        
        return df

    def run(self, df):
        # Resample
        df_h4 = df.resample(self.params['TIMEFRAME']).agg({
            'Open': 'first', 'High': 'max', 'Low': 'min', 'Close': 'last'
        }).dropna()
        
        df_h4 = self.calculate_indicators(df_h4)
        
        times = df_h4.index
        opens = df_h4['Open'].values
        highs = df_h4['High'].values
        lows = df_h4['Low'].values
        closes = df_h4['Close'].values
        atrs = df_h4['ATR'].values
        adxs = df_h4['ADX'].values
        
        start_idx = 20
        
        for i in range(start_idx, len(df_h4)):
            if self.ruined: break
            
            timestamp = times[i]
            op, hi, lo, cl = opens[i], highs[i], lows[i], closes[i]
            
            # --- MANAGEMENT ---
            current_equity_calc = self.balance
            
            for t_idx in range(len(self.active_trades) - 1, -1, -1):
                trade = self.active_trades[t_idx]
                entry = trade['entry_price']
                lot = trade['lot']
                sl = trade['sl']
                tp = trade['tp']
                direction = trade['type']
                
                closed = False
                pnl = 0.0
                
                if direction == 'LONG':
                    if lo <= sl:
                        pnl = (sl - entry) * lot * CONTRACT_SIZE
                        self.close_trade(trade, sl, timestamp, pnl, "SL")
                        closed = True
                    elif hi >= tp:
                        pnl = (tp - entry) * lot * CONTRACT_SIZE
                        self.close_trade(trade, tp, timestamp, pnl, "TP")
                        closed = True
                    else:
                        current_equity_calc += (cl - entry) * lot * CONTRACT_SIZE
                        
                else: # SHORT
                    if (hi + SPREAD) >= sl:
                        pnl = (entry - sl) * lot * CONTRACT_SIZE
                        self.close_trade(trade, sl, timestamp, pnl, "SL")
                        closed = True
                    elif (lo + SPREAD) <= tp:
                        pnl = (entry - tp) * lot * CONTRACT_SIZE
                        self.close_trade(trade, tp, timestamp, pnl, "TP")
                        closed = True
                    else:
                        current_equity_calc += (entry - (cl + SPREAD)) * lot * CONTRACT_SIZE
                
                if closed:
                    self.active_trades.pop(t_idx)

            self.equity = current_equity_calc
            if self.equity > self.peak_equity: self.peak_equity = self.equity
            dd = (self.peak_equity - self.equity) / self.peak_equity
            
            if self.equity < (INITIAL_BALANCE * 0.30):
                self.ruined = True
                self.close_all(timestamp, cl, "RUIN")
                break
                
            # --- ENTRIES ---
            prev_atr = atrs[i-1]
            prev_adx = adxs[i-1]
            prev_range = highs[i-1] - lows[i-1]
            
            if np.isnan(prev_atr) or np.isnan(prev_adx): continue
            
            if prev_adx < self.params['ADX_THRESH']: continue
            if prev_range >= (prev_atr * self.params['NR_FACTOR']): continue
            
            current_risk = self.params['RISK_PCT']
            if dd > 0.10: current_risk /= 2.0
            if dd > 0.20: current_risk /= 2.0
            
            trig_high = highs[i-1] + SPREAD
            trig_low = lows[i-1]
            
            if len(self.active_trades) == 0:
                # LONG
                if hi >= trig_high:
                    stop_p = lows[i-1]
                    dist = trig_high - stop_p
                    if dist > 0.5:
                        target = trig_high + (dist * self.params['TP_RATIO'])
                        risk_amt = self.equity * current_risk
                        lot = round(risk_amt / (dist * CONTRACT_SIZE), 2)
                        
                        if lot >= 0.01:
                             if (trig_high * lot * CONTRACT_SIZE) / MAX_LEVERAGE < self.equity:
                                self.open_trade('LONG', trig_high, lot, stop_p, target, timestamp)
                                continue 
                
                # SHORT
                if lo <= trig_low:
                    stop_p = highs[i-1]
                    dist = stop_p - trig_low
                    if dist > 0.5:
                        target = trig_low - (dist * self.params['TP_RATIO'])
                        risk_amt = self.equity * current_risk
                        lot = round(risk_amt / (dist * CONTRACT_SIZE), 2)
                        if lot >= 0.01:
                             if (trig_low * lot * CONTRACT_SIZE) / MAX_LEVERAGE < self.equity:
                                self.open_trade('SHORT', trig_low, lot, stop_p, target, timestamp)

        return self.equity

    def open_trade(self, type, price, lot, sl, tp, time):
        self.trade_counter += 1
        t = {
            'ticket': self.trade_counter,
            'type': type,
            'open_time': time,
            'entry_price': price,
            'lot': lot,
            'sl': sl,
            'tp': tp,
            'status': 'OPEN',
            'profit': 0.0
        }
        self.active_trades.append(t)
        
    def close_trade(self, trade, price, time, pnl, reason):
        trade['status'] = 'CLOSED'
        trade['close_time'] = time
        trade['close_price'] = price
        trade['profit'] = pnl
        trade['exit_reason'] = reason
        self.balance += pnl
        self.trades.append(trade)
        
    def close_all(self, time, price, reason):
        for trade in self.active_trades:
            if trade['type'] == 'LONG':
                pnl = (price - trade['entry_price']) * trade['lot'] * CONTRACT_SIZE
                exit_p = price
            else:
                pnl = (trade['entry_price'] - (price + SPREAD)) * trade['lot'] * CONTRACT_SIZE
                exit_p = price + SPREAD
            
            self.close_trade(trade, exit_p, time, pnl, reason)
        self.active_trades = []

def process_file(file_path, config):
    year = os.path.basename(file_path).split('_')[-1].replace('.csv', '')
    
    try:
        df = pd.read_csv(file_path, names=['Date', 'Time', 'Open', 'High', 'Low', 'Close', 'Vol'])
        df['Datetime'] = pd.to_datetime(df['Date'] + ' ' + df['Time'])
        df.set_index('Datetime', inplace=True)
        df.drop(['Date', 'Time', 'Vol'], axis=1, inplace=True)
        
        engine = ApexEngine(INITIAL_BALANCE, config)
        final_equity = engine.run(df)
        
        if engine.trades:
            ledger_df = pd.DataFrame(engine.trades)
            ledger_df['Balance'] = INITIAL_BALANCE + ledger_df['profit'].cumsum()
            ledger_df.to_csv(f"{config['NAME']}_{year}.csv", index=False)
            
        ret_pct = ((final_equity - INITIAL_BALANCE) / INITIAL_BALANCE) * 100
        
        return {
            'Config': config['NAME'],
            'Year': year,
            'Return_Pct': ret_pct,
            'Trades': len(engine.trades),
            'Final_Balance': final_equity,
            'Ruined': engine.ruined
        }
        
    except Exception as e:
        print(f"Error {year}: {e}")
        return None

def main():
    files = glob.glob(os.path.join(DATA_DIR, "DAT_MT_XAUUSD_M1_20*.csv"))
    # files = [f for f in files if any(y in f for y in ['2019','2020','2021','2022','2023','2024','2025'])]
    
    all_results = []
    
    print(f"Running Comparison for {len(CONFIGS)} Configs over {len(files)} Years...")
    
    for cfg in CONFIGS:
        print(f"\n--- Running {cfg['NAME']} ---")
        for f in files:
            res = process_file(f, cfg)
            if res:
                all_results.append(res)
                print(f"{res['Year']}: {res['Return_Pct']:.2f}%")

    if all_results:
        df_res = pd.DataFrame(all_results)
        df_res.to_csv("Comparison_Results.csv", index=False)
        
        print("\n=== FINAL COMPARISON ===")
        pivot = df_res.pivot(index='Year', columns='Config', values='Return_Pct')
        print(pivot.to_string())
        
        # Stats
        print("\n--- Average Return ---")
        print(df_res.groupby('Config')['Return_Pct'].mean())
        
        print("\n--- Min Return ---")
        print(df_res.groupby('Config')['Return_Pct'].min())

if __name__ == "__main__":
    main()
