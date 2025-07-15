import telebot
import requests
import time
import numpy as np
from tradingview_ta import TA_Handler, Interval
import talib

# Configurare
TOKEN = "TOKENUL_TAU_TELEGRAM"
CHAT_ID = 7048836653
BING_API_KEY = "CHEIA_TA_BING_NEWS"

bot = telebot.TeleBot(TOKEN)

def get_tradingview_price(interval=Interval.INTERVAL_1_MINUTE):
    try:
        handler = TA_Handler(
            symbol="XAUUSD",
            exchange="FOREXCOM",
            screener="forex",
            interval=interval
        )
        analysis = handler.get_analysis()
        return float(analysis.indicators['close'])
    except Exception as e:
        print(f"Eroare TradingView ({interval}): {e}")
        return None

def get_metalslive_price():
    try:
        r = requests.get("https://api.metals.live/v1/spot", timeout=5)
        data = r.json()
        for item in data:
            if "gold" in item:
                return float(item['gold'])
    except Exception as e:
        print(f"Eroare metals.live: {e}")
    return None

def get_news_signal():
    try:
        url = "https://api.bing.microsoft.com/v7.0/news/search"
        params = {"q": "gold OR XAUUSD", "count": 5, "mkt": "en-US"}
        headers = {"Ocp-Apim-Subscription-Key": BING_API_KEY}
        r = requests.get(url, headers=headers, params=params, timeout=5)
        data = r.json()
        news_titles = [item['name'] for item in data.get('value', [])]
        bullish_words = ["rise", "rally", "soar", "up", "increase", "safe haven", "record high", "surge"]
        bearish_words = ["fall", "drop", "down", "decline", "sell-off", "plunge", "low"]
        bull = any(any(word in t.lower() for word in bullish_words) for t in news_titles)
        bear = any(any(word in t.lower() for word in bearish_words) for t in news_titles)
        if bull and not bear:
            return "BUY"
        elif bear and not bull:
            return "SELL"
        else:
            return None
    except Exception as e:
        print(f"Eroare Bing News: {e}")
        return None

def scalping_signal(prices):
    if len(prices) < 20:
        return None
    closes = np.array(prices)
    rsi = talib.RSI(closes, timeperiod=7)[-1]
    fast_ema = talib.EMA(closes, timeperiod=5)[-1]
    slow_ema = talib.EMA(closes, timeperiod=13)[-1]
    macd, macdsignal, _ = talib.MACD(closes, fastperiod=6, slowperiod=13, signalperiod=5)
    if rsi < 30 and fast_ema > slow_ema and macd[-1] > macdsignal[-1]:
        return "BUY"
    elif rsi > 70 and fast_ema < slow_ema and macd[-1] < macdsignal[-1]:
        return "SELL"
    else:
        return None

def format_signal_message(direction, entry, tp1, tp2, sl, profit):
    return (
        f"📊 Semnal – {direction}\n"
        f"🔴 Intrare: ~{entry:.2f}\n"
        f"🎯 TP1: ~{tp1:.2f}\n"
        f"🎯 TP2: ~{tp2:.2f}\n"
        f"🛑 SL: ~{sl:.2f}\n"
        f"✅ Profit estimat: {profit}\n"
    )

def calculate_levels(entry, direction, scalping=True):
    pip = 0.01  # 1 pip XAUUSD
    # Pentru scalping TP/SL mai aproape!
    if scalping:
        tp1_pips, tp2_pips, sl_pips = 15, 25, 12
        profit = "+15 – 25 pips"
    else:
        tp1_pips, tp2_pips, sl_pips = 38, 65, 47
        profit = "+70 – 100 pips"
    if direction == "BUY":
        tp1 = entry + tp1_pips * pip
        tp2 = entry + tp2_pips * pip
        sl = entry - sl_pips * pip
    else:
        tp1 = entry - tp1_pips * pip
        tp2 = entry - tp2_pips * pip
        sl = entry + sl_pips * pip
    return tp1, tp2, sl, profit

def main_loop():
    tv_prices_1m = []
    tv_prices_5m = []
    last_signal = None

    while True:
        entry = get_tradingview_price(Interval.INTERVAL_1_MINUTE)
        entry5 = get_tradingview_price(Interval.INTERVAL_5_MINUTES)
        metals_entry = get_metalslive_price()
        news = get_news_signal()
        # Folosim tradingview ca preț de bază
        if entry:
            tv_prices_1m.append(entry)
            if len(tv_prices_1m) > 100:
                tv_prices_1m = tv_prices_1m[-100:]
        if entry5:
            tv_prices_5m.append(entry5)
            if len(tv_prices_5m) > 100:
                tv_prices_5m = tv_prices_5m[-100:]

        # Scalping signal
        direction = scalping_signal(tv_prices_1m)
        direction5 = scalping_signal(tv_prices_5m)
        # Majoritatea de semnale sau consens
        dir_list = [direction, direction5, news]
        buy_count = dir_list.count("BUY")
        sell_count = dir_list.count("SELL")

        # Folosim prețul de la tradingview, dacă nu există, metals.live
        chosen_entry = entry if entry else metals_entry
        if not chosen_entry:
            time.sleep(60)
            continue

        if buy_count >= 2:
            msg_dir = "BUY"
        elif sell_count >= 2:
            msg_dir = "SELL"
        else:
            msg_dir = None

        if msg_dir and msg_dir != last_signal:
            tp1, tp2, sl, profit = calculate_levels(chosen_entry, msg_dir, scalping=True)
            msg = format_signal_message(msg_dir, chosen_entry, tp1, tp2, sl, profit)
            bot.send_message(CHAT_ID, msg)
            last_signal = msg_dir

        time.sleep(60)  # Scalping: verifică la fiecare minut

if __name__ == "__main__":
    print("Bot scalping XAUUSD pornit.")
    main_loop()
