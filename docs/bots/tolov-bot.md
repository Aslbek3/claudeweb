# tolov-bot (To'lov bot) — @Sukunaro_bot

**Turi:** To'lov / oylik obuna kuzatuv boti
**PM2 nomi:** `tolov-bot`
**Joylashuvi:** `/root/vps/claudeweb/just/To'lov bot`
**Telegram:** [@Sukunaro_bot](https://t.me/Sukunaro_bot) — ko'rinadigan nomi: "Toʻloʻvlar"

> To'liq foydalanuvchi/admin qo'llanmasi loyihaning o'zida bor: [`just/To'lov bot/README.md`](../../just/To'lov%20bot/README.md). Bu fayl — markaziy hujjatlashtirish uchun qisqacha xulosa + boshqarish buyruqlari.

## Nima qiladi

Boshqa Telegram botlarning egalari (masalan yuqoridagi `kino-bot` kabi) o'z botini shu bot orqali **ro'yxatdan o'tkazadi**. Admin so'rovni ko'rib chiqib oylik to'lov summasini belgilaydi (yoki rad etadi). Shundan keyin bot egasi har oy ko'rsatilgan kartaga pul o'tkazib, chek skrinshotini yuboradi — admin ✅/❌ tugma bilan tasdiqlaydi/rad etadi. Tasdiqlansa, botning faol muddati avtomatik 1 oyga uzayadi. Muddat tugashiga 3 kun qolganda yoki o'tib ketganda foydalanuvchi va adminlarga avtomatik eslatma boradi (`reminders.py`, har 12 soatda tekshiriladi).

### Asosiy oqim
1. `/start` — ro'yxatdan o'tish, asosiy menyu.
2. **➕ Bot qo'shish** — nomi/username/tavsif kiritiladi, so'rov barcha adminlarga boradi.
3. Admin tasdiqlab, oylik summani belgilaydi (yoki admin o'zi to'g'ridan-to'g'ri tasdiqlangan holda qo'sha oladi).
4. **🤖 Mening botlarim** — holat (⏳ kutilmoqda / ✅ faol / ❌ rad etilgan), muddat va qolgan kunlar.
5. **💳 To'lov qilish** — bazadagi kartalar ko'rsatiladi, chek yuboriladi, admin tasdiqlaydi.
6. Muddat yaqinlashsa/o'tsa — avtomatik eslatma.

### Admin panel (`/admin`)
- **📊 Statistika** — foydalanuvchilar, botlar (kutilayotgan/faol/rad etilgan/muddati o'tgan), to'lovlar, shu oy/jami tushum.
- **🤖 Botlar** — ro'yxat + yangi bot admin tomonidan to'g'ridan-to'g'ri qo'shish.
- **💳 To'lovlar** — tasdiqlash/rad etish, **💳 Kartalar** — karta qo'shish/o'chirish (bir nechta karta bo'lishi mumkin).
- **📢 Xabar yuborish** — broadcast (oldindan ko'rib tasdiqlash bilan).

## Texnik tuzilma

```
To'lov bot/
├── bot.py           # kirish nuqtasi, dispatcher, eslatmalar fon vazifasi
├── config.py         # .env: BOT_TOKEN, ADMIN_IDS, CARD_NUMBER, CARD_HOLDER, DEFAULT_MONTHLY_FEE, DB_PATH
├── database.py         # SQLite: users / bots / payments / cards / tariffs
├── utils.py             # sana hisob-kitoblari, formatlash
├── states.py              # FSM holatlari
├── keyboards.py            # reply/inline klaviaturalar
├── reminders.py             # muddat eslatmalari (12 soatda bir)
├── bot.db                    # SQLite baza fayli
├── venv/                      # mustaqil Python virtualenv
└── handlers/
    ├── admin.py                # admin panel va tasdiqlash oqimi
    └── user.py                  # foydalanuvchi oqimi
```

## Baza sxemasi (`bot.db`)

| Jadval | Vazifasi |
|---|---|
| `users` | Ro'yxatdan o'tgan foydalanuvchilar |
| `bots` | Ro'yxatga olingan botlar (egasi, nomi, oylik summa, holati, muddati) |
| `payments` | To'lov tarixi (summa, chek file_id, holati, kim tasdiqlagan) |
| `cards` | Foydalanuvchiga ko'rsatiladigan kartalar ro'yxati |
| `tariffs` | Taklif etiladigan tarif summalari |

## Boshqarish

```bash
cd "/root/vps/claudeweb/just/To'lov bot"

# Holat / loglar
/usr/local/bin/pm2 describe tolov-bot
/usr/local/bin/pm2 logs tolov-bot --lines 50

# Oddiy qayta ishga tushirish
/usr/local/bin/pm2 restart tolov-bot --update-env

# To'liq 0'dan (restart hisoblagichi ham nollanadi)
/usr/local/bin/pm2 delete tolov-bot
/usr/local/bin/pm2 start bot.py --name tolov-bot --interpreter ./venv/bin/python3
/usr/local/bin/pm2 save

# Statistikani (bazani) nolga tushirish — DIQQAT: qaytarib bo'lmaydi, avval nusxa oling
cp bot.db "bot.db.backup_$(date +%Y%m%d_%H%M%S)"
python3 -c "
import sqlite3
conn = sqlite3.connect('bot.db')
cur = conn.cursor()
for t in ['users','bots','payments','cards','tariffs']:
    cur.execute(f'DELETE FROM {t}')
cur.execute('DELETE FROM sqlite_sequence')
conn.commit()
"
```

## Joriy holat (2026-08-14)

- Foydalanuvchilar: **0**
- Ro'yxatga olingan botlar: **0**
- To'lovlar: **0**
- Kartalar: **1** (`.env` dagi `CARD_NUMBER`/`CARD_HOLDER` dan avtomatik yaratiladi)
- Tariflar: **0**

> 2026-08-14'da foydalanuvchi so'rovi bilan barcha statistika (users/bots/payments/cards/tariffs) 0'ga tushirilgan va PM2 jarayoni to'liq qayta yaratilgan (restart hisoblagichi ham 0). Eski holatning zaxira nusxasi: `bot.db.backup_20260814_080736`.

## Holat — 2026-09-03: `billing_type` + umumiy obuna avto-stop qo'shildi (root sessiyasi orqali)

Foydalanuvchi Telegramdan (`@Sukunaro_bot`ga qarata) yangi kod zip'ini (`tolov-bot_2026-09-03_kengaytirilgan.zip`, sandbox'da tayyorlangan, VPS'da sinalmagan deb belgilangan) yubordi — **root sessiyasidan** ko'rib chiqilib, sinalib, joylashtirildi (claudeweb'ning o'z izolyatsiyasiga/ACL'iga tegilmadi, faqat `just/To'lov bot/` ichidagi fayllar yangilandi, xuddi shu CLAUDE.md'dagi "O'z uy papkasi ichidagi qo'shimcha loyihalar" qoidasiga ko'ra bu joy claudeweb domenida to'liq ochiq).

