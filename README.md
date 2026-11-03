# Testy.py
import time
import logging
import MetaTrader5 as mt5
import pandas as pd
import numpy as np
import requests
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes

# ==========================================
# 1. YOUR CONFIGURATION (UPDATE THESE LINES)
# ==========================================
TELEGRAM_TOKEN = "9167367370:AAEMEJ6HYT800lp5d8YBxuhlICW8xU4h2rk"  # From @BotFather
CHAT_ID = "7267946619"          # From @userinfobot

# Your Broker Demo Credentials
MT5_LOGIN = 6240650                       # Your demo account number
MT5_PASSWORD = "$jYK2oLz"        # Your demo password
MT5_SERVER = "Headway-Demo"                  # e.g., Deriv-Demo, Exness-Trial, MetaQuotes-Demo
SYMBOL = "XAUUSD"
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
    tp = price + ((price - signal['zone_low']) * 2.5) if order_type == mt5.ORDER_TYPE_BUY else price - ((signal['zone_high'] - price) * 2.5)

    request = {
        "action": mt5.TRADE_ACTION_DEAL,
        "symbol": SYMBOL,
        "volume": LOT_SIZE,
        "type": order_type,
        "price": price,
        "sl": sl,
        "tp": tp,
        "deviation": 20,
        "magic": 234000,
        "comment": "ICT BPR Scalper Bot",
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }
    
    result = mt5.order_send(request)
    return result, price, sl, tp

# ==========================================
# 3. INTERACTIVE TELEGRAM COMMAND HANDLERS
# ==========================================
async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("▶️ Resume", callback_data="resume"), InlineKeyboardButton("⏸️ Pause", callback_data="pause")],
        [InlineKeyboardButton("📊 Account Balance", callback_data="balance"), InlineKeyboardButton("⚡ Force BPR Scan", callback_data="scan")]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text("🤖 *ICT BPR & MSNR Mobile Bot Control Center*\nChoose an action below:", reply_markup=reply_markup, parse_mode="Markdown")

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global BOT_PAUSED
    query = update.callback_query
    await query.answer()

    if query.data == "pause":
        BOT_PAUSED = True
        await query.edit_message_text("⏸️ *Bot Paused.* Scanning suspended.")
    elif query.data == "resume":
        BOT_PAUSED = False
        await query.edit_message_text("▶️ *Bot Resumed.* Scanning M1 BPR zones...")
    elif query.data == "balance":
        acc = mt5.account_info()
        bal_text = f"💰 *Demo Balance:* ${acc.balance}\n📈 *Equity:* ${acc.equity}\n🏢 *Server:* `{MT5_SERVER}`" if acc else "❌ MT5 Disconnected."
        await query.message.reply_text(bal_text, parse_mode="Markdown")
    elif query.data == "scan":
        df = fetch_candles(SYMBOL)
        sig = scan_ict_bpr_msnr(df)
        if sig:
            await query.message.reply_text(f"🎯 *BPR Zone Found!*\nType: {sig['type']}\nHigh: {sig['zone_high']}\nLow: {sig['zone_low']}")
        else:
            await query.message.reply_text("🔍 *Scan Complete:* No active BPR overlaps on M1 right now.")

# ==========================================
# 4. MAIN APPLICATION LOOP
# ==========================================
if __name__ == "__main__":
    if initialize_mt5():
        print("Connected to MT5 Demo successfully!")
        app = Application.builder().token(TELEGRAM_TOKEN).build()
        app.add_handler(CommandHandler("start", start_command))
        app.add_handler(CallbackQueryHandler(button_handler))
        app.run_polling()

