import os
import logging
import requests

from flask import Flask, request, jsonify

# ==========================================
# ENGINEERING AI - Cloud Telegram Bot
# ==========================================

app = Flask(__name__)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s"
)

logger = logging.getLogger("ENGINEERING_AI")

# ---------- Configuration ----------

TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN", "").strip()
OPENROUTER_API_KEY = os.getenv("OPENROUTER_API_KEY", "").strip()
APP_URL = os.getenv("APP_URL", "").strip().rstrip("/")
WEBHOOK_SECRET = os.getenv("WEBHOOK_SECRET", "").strip()

OPENROUTER_MODEL = os.getenv(
    "OPENROUTER_MODEL",
    "openrouter/auto"
).strip()

TELEGRAM_API = (
    f"https://api.telegram.org/bot{TELEGRAM_TOKEN}"
)

OPENROUTER_API = "https://openrouter.ai/api/v1/chat/completions"

SYSTEM_PROMPT = """
تو ENGINEERING AI هستی؛ یک دستیار تخصصی فارسی‌زبان
برای مهندسی برق، الکترونیک، کامپیوتر و برنامه‌نویسی.

وظایف تو:

1. آموزش الکترونیک از مقدماتی تا پیشرفته.
2. تحلیل مدارهای الکترونیکی و توضیح عملکرد قطعات.
3. راهنمایی عیب‌یابی بردهای الکترونیکی.
4. آموزش میکروکنترلرها، برنامه‌نویسی C و C++ و Python.
5. آموزش مبانی دیجیتال، گیت‌های منطقی و تبدیل مبناها.
6. کمک به حل مسائل ریاضی، فیزیک و دروس فنی.
7. توضیح شبکه‌های کامپیوتری و سیستم‌های نظارتی.
8. نوشتن و اصلاح کدهای برنامه‌نویسی.
9. ارائه پروژه‌های عملی مناسب دانش‌آموزان الکترونیک.

قوانین پاسخ‌گویی:

- فارسی روان و قابل فهم بنویس.
- پاسخ‌ها را مرحله‌به‌مرحله توضیح بده.
- در مسائل محاسباتی، فرمول و مراحل حل را نشان بده.
- اگر اطلاعات مسئله کافی نیست، سؤال بپرس.
- اگر از جواب مطمئن نیستی، صادقانه بگو.
- برای عیب‌یابی، ابتدا قطعات و آزمایش‌های ساده را بررسی کن.
- در کار با برق شهر و خازن‌های ولتاژ بالا، هشدار ایمنی بده.
- هیچ‌وقت مقدار اندازه‌گیری‌نشده را به‌عنوان واقعیت اعلام نکن.
- برای کدهای برنامه‌نویسی، کد کامل و روش اجرای آن را ارائه بده.
"""

# ---------- Telegram Functions ----------

def telegram_request(method, payload):
    if not TELEGRAM_TOKEN:
        raise RuntimeError("TELEGRAM_TOKEN is not configured.")

    response = requests.post(
        f"{TELEGRAM_API}/{method}",
        json=payload,
        timeout=20
    )

    response.raise_for_status()
    result = response.json()

    if not result.get("ok"):
        raise RuntimeError(
            result.get("description", "Telegram API error")
        )

    return result


def send_message(chat_id, text):
    if not text:
        text = "متأسفانه پاسخی دریافت نشد. دوباره تلاش کن."

    # Telegram messages have a length limit.
    chunks = [
        text[i:i + 4000]
        for i in range(0, len(text), 4000)
    ]

    for chunk in chunks:
        telegram_request(
            "sendMessage",
            {
                "chat_id": chat_id,
                "text": chunk
            }
        )


# ---------- AI Function ----------

def ask_ai(user_text):
    if not OPENROUTER_API_KEY:
        return (
            "سرویس هوش مصنوعی هنوز تنظیم نشده است. "
            "مدیر ربات باید کلید API را در متغیرهای "
            "محیطی سرور قرار دهد."
        )

    headers = {
        "Authorization": f"Bearer {OPENROUTER_API_KEY}",
        "Content-Type": "application/json",
        "X-Title": "ENGINEERING AI"
    }

    payload = {
        "model": OPENROUTER_MODEL,
        "messages": [
            {
                "role": "system",
                "content": SYSTEM_PROMPT
            },
            {
                "role": "user",
                "content": user_text
            }
        ],
        "temperature": 0.4,
        "max_tokens": 2000
    }

    try:
        response = requests.post(
            OPENROUTER_API,
            headers=headers,
            json=payload,
            timeout=90
        )

        if response.status_code != 200:
            logger.error(
                "OpenRouter returned HTTP %s",
                response.status_code
            )

            return (
                "ارتباط با هوش مصنوعی با مشکل روبه‌رو شد. "
                "کمی بعد دوباره امتحان کن."
            )

        data = response.json()

        choices = data.get("choices", [])

        if not choices:
            return "هوش مصنوعی پاسخی برنگرداند. دوباره تلاش کن."

        answer = choices[0]["message"].get("content", "")

        if isinstance(answer, list):
            answer = "\n".join(
                item.get("text", "")
                for item in answer
                if isinstance(item, dict)
            )

        answer = str(answer).strip()

        if not answer:
            return "پاسخ خالی دریافت شد. دوباره تلاش کن."

        return answer

    except requests.Timeout:
        logger.warning("OpenRouter request timed out.")
        return "پاسخ‌گویی زمان‌بر شد. لطفاً دوباره تلاش کن."

    except requests.RequestException:
        logger.exception("OpenRouter connection error.")
        return "در اتصال به سرویس هوش مصنوعی مشکلی پیش آمد."

    except (KeyError, ValueError, TypeError):
        logger.exception("Invalid OpenRouter response.")
        return "پاسخ سرویس قابل پردازش نبود. دوباره تلاش کن."


