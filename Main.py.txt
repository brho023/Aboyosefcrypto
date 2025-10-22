# main.py
# MA50 Telegram watcher for MEXC — ready for Render.com
import os
import time
import json
import requests
import pandas as pd
import ccxt
from datetime import datetime, timedelta

# -------- CONFIG ----------
BOT_TOKEN = os.environ.get("BOT_TOKEN")            # ضع التوكن في متغير بيئة على Render
OWNER_CHAT_ID = os.environ.get("OWNER_CHAT_ID")    # اختياري: لتلقي رسالة بدء التشغيل
STATE_FILE = "ma50_bot_state.json"
TIMEFRAME = "5m"
MA_PERIOD = 50
CHECK_INTERVAL = 30           # ثانية بين جولات الفحص (يمكن تغييره)
MONITOR_DURATION_HOURS = 24
# --------------------------

if not BOT_TOKEN:
    print("⚠️ خطأ: متغير البيئة BOT_TOKEN غير مُحمّل. ضع المتغير ثم أعد التشغيل.")
    raise SystemExit(1)

BASE_URL = f"https://api.telegram.org/bot{BOT_TOKEN}"
exchange = ccxt.mexc()

def load_state():
    try:
        with open(STATE_FILE, "r") as f:
            return json.load(f)
    except Exception:
        return {"offset": 0, "monitored": {}}

def save_state(state):
    try:
        with open(STATE_FILE, "w") as f:
            json.dump(state, f)
    except Exception as e:
        print("Save state error:", e)

def send_message(chat_id, text):
    try:
        url = f"{BASE_URL}/sendMessage"
        data = {"chat_id": chat_id, "text": text, "parse_mode": "Markdown"}
        requests.post(url, data=data, timeout=10)
    except Exception as e:
        print("Telegram send error:", e)

def get_updates(offset):
    try:
        url = f"{BASE_URL}/getUpdates"
        params = {"timeout": 30}
        if offset:
            params["offset"] = offset + 1
        r = requests.get(url, params=params, timeout=35)
        res = r.json()
        return res.get("result", [])
    except Exception as e:
        print("Get updates error:", e)
        return []

def normalize_symbol(text):
    s = text.strip().upper().replace(" ", "")
    if "/" in s:
        return s
    if s.endswith("USDT"):
        base = s[:-4]
        return f"{base}/USDT"
    return f"{s}/USDT"

def market_exists(symbol):
    try:
        markets = exchange.load_markets()
        return symbol in markets and markets[symbol].get("spot", False)
    except Exception:
        return False

def fetch_ohlcv_df(symbol, limit):
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe=TIMEFRAME, limit=limit)
    df = pd.DataFrame(ohlcv, columns=["time","open","high","low","close","volume"])
    df["close"] = df["close"].astype(float)
    return df

