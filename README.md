import os
import glob
import json
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, filters

TOKEN = os.environ["TELEGRAM_TOKEN"]
PORT = int(os.environ.get("PORT", 10000))
RENDER_URL = os.environ["RENDER_EXTERNAL_URL"]

MALUMOT_PAPKA = "malumotlar"
os.makedirs(MALUMOT_PAPKA, exist_ok=True)

KODLAR_FAYL = "kodlar.json"
if os.path.exists(KODLAR_FAYL):
    with open(KODLAR_FAYL, "r", encoding="utf-8") as f:
        kodlar = json.load(f)
else:
    kodlar = {}

def kodlarni_saqlash():
    with open(KODLAR_FAYL, "w", encoding="utf-8") as f:
        json.dump(kodlar, f, ensure_ascii=False, indent=2)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    if user_id not in kodlar:
        context.user_data["holat"] = "yangi_kod"
        await update.message.reply_text(
            "Xush kelibsiz!🙌 Sizda hali kod yo'q😒.\nO'zingiz uchun yangi kod o'ylab toping va yuboring✌️ (masalan: 1234🤷‍♂️):"
        )
    else:
        context.user_data["holat"] = "kod_kutilmoqda"
        await update.message.reply_text("Kodingizni kiriting:")


async def matn_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    matn = update.message.text.strip()
    holat = context.user_data.get("holat")

    if holat == "yangi_kod":
        kodlar[user_id] = matn
        kodlarni_saqlash()
        context.user_data["holat"] = "ochilgan"
        await update.message.reply_text("Kod saqlandi✅👌 Endi botdan foydalanishingiz mumkin.¬_¬")
        return

    if holat != "ochilgan":
        if user_id in kodlar and kodlar[user_id] == matn:
            context.user_data["holat"] = "ochilgan"
            await update.message.reply_text("•Kod to'g'ri ✅ Xush kelibsiz!•")
        else:
            context.user_data["holat"] = "kod_kutilmoqda"
            await update.message.reply_text("Kod noto'g'ri🤦‍♂️. Qaytadan kiriting😤, yoki /start bosing🙂‍↕️:")
        return

    foydalanuvchi_papka = f"{MALUMOT_PAPKA}/{user_id}"
    mos_fayllar = glob.glob(f"{foydalanuvchi_papka}/{matn}.*")

    if mos_fayllar:
        fayl_nomi = mos_fayllar[0]
        kengaytma = fayl_nomi.split(".")[-1].lower()
        if kengaytma in ("jpg", "jpeg", "png"):
            await update.message.reply_photo(photo=open(fayl_nomi, "rb"))
        else:
            await update.message.reply_document(document=open(fayl_nomi, "rb"))
    else:
        await update.message.reply_text(f"'{matn}' nomli malumot topilmadi😕")


async def rasm_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    holat = context.user_data.get("holat")
    if holat != "ochilgan":
        await update.message.reply_text("Avval /start👈 bosib, kodingizni kiriting🥱.")
        return

    caption = update.message.caption
    if not caption:
        await update.message.reply_text("Rasmni va boshqa ma'lumotlaringiz saqlanishi uchun izoh kiritib qayta urinib ko`ring , masalan: 'selfi'")
        return

    foydalanuvchi_papka = f"{MALUMOT_PAPKA}/{user_id}"
    os.makedirs(foydalanuvchi_papka, exist_ok=True)

    photo = update.message.photo[-1]
    file = await context.bot.get_file(photo.file_id)
    fayl_nomi = f"{foydalanuvchi_papka}/{caption.strip()}.jpg"
    await file.download_to_drive(fayl_nomi)

    try:
        await context.bot.delete_message(chat_id=update.effective_chat.id, message_id=update.message.message_id)
    except Exception:
        pass

    await context.bot.send_message(chat_id=update.effective_chat.id, text=f"Saqlandi✅ ('{caption.strip()}' nomi bilan)")


async def hujjat_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    holat = context.user_data.get("holat")
    if holat != "ochilgan":
        await update.message.reply_text("Avval /start👈 bosib, kodingizni kiriting🥱.")
        return

    caption = update.message.caption
    if not caption:
        await update.message.reply_text("Hujjat saqlanishi uchun izoh kiritib qayta urinib ko`ring, masalan: 'diplom'")
        return

    foydalanuvchi_papka = f"{MALUMOT_PAPKA}/{user_id}"
    os.makedirs(foydalanuvchi_papka, exist_ok=True)

    document = update.message.document
    file = await context.bot.get_file(document.file_id)
    asl_kengaytma = os.path.splitext(document.file_name)[1] if document.file_name else ""
    fayl_nomi = f"{foydalanuvchi_papka}/{caption.strip()}{asl_kengaytma}"
    await file.download_to_drive(fayl_nomi)

    try:
        await context.bot.delete_message(chat_id=update.effective_chat.id, message_id=update.message.message_id)
    except Exception:
        pass

    await context.bot.send_message(chat_id=update.effective_chat.id, text=f"Hujjat saqlandi✅ ('{caption.strip()}' nomi bilan)")


async def yordam(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Bu bot sizga rasmlarni va boshqa muhim ma'lumotlarni saqlashga yordam beradi.🥱\n 1. Birinchi o'z kodingizni yarating.\n 2. Har safar ushbu yaratgan kodingiz bilan kirishingiz memkin bo'ladi.\n 3. Rasm yoki biror hujjat tashlashdan oldin izoh qoldiring\n 4. Izoh bu kalit so'zi bo`lib kerakli rasm yoki faylni topishga yordam beradi. \n/start bilan boshlang.")


app = ApplicationBuilder().token(TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(MessageHandler(filters.PHOTO, rasm_handler))
app.add_handler(MessageHandler(filters.Document.ALL, hujjat_handler))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, matn_handler))
app.add_handler(CommandHandler("yordam", yordam))

app.run_webhook(
    listen="0.0.0.0",
    port=PORT,
    url_path=TOKEN,
    webhook_url=f"{RENDER_URL}/{TOKEN}",
)
