# Testy.py
import time
import logging
import MetaTrader5 as mt5

LOT_SIZE = 0.1

# Bot State Control
BOT_PAUSED = False

logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

# ==========================================
# 2. MT5 & ICT BPR TRADING STRATEGY ENGINE
# ==========================================
def initialize_mt5():
    if not mt5.initialize():
        logging.error("MT5 Initialization failed.")
        return False
    
    authorized = mt5.login(login=MT5_LOGIN, password=MT5_PASSWORD, server=MT5_SERVER)
    if not authorized:
        logging.error(f"MT5 Login failed for account {MT5_LOGIN}")
        return False
    return True

def fetch_candles(symbol, timeframe=mt5.TIMEFRAME_M1, n=200):
    rates = mt5.copy_rates_from_pos(symbol, timeframe, 0, n)
    if rates is None:
        return pd.DataFrame()
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')
    return df

def scan_ict_bpr_msnr(df):
    """Detects ICT Balanced Price Ranges (Overlap of Bullish & Bearish FVGs)"""
    df['Bull_FVG_Top'] = np.where(df['low'] > df['high'].shift(2), df['low'], np.nan)
    df['Bull_FVG_Bot'] = np.where(df['low'] > df['high'].shift(2), df['high'].shift(2), np.nan)
    
    df['Bear_FVG_Top'] = np.where(df['high'] < df['low'].shift(2), df['low'].shift(2), np.nan)
    df['Bear_FVG_Bot'] = np.where(df['high'] < df['low'].shift(2), df['low'], np.nan)

    for i in range(3, len(df)):
        bull_t, bull_b = df['Bull_FVG_Top'].iloc[i], df['Bull_FVG_Bot'].iloc[i]
        bear_t, bear_b = df['Bear_FVG_Top'].iloc[i-1], df['Bear_FVG_Bot'].iloc[i-1]

        if pd.notna(bull_t) and pd.notna(bear_t):
            bpr_top = min(bull_t, bear_t)
            bpr_bot = max(bull_b, bear_b)
            
            if bpr_top > bpr_bot: # Overlap present
                current_price = df['close'].iloc[-1]
                if bpr_bot <= current_price <= bpr_top:
                    return {
                        "type": "BULLISH_BPR" if df['close'].iloc[i] > df['open'].iloc[i] else "BEARISH_BPR",
                        "zone_high": bpr_top,
                        "zone_low": bpr_bot,
                        "price": current_price
                    }
    return None

def execute_trade(signal):
    tick = mt5.symbol_info_tick(SYMBOL)
    if not tick:
        return None
        
    price = tick.ask if signal['type'] == "BULLISH_BPR" else tick.bid
    order_type = mt5.ORDER_TYPE_BUY if signal['type'] == "BULLISH_BPR" else mt5.ORDER_TYPE_SELL
    sl = signal['zone_low'] - 0.0008 if order_type == mt5.ORDER_TYPE_BUY else signal['zone_high'] + 0.0008
    tp = price 

