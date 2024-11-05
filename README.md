import logging
import subprocess
import sys

# التأكد من تثبيت المكتبات المطلوبة
try:
    import openai
    import requests
except ImportError as e:
    package_name = str(e).split("'")[1]
    subprocess.check_call([sys.executable, "-m", "pip", "install", package_name])
    import openai
    import requests

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, Sticker
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, CallbackQueryHandler, filters, ContextTypes

# إعدادات التسجيل
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# معرف المطور وتوكن البوت وAPI مفتاح OpenAI
DEV_ID = '7176809828'
TOKEN = '7913052380:AAFq3sZEO118kK1cvWm2OTW-7InneCjsFfE'
OPENAI_API_KEY = 'YOUR_OPENAI_API_KEY'

openai.api_key = OPENAI_API_KEY

# وظيفة البدء
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text("مرحباً! أنا كونان، مساعدك الذكي. كيف يمكنني مساعدتك اليوم؟")

# وظيفة واجهة التحكم للمطور
async def admin_panel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if str(update.effective_user.id) == DEV_ID:
        keyboard = [
            [InlineKeyboardButton("الإحصائيات", callback_data='stats')],
            [InlineKeyboardButton("إعدادات البوت", callback_data='settings')],
            [InlineKeyboardButton("إضافة نكتة جديدة", callback_data='add_joke')],
            [InlineKeyboardButton("إضافة رد تلقائي", callback_data='add_response')],
            [InlineKeyboardButton("تحكم الردود", callback_data='manage_responses')]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        await update.message.reply_text("مرحبًا بك في لوحة التحكم للبوت!", reply_markup=reply_markup)
    else:
        await update.message.reply_text("ليس لديك صلاحيات الوصول إلى لوحة التحكم.")

# وظيفة للتفاعل مع الرسائل
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    text = update.message.text.lower()
    
    if "مرحبا" in text or "اهلا" in text:
        await update.message.reply_text("مرحباً بك! كيف أستطيع مساعدتك؟")

    elif "ملصق" in text:
        sticker_name = text.split("ملصق ")[-1]
        sticker_url = search_for_sticker(sticker_name)
        if sticker_url:
            await update.message.reply_sticker(sticker=Sticker(file_id=sticker_url))
        else:
            await update.message.reply_text("لم أتمكن من العثور على الملصق المطلوب.")

    elif "من هو" in text or "ما هو" in text or "سؤال" in text:
        response = await get_openai_answer(text)
        await update.message.reply_text(response)
    
    elif "أنمي" in text:
        anime_info = search_anime_info(text)
        await update.message.reply_text(anime_info)

    else:
        await update.message.reply_text("أعتذر، لم أفهم الرسالة. يمكنك سؤالي عن الأنمي أو أي استفسار عام.")

# وظيفة جلب إجابة من OpenAI
async def get_openai_answer(query: str) -> str:
    response = openai.Completion.create(
        engine="text-davinci-003",
        prompt=query,
        max_tokens=100
    )
    return response.choices[0].text.strip()

# وظيفة البحث عن معلومات الأنمي
def search_anime_info(query: str) -> str:
    search_url = f"https://www.googleapis.com/customsearch/v1?q={query}&key=YOUR_GOOGLE_API_KEY"
    response = requests.get(search_url)
    data = response.json()
    if "items" in data:
        return data["items"][0]["snippet"]
    else:
        return "لم أتمكن من العثور على معلومات حول هذا الأنمي."

# وظيفة البحث عن ملصق
def search_for_sticker(sticker_name: str) -> str:
    # بحث باستخدام API لموقع ملصقات متاح (قد تحتاج تعديل حسب المتاح)
    return "STICKER_FILE_ID"

# تشغيل التطبيق
def main() -> None:
    app = ApplicationBuilder().token(TOKEN).build()

    # إضافة الأوامر إلى المدخلات
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("Admin", admin_panel))
    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    # معالجة الأخطاء
    app.add_error_handler(error)

    # بدء البوت
    app.run_polling()

if __name__ == '__main__':
    main()
