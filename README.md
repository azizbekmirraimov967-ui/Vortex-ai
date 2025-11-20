import os
import json
import requests
from flask import Flask, request

# ====== SETTINGS ======
BOT_TOKEN = "8533013457:AAFSj7idL7vU5aatGcbwM0LT6DGqoL-39NM"
ADMIN_ID = 6184333370  # O'z admin ID raqamingizni kiriting
OPENROUTER_API_KEY = "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"
MODEL = "anthropic/claude-3.5-sonnet"

# Telegram API
API_URL = f"https://api.telegram.org/bot{BOT_TOKEN}/"

# Settings fayl
SETTINGS_FILE = "ai_settings.json"

# Flask app
app = Flask(__name__)

# ====== AI CONFIG FILE ======
def load_settings():
    """AI sozlamalarini yuklash"""
    if not os.path.exists(SETTINGS_FILE):
        default_settings = {
            "personality": "Sen foydalanuvchilarga yordam beradigan foydali AI botsan.",
            "instructions": ""
        }
        with open(SETTINGS_FILE, "w", encoding="utf-8") as f:
            json.dump(default_settings, f, ensure_ascii=False, indent=4)
        return default_settings
    
    with open(SETTINGS_FILE, "r", encoding="utf-8") as f:
        return json.load(f)

def save_settings(settings):
    """AI sozlamalarini saqlash"""
    with open(SETTINGS_FILE, "w", encoding="utf-8") as f:
        json.dump(settings, f, ensure_ascii=False, indent=4)

# ====== SEND MESSAGE ======
def send_message(chat_id, text, keyboard=None):
    """Telegram orqali xabar yuborish"""
    data = {
        "chat_id": chat_id,
        "text": text,
        "parse_mode": "HTML"
    }
    
    if keyboard:
        data["reply_markup"] = json.dumps(keyboard)
    
    requests.post(API_URL + "sendMessage", data=data)

def answer_callback_query(callback_query_id):
    """Callback query javobini yuborish"""
    requests.post(API_URL + "answerCallbackQuery", 
                  data={"callback_query_id": callback_query_id})

# ====== OPENROUTER AI FUNKSIYASI ======
def ask_ai(user_message, settings):
    """OpenRouter AI dan javob olish"""
    system_content = settings["personality"]
    if settings["instructions"]:
        system_content += "\n\n" + settings["instructions"]
    
    payload = {
        "model": MODEL,
        "messages": [
            {"role": "system", "content": system_content},
            {"role": "user", "content": user_message}
        ]
    }
    
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {OPENROUTER_API_KEY}",
        "HTTP-Referer": "https://your-domain.com",
        "X-Title": "TelegramAI"
    }
    
    try:
        response = requests.post(
            "https://openrouter.ai/api/v1/chat/completions",
            json=payload,
            headers=headers,
            timeout=60
        )
        
        if response.status_code != 200:
            return f"⚠️ AI serverida xatolik (HTTP {response.status_code}). Keyinroq urinib ko'ring."
        
        result = response.json()
        return result.get("choices", [{}])[0].get("message", {}).get("content", 
            "⚠️ Javob olishda xatolik. Keyinroq urinib ko'ring.")
    
    except requests.exceptions.Timeout:
        return "⚠️ So'rov vaqti tugadi. Iltimos, qaytadan urinib ko'ring."
    except Exception as e:
        return f"⚠️ Xatolik yuz berdi: {str(e)}"

# ====== WEBHOOK HANDLER ======
@app.route('/', methods=['POST'])
def webhook():
    """Telegram webhook handleri"""
    update = request.get_json()
    
    if not update:
        return "OK"
    
    settings = load_settings()
    
    # ====== CALLBACK QUERY ======
    if "callback_query" in update:
        callback_query = update["callback_query"]
        chat_id = callback_query["message"]["chat"]["id"]
        user_id = callback_query["from"]["id"]
        callback_data = callback_query["data"]
        
        # Callback javobini yuborish
        answer_callback_query(callback_query["id"])
        
        # Admin callback uchun
        if user_id == ADMIN_ID and callback_data == "ai_settings":
            msg = "🤖 <b>AI Sozlamalari</b>\n\n"
            msg += f"<b>Hozirgi shaxsiyat:</b>\n{settings['personality']}\n\n"
            msg += f"<b>Ko'rsatmalar:</b>\n{settings['instructions']}\n\n"
            msg += "O'zgartirish uchun:\n"
            msg += "/setpersonality [yangi matn]\n"
            msg += "/setinstructions [yangi ko'rsatma]"
            
            send_message(chat_id, msg)
        
        return "OK"
    
    # ====== ODDIY XABAR ======
    if "message" not in update:
        return "OK"
    
    message = update["message"]
    chat_id = message["chat"]["id"]
    user_id = message["from"]["id"]
    text = message.get("text", "")
    
    if not text:
        return "OK"
    
    # ====== ADMIN BUYRUQLARI ======
    if user_id == ADMIN_ID:
        
        if text in ["/start", "/admin"]:
            keyboard = {
                "inline_keyboard": [
                    [{"text": "🤖 AI Sozlamalari", "callback_data": "ai_settings"}]
                ]
            }
            send_message(chat_id, "👨‍💼 Admin panelga xush kelibsiz!", keyboard)
            return "OK"
        
        if text.startswith("/setpersonality "):
            new_personality = text[16:].strip()
            if not new_personality:
                send_message(chat_id, 
                    "❌ Iltimos, matn kiriting.\n\nMisol:\n/setpersonality Sen do'stona AI yordamchisan")
                return "OK"
            
            settings["personality"] = new_personality
            save_settings(settings)
            send_message(chat_id, f"✅ Shaxsiyat yangilandi:\n\n{new_personality}")
            return "OK"
        
        if text.startswith("/setinstructions "):
            new_instructions = text[17:].strip()
            if not new_instructions:
                send_message(chat_id, 
                    "❌ Iltimos, matn kiriting.\n\nMisol:\n/setinstructions Har doim qisqa javob ber")
                return "OK"
            
            settings["instructions"] = new_instructions
            save_settings(settings)
            send_message(chat_id, "✅ Ko'rsatmalar yangilandi!")
            return "OK"
    
    # ====== ODDIY FOYDALANUVCHI UCHUN AI JAVOB ======
    if text and not text.startswith('/'):
        send_message(chat_id, "⏳ Javob tayyorlanmoqda...")
        ai_response = ask_ai(text, settings)
        send_message(chat_id, ai_response)
    
    return "OK"

# ====== WEBHOOK O'RNATISH ======
def set_webhook(webhook_url):
    """Telegram webhook ni o'rnatish"""
    url = API_URL + "setWebhook"
    response = requests.post(url, data={"url": webhook_url})
    print(response.json())

if __name__ == '__main__':
    # Webhook URL ni o'rnating (masalan: https://your-domain.com/webhook)
    # set_webhook("https://your-domain.com/")
    
    # Local test uchun
    app.run(host='0.0.0.0', port=5000, debug=True)