def handle_updates(state):
    offset = state.get("offset", 0)
    updates = get_updates(offset)
    if not updates:
        return state
    for u in updates:
        state["offset"] = u.get("update_id", state.get("offset", 0))
        msg = u.get("message")
        if not msg:
            continue
        chat = msg.get("chat", {})
        chat_id = chat.get("id")
        text = msg.get("text", "")
        if not text:
            if chat_id:
                send_message(chat_id, "أرسل اسم الزوج الذي تريد مراقبته، مثال: `/watch BTC/USDT`")
            continue
        text = text.strip()
        low = text.lower()

        if low.startswith("/start"):
            send_message(chat_id, (
                "👋 أهلاً! استخدم:\n"
                "`/watch SYMBOL` — لبدء مراقبة لمدة 24 ساعة (مثال: `/watch BTC` أو `/watch BTC/USDT`).\n"
                "`/list` لعرض المراقبات. `/stop SYMBOL` لإيقاف المراقبة."
            ))
            continue

        if low == "/list":
            monitored = state.get("monitored", {})
            if not monitored:
                send_message(chat_id, "📭 لا توجد أزواج قيد المراقبة حاليًا.")
            else:
                lines = []
                for s, info in monitored.items():
                    end_dt = datetime.utcfromtimestamp(info["end_ts"]).strftime("%Y-%m-%d %H:%M UTC")
                    lines.append(f"{s} — انتهاء: {end_dt} — الحالة: {info.get('state','unknown')}")
                send_message(chat_id, "🔎 المراقبات:\n" + "\n".join(lines[:50]))
            continue

        if low.startswith("/stop"):
            parts = text.split()
            if len(parts) >= 2:
                sym = normalize_symbol(parts[1])
                monitored = state.get("monitored", {})
                if sym in monitored:
                    monitored.pop(sym, None)
                    state["monitored"] = monitored
                    save_state(state)
                    send_message(chat_id, f"🛑 توقفت مراقبة {sym}.")
                else:
                    send_message(chat_id, f"❌ {sym} ليست في قائمة المراقبة.")
            else:
                send_message(chat_id, "استخدام: `/stop BTC` لإيقاف المراقبة.")
            continue

        if low.startswith("/watch"):
            parts = text.split()
            if len(parts) < 2:
                send_message(chat_id, "أرسل مثال: `/watch BTC/USDT` أو `/watch BTC`")
                continue
            sym_raw = parts[1]
            sym = normalize_symbol(sym_raw)
            if not market_exists(sym):
                send_message(chat_id, f"⚠️ السوق {sym} غير موجود على MEXC.")
                continue
            monitored = state.setdefault("monitored", {})
            end_ts = int((datetime.utcnow() + timedelta(hours=MONITOR_DURATION_HOURS)).timestamp())
            monitored[sym] = {"chat_id": chat_id, "end_ts": end_ts, "state": "unknown"}
            state["monitored"] = monitored
            save_state(state)
            send_message(chat_id, f"✅ بدأت مراقبة {sym} لمدة {MONITOR_DURATION_HOURS} ساعة.")
            continue

        # fallback: لو كتب رمز بدون /watch
        sym_try = normalize_symbol(text)
        try:
            if market_exists(sym_try):
                monitored = state.setdefault("monitored", {})
                end_ts = int((datetime.utcnow() + timedelta(hours=MONITOR_DURATION_HOURS)).timestamp())
                monitored[sym_try] = {"chat_id": chat_id, "end_ts": end_ts, "state": "unknown"}
                state["monitored"] = monitored
                save_state(state)
                send_message(chat_id, f"✅ بدأت مراقبة {sym_try} لمدة {MONITOR_DURATION_HOURS} ساعة.")
                continue
        except Exception:
            pass

        send_message(chat_id, "لم أفهم الطلب. أرسل `/watch BTC` أو `/watch BTC/USDT` للبدء.")
    return state

def monitor_loop(state):
    monitored = state.get("monitored", {})
    to_remove = []
    for sym, info in list(monitored.items()):
        chat_id = info.get("chat_id")
        end_ts = info.get("end_ts")
        if int(datetime.utcnow().timestamp()) > end_ts:
            send_message(chat_id, f"⏳ انتهت مدة مراقبة {sym}.")
            to_remove.append(sym)
            continue
        try:
            df = fetch_ohlcv_df(sym, MA_PERIOD + 3)
            if df is None or df.shape[0] < MA_PERIOD + 2:
                continue
            df["ma"] = df["close"].rolling(window=MA_PERIOD).mean()
            prev = df.iloc[-2]
            last = df.iloc[-1]
            prev_state = info.get("state", "unknown")
            current_pos = "above" if last["close"] > last["ma"] else "below"
            cross_up = (prev["close"] < prev["ma"]) and (last["close"] > last["ma"])

            if cross_up and prev_state != "above":
                price = float(last["close"])
                tstr = datetime.utcfromtimestamp(int(last["time"]) / 1000).strftime("%Y-%m-%d %H:%M UTC")
                send_message(chat_id, (
                    f"🚀 *اختراق MA50* — {sym}\n\n"
                    f"السعر: `{price:.6f}`\n"
                    f"الوقت: {tstr}\n"
                    f"_ستتلقى إشعارات لاحقًا فقط بعد نزول السعر تحت MA ثم اختراق جديد._"
                ))
                monitored[sym]["state"] = "above"
                save_state(state)
            elif current_pos == "below":
                monitored[sym]["state"] = "below"
                save_state(state)
        except Exception as e:
            print(f"Monitor error for {sym}: {e}")
    for s in to_remove:
        monitored.pop(s, None)
    state["monitored"] = monitored
    return state

def main():
    state = load_state()
    print("✅ بوت MA50 يعمل — جاهز لاستقبال /watch")
    if OWNER_CHAT_ID:
        try:
            send_message(OWNER_CHAT_ID, "🤖 البوت بدأ العمل على Render.")
        except Exception:
            pass
    while True:
        try:
            state = handle_updates(state)
            state = monitor_loop(state)
            save_state(state)
        except Exception as e:
            print("Main loop error:", e)
        time.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    main()