O'zgargan/yangi fayllar: `config.py`, `bot.py`, `database.py` (yangilandi), `subscription_lifecycle.py` (yangi).

**Nima qo'shildi:**
1. **`billing_type` ustuni** (`bots` jadvali, standart `'subscription'`, muqobili `'one_time'`) — `one_time` botlar (mas. admin-abadiy) `paid_until`/avto-stop tizimiga kirmaydi.
2. **Umumiy obuna avto-stop** (`subscription_lifecycle.py`, yangi fon vazifasi `subscription_loop`, `bot.py`ga ulandi) — avval faqat 48 soatlik trial botlar avtomatik to'xtatilardi; endi `billing_type='subscription'` bo'lgan, trial bo'lmagan, `pm2_name`ga ega BARCHA botlar uchun ham muddat o'tsa `pm2 stop` + admin/mijozga xabar. Sozlash: `SUBSCRIPTION_STOP_GRACE_DAYS` (standart 0), `SUBSCRIPTION_CHECK_INTERVAL_MINUTES` (standart 60). Qayta yoqish yangi kod talab qilmadi — mavjud `_reactivate_if_stopped()` (`handlers/admin.py`) trial'ga bog'liq emas edi.
3. **`database.link_existing_bot()`** — allaqachon ishlab turgan pm2 process'ni (mas. poster-01) qayta deploy qilmasdan bazaga bog'lash uchun. **Hali admin panelda UI ulanmagan** (faqat DB funksiyasi tayyor) — keyingi bosqich.

