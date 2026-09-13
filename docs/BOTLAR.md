# VPS botlari — asosiy ro'yxat

Bu fayl `claudeweb` foydalanuvchisi ostida (o'zining alohida PM2 daemonida) ishlab turgan barcha Telegram botlarining markaziy ro'yxati, turi bo'yicha guruhlangan. Har bir bot uchun batafsil hujjat — `docs/bots/<bot-nomi>.md`.

> Umumiy boshqaruv, izolyatsiya va infratuzilma haqida — loyihaning ildizidagi `CLAUDE.md`ga qarang. Bu fayl faqat botlarning o'zi (nima qilishi, qanday tuzilgani) haqida.

## Tezkor holat tekshiruvi

```bash
/usr/local/bin/pm2 list
```

---

## 1-tur: Kino/Anime tarqatuvchi botlar

Foydalanuvchi kino/anime kodini yuboradi → bot faylni (video/hujjat/rasm) qaytaradi. Majburiy obuna (kanal/guruh), avtopost (rejalashtirilgan e'lonlar), statistika va admin panel bilan.

| PM2 nomi | Telegram | Joylashuvi | Baza | Hujjat |
|---|---|---|---|---|
| `kino-bot` | [@TarjimaKinoKodlari_bot](https://t.me/TarjimaKinoKodlari_bot) | `workspace/kino-bot-pro` | `kino_bot.db` (SQLite) | [kino-bot-pro.md](bots/kino-bot-pro.md) |
| `kino-bot-2` | [@avtopost4_bot](https://t.me/avtopost4_bot) | `workspace/kino-bot-2` | `kino_bot.db` (SQLite) | [kino-bot-2.md](bots/kino-bot-2.md) |

Ikkalasi ham **bir xil kodga** asoslangan (bitta loyihaning ikkita mustaqil nusxasi/instansi) — faqat `.env` dagi `BOT_TOKEN` (demak, Telegram bot hisobi) va o'zining baza fayli farq qiladi. Kod farqi yo'q (`diff` bilan tekshirilgan).

## 2-tur: To'lov / obuna kuzatuv boti

Boshqa botlarning egalari o'z botini ro'yxatdan o'tkazadi, admin oylik to'lov summasini belgilaydi, foydalanuvchi har oy kartaga pul o'tkazib chek yuboradi, admin tasdiqlaydi — muddat avtomatik uzayadi, tugashiga yaqin eslatma keladi.

| PM2 nomi | Telegram | Joylashuvi | Baza | Hujjat |
|---|---|---|---|---|
| `tolov-bot` | [@Sukunaro_bot](https://t.me/Sukunaro_bot) | `just/To'lov bot` | `bot.db` (SQLite) | [tolov-bot.md](bots/tolov-bot.md) |

---

## Umumiy texnologik stek (barcha botlar)

- **Til/runtime:** Python 3 + [aiogram 3.x](https://docs.aiogram.dev/) (async Telegram Bot API framework), har birining o'z `venv/` papkasi bor.
- **Baza:** SQLite (bot papkasi ichida, tashqi DB server yo'q).
- **Ishga tushirish:** har biri PM2 orqali `--interpreter <bot-papkasi>/venv/bin/python3` bilan, `bot.py` skripti.
- **Sozlamalar:** `.env` fayl (`BOT_TOKEN`, `ADMIN_IDS`, va h.k.) — repo'ga kirmaydi (`.gitignore`).

## Boshqarish (barcha botlar uchun umumiy)

```bash
# Holat
/usr/local/bin/pm2 list
/usr/local/bin/pm2 describe <pm2-nomi>

# Loglar
/usr/local/bin/pm2 logs <pm2-nomi> --lines 50

# Qayta ishga tushirish (holatni/hisoblagichni saqlab)
/usr/local/bin/pm2 restart <pm2-nomi> --update-env

# To'liq 0'dan qayta yaratish (restart hisoblagichini ham nolga tushiradi)
/usr/local/bin/pm2 delete <pm2-nomi>
cd "<bot-papkasi>" && /usr/local/bin/pm2 start bot.py --name <pm2-nomi> --interpreter ./venv/bin/python3
/usr/local/bin/pm2 save

# O'zgarishlarni doimiy saqlash (reboot'dan keyin ham tiklanishi uchun)
/usr/local/bin/pm2 save
```

> Root/SSH shart emas — bularning barchasi claudeweb sessiyasining o'zidan (shu veb-interfeys orqali) bajariladi.

## Joriy holat (oxirgi tekshiruv: 2026-08-14)

| Bot | Foydalanuvchilar | Kontent/yozuvlar | Eslatma |
|---|---|---|---|
| kino-bot (@TarjimaKinoKodlari_bot) | 30 | 4 ta kino/anime, 1 majburiy kanal, 2 admin | — |
| kino-bot-2 (@avtopost4_bot) | 78 | 2 ta kino/anime, 2 majburiy kanal, 1 admin | — |
| tolov-bot (@Sukunaro_bot) | 0 | 0 ta bot, 0 to'lov, 1 karta | 2026-08-14'da statistikasi 0'dan qaytadan boshlangan (zaxira: `bot.db.backup_20260814_080736`) |
