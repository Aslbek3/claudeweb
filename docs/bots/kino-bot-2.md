# kino-bot-2 — @avtopost4_bot

**Turi:** Kino/Anime tarqatuvchi bot
**PM2 nomi:** `kino-bot-2`
**Joylashuvi:** `/root/vps/claudeweb/workspace/kino-bot-2`
**Telegram:** [@avtopost4_bot](https://t.me/avtopost4_bot)

## Nima qiladi

`kino-bot` (`kino-bot-pro`) bilan **bir xil manba kodiga** ega — mustaqil ikkinchi nusxa/instansi, o'z Telegram bot hisobi (`@avtopost4_bot`) va o'z SQLite bazasi bilan ishlaydi. To'liq funksionallik tavsifi uchun [kino-bot-pro.md](kino-bot-pro.md) ga qarang — bu yerda faqat shu instansiga xos farqlar va joriy holat keltirilgan.

`bot.py`, `config.py`, `database.py`, `handlers/user.py` fayllari ikkala loyihada **bayt-baytiga bir xil** (`diff` bilan tekshirilgan, farq yo'q).

## Texnik tuzilma

```
kino-bot-2/
├── bot.py, config.py, database.py, requirements.txt   # kino-bot-pro bilan bir xil
├── kino_bot.db          # MUSTAQIL baza fayli (kino-bot-pro'nikidan alohida)
├── venv/                 # MUSTAQIL Python virtualenv
├── handlers/ (admin.py, user.py, stats_logger.py)
└── utils/ (subscription.py, keyboards.py)
```

## Boshqarish

```bash
cd "/root/vps/claudeweb/workspace/kino-bot-2"

# Holat / loglar
/usr/local/bin/pm2 describe kino-bot-2
/usr/local/bin/pm2 logs kino-bot-2 --lines 50

# Oddiy qayta ishga tushirish
/usr/local/bin/pm2 restart kino-bot-2 --update-env

# To'liq 0'dan (restart hisoblagichi ham nollanadi)
/usr/local/bin/pm2 delete kino-bot-2
/usr/local/bin/pm2 start bot.py --name kino-bot-2 --interpreter ./venv/bin/python3
/usr/local/bin/pm2 save

# Statistikani (bazani) nolga tushirish — DIQQAT: qaytarib bo'lmaydi, avval nusxa oling
cp kino_bot.db "kino_bot.db.backup_$(date +%Y%m%d_%H%M%S)"
python3 -c "
import sqlite3
conn = sqlite3.connect('kino_bot.db')
cur = conn.cursor()
for t in ['users','movies','channels','post_channels','settings','scheduled_posts','admins','preset_times','join_requests','member_events']:
    cur.execute(f'DELETE FROM {t}')
cur.execute('DELETE FROM sqlite_sequence')
conn.commit()
"
```

## Joriy holat (2026-08-14)

- Foydalanuvchilar: **78**
- Kino/anime: **2**
- Majburiy kanal: **2**
- Admin: **1**
- Bloklash/blokdan chiqarish hodisasi: **85**
- A'zolik so'rovi (join request): **64**

## Eslatmalar

- Baza sxemasi va admin panel funksiyalari `kino-bot-pro` bilan bir xil — batafsil jadval tavsifi uchun [kino-bot-pro.md](kino-bot-pro.md#baza-sxemasi-kino_botdb) ga qarang.
- `.env` faylida o'ziga xos `BOT_TOKEN` bor (kino-bot-pro'nikidan farqli), `ADMIN_IDS` bir xil.