# ---------- Message Handler ----------

def handle_update(update):
    message = update.get("message")

    if not message:
        return

    chat = message.get("chat", {})
    chat_id = chat.get("id")
    text = message.get("text", "")

    if not chat_id or not text:
        return

    text = text.strip()

    if text.startswith("/start"):
        send_message(
            chat_id,
            "⚡ به ENGINEERING AI خوش آمدی!\n\n"
            "دستیار تخصصی الکترونیک، برنامه‌نویسی و مهندسی.\n\n"
            "می‌توانی درباره این موضوعات سؤال کنی:\n"
            "🔌 الکترونیک و عیب‌یابی مدار\n"
            "💻 Python، C و C++\n"
            "🔢 مبانی دیجیتال و تبدیل مبنا\n"
            "📐 ریاضی و فیزیک\n"
            "🛠 پروژه‌های عملی مهندسی\n\n"
            "سؤالت را بنویس تا شروع کنیم!"
        )
        return

    if text.startswith("/help"):
        send_message(
            chat_id,
            "راهنمای ENGINEERING AI\n\n"
            "سؤالت را به زبان فارسی بنویس.\n"
            "برای مثال:\n"
            "• ترانزیستور چگونه کار می‌کند؟\n"
            "• کد تبدیل باینری به ده‌دهی در پایتون چیست؟\n"
            "• چطور یک مدار را عیب‌یابی کنم؟"
        )
        return

    if text.startswith("/"):
        send_message(
            chat_id,
            "این دستور شناخته‌شده نیست. "
            "از /start استفاده کن یا سؤالت را بنویس."
        )
        return

    send_message(chat_id, "⏳ در حال بررسی سؤال تو هستم...")

    answer = ask_ai(text)

    send_message(chat_id, answer)


# ---------- Webhook Routes ----------

@app.route("/", methods=["GET"])
def home():
    return jsonify({
        "name": "ENGINEERING AI",
        "status": "running",
        "service": "Telegram Webhook"
    })


@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok"}), 200


@app.route("/webhook/<secret>", methods=["POST"])
def webhook(secret):
    if not WEBHOOK_SECRET or secret != WEBHOOK_SECRET:
        return jsonify({"error": "Unauthorized"}), 403

    update = request.get_json(silent=True)

    if not isinstance(update, dict):
        return jsonify({"error": "Invalid JSON"}), 400

    try:
        handle_update(update)
    except Exception:
        logger.exception("Error handling Telegram update.")

    # Telegram receives confirmation that the update was received.
    return jsonify({"ok": True}), 200


# ---------- Register Telegram Webhook ----------

def configure_webhook():
    if not all([
        TELEGRAM_TOKEN,
        OPENROUTER_API_KEY,
        APP_URL,
        WEBHOOK_SECRET
    ]):
        logger.warning(
            "Webhook not configured. Set all required "
            "environment variables on the hosting service."
        )
        return

    webhook_url = f"{APP_URL}/webhook/{WEBHOOK_SECRET}"

    try:
        response = requests.post(
            f"{TELEGRAM_API}/setWebhook",
            json={
                "url": webhook_url,
                "secret_token": WEBHOOK_SECRET,
                "drop_pending_updates": False
            },
            timeout=20
        )

        response.raise_for_status()
        result = response.json()

        if result.get("ok"):
            logger.info("Telegram webhook configured successfully.")
        else:
            logger.error(
                "Telegram webhook configuration failed: %s",
                result.get("description", "Unknown error")
            )

    except requests.RequestException:
        logger.exception("Could not configure Telegram webhook.")


configure_webhook()


if __name__ == "__main__":
    port = int(os.getenv("PORT", "10000"))

    app.run(
        host="0.0.0.0",
        port=port
    )
