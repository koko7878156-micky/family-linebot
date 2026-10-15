import sys
sys.stdout.reconfigure(line_buffering=True)

from flask import Flask, request
import os, json, datetime, threading, re, hmac, hashlib, base64
import gspread
from google.oauth2.service_account import Credentials
from openai import OpenAI
import requests as req

app = Flask(__name__)

LINE_CHANNEL_SECRET = os.environ.get("LINE_CHANNEL_SECRET", "")
LINE_CHANNEL_ACCESS_TOKEN = os.environ.get("LINE_CHANNEL_ACCESS_TOKEN", "")
OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY", "")
GOOGLE_SHEET_ID = os.environ.get("GOOGLE_SHEET_ID", "")
GOOGLE_CREDENTIALS_JSON = os.environ.get("GOOGLE_CREDENTIALS_JSON", "")

client = OpenAI(api_key=OPENAI_API_KEY)

TW = datetime.timezone(datetime.timedelta(hours=8))

def get_sheet():
    creds_dict = json.loads(GOOGLE_CREDENTIALS_JSON)
    scopes = ["https://www.googleapis.com/auth/spreadsheets"]
    creds = Credentials.from_service_account_info(creds_dict, scopes=scopes)
    gc = gspread.authorize(creds)
    sh = gc.open_by_key(GOOGLE_SHEET_ID)
    try:
        ws = sh.worksheet("記帳")
    except:
        ws = sh.add_worksheet(title="記帳", rows="1000", cols="7")
        ws.append_row(["日期", "時間", "傳送者", "項目", "金額", "類別", "原始訊息"])
    return ws

CATEGORIES = ["餐飲", "交通", "生活", "醫療健康", "美容保養", "服飾",
              "訂閱服務", "娛樂休閒", "運動健身", "旅遊", "教育", "保險",
              "稅費", "捐款公益", "其他"]

def fallback_expense(text):
    """AI 無法判斷時，只要訊息含數字就強制記錄為其他。"""
    match = re.search(r'^(.*?)\s*(\d+)\s*$', text.strip())
    if match:
        item = match.group(1).strip() or text.strip()
        amount = int(match.group(2))
        return [{"item": item, "amount": amount, "category": "其他"}]
    return None

def parse_expense(text):
    sys.stdout.write(f"[PARSE] 開始解析: {text}\n")
    sys.stdout.flush()
    prompt = f"""你是家庭記帳助手。分析訊息是否為記帳內容。

訊息：{text}

規則：只要訊息包含「任何文字 + 數字」的模式，一律視為記帳，不管項目名稱是什麼。
- item: 項目名稱（數字前的所有文字，保留原文包含特殊符號）
- amount: 金額數字（找到的第一個整數）
- category: 最符合的類別

分類規則（優先於其他判斷）：
- 含有「食材、食物、食品、生鮮、蔬菜、水果、肉類、海鮮、零食、點心、飲料、早餐、午餐、晚餐、宵夜、飯、麵、鍋、滷味」等詞 → 餐飲
- 含有「規費、手續費、辦理費、地政、戶政、登記費、代辦、公證、規費」等詞 → 稅費
- 含有「稅、罰款、停車費、道路」等詞 → 稅費
- 即使是全聯、好市多、家樂福等賣場，只要品項名稱含食物相關詞就歸餐飲

可用類別：{", ".join(CATEGORIES)}

不是記帳訊息（純問候、沒有數字的句子）才回傳 []
只回傳 JSON，不要其他文字

範例：
- "早餐 100" → [{{"item":"早餐","amount":100,"category":"餐飲"}}]
- "好市多食材 2652" → [{{"item":"好市多食材","amount":2652,"category":"餐飲"}}]
- "全聯生鮮 350" → [{{"item":"全聯生鮮","amount":350,"category":"餐飲"}}]
- "規費 44" → [{{"item":"規費","amount":44,"category":"稅費"}}]
- "戶政&地政辦理 124" → [{{"item":"戶政&地政辦理","amount":124,"category":"稅費"}}]
- "Claude 629" → [{{"item":"Claude","amount":629,"category":"其他"}}]
- "停車 80" → [{{"item":"停車","amount":80,"category":"交通"}}]"""

    try:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=500
        )
        raw = response.choices[0].message.content.strip()
        raw = raw.replace("```json", "").replace("```", "").strip()
        sys.stdout.write(f"[PARSE] OpenAI: {raw}\n")
        sys.stdout.flush()
        data = json.loads(raw)
        return data if isinstance(data, list) and len(data) > 0 else None
    except Exception as e:
        sys.stdout.write(f"[PARSE] 錯誤: {e}\n")
        sys.stdout.flush()
        return None

def reply_message(reply_token, text):
    url = "https://api.line.me/v2/bot/message/reply"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {LINE_CHANNEL_ACCESS_TOKEN}"
    }
    req.post(url, headers=headers, json={
        "replyToken": reply_token,
        "messages": [{"type": "text", "text": text}]
    })