**Tekshirilgan (root sessiyasida, joylashtirishdan oldin va keyin):**
- Barcha o'zgargan fayl + import zanjiri `py_compile` va haqiqiy `import bot` bilan xatosiz (claudeweb venv, `sudo -u claudeweb` ostida).
- `billing_type` migratsiyasi (`ALTER TABLE`) production `bot.db`ning nusxasida oldindan sinaldi, keyin haqiqiy bazada ham xatosiz qo'llanildi (ustun qo'shildi, mavjud botlar `billing_type='subscription'` bilan qoldi, hech narsa yo'qolmadi).
- Joylashtirishdan oldin `bot.db`/`config.py`/`bot.py`/`database.py`ning zaxira nusxasi olindi: `backups/pre_update_20260903_130830/`.
- `pm2 restart tolov-bot --update-env` xatosiz, jarayon barqaror **online** (`unstable restarts: 0`, crash-loop yo'q), loglarda "Run polling for bot @Sukunaro_bot" tasdiqlandi, `pm2 save` bilan saqlandi.

**Hali qilinmagan (keyingi bosqich, O'ZGARISHLAR_2026-09-03.md faylida ham qayd etilgan):**
1. Click/Payme (va Paynet) webhook — hozircha faqat qo'lda chek tasdiqlash bor.
2. `link_existing_bot()` uchun admin panelda UI (yoki mavjud obunachilarni import qilish uchun bir martalik skript).
3. `products` jadvaliga `billing_type` maydoni qo'shish (hozir faqat `bots`da).
4. Real foydalanuvchi oqimi bilan production sinov (funksiyalar hali jonli trafik bilan ishlatilmagan, faqat statik/migratsiya darajasida tekshirilgan).

## Holat — 2026-09-03 (2): Mahsulot katalogiga tahrirlash qo'shildi

Admin so'roviga ko'ra 📦 Mahsulotlar bo'limiga **tahrirlash** imkoniyati qo'shildi — avval faqat qo'shish/o'chirish bor edi, narxni to'g'rilash uchun mahsulotni o'chirib qaytadan qo'shishga to'g'ri kelardi.

- **`database.py`:** yangi `update_product(product_id, *, name=None, description=None, monthly_fee=None)` — faqat berilgan maydonni yangilaydi.
- **`states.py`:** yangi `EditProduct` (name/description/fee) FSM guruhi.
- **`keyboards.py`:** `admin_product_detail_kb` ga "✏️ Nomi" / "✏️ Tavsifi" / "✏️ Narxi" tugmalari qo'shildi.
- **`handlers/admin.py`:** har bir maydon uchun alohida callback+FSM oqim (`editproductname:`/`editproductdesc:`/`editproductfee:`), "❌ Bekor qilish" global cancel ro'yxatiga ham qo'shildi.

Tekshirilgan: `py_compile`+`import bot` xatosiz; `update_product()` bazaning nusxasida (nom/tavsif/narx alohida-alohida) funksional sinaldi; joylashtirishdan oldin `bot.db` zaxira nusxasi olindi (`backups/pre_editproduct_<stamp>/`); `pm2 restart tolov-bot --update-env` xatosiz, jarayon barqaror **online** (`unstable restarts: 0`), `pm2 save` bilan saqlandi.

## Holat — 2026-09-03 (3): Tugma bosishlardagi sekinlik (ba'zan ~1 soniya) tuzatildi

Admin "tugma bosganda sekin javob keladi, ba'zan 1 soniyagacha" deb xabar berdi. Production error logidan (`tolov-bot-error.log`, 243 ta so'nggi "Duration" o'lchovi) statistikaga qaraldi: median 82ms (yaxshi), lekin **p99 ≈ 577ms, maksimum ≈ 1047ms** — ya'ni ko'pchilik tugma tez, lekin vaqti-vaqti bilan (ayniqsa **yozuvchi**, ya'ni tasdiqlash/qo'shish/tahrirlash kabi) tugmalarda sezilarli sakrash bor edi. Ikkita real sabab topildi va tuzatildi:

1. **`database.py`, `get_db()`** — `PRAGMA synchronous` standart bo'yicha `FULL` (2) edi: WAL rejimida bu har bir yozuv (commit)da 2 marta diskka `fsync` qilishni talab qiladi — eng xavfsiz, lekin eng sekin variant. WAL bilan rasmiy tavsiya qilingan `NORMAL` (1)ga o'zgartirildi — ma'lumot xavfsizligi WAL orqali baribir ta'minlanadi (faqat butun OS keskin yiqilib tushgan taqdirdagina so'nggi bir necha commit yo'qolishi mumkin, korruptsiya emas), lekin fsync yukini sezilarli kamaytiradi.
2. **`backup.py`, `backup_db()`** — `shutil.copy2()` (baza faylini nusxalash) va eski nusxalarni o'chirish **sinxron/bloklovchi** chaqiriq edi, asyncio event loop'ning o'zida to'g'ridan-to'g'ri ishlatilardi — shu paytda kelgan HAR QANDAY boshqa tugma bosish/xabar navbatda kutib qolardi (bot bitta process, bitta event loop). Bu ishga tushganda (start'da va har `BACKUP_INTERVAL_HOURS`da) darhol keladigan tugma bosishlari aynan shu sababdan sekinlashishi mumkin edi. `asyncio.to_thread(...)` bilan o'raldi — endi alohida thread'da bajariladi, event loop'ni bloklamaydi.

Tekshirilgan: `py_compile`+`import bot` xatosiz; bazaning nusxasida `PRAGMA synchronous` haqiqatan `1` (NORMAL) qaytarishi va `backup_db()` xatosiz/tezkor (~25ms) ishlashi tasdiqlandi; production `bot.db`+kod zaxirasi olingandan keyin (`backups/pre_perffix_<stamp>/`) joylashtirildi, `pm2 restart tolov-bot --update-env` xatosiz, jarayon barqaror **online**, `pm2 save` bilan saqlandi.

**Eslatma:** qolgan tomonlama sekinlik (Telegram API'gacha bo'lgan tarmoq kechikishi, ba'zi handlerlarda ketma-ket bir necha `answer()`/`edit_reply_markup()` chaqiruvi) hali ham bor va butunlay yo'qolmasligi mumkin — bular kamroq ta'sirli, lekin agar sekinlik davom etsa, keyingi qadam sifatida shu handlerlarni ham ko'rib chiqish mumkin.

## Eslatmalar

- `.env` faylida `BOT_TOKEN`, `ADMIN_IDS`, `CARD_NUMBER`, `CARD_HOLDER` bor — hech qachon oshkor qilinmasin/commit qilinmasin.
- Bazani nolga tushirishdan oldin **har doim** zaxira nusxa oling (`cp bot.db bot.db.backup_...`) — bu qaytarib bo'lmas amal.
