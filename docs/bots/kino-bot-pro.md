# kino-bot (kino-bot-pro) — @TarjimaKinoKodlari_bot

**Turi:** Kino/Anime tarqatuvchi bot
**PM2 nomi:** `kino-bot`
**Joylashuvi:** `/root/vps/claudeweb/workspace/kino-bot-pro`
**Telegram:** [@TarjimaKinoKodlari_bot](https://t.me/TarjimaKinoKodlari_bot)

## Nima qiladi

Foydalanuvchi botga kino yoki anime **kodini** (masalan `1234`, yoki deep-link orqali `k1234` / `a1234`) yuboradi, bot mos faylni (video/hujjat/rasm/animatsiya/audio) qaytaradi. Kodni yuborishdan oldin foydalanuvchi **majburiy kanal/guruhlarga a'zo** bo'lishi talab qilinadi (obuna tekshiruvi). Admin panel orqali kino/anime yuklash, majburiy kanallarni boshqarish, botni sozlash, statistikani ko'rish va **rejalashtirilgan avtopost** (belgilangan vaqtda kanalga e'lon yuborish) qilish mumkin.

### Foydalanuvchi oqimi
1. `/start` (yoki `/start k1234` / `/start a1234` deep-link) — ro'yxatdan o'tadi.
2. Bot majburiy kanallarga obunani tekshiradi (`utils/subscription.py`); obuna bo'lmasa — obuna tugmalari va "✅ Tekshirish" tugmasi ko'rsatiladi.
3. Obuna tasdiqlansa (yoki oldindan obuna bo'lsa) — so'ralgan kino/anime yuboriladi.
4. Agar oddiy raqam yuborilsa va shu kod ostida ham kino, ham anime bo'lsa — foydalanuvchiga tanlov tugmalari chiqadi.
5. "Maxfiy va so'rovli" (join-request) kanallar uchun: admin so'rovni Telegramning o'zida tasdiqlashi shart emas — bot foydalanuvchi so'rov yuborganini bazaga yozadi va shuni "obuna bor" deb hisoblaydi (`join_requests` jadvali), shu bilan foydalanuvchi bloklanib qolmaydi.
6. Foydalanuvchi botni bloklasa/blokdan chiqarsa — `handlers/stats_logger.py` buni kuzatib, bazadagi statusni yangilaydi (`active`/`left`).

### Admin panel (`/admin` yoki "🔧 Boshqarish" tugmasi)
- **🎬 Kino/Anime yuklash** — fayl + kod + tavsif bilan yangi kontent qo'shish, kodi bo'yicha o'chirish (tasdiqlash bilan).
- **📊 Statistika** — foydalanuvchilar/kontent sonlari.
- **📢 Majburiy obunalar** — kanal/guruh qo'shish, ikki turi bor: (1) ochiq va cheksiz, (2) maxfiy va so'rovli (join-request).
- **🔗 Qo'shimcha kanallar** — avtopost yuboriladigan kanallar ro'yxati.
- **⏳ Avto vaqtlar** — kunlik rejalashtirilgan post vaqtlarini belgilash (preset times); **📋 Rejalashtirilgan postlar** — navbatdagi/yuborilgan postlar ro'yxati va o'chirish.
- **✉️ Xabar yuborish** — broadcast.
- **👥 Userlar** / **👤 Adminlar** — foydalanuvchi/admin boshqaruvi (admin qo'shish/o'chirish).
- **⚙️ Sozlamalar** — masalan `protect_content` (fayllarni forward/saqlashdan himoya qilish) kabi bot darajasidagi sozlamalar.

### Fon vazifasi (`bot.py`)
`check_scheduled_posts_loop` — har 15 soniyada bazadagi kutilayotgan (`pending`) rejalashtirilgan postlarni tekshiradi va vaqti kelganlarini tegishli kanal(lar)ga "🍿 Kino/Animeni ko'rish" tugmasi (deep-link) bilan yuboradi.

## Texnik tuzilma

```
kino-bot-pro/
├── bot.py                  # kirish nuqtasi: dispatcher, router ro'yxati, avtopost fon vazifasi
├── config.py                # .env dan BOT_TOKEN, ADMIN_IDS o'qiydi
├── database.py               # SQLite: users, movies, channels, admins va h.k. (630 qator)
├── kino_bot.db                # SQLite baza fayli
├── requirements.txt          # aiogram>=3.0.0, python-dotenv
├── venv/                      # mustaqil Python virtualenv
├── handlers/
│   ├── admin.py                # admin panel — eng katta fayl (~1400 qator)
│   ├── user.py                  # foydalanuvchi oqimi, kod qidirish, obuna tekshiruvi
│   └── stats_logger.py           # bot bloklash/blokdan chiqarish hodisalarini kuzatish
└── utils/
    ├── subscription.py           # kanal a'zoligini Telegram API orqali tekshirish
    └── keyboards.py                # reply/inline klaviaturalar
```

## Baza sxemasi (`kino_bot.db`)

| Jadval | Vazifasi |
|---|---|
| `users` | Foydalanuvchilar (id, username, status: active/left) |
| `movies` | Kino/anime kontenti (kod, turi, file_id, tavsif) |
| `channels` | Majburiy obuna kanallari (ochiq yoki so'rovli) |
| `post_channels` | Avtopost yuboriladigan qo'shimcha kanallar |
| `settings` | Bot darajasidagi sozlamalar (masalan `protect_content`) |
| `scheduled_posts` | Rejalashtirilgan/yuborilgan postlar navbati |
| `admins` | Qo'shimcha adminlar ro'yxati (`.env`dagi `ADMIN_IDS`dan tashqari) |
| `preset_times` | Avtopost uchun kunlik vaqt shablonlari |
| `join_requests` | Maxfiy/so'rovli kanallarga yuborilgan a'zolik so'rovlari |
| `member_events` | Bloklash/blokdan chiqarish hodisalari logi |

## Boshqarish

```bash
cd "/root/vps/claudeweb/workspace/kino-bot-pro"

# Holat / loglar
/usr/local/bin/pm2 describe kino-bot
/usr/local/bin/pm2 logs kino-bot --lines 50

# Oddiy qayta ishga tushirish
/usr/local/bin/pm2 restart kino-bot --update-env

# To'liq 0'dan (restart hisoblagichi ham nollanadi)
/usr/local/bin/pm2 delete kino-bot
/usr/local/bin/pm2 start bot.py --name kino-bot --interpreter ./venv/bin/python3
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

- Foydalanuvchilar: **30**
- Kino/anime: **4**
- Majburiy kanal: **1**
- Admin: **2**
- Bloklash/blokdan chiqarish hodisasi: **32**
- A'zolik so'rovi (join request): **2**

## Eslatmalar

- `kino-bot-2` bilan **bir xil manba kodiga** ega (`diff` bo'yicha farqsiz) — bu ikkinchi mustaqil nusxa, o'z Telegram bot hisobi va o'z bazasi bilan.
- `.env` faylida `BOT_TOKEN` va `ADMIN_IDS` bor — hech qachon oshkor qilinmasin/commit qilinmasin.