def query_stats(period):
    try:
        ws = get_sheet()
        rows = ws.get_all_records()
    except Exception as e:
        return f"📭 無法連接試算表：{e}"
    if not rows:
        return "📭 目前還沒有記帳資料。"
    today = datetime.datetime.now(TW).date()
    if period == "today":
        filtered = [r for r in rows if r.get("日期") == str(today)]
        title = f"📅 {today.strftime('%m/%d')} 今日支出"
    elif period == "month":
        ym = today.strftime("%Y-%m")
        filtered = [r for r in rows if str(r.get("日期", "")).startswith(ym)]
        title = f"📊 {today.strftime('%Y年%m月')} 本月支出"
    else:
        ym = today.strftime("%Y-%m")
        filtered = [r for r in rows if str(r.get("日期", "")).startswith(ym)]
        title = f"👤 {today.strftime('%Y年%m月')} 各人支出"
    if not filtered:
        return f"{title}\n\n還沒有資料。"
    if period == "stats":
        person_totals = {}
        for r in filtered:
            p = r.get("傳送者", "未知")
            amt = int(r.get("金額", 0))
            person_totals[p] = person_totals.get(p, 0) + amt
        total = sum(person_totals.values())
        lines = [f"💰 {title}", ""]
        for p, amt in sorted(person_totals.items(), key=lambda x: -x[1]):
            pct = amt / total * 100 if total else 0
            lines.append(f"  {p}：${amt:,}（{pct:.0f}%）")
        lines.append(f"\n  合計：${total:,}")
        return "\n".join(lines)
    else:
        cat_totals = {}
        grand = 0
        for r in filtered:
            cat = r.get("類別", "其他")
            amt = int(r.get("金額", 0))
            cat_totals[cat] = cat_totals.get(cat, 0) + amt
            grand += amt
        lines = [f"💰 {title}", f"合計：${grand:,}", ""]
        for cat, amt in sorted(cat_totals.items(), key=lambda x: -x[1]):
            lines.append(f"  {cat}　${amt:,}")
        return "\n".join(lines)

def handle_event(event):
    if event.get("type") != "message":
        return
    msg = event.get("message", {})
    if msg.get("type") != "text":
        return
    text = msg.get("text", "").strip()
    reply_token = event.get("replyToken", "")
    sender = event.get("source", {}).get("userId", "未知")
    sys.stdout.write(f"[MSG] '{text}' from {sender}\n")
    sys.stdout.flush()
    if text in ["/今日", "/today"]:
        reply_message(reply_token, query_stats("today"))
    elif text in ["/本月", "/month"]:
        reply_message(reply_token, query_stats("month"))
    elif text in ["/統計", "/stats"]:
        reply_message(reply_token, query_stats("stats"))
    elif text in ["/說明", "/help"]:
        reply_message(reply_token, "記帳小天才\n\n直接傳費用記錄：\n  早餐 100\n  停車費｜80\n\n查詢：\n  /今日 /本月 /統計 /說明")
    else:
        expenses = parse_expense(text) or fallback_expense(text)
        if expenses:
            try:
                ws = get_sheet()
                today = str(datetime.datetime.now(TW).date())
                now_time = datetime.datetime.now(TW).strftime("%H:%M")
                lines = []
                for exp in expenses:
                    ws.append_row([today, now_time, sender, exp["item"], exp["amount"], exp["category"], text])
                    lines.append(f"✅ {exp['item']} ${exp['amount']:,}（{exp['category']}）")
                    sys.stdout.write(f"[SHEETS] 寫入: {exp}\n")
                    sys.stdout.flush()
                reply_message(reply_token, "\n".join(lines))
            except Exception as e:
                sys.stdout.write(f"[SHEETS] 失敗: {e}\n")
                sys.stdout.flush()

@app.route("/callback", methods=["POST"])
def callback():
    body = request.get_data(as_text=True)
    # 確認真的是 LINE 送來的（用 Channel Secret 驗簽章），不然任何人都能往試算表塞資料
    if not LINE_CHANNEL_SECRET:
        sys.stdout.write("[WEBHOOK] 沒設定 LINE_CHANNEL_SECRET，拒收\n")
        return "LINE_CHANNEL_SECRET not set", 500
    expected = base64.b64encode(hmac.new(LINE_CHANNEL_SECRET.encode(), body.encode(), hashlib.sha256).digest()).decode()
    if not hmac.compare_digest(expected, request.headers.get("X-Line-Signature", "")):
        sys.stdout.write("[WEBHOOK] 簽章不對，拒收\n")
        return "bad signature", 403
    sys.stdout.write(f"[WEBHOOK] 收到訊息\n")
    sys.stdout.flush()
    try:
        data = json.loads(body)
    except:
        return "OK"
    for event in data.get("events", []):
        t = threading.Thread(target=handle_event, args=(event,))
        t.daemon = True
        t.start()
    return "OK"

@app.route("/", methods=["GET"])
def health():
    return "✅ OK"

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port)
