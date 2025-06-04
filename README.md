# Samsa

import os
import re
import logging
from telegram import Update
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext
import pdfplumber
import requests
from datetime import datetime, timedelta

# تكوين السجل
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# متغيرات البيئة
TELEGRAM_TOKEN = os.getenv('@Sultanaah_bot')
SMSA_API_KEY = os.getenv('SMSA_API_KEY', 'default_api_key')  # استبدل بواجهة سمسا الفعلية

# تخزين مؤقت للشحنات
shipments_cache = {}

class SMSATracker:
    @staticmethod
    def extract_tracking_numbers(pdf_path):
        numbers = []
        try:
            with pdfplumber.open(pdf_path) as pdf:
                for page in pdf.pages:
                    text = page.extract_text()
                    if text:
                        # نمط أرقام تتبع سمسا (اضبطه حسب تنسيق بوليصاتك)
                        found_numbers = re.findall(r'\b[A-Z]{2}\d{8,12}[A-Z]?\b', text)
                        numbers.extend(found_numbers)
        except Exception as e:
            logger.error(f"Error extracting from PDF: {e}")
        return list(set(numbers))

    @staticmethod
    def get_shipment_status(tracking_number):
        try:
            # هذه دالة وهمية - استبدلها بواجهة سمسا الفعلية
            # يمكنك استخدام requests للاتصال بخدمة الويب الخاصة بسمسا
            if not tracking_number:
                return "رقم تتبع غير صالح"
            
            # لمحاكاة واجهة سمسا (استبدل هذا الجزء)
            statuses = [
                "تم استلام الطلب",
                "في الانتقال",
                "في التوصيل",
                "تم التسليم"
            ]
            fake_status = statuses[hash(tracking_number) % len(statuses)]
            
            return fake_status
        except Exception as e:
            logger.error(f"Tracking error: {e}")
            return "خطأ في الاستعلام"

# أوامر البوت
def start(update: Update, context: CallbackContext) -> None:
    user = update.effective_user
    update.message.reply_markdown_v2(
        fr'مرحباً {user.mention_markdown_v2()}\! أرسل لي ملف PDF يحتوي على بوليصات شحن سمسا وسأخبرك بحالة كل شحنة\.'
    )

def help_command(update: Update, context: CallbackContext) -> None:
    update.message.reply_text('مساعدة: أرسل ملف PDF يحتوي على بوليصات شحن سمسا وسأقوم بتتبعها لك.')

def handle_document(update: Update, context: CallbackContext) -> None:
    if update.message.document.mime_type != 'application/pdf':
        update.message.reply_text('⚠️ يرجى إرسال ملف PDF فقط.')
        return

    # تحميل الملف
    file = context.bot.get_file(update.message.document.file_id)
    file_path = f'temp_{update.message.document.file_id}.pdf'
    file.download(file_path)
    
    # استخراج أرقام التتبع
    tracker = SMSATracker()
    tracking_numbers = tracker.extract_tracking_numbers(file_path)
    
    if not tracking_numbers:
        update.message.reply_text('❌ لم أتمكن من العثور على أرقام تتبع في الملف.')
        os.remove(file_path)
        return
    
    # الحصول على الحالات
    results = ["🚚 حالة الشحنات:"]
    for num in tracking_numbers[:10]:  # حد أقصى 10 شحنات لكل ملف
        if num in shipments_cache and shipments_cache[num]['expiry'] > datetime.now():
            status = shipments_cache[num]['status']
        else:
            status = tracker.get_shipment_status(num)
            shipments_cache[num] = {
                'status': status,
                'expiry': datetime.now() + timedelta(hours=1)
            }
        results.append(f"\n📦 {num}: {status}")
    
    update.message.reply_text('\n'.join(results))
    os.remove(file_path)

def error_handler(update: Update, context: CallbackContext) -> None:
    logger.error(msg="حدث خطأ في البوت:", exc_info=context.error)
    if update and update.message:
        update.message.reply_text('❌ حدث خطأ غير متوقع. يرجى المحاولة لاحقاً.')

def main() -> None:
    if not TELEGRAM_TOKEN:
        logger.error("TELEGRAM_TOKEN غير محدد!")
        return

    updater = Updater(TELEGRAM_TOKEN)
    dispatcher = updater.dispatcher

    # تسجيل الأوامر
    dispatcher.add_handler(CommandHandler("start", start))
    dispatcher.add_handler(CommandHandler("help", help_command))
    dispatcher.add_handler(MessageHandler(Filters.document, handle_document))
    
    # تسجيل معالج الأخطاء
    dispatcher.add_error_handler(error_handler)

    # بدء البوت
    updater.start_polling()
    logger.info("Bot is running...")
    updater.idle()

if __name__ == '__main__':
    main()
