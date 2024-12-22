from telegram import Update, KeyboardButton, ReplyKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes
import logging

# إعداد السجلات
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
                    level=logging.INFO)
logger = logging.getLogger(__name__)

# توكن البوت
TOKEN = "7775472596:AAF3UZfaUeEiu9mfV9od3_IVxRQtiw_AObo"
CHANNEL_LINK = "https://t.me/telebot_20"

# دالة التحقق من الاشتراك في القناة
async def check_subscription(update: Update, context: ContextTypes.DEFAULT_TYPE) -> bool:
    user_id = update.message.from_user.id
    try:
        chat_member = await context.bot.get_chat_member('@telebot_20', user_id)
        if chat_member.status in ['member', 'administrator']:
            return True
    except Exception as e:
        logger.error(f"Error checking subscription for user {user_id}: {e}")
    return False

# دالة بدء البوت
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # التحقق من الاشتراك في القناة قبل السماح للمستخدم بالاستمرار
    if not await check_subscription(update, context):
        await update.message.reply_text(
            f"أهلاً بك! يجب عليك الاشتراك في القناة أولاً: {CHANNEL_LINK}\n"
            "يرجى الاشتراك ثم اضغط على /start مرة أخرى."
        )
        return
    
    context.user_data['state'] = 'await_word'
    # أزرار إضافية: زر لحذف السجل
    keyboard = [
        [KeyboardButton("ابدأ")],
        [KeyboardButton("حذف السجل")]
    ]  # زر في أسفل الترويسة
    reply_markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True, one_time_keyboard=True)
    await update.message.reply_text("مرحباً بك! ابدأ بكتابة الكلمة التي تريد نسخها.", reply_markup=reply_markup)

# دالة لحذف السجل
async def delete_record(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data.clear()  # مسح السجل بالكامل
    # رسالة توضح للمستخدم أنه تم مسح السجل
    await update.message.reply_text("تم مسح السجل. يمكنك الآن البدء من جديد.")
    context.user_data['state'] = 'await_word'  # إعادة تعيين الحالة
    await update.message.reply_text("مرحباً بك! ابدأ بكتابة الكلمة التي تريد نسخها.")

# دالة لمعالجة الرسائل
async def process_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text == "ابدأ":
        context.user_data['state'] = 'await_word'
        await update.message.reply_text("مرحباً بك! ابدأ بكتابة الكلمة التي تريد نسخها.")
        return
    elif text == "حذف السجل":
        await delete_record(update, context)
        return

    state = context.user_data.get('state', None)
    if state == 'await_word':
        context.user_data['word'] = text
        context.user_data['state'] = 'await_quantity'
        await update.message.reply_text("كم مرة تريد تكرار الكلمة؟ اكتب رقماً أو مجالاً (مثل: 100,200).")
    elif state == 'await_quantity':
        try:
            quantity = text
            # التحقق إذا كانت الكمية عبارة عن نطاق (مثل: 100,200)
            if ',' in quantity:
                start, end = map(int, quantity.split(','))
                if start > end:
                    raise ValueError("Invalid range")
                
                # إذا كان الـ end أكبر من 1000، نقوم بتحديده إلى 1000
                if end > 1000:
                    await update.message.reply_text("الكمية القصوى هي 1000. تم تعديل القيمة تلقائيًا.")
                    end = 1000
                
                repetitions = range(start, end + 1)
            else:
                num = int(quantity)
                if num < 1 or num > 1000:  # الحد الأقصى 1000
                    await update.message.reply_text("الكمية القصوى هي 1000. حاول مرة أخرى.")
                    context.user_data.clear()  # إعادة التهيئة للبيانات
                    await update.message.reply_text("اكتب الكلمة التي تريد نسخها من جديد.")
                    return
                repetitions = range(1, num + 1)

            word = context.user_data['word']
            response_parts = []
            current_part = []

            # تقسيم النتيجة إلى أجزاء صغيرة (100 سطر في كل جزء)
            for i in repetitions:
                current_part.append(f"{i}- {word}")
                if len(current_part) == 100:  # تقسيم كل 100 سطر
                    response_parts.append("\n".join(current_part))
                    current_part = []

            # إضافة الجزء الأخير إذا تبقى عناصر
            if current_part:
                response_parts.append("\n".join(current_part))

            # إرسال كل جزء في رسالة منفصلة
            for part in response_parts:
                await update.message.reply_text(part)

            # بعد الانتهاء من تنفيذ الطلب، يطلب الكلمة مرة أخرى
            context.user_data.clear()  # إعادة التهيئة للبيانات
            await update.message.reply_text("اكتب الكلمة التي تريد نسخها من جديد.")
            
        except ValueError as e:
            if str(e) == "Out of range":
                await update.message.reply_text("النطاق خارج الحدود المسموح بها. الحد الأقصى هو 1000.")
            elif str(e) == "Invalid range":
                await update.message.reply_text("أنت مخطئ في التنسيق. يرجى كتابة المجال كالتالي: 100,200")
            else:
                await update.message.reply_text("حدث خطأ غير متوقع أثناء معالجة الكمية. حاول مرة أخرى.")
            context.user_data.clear()  # إعادة التهيئة للبيانات
            await update.message.reply_text("اكتب الكلمة التي تريد نسخها من جديد.")
        except Exception as e:
            await update.message.reply_text(f"حدث خطأ غير متوقع: {e}. حاول مرة أخرى.")
            context.user_data.clear()  # إعادة التهيئة للبيانات
            await update.message.reply_text("اكتب الكلمة التي تريد نسخها من جديد.")
    else:
        context.user_data['state'] = 'await_word'
        await update.message.reply_text("اكتب الكلمة التي تريد نسخها.")

def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, process_message))
    app.run_polling()

if __name__ == "__main__":
    main()
