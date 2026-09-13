# claude-code-web

Telefon/brauzer uchun Claude Code veb-interfeysi. Express + WebSocket server, `@anthropic-ai/claude-agent-sdk` orqali ishlaydi.

## Manzil va kirish

- **URL:** https://claudeuz.duckdns.org (duckdns domeni, 161.97.105.98'ga yo'naltirilgan)
- **Parol:** `.env` dagi `APP_PASSWORD`
- nginx (`/etc/nginx/sites-available/claudeuz`) + certbot HTTPS orqali `127.0.0.1:3210`ga proxy qiladi. Server o'zi faqat `127.0.0.1:3210`da tinglaydi (HOST=127.0.0.1, TRUST_PROXY=1) — tashqaridan to'g'ridan-to'g'ri portga kirish yo'q.

## Izolyatsiya (MUHIM — buzmang)

Bu loyiha VPS'dagi boshqa hech qanday narsaga (botlar, `.env` fayllar, admin Aslbekning shaxsiy Claude sessiyasi) tegmasligi va aloqasi bo'lmasligi kerak:

- **Alohida tizim foydalanuvchisi:** `claudeweb` (non-root, uid 1001), uy papkasi = shu loyiha papkasining o'zi (`/root/vps/claudeweb`).
- **ACL bilan yopilgan yo'l:** `/root` (710) va `/root/vps` (755) papkalarida `claudeweb` uchun `setfacl -m u:claudeweb:--x` bilan faqat "o'tish" huquqi bor, `/root` va `/root/vps` ichidagi HAR BIR boshqa element (fayl va papka) uchun esa aniq `setfacl -m u:claudeweb:0` (taqiq) qo'yilgan. Natija: `claudeweb` faqat `/root/vps/claudeweb/` ga yeta oladi, hech narsani list/read qila olmaydi.
  - Tekshirish: `sudo -u claudeweb ls /root/vps/projects` → "Permission denied" bo'lishi kerak.
  - **Diqqat:** kelajakda `/root` yoki `/root/vps` ichiga yangi fayl/papka qo'shilsa, ularga ham xuddi shunday `setfacl -m u:claudeweb:0` qo'yish kerak — aks holda o'sha yangi element `claudeweb`ga ochiq (world-readable, 755/644) bo'lib qolishi mumkin.
- **Mustaqil Node/PM2:** `/root/.nvm`, `/root/.npm-global` `claudeweb`ga yopiq bo'lgani uchun tizim darajasida (hammaga ochiq) infratuzilma o'rnatilgan:
  - `/usr/local/bin/node22` — Node v22.23.1 (nusxa, `/root/.nvm`dan ko'chirilgan)
  - `/usr/local/bin/pm2` → `/usr/local/lib/node_modules/pm2` (global npm o'rnatma, `npm install -g pm2 --prefix /usr/local`)
- **Mustaqil PM2 daemon:** root'ning PM2'sidan butunlay alohida. `PM2_HOME=/root/vps/claudeweb/.pm2` (HOME shu bo'lgani uchun standart joy).
- **Autentifikatsiya izolyatsiyasi:** `claude auth login` `claudeweb` nomidan bajarilgan — tokenlar `/root/vps/claudeweb/.claude/.credentials.json`da, admin Aslbekning shaxsiy `/root/.claude/.credentials.json`iga HECH aloqasi yo'q. **Alohida (ikkinchi) Claude hisobi ulangan — Aslbekning shaxsiy Pro/Max obunasi bilan bog'liq emas.**

## O'z uy papkasi ichidagi qo'shimcha loyihalar (MUHIM — ACL bilan aralashtirmang)

Yuqoridagi ACL izolyatsiyasi **faqat** `/root` va `/root/vps` ichidagi **sibling** (o'zidan tashqari) fayl/papkalarga tegishli. `/root/vps/claudeweb/` ning **o'zi ichidagi** HAR QANDAY papka/fayl — nomidan qat'iy nazar, `.env`dagi `PROJECT_DIR=./workspace`dan tashqarida bo'lsa ham — claudeweb uchun to'liq ochiq, hech qanday ACL taqig'i yo'q.

Hozirgi ma'lum holat: bir nechta Telegram bot loyihasi `workspace/` bilan bir qatorda `just/` papkasida ham joylashgan, va ularning barchasi claudeweb'ning **o'z alohida PM2 daemonida** ishlaydi (`sudo -u claudeweb pm2 list` orqali ko'rinadi):

| PM2 nomi | Joylashuvi |
|---|---|
| `kino-bot` | `workspace/kino-bot-pro` |
| `kino-bot-2` | `workspace/kino-bot-2` |
| `kino-uploader` | `workspace/kino-uploader` |
| `tolov-bot` | `just/To'lov bot` |
| `vip-chopar` | `workspace/Vip chopar` |
| `kanal-01` | `Post bot 10` (uy papkasining tepasida, `workspace`/`just`dan tashqarida, lekin baribir claudeweb uy papkasi ICHIDA — to'liq ochiq) |
| `savdo-hisob` | `savdo-hisob` (uy papkasining tepasida — Telegram bot emas, mustaqil Node/Express veb-ilova, lokal `127.0.0.1:3212`da; foydalanuvchida root/SSH yo'qligi sababli nginx/domen o'rniga shu serverning o'zida `/savdo/` proksisi ishlatiladi — 2026-08-25'da faollashtirildi, `https://claudeuz.duckdns.org/savdo/` orqali ochiladi; tafsilot uchun `savdo-hisob/CLAUDE.md`ning "2026-08-24/25: /savdo/ proksi" bo'limiga qarang) |
| `polat` | `polat` (uy papkasining tepasida — Telegram bot emas, mustaqil Node/Express veb-ilova, lokal `127.0.0.1:3213`da; Po'lat restorani uchun afitsiant/oshpaz/dastavkachi/admin buyurtma-hisob tizimi + mijozlar uchun login shart bo'lmagan landing/buyurtma (savat+checkout) sahifasi; **`https://claudeuz.duckdns.org/polat/` proksisi olib tashlangan** — endi o'zining alohida subdomeni **`https://polatuz.duckdns.org/`** orqali ochiladi (o'z nginx server bloki + Certbot sertifikati); tafsilot uchun `polat/CLAUDE.md`ga qarang) |

Bularning har biri `sudo -u claudeweb -H bash -lc '/usr/local/bin/pm2 restart <nom>'` bilan claudeweb sessiyasining **o'zi** boshqara oladi — root/SSH aralashuvi shart emas.

**Qoida:** bot/loyiha nomi tanish bo'lmasa, avval `pm2 list` bilan tekshiring. Agar u shu ro'yxatda (claudeweb'ning o'z PM2'sida) chiqsa — demak u uy papkasi ichida va to'liq boshqarish mumkin. Faqat `pm2 list`da ko'rinmagan / boshqa user (masalan `root`) ostidagi jarayonlar chindan ham tashqi va claudeweb uchun yopiq.

**Chegara qanchalik ishonchli — bu taxmin emas, sinalgan haqiqat:** 2026-08-13'da root sifatida 3 xil hujum vektori maxsus sinaldi — (1) absolyut yo'l bilan tashqi faylni o'qish (`cat /root/vps/projects/.../*.env`), (2) o'z papkasi ichidan nisbiy yo'l bilan chiqishga urinish (`cd workspace && cat ../../projects/...`), (3) tashqi papkalarni ro'yxatlash (`/root`, `/root/vps`, boshqa loyiha papkalari, `/root/.claude`). **Uchalasi ham `Permission denied`** — hatto `/root` va `/root/vps`ning o'zini ham ro'yxatlab bo'lmaydi (`claudeweb` uchun faqat `--x` "o'tish" huquqi bor, `r` yo'q). Demak chegara OS/ACL darajasida jismonan bloklangan, AI'ning "to'g'ri tushunishi"ga bog'liq emas.

**Xulq-atvor qoidasi:** `/root/vps/claudeweb/` ichida — hech ikkilanmasdan, to'liq erkin harakat qiling (fayllar, PM2 botlar, hammasi). Undan tashqarida biror narsa so'ralsa (boshqa VPS loyihasi, boshqa foydalanuvchi jarayoni, `/root` ostidagi narsa) — ACL buni jismonan bloklaydi, shuning uchun urinib ko'rish shart emas: darhol va ishonch bilan "bu mendan tashqarida, ruxsatim yo'q" deb javob bering, keyin kerak bo'lsa foydalanuvchini to'g'ridan-to'g'ri VPS/SSH sessiyasiga yo'naltiring.

**2026-08-13 voqeasi:** claudeweb sessiyasi foydalanuvchining "to'lov botni qayta ishga tushir" so'rovini rad etib, buni ACL bilan yopilgan tashqi loyiha deb noto'g'ri xulosa chiqargan edi — aslida bot `just/To'lov bot/` da, to'liq claudeweb domenida edi. Bu holatda haqiqatan tashqi ekanini tasdiqlashdan oldin har doim avval `pm2 list` bilan tekshiring.

## Boshqarish

```bash
# Holatni ko'rish
sudo -u claudeweb -H bash -lc '/usr/local/bin/pm2 list'
sudo -u claudeweb -H bash -lc '/usr/local/bin/pm2 logs claude-code-web --lines 50'

# Qayta ishga tushirish
sudo -u claudeweb -H bash -lc '/usr/local/bin/pm2 restart claude-code-web --update-env'

# Saqlash (kod o'zgargandan keyin)
sudo -u claudeweb -H bash -lc '/usr/local/bin/pm2 save'
```

`chown -R claudeweb:claudeweb /root/vps/claudeweb` — root sifatida fayl tahrirlagandan keyin egalikni tiklashni unutmang.

## Reboot'dan keyin

Alohida systemd xizmati bor: `pm2-claudeweb.service` (`systemctl status pm2-claudeweb`), `enable` qilingan — VPS qayta yuklanganda `pm2 resurrect` orqali avtomatik ko'tariladi. Root'ning `pm2-root.service`idan mustaqil.

## Qo'shimcha funksiyalar (2026-08-08)

- **Chatni tozalash** — topbar `[clear]` tugmasi, WS `clear_chat` xabari orqali (`sessionManager.resetSession`) — Claude bilan suhbat butunlay yangidan boshlanadi.
- **Yangi papka yaratish** — Fayllar panelidagi `[+dir]`, `POST /api/files/mkdir` (`fileApi.mkdir`).
- **Fayl yuklash** — Fayllar panelidagi `[up]`, `POST /api/files/upload` (multer, 100MB limit, `node_modules`ga `npm install multer` bilan qo'shilgan).
- **Fayl yuklab olish** — har bir fayl qatorida `[dl]` va fayl ko'ruvchisida `[dl]`, `GET /api/file/download`.

## Saytdan Claude hisobini ulash/almashtirish

SSH/terminalga kirish shart emas — `/auth.html` sahifasida (yoki bosh sahifadagi `[auth]` tugmasi orqali) "Boshlash" bosiladi, chiqqan havola brauzerda ochilib kerakli Claude hisobi bilan tasdiqlanadi, natijada chiqadigan kod saytga qaytarib kiritiladi. Buni `server/authManager.js` (`script -qec` orqali haqiqiy pty ochib, `claude auth login --claudeai`ni boshqaradi — oddiy pipe orqali kod qabul qilinmaydi, real terminal SHART) va `server/index.js`dagi `/api/auth/*` route'lari ta'minlaydi.

## Ruxsat rejimlari (manual / plan / avto)

Topbar composer ustidagi `[manual]`/`[plan]`/`[avto]` tugmasi bosilganda aylanadi (`public/app.js`), WS orqali `set_permission_mode` yuboradi, server `sessionManager.js`da SDK'ning `setPermissionMode()`sini chaqiradi.

- **manual** (`default`) — hamma narsa (Write/Edit/Bash/...) ruxsat so'raydi, faqat Read/Grep/Glob/TodoWrite avtomatik.
- **plan** — hech narsa bajarilmaydi, faqat reja tuziladi (`ExitPlanMode` avto-ruxsat, lekin haqiqiy amal baribir so'raydi).
- **avto** (`acceptEdits`) — Write/Edit/NotebookEdit avto-ruxsat; **Bash ham** — `DANGEROUS_BASH_PATTERNS` denylist'iga (`sessionManager.js`, `isDangerousBash()`) tushmagan har qanday buyruq so'rovsiz bajariladi (zip/unzip, mkdir, cp/mv, git status/diff/log/add/commit, npm/pip install va hokazo). Denylist: `sudo`, `rm -rf`/`-r -f` (alohida flag bo'lsa ham), `dd if=`, `mkfs`, fork bomb, `chmod 777`, `chown -R`, `curl|wget ... | sh/bash`, `git push --force`, `shutdown/reboot`, `killall`, `iptables/ufw/firewall-cmd`, `> /etc/...`, `crontab -r`, `--no-preserve-root` — bular avto rejimda ham ruxsat kartochkasi chiqaradi. Bu **denylist** (allowlist emas) — yangi xavfli pattern topilsa shu massivga regex qo'shiladi.
- `bypassPermissions` HECH QACHON ishlatilmaydi — avto rejim ham SDK darajasidagi haqiqiy tekshiruv ostida qoladi, faqat denylist orqali filtrlanadi.

## Holat — 2026-08-12: rootweb'dagi UI yangilanishlari ko'chirildi, sayt tayyor

`/root/vps/rootweb`da qilingan 3 ta commit (Claude.ai uslubidagi qayta dizayn + fayl biriktirish + AskUserQuestion, Windows→Linux control-bayt regex tuzatishi, fayllar panelida papka/fayl ikonkalari) shu kuni claudeweb'ga ham qo'lda ko'chirildi — **izolyatsiya (ACL, alohida foydalanuvchi/PM2, `permissionMode='default'`, `DANGEROUS_BASH_PATTERNS`) va "qurilmalar" (devices) tabi hech tegilmadi**, faqat UI/feature kodi (chat dizayni, `app.js`/`style.css`/`index.html`/`server/index.js`/`server/sessionManager.js`) yangilandi.

- rootweb'ning `resolveBrowseRoot()`/`root=` query-parametri, `DELETE /api/file`/`rename` marshrutlari, `..` qatori — claudeweb'ga **ataylab ko'chirilmadi** (rootweb-only, path-sandbox chegarasidan chiqish imkonini beradi, faqat root ilova uchun xavfsiz deb hisoblanadi).
- claudeweb'ning `server/index.js`sida ham xuddi shu Windows-korruptsiya bagi (fayl-nomi tozalash regexida haqiqiy NUL/control baytlar) mustaqil ravishda mavjud edi — bu ham tuzatildi.
- Composer HTML strukturasi (`modeBtn` joylashuvi) va CSS'dagi eski `.mode-row`/`.mode-pill` bloki claudeweb'ning o'ziga xos joylashuviga moslab qo'lda tahrirlandi (rootweb'nikidan farqli edi).
- Zaxira: `/root/vps/claudeweb_backup_20260812/` (tahrirlangan 8 ta faylning originali).
- Tekshirilgan: `pm2 restart claude-code-web --update-env` (claudeweb'ning o'z alohida PM2 daemoni orqali) xatosiz, HTTPS login oqimi (`/api/login` → bosh sahifa → `app.js`/`style.css`) 200 qaytardi, yangi kod (`file-icon`, `question_request`, `pendingAttachments`) xizmat qilinayotgani tasdiqlandi, fayl egaligi (`claudeweb:claudeweb`) va ACL (`sudo -u claudeweb ls /root/vps/projects` → Permission denied) o'zgarmagani tasdiqlandi.
- Vizual (brauzerda ko'rinish) tekshiruv Chrome kengaytmasi ulanmagani sababli hali qilinmagan.

## Holat — 2026-08-16: `tolov-bot`ga (just/To'lov bot) yangi admin funksiyalar qo'shildi

Foydalanuvchi so'rovi bilan `just/To'lov bot/` (PM2'da `tolov-bot`) Python/aiogram botiga 6 ta yangi funksiya qo'shildi va production'da tekshirilib ishga tushirildi. **claudeweb'ning o'z izolyatsiyasiga (ACL, alohida user/PM2) hech tegilmadi** — bu faqat `just/` ichidagi bitta loyihaga oid o'zgarish, chunki `just/` claudeweb uy papkasi ichida (to'liq ochiq, CLAUDE.mdning "O'z uy papkasi ichidagi qo'shimcha loyihalar" bo'limiga qarang).

Qo'shilganlar:
- **Avtomatik backup** (`backup.py`, yangi) — ishga tushganda va har `BACKUP_INTERVAL_HOURS` (standart 24) soatda `backups/` papkasiga WAL-checkpoint qilib bazani nusxalaydi, eskilarini (`BACKUP_KEEP`, standart 14) o'chiradi. Admin panelda qo'lda ham olish mumkin (**📦 Backup** tugmasi).
- **CSV eksport** (`export.py`, yangi) — admin panelda **📤 Eksport** tugmasi foydalanuvchilar/botlar/to'lovlar ro'yxatini CSV (Excel'da ochiladi) qilib yuboradi.
- **Global xato ushlash + flood-control** (`middlewares.py` yangi + `bot.py`dagi `@dp.error()`) — handlerdagi ushlanmagan xatolik endi botni yiqitmaydi, foydalanuvchiga umumiy xabar ko'rsatiladi va logga yoziladi; bitta foydalanuvchidan juda tez-tez kelayotgan so'rovlar (`THROTTLE_SECONDS`, standart 0.6s) e'tiborsiz qoldiriladi.
- **Segmentlangan broadcast** — **📢 Xabar yuborish** endi avval guruh so'raydi: barcha foydalanuvchilar / qarzdor (muddati o'tgan) bot egalari / faol bot egalari.
- **Foydalanuvchini o'chirish** — 👥 Foydalanuvchilar bo'limida, faqat **botsiz** foydalanuvchi tafsilotida **🗑 O'chirish** tugmasi chiqadi (tasdiqlash bilan) — boti bor foydalanuvchini tarixni yo'qotmaslik uchun o'chirib bo'lmaydi.
- **Qo'lda to'lov qo'shish** — faol bot tafsilotida **💰 To'lov qo'shish** tugmasi bilan admin chek/skrinshotsiz (masalan naqd qabul qilingan) to'lovni qayd etib, muddatni (`paid_until`) darhol uzaytiradi.

O'zgargan asosiy fayllar: `bot.py`, `config.py`, `database.py`, `keyboards.py`, `states.py`, `handlers/admin.py`. Yangi fayllar: `backup.py`, `export.py`, `middlewares.py`. `README.md` shu funksiyalar bilan yangilandi.

Tekshirilgan: `py_compile` va import darajasida barcha fayl xatosiz; `backup_db()`/`export.*`/`delete_user()`/`add_manual_payment()` real bazaga qarshi (vaqtinchalik test yozuvlar bilan, keyin tozalab) funksional sinaldi; `pm2 restart tolov-bot --update-env` xatosiz, jarayon barqaror **online** (restart soni ortmadi, crash-loop yo'q), `pm2 save` bilan saqlandi; sessiya davomida haqiqiy admin Telegram orqali botga ulanib bir nechta buyruqni muvaffaqiyatli yuborgani logda tasdiqlandi.

## Botlar haqida batafsil (har birining o'z README/CLAUDE.mdidan birlashtirilgan)

Quyida uy papkasi ichidagi (`workspace/`, `just/`, va tepasidagi `Post bot 10`/`savdo-hisob`) har bir loyihaning o'z hujjatidan (README.md/CLAUDE.md) qisqartirilgan mazmuni — asl fayllar ham joyida qoladi, bu yerda faqat tezkor umumiy ko'rinish uchun. (`savdo-hisob` Telegram bot emas, mustaqil veb-ilova — pastda alohida belgilangan.)

### `kino-bot` (`workspace/kino-bot-pro`) va `kino-bot-2` (`workspace/kino-bot-2`)

Python/aiogram 3 bot juftligi — kino/anime kodini raqam bilan tarqatish. Foydalanuvchi kod yuboradi (yoki deep-link `?start=k123`/`a123`) va video/hujjat/rasm qaytariladi, majburiy kanal-a'zolik tekshiruvi orqali (ochiq, yopiq/join-request va tasdiqlab bo'lmaydigan "boshqa" havolalar). Admin FSM orqali kontent yuklaydi/o'chiradi, majburiy va reklama kanallarini boshqaradi, 20 tagacha kanalga avto-post rejalashtiradi, foydalanuvchilarga broadcast (forward yoki copy) yuboradi, foydalanuvchilarni bloklaydi, kontent-himoyani (forward/save taqig'i) yoqadi/o'chiradi, qo'shimcha adminlar boshqaradi. Bitta SQLite baza (`users`, `movies`, `channels`, `post_channels`, `settings`, `scheduled_posts`, `admins`, `preset_times`, `member_events`, `join_requests` jadvallari).

`kino-bot` (`workspace/kino-bot-pro`) va `kino-bot-2` (`workspace/kino-bot-2`) — kod bayt-baytiga bir xil (`diff -rq` bilan tasdiqlangan), ikkitasi mustaqil deploy: har birining o'z `.env` (`BOT_TOKEN`/`ADMIN_IDS`) va o'z SQLite bazasi, alohida PM2 jarayoni sifatida ishlaydi. To'liq hujjat: `workspace/kino-bot-pro/CLAUDE.md` (to'liq fayl/handler/baza tafsiloti) va `workspace/kino-bot-2/CLAUDE.md` (qisqa, birinchisiga havola beradi).

### `kino-uploader` (`workspace/kino-uploader`) — 1 video → 5 kino botga avtomatik tarqatish

Admin (barcha 5 ta kino botda admin bo'lgan shaxsiy Telegram raqami) videoni boshqaruv botga (@Birlashtrbot) yuborsa, tizim uni har bir nishon botning `/admin` → "➕ Kino qo'shish" jarayonini avtomatik takrorlab, hammasiga tarqatadi — qo'lda 5 marta bir xil ishni qilish shart emas. Ikki qismli (`vip-chopar` bilan bir xil naqsh):

- **Userbot** (Telethon, MTProto) — `distributor.py`: boshqaruv bot video kelgan xabarni `forward_message()` bilan (bayt yuklamasdan) userbotga forward qiladi, userbot esa MTProto orqali (Bot API'ning 20MB `getFile` chegarasisiz) yuklab, bitta yuklamani (`upload_file()`) barcha 5 nishonga qayta ishlatib yuboradi — shu sabab tarqatish tez.
- **Boshqaruv bot** (`bot.py`, aiogram) — nishonlar ro'yxati (`targets.json`, har biri o'z `steps` JSON-shablonига ega), tarix (`history.json`), foydalanilgan kodlar (`used_codes.json`), "🧮 Kod skanerlash" (0-9999 kodlarni nishon botlardan avtomatik tekshirish).
- **Bir martalik login:** `login.py request` → `login.py signin <KOD> [2FA]` → `kinouploader.session`.
- **`.env` kalitlari:** `API_ID`/`API_HASH`, `PHONE_NUMBER`, `BOT_TOKEN`, `ADMIN_IDS`, `STEP_TIMEOUT_SECONDS`/`TOTAL_TIMEOUT_SECONDS`/`DISTRIBUTE_DELAY_SECONDS`, `SCAN_DELAY_SECONDS`/`SCAN_PAUSE_EVERY`/`SCAN_PAUSE_SECONDS`, `CODE_NOT_FOUND_PHRASE`, `AUTO_UPLOAD_DELAY_SECONDS`, `REPORT_CHANNEL_ID`.
- **2026-09-06'dan:** **📡 Kanal** (manba kanal ulash/boshqarish, `channels.json`) + **🚀 Avto video yuklash** (ulangan kanaldan ko'p videoni admin belgilagan sonda ketma-ket avtomatik olib tarqatadi, har video uchun manba/nishon havolalari va ixtiyoriy alohida hisobot-kanaliga post) — tafsilot pastdagi "Holat — 2026-09-06" bo'limida va `workspace/kino-uploader/CLAUDE.md`da.
- To'liq tafsilot: `workspace/kino-uploader/README.md` va `workspace/kino-uploader/CLAUDE.md`. PM2 nomi: `kino-uploader`.

### `kanal-01` (`Post bot 10`) — kanalga avto-post/rejalashtiruvchi bot

2026-08-18'da yaratilgan/ishga tushirilgan aiogram 3 + MongoDB (Motor) + APScheduler bot. Admin kontent (matn/rasm/video/GIF) + "kino kodi" yuklaydi, FSM-master orqali maqsad kanal(lar) va yuborish vaqti/sanasini tanlaydi — post MongoDB'ga (`posts` jamlanmasi, `pending` holat) navbatga qo'yiladi va yashirin "navbat kanali"da oldindan ko'rsatiladi. Har daqiqada ishlaydigan scheduler joyi vaqti kelgan postlarni maqsad kanal(lar)ga yuboradi (kod asosida shakllangan "🎬 Kino ko'rish" deep-link tugmasi bilan), so'ng `sent`/`failed` deb belgilaydi. Admin bo'lmagan foydalanuvchiga o'z bot nusxasini sotib olish haqida reklama xabari ko'rsatiladi — kod bir nechta alohida tokenli bot nusxasi sifatida bitta umumiy MongoDB baza (`avtopost_db`) ustida ishlashga mo'ljallangan ko'rinadi (har biri `bots` jamlanmasida o'z `@username`i bilan, o'z admin/kanal/avto-vaqt/doimiy matn sozlamalari bilan).

`config.py`da (`.env` emas) `BOT_TOKEN`, `ADMINS`, `MONGO_URL`, `DB_NAME`, `TIMEZONE`, `QUEUE_CHANNEL_ID` — maxfiy qiymatlar to'g'ridan-to'g'ri faylda (CLAUDE.mdda ko'rsatilmagan). To'liq fayl/funksiya tafsiloti: `Post bot 10/CLAUDE.md`. PM2 nomi: `kanal-01`, ishga tushirish `./venv/bin/python3` interpretatori bilan.

### `tolov-bot` (`just/To'lov bot`) — oylik obuna/to'lov boti + 48 soatlik avto-deploy sinov

Telegram bot: foydalanuvchilar o'z botlari haqida ma'lumot qo'shadi, admin so'rovni ko'rib tasdiqlaydi/rad etadi va oylik to'lov summasini belgilaydi; foydalanuvchi har oy shu bot uchun to'lov qiladi (kartaga o'tkazib chek skrinshotini yuboradi), admin ✅/❌ bilan tasdiqlaydi/rad etadi — tasdiqlansa muddat avtomatik 1 oyga uzayadi. Muddat tugashiga 3 kun qolganda/tugagach avtomatik eslatma boradi.

2026-08-19'da qo'shilgan: **📦 Botlar katalogi** — admin (bot ichidan, 📦 Mahsulotlar bo'limida) tayyor bot xizmatlarini (hozircha `kino-bot-pro` shabloni) nom+tavsif+narx bilan katalogga qo'shadi; oddiy foydalanuvchi shundan birini tanlab, @BotFather tokenini yuborsa, tizim **avtomatik ravishda** shu shablon kodining nusxasini (`just/To'lov bot/deployed/<pm2-nom>/`) yaratib, foydalanuvchi tokeni bilan sozlab, **claudeweb'ning o'z PM2 daemonida** ishga tushiradi (`deploy.py`, `subprocess` orqali `pm2 start/stop/delete`; shablon manba kodining umumiy `venv`sidan interpretator sifatida to'g'ridan-to'g'ri foydalaniladi — har safar alohida virtual muhit o'rnatilmaydi). 48 soatlik bepul sinov boshlanadi: 24 soatdan keyin to'lov eslatmasi, 48 soatdan keyin (hali to'lanmagan bo'lsa) `pm2 stop` bilan avtomatik to'xtatish, to'lov qilingach (chek tasdiqlansa yoki admin qo'lda qo'shsa) avtomatik `pm2 start` bilan qayta yoqish — nazorat `trial_lifecycle.py`dagi fon vazifasi orqali har 15 daqiqada tekshiriladi. Faqat token bilan sozlanadigan, o'z ichiga bazasi bilan mustaqil ishlaydigan shablonlar shu oqimga mos (Telethon-userbot login talab qiladigan `vip-chopar`/`kino-uploader` mos emas, ular alohida qo'lda sozlanadi).

- **Sozlash:** `.env` — `BOT_TOKEN`, `ADMIN_IDS` (vergul bilan, bir nechta admin), `CARD_NUMBER`/`CARD_HOLDER` (boshlang'ich karta), `DEFAULT_MONTHLY_FEE`, `DB_PATH` (SQLite). Ishga tushirish: `python bot.py`.
- **Admin panel** (`/admin` yoki **🛠 Boshqarish**): 📊 Statistika, 👥 Foydalanuvchilar (botsiz foydalanuvchini 🗑 o'chirish mumkin), 🤖 Botlar (🆕 yangi bot qo'lda qo'shish, faol botga 💰 to'lov qo'shish — chek/skrinshotsiz), 💳 To'lovlar + 💳 Kartalar (➕ karta qo'shish/o'chirish), 📦 Mahsulotlar (➕ mahsulot qo'shish — nom/tavsif/shablon/narx, 🗑 o'chirish), 📢 Xabar yuborish (segmentlangan: barchasi / qarzdorlar / faollar), 📤 Eksport (CSV: foydalanuvchilar/botlar/to'lovlar), 📦 Backup (qo'lda yoki avtomatik har `BACKUP_INTERVAL_HOURS`, standart 24 soat, `BACKUP_KEEP`=14 saqlanadi).
- **Barqarorlik:** global xatolik ushlagich (`@dp.error()`, `bot.py`) — handler xatosi botni yiqitmaydi; flood-control middleware (`THROTTLE_SECONDS`, standart 0.6s) tez-tez so'rovlarni e'tiborsiz qoldiradi.
- **Fayllar:** `bot.py` (kirish nuqtasi), `config.py`, `database.py` (users/bots/payments/products), `deploy.py` (sinov-deploy: shablon nusxalash + PM2), `trial_lifecycle.py` (24/48 soatlik sinov nazorati), `utils.py`, `states.py` (FSM), `keyboards.py`, `reminders.py`, `backup.py`, `export.py`, `middlewares.py`, `handlers/user.py`, `handlers/admin.py`, `deployed/` (joylashtirilgan sinov botlari — `.gitignore`da, tokenlar saqlanadi).
- 2026-08-16'da backup/eksport/xatolik ushlash/flood-control/segmentlangan broadcast/foydalanuvchi o'chirish/qo'lda to'lov qo'shish funksiyalari qo'shilib production'da tekshirildi (tafsilotlar yuqorida, "Holat — 2026-08-16" bo'limida). 2026-08-19'da katalog/avto-deploy/sinov nazorati funksiyasi qo'shildi (tafsilot: "Holat — 2026-08-19" bo'limi).

### `vip-chopar` (`workspace/Vip chopar`) — video forwarder + boshqaruv bot

Manba Telegram kanalidagi video xabarlarni maqsad kanalga **native forward** ("Forwarded from...") qiluvchi tizim, ikki qismli:

- **Userbot** (Telethon, haqiqiy Telegram akkaunt nomidan — `+998705146070`, @Kinoadminn2) — `forward_videos.py`, botga a'zo bo'lmagan/yopiq kanallarning ham to'liq tarixini o'qiy oladi, video-only filter bilan eskisidan yangisiga forward qiladi, progressni `state.json`ga yozadi (to'xtasa davom etadi), `FloodWaitError`ni avtomatik kutib qayta uradi. Bir martalik login: `login.py request` → kod kelgach `login.py signin <KOD> [2FA_PAROL]` → `vipchopar.session` yaratiladi (juda maxfiy, gitga tushmaydi).
- **Boshqaruv bot** (`bot.py`, aiogram) — PM2'da `vip-chopar` nomi bilan doimiy ishlaydi, inline tugmalar: ▶️ Forward (subprocess sifatida `forward_videos.py`ni ishga tushiradi, PID/log `forward.pid`/`forward.log`), 📊 Status, ⏹ Stop, 📡 Manba/🎯 Maqsad kanal sozlash (`channels.json`ga saqlanadi, kanal o'zgarsa progress avtomatik tozalanadi). Faqat `.env`dagi `ADMIN_IDS` buyruq bera oladi.
- **Sozlash:** `.env` — `API_ID`/`API_HASH` (my.telegram.org), `PHONE_NUMBER`, `SOURCE_CHANNEL`/`DEST_CHANNEL` (yoki bot orqali `channels.json`), `FORWARD_DELAY_SECONDS` (standart 3s). `config.py`da `load_dotenv(override=True)` — doim diskdagi `.env`ni ustuvor qiladi.
- **2026-08-14/15'da bajarilgan:** loyiha yaratildi, Telethon login yakunlandi, manba ("vip kana1") va maqsad ("Vip") kanallar sozlandi, 2 ta bug tuzatildi (env meros qilish bo'sh qiymat muammosi; kanal ID matn/int mos kelmasligi — `parse_channel()` bilan tuzatildi), 500+ video muvaffaqiyatli fon rejimida forward qilindi (xatosiz, FloodWait yo'q).
- **Muhim:** `forward_videos.py` PM2'da emas (bir martalik ish, `bot.py` uni subprocess sifatida boshqaradi); kelajakda yangi kelayotgan videolarni ham avtomatik forward qilish kerak bo'lsa, alohida real-time skript (`events.NewMessage` handler) yozib PM2'ga qo'shish kerak bo'ladi.
- `.env`, `*.session`, `channels.json`, `state.json` — hech qachon commit qilinmasin/hech kimga yuborilmasin (akkauntga to'liq kirish huquqi beradi).

### `savdo-hisob` (`savdo-hisob`) — savdo-sotiq hisob-kitobi veb-ilovasi (Telegram bot emas)

Repo ildizidagi asosiy `claude-code-web`dan **butunlay mustaqil** Node.js/Express 5 + better-sqlite3 veb-ilova (o'z `package.json`, o'z serveri, o'z SQLite bazasi — `data/savdo.db`), yagona admin (parol bilan kirish) uchun: mahsulot/ombor hisobi (qoldiq, kam qoldiq ogohlantirishi), sotuv/xarid (bir nechta mahsulotli, ombor qoldig'ini avtomatik yangilaydi, `db.transaction()` bilan atomik), qarzdorlik (mijozga nasiya / yetkazib beruvchiga qarz, qisman to'lovlar, yagona `debt_payments` ledger), hisobot/dashboard (Chart.js grafiklar). Mobil-qulay (responsive, pastki tab-bar, PWA-tayyor manifest, service-worker hali yo'q).

- **Fayl strukturasi:** `server/index.js` (bootstrap), `server/db.js`/`schema.sql`/`migrate.js`, `server/auth.js` (signed-cookie, asosiy claude-code-web naqshidan mustaqil nusxa), `server/routes/*.js` (products/contacts/sales/purchases/debts/reports/settings), `server/services/stock.js`+`debtCalc.js`, `public/*.html`+`app.js`+`style.css`.
- **Ishga tushirilgan holat:** claudeweb'ning o'z PM2 daemonida `savdo-hisob` nomi bilan (`.env`: `PORT=3212`, `HOST=127.0.0.1`) — 2026-08-23 MVP yaratildi, 2026-08-24 PM2'ga ulandi. Foydalanuvchida root/SSH yo'qligi sababli alohida nginx/domen o'rniga **asosiy `claude-code-web`ning o'zi ichida `/savdo/` yo'lida reverse-proksi** qilib ulangan (`server/index.js`ga qo'shilgan qo'lda yozilgan middleware, `Location`/`Set-Cookie` header'larini `/savdo` prefiksiga moslab qayta yozadi) — 2026-08-25'da faollashtirildi, `https://claudeuz.duckdns.org/savdo/` orqali ochiladi. Cookie nomi to'qnashuvining oldini olish uchun `"savdo_session"` (asosiy ilova `"session"` ishlatadi).
- **2026-08-25'da tuzatilgan bug:** `/savdo/` cheksiz redirect halqasi (strict routing o'chirilgani sababli `/savdo` redirect handler'i `/savdo/`gа ham mos kelib qolgan edi) — asosiy `server/index.js`dagi redirect shartiga aniq tekshiruv qo'shildi.
- **2026-08-25'da qo'shilgan feature:** mahsulot formasida/ro'yxatida jonli hisoblanadigan "foyda" ko'rsatkichi (frontend-only, backend/baza o'zgarmadi).
- To'liq tafsilot (API ro'yxati, baza sxemasi, holat tarixi): `savdo-hisob/CLAUDE.md`.

### `polat` (`polat`) — restoran buyurtma/hisob-kitob tizimi (Telegram bot emas)

`savdo-hisob` bilan bir xil naqshdagi mustaqil Node.js/Express 5 + better-sqlite3 veb-ilova (o'z SQLite bazasi — `data/polat.db`), lekin **to'rt turdagi login** (admin/afitsiant/oshpaz/**dastavkachi**) + **login shart bo'lmagan mijozlar landing sahifasi** bilan ancha kengroq: afitsiant stolga buyurtma qo'shib hisob-kitob/chek qiladi, oshpaz oshxona ekranida band stollar+onlayn buyurtmalarni (olib ketish/yetkazib berish turi bilan) ko'radi, dastavkachi (2026-09-08'da qo'shildi) faqat yetkazib berish buyurtmalarini ko'rib yetkazganini belgilaydi, admin menyu/stol/xodim/xarajat/bron/onlayn-buyurtma (chek chiqarish bilan)/hisobotni boshqaradi, mijoz esa (`/landing/`, login shart emas) haqiqiy menyudan savatga taom qo'shib olib ketish/yetkazib berish buyurtmasi beradi (yetkazib berishda ixtiyoriy GPS lokatsiya bilan) yoki stol bron qiladi.

- **Ishga tushirilgan holat:** claudeweb'ning o'z PM2 daemonida `polat` nomi bilan (`.env`: `PORT=3213`). Ilgari `savdo-hisob` naqshida `/polat/` proksisi orqali `claudeuz.duckdns.org`da ochilardi, lekin bu proksi keyinchalik `claude-code-web/server/index.js`dan olib tashlangan (kodda shunday izoh qoldirilgan) — endi **o'z alohida subdomeni** orqali ochiladi: **`https://polatuz.duckdns.org/`** (`/etc/nginx/sites-available/polatuz`, mustaqil Certbot sertifikati, to'g'ridan-to'g'ri `127.0.0.1:3213`ga proxy_pass). Bosh sahifa `/` avtomatik `/landing/`ga yo'naltiradi. Cookie nomi `"polat_session"`. (2026-09-04'da tekshiruv paytida aniqlangan/tuzatilgan — asosiy va `polat/CLAUDE.md` hujjatlari eski `/polat/` manzilni yozib qolgan edi.)
- **2026-08-25/26'da qo'shilgan:** qorong'i/oltin ("lyuks") dizayn (admin/afitsiant/oshpaz), mustaqil terracotta/qora/oltin dizaynli mijozlar landing sahifasi (menyu+savat+buyurtma, stol bron qilish oynasi, galereya, biz haqimizda, aloqa), yangi **oshpaz** roli (SQLite CHECK constraint'ini xavfsiz qayta qurish orqali qo'shildi — `polat/server/db.js`dagi `migrateAddChefRole()`), va xavfsizlik tekshiruvi (`TRUST_PROXY` tuzatildi — sessiya cookie'sida `Secure` yo'q edi; 500-xato xabarlari mijozga endi tafsilotsiz qaytadi; xodim paroli minimal uzunligi 6ga oshirildi).
- **2026-09-08'da qo'shilgan (bitta uzun sessiyada):** landing'dagi savat/checkout oqimi (ilgari CSS tayyor edi, HTML/JS yo'q edi) haqiqatan ishga tushirildi + ixtiyoriy GPS lokatsiya; yangi **dastavkachi (courier)** roli (`/courier/orders.html`, xavfsiz CHECK-migratsiya bilan) — yetkazib berish buyurtmalarini ko'rib "🚚 Yetkazildi" deb belgilaydi; yangi yetkazib berish buyurtmasi kelganda admin+oshpaz+dastavkachiga BARAVAR bildirishnoma (`notifications.customer_order_id`); admin panelda mijoz buyurtmasi uchun ham QZ Tray chek chiqarish qo'shildi; ikkita real bug tuzatildi (admin'da yetkazib berish "tayyor" buyurtmasi chalg'ituvchi "Bajarildi" deb ko'rsatilardi — endi `delivered_at`ga qarab aniqlashtiriladi; oshpaz ekrani buyurtma turini — olib ketish/yetkazib berish — umuman ko'rsatmasdi) + admin "Buyurtmalar" sahifasiga 15s avto-yangilanish qo'shildi (ilgari faqat qo'lda F5 bilan yangilanardi).
- To'liq tafsilot (fayl strukturasi, baza sxemasi, API, rollar, xavfsizlik holati): `polat/CLAUDE.md`.

## Holat — 2026-08-18: barcha loyihalarga CLAUDE.md yozildi, asosiy hujjat yangilandi

Uy papkasi ichidagi barcha bot loyihalari uchun alohida `CLAUDE.md` yaratildi/tekshirildi (`workspace/kino-bot-pro`, `workspace/kino-bot-2`, `Post bot 10` — yangi yozildi; `workspace/kino-uploader`, `just/To'lov bot`, `workspace/Vip chopar` — allaqachon bor edi). Shu bilan birga bugun ma'lum bo'lgan ikkita yangi loyiha (`kino-uploader` va `kanal-01`/`Post bot 10`) yuqoridagi PM2 jadvali va "Botlar haqida batafsil" bo'limiga qo'shildi — ilgari faqat `pm2 list`da ko'rinib, bu hujjatda yo'q edi.

- `kino-bot-pro`/`kino-bot-2` — kod bayt-baytiga bir xil ekani `diff -rq` bilan tasdiqlandi; ikkalasi ham mustaqil `.env`/baza bilan kino/anime-kod tarqatish boti.
- `Post bot 10` (`kanal-01`) — aiogram 3 + MongoDB (Motor) + APScheduler asosidagi kanalga avto-post/rejalashtirish boti, ko'p-nusxali (bir nechta alohida tokenli bot instansi bitta umumiy MongoDB'ni ishlatishga mo'ljallangan) arxitektura bilan.
- Har bir loyihaning o'z `CLAUDE.md`si endi mustaqil manba hisoblanadi (fayllar, `.env`/`config.py` kalitlari, admin panel funksiyalari, baza tuzilishi) — shu asosiy fayldagi "Botlar haqida batafsil" bo'limi faqat qisqa umumiy ko'rinish beradi.

## Holat — 2026-08-19: `tolov-bot`ga (just/To'lov bot) katalog + avtomatik sinov-deploy funksiyasi qo'shildi

Foydalanuvchi so'rovi bilan `tolov-bot`ga yangi bo'lim qo'shildi: **📦 Botlar katalogi** (foydalanuvchiga) + **📦 Mahsulotlar** (adminga). Admin katalogga tayyor bot xizmatini (nom, tavsif, texnik shablon, sinovdan keyingi oylik narx) qo'shadi; foydalanuvchi katalogdan birini tanlab, **🎁 48 soat sinab ko'rish** tugmasini bosadi, @BotFather orqali yaratgan bot tokenini yuboradi — tizim shu tokendan foydalanib **yangi bot nusxasini avtomatik VPS'ga (claudeweb'ning o'z PM2 daemoniga) joylashtiradi**, 48 soatlik bepul sinov boshlanadi, 24 soatdan keyin to'lov eslatmasi, 48 soatdan keyin (to'lanmagan bo'lsa) avtomatik to'xtatish, to'lov qilingach avtomatik qayta yoqish.

- **Yangi fayllar:** `deploy.py` (shablon registri + `subprocess` orqali `pm2 start/stop/delete`, shablon manba kodidan `venv`/baza/`.env`siz nusxa olib, umumiy `venv` interpretatoriga ishora qiladi — har safar alohida virtual muhit o'rnatilmaydi), `trial_lifecycle.py` (24/48 soatlik nazorat fon vazifasi, `bot.py`ga `trial_loop` sifatida ulandi, har 15 daqiqada tekshiradi).
- **Baza:** yangi `products` jadvali (katalog, admin boshqaradi) + `bots` jadvaliga `ALTER TABLE` orqali (mavjud ma'lumot yo'qolmasdan) qo'shilgan ustunlar: `product_id`, `is_trial`, `trial_pending`, `deploy_status`, `pm2_name`, `bot_token`, `trial_started_at`, `trial_reminder_sent`. Sinov boti darhol `status='approved'` bilan yaratiladi — shu sabab mavjud "Mening botlarim"/to'lov oqimi o'zgarishsiz ishlayveradi; `trial_pending=1` bo'lgan botlar oddiy kunlik (`get_due_bots`) eslatma so'rovidan chetlashtirilgan (ular 24/48 soatlik alohida nazoratda).
- **Faqat bitta shablon hozircha ulangan:** `kino_bot` (`workspace/kino-bot-pro` asosida — mustaqil SQLite bazali, faqat `BOT_TOKEN`/`ADMIN_IDS` bilan sozlanadigan bot). Telethon-userbot login talab qiladigan loyihalar (`vip-chopar`, `kino-uploader`) bu avto-deploy oqimiga texnik jihatdan mos emas (faqat token yetarli emas — telefon raqam/SMS-kod login kerak), shuning uchun ulanmagan; yangi shablon qo'shish `deploy.py`dagi `TEMPLATES` registriga qo'lda (developer tomonidan) qo'shilishi kerak.
- Tekshirilgan: barcha o'zgargan/yangi fayl `py_compile` va import darajasida xatosiz; DB migratsiyasi production `bot.db`ning nusxasida avval sinalgan, so'ng haqiqiy bazada `pm2 restart` orqali xatosiz qo'llanilgani tasdiqlangan (`products` jadvali va `bots`dagi yangi ustunlar mavjud); to'liq deploy/stop/delete tsikli soxta token bilan haqiqiy PM2'da sinaldi (jarayon yaratildi → to'xtatildi → o'chirildi → tozalandi, production `tolov-bot` jarayoniga tegmasdan); `pm2 restart tolov-bot --update-env` xatosiz, jarayon barqaror **online** (restart soni ortmadi, crash-loop yo'q), `pm2 save` bilan saqlandi. `deployed/` papkasi `.gitignore`ga qo'shildi (foydalanuvchi tokenlarini o'z ichiga oladi).

**Shu kuni davomida qo'shildi:** foydalanuvchi "admin bilan bog'laning" matni nega adminpanelda tahrirlanmasligini so'radi — sabab, u kodda hardcode qilingan matn edi (haqiqiy tugma emas). Tuzatildi: yangi umumiy key-value `settings` jadvali (`database.py`), admin panelida **📞 Admin kontakti** tugmasi (`handlers/admin.py`, `SetAdminContact` FSM holati) — admin `@username` kiritadi (yoki **🗑 Kontaktni tozalash**), `utils.py`dagi `admin_contact_line()` orqali botning 4 ta joyidagi ("Yordam" matni, karta yo'qligi, bot rad etilganda, to'lov rad etilganda) barcha "Admin bilan bog'laning" xabarlari endi shu saqlangan qiymatni dinamik ko'rsatadi (qiymat bo'lmasa umumiy matnga qaytadi). `settings` jadvali migratsiyasi ham avval nusxada, so'ng production bazada sinalib, `pm2 restart tolov-bot --update-env` xatosiz o'tgani va jarayon barqaror online ekani tasdiqlandi.

## Holat — 2026-08-25: sayt dizayni Claude.ai uslubiga chuqurlashtirildi

Foydalanuvchi so'rovi bilan asosiy chat interfeysi (`public/style.css`, `public/app.js`, `public/icon.svg`, `public/login.html`) dizayn jihatdan yangilandi — **izolyatsiya, ID/class nomlari va JS mantig'iga tegilmadi**, faqat vizual qatlam:

- **Editorial serif shrift** — assistant xabar matni endi `--serif` (`Iowan Old Style`/`Palatino`/`Georgia` zaxira zanjiri) bilan chiqadi, kod bloklari (`code`/`pre`) hamon `--mono`da qoladi; foydalanuvchi xabar pufakchalari sans holida qoladi.
- **Asterisk belgisi (✳)** — avval `display:none` bo'lgan `.marker` span endi faqat assistant qatorlarida ko'rinadi (accent rangda, kichik), `app.js`dagi `addBubble()` ichida marker matni `'·'` dan `'✳'`ga o'zgartirildi.
- **Nozik fon gradienti** — `body`ga yuqori markazda juda xira (`opacity 0.07`) accent-rang radial gradient qo'shildi, `.app-shell`/`#messages` foni shaffof qilindi shu gradient ko'rinishi uchun (header/composer hamon qattiq fonda, chegara sifatida).
- **Chuqurlik/soyalar** — `#composer`, `.tool-card`, `.permission-card`ga yumshoq box-shadow qo'shildi; `.tool-card`ga chapdan ingichka accent chiziq (border-left) qo'shildi.
- **Silliqroq animatsiya egri chizig'i** — yangi `--ease: cubic-bezier(0.16, 1, 0.3, 1)` o'zgaruvchisi drawer/overlay/file-viewer transition'lariga qo'llandi (avvalgi oddiy `ease-out` o'rniga).
- **Typing-indikator** — "generating▋" matn-kursor o'rniga uch nuqtali sakrab turuvchi animatsiya (`.typing-dots`, `app.js`dagi `showTyping()` ichida yangi markup).
- **Logo/ikonka** — `public/icon.svg` (PWA ikonkasi) va login sahifasidagi `.login-mark` endi `>_` terminal belgisi o'rniga accent fonli, uch chiziqli sunburst/asterisk belgisi (Claude.ai uslubiga yaqin, lekin mustaqil chizilgan).
- **Login sahifa** — karta foniga nozik gradient (`--bg-elev-2` → `--bg-elev`), sahifa foniga ham xira radial gradient, mark atrofida accent glow soyasi qo'shildi.

**Tekshirilgan/tekshirilmagan:** `node -e` bilan CSS qavslar balansi va `node -c public/app.js` bilan JS sintaksisi, `python3 xml.dom.minidom` bilan `icon.svg` — barchasi xatosiz tasdiqlangan. **PM2 restart hali bajarilmagan** — foydalanuvchi ruxsat kartochkasini bir necha marta rad etdi (sabab aniqlanmadi); umuman olganda restart shart emas, chunki bu fayllar (`public/`) statik va Express ularni to'g'ridan-to'g'ri diskdan xizmat qiladi (build/bundle bosqichi yo'q) — brauzerda hard-refresh qilish yetarli bo'lishi kerak. Ammo haqiqiy brauzerda vizual tekshiruv hali qilinmagan.

## Holat — 2026-08-25/26: `polat` (restoran tizimi) uy papkasidagi loyihalar ro'yxatiga qo'shildi

`polat` loyihasi (avvalroq foydalanuvchi tomonidan qurilgan, afitsiant/stol/buyurtma/chek asosiy qismi bilan) bu kuni bitta uzun sessiyada sezilarli darajada kengaytirildi va endi bu asosiy hujjatda ham to'liq qayd etildi (ilgari faqat `pm2 list`da ko'rinib, bu faylda umuman yo'q edi — yuqoridagi PM2 jadvali va "Botlar haqida batafsil" bo'limiga `savdo-hisob`dan keyin qo'shildi).

Qisqacha: qorong'i/oltin dizayn yangilanishi, mijozlar uchun mustaqil landing sahifa (menyu+savat+buyurtma, stol bron qilish, galereya, biz haqimizda, aloqa), yangi "oshpaz" (chef) roli va oshxona ekrani, hamda to'liq xavfsizlik tekshiruvi (bitta real bug — `TRUST_PROXY` — topilib tuzatildi). To'liq tafsilot: `polat/CLAUDE.md`.

## Holat — 2026-08-28: `kino-uploader`da ikkita production bug topildi va tuzatildi

Foydalanuvchi so'rovi ("xatoliklar bormi tekshir va tuzat") bilan `workspace/kino-uploader/` (PM2 nomi `kino-uploader`) production xato logi (`.pm2/logs/kino-uploader-error.log`, ~4400 qator) tekshirildi — ikkita bog'liq, haqiqiy bug topilib tuzatildi ("video umuman yuklanmayapti" muammosining haqiqiy ildizi):

1. `distributor.py`, `receive_relayed_video()` — Telethon `conv.wait_event(events.NewMessage(incoming=True))` `chats=` cheklovisiz edi, shu sabab boshqa nishon bot bilan ketayotgan suhbat javobi (yoki boshqa kanal xabari) "forward qilingan video" deb noto'g'ri qabul qilinib, video fayl umuman yaratilmasdi (keyin `ffprobe`/`upload_file` shu yo'q faylga qarab yiqilardi). Fix: `chats=bot_username` + `func=lambda e: e.video or e.document`.
2. `distributor.py`, `distribute_video()` — `_probe_video_attributes()` ichidagi bloklovchi `subprocess.run()` (ffprobe) butun asyncio event loop'ni (bot+userbot bir processda) to'xtatib qo'yardi, shu sabab callback tugmalar "query is too old" xatosi bilan rad etilardi. Fix: `asyncio.to_thread(...)`.

Tekshirildi: `py_compile` xatosiz, `pm2 restart kino-uploader --update-env` xatosiz, jarayon **online** (`unstable restarts: 0`), Telethon userbot qayta ulandi. To'liq tafsilot: `workspace/kino-uploader/CLAUDE.md` ("Ishlab chiqish tarixi", 13-band).

## Holat — 2026-09-06: `kino-uploader`ga "Kanal" va "Avto video yuklash" bo'limlari qo'shildi

Foydalanuvchi so'rovi bilan `workspace/kino-uploader/` (PM2 nomi `kino-uploader`) bir necha bosqichda sezilarli kengaytirildi — avval qo'lda (kod/videoni admin o'zi yuborib) ishlagan tizimga endi ulangan manba KANALDAN ko'p videoni avtomatik olib tarqatish imkoniyati qo'shildi:

1. **📡 Kanal** — manba kanal(lar)ni ulash/ro'yxatlash/yoqish-o'chirish/uzish (`channels.json`). Ulashda userbot (`distributor.resolve_channel()`) shu kanalga (`@username`/`-100...` ID/`t.me/...` havola) haqiqatan kirish huquqi bor-yo'qligini darhol tekshiradi.
2. **🚀 Avto video yuklash** — bitta tugma bilan: admin nechta video kerakligini kiritadi → har video uchun tasdiqlangan bo'sh kod hovuzidan (🔢 Raqam bilan bir xil) tasodifiy kod tanlanadi → ulangan kanaldan ESKIDAN YANGIGA (`distributor.iter_channel_videos()`) navbatdagi video olinadi (bayt yuklab olmasdan) → barcha faol nishon botlarga tarqatiladi. Progress jonli yangilanadi, **⏹ To'xtatish** mavjud, kanal progressi (`last_msg_id`) resumable (to'xtatilsa/restart bo'lsa oxirgi ko'rilgan xabardan davom etadi).
3. **Havolalar** — har video uchun manba (kanaldagi) xabarga HAM, har nishon botga tushgan videoning O'SHA BOTDAGI o'z xabariga HAM `tg://openmessage?user_id=<peer>&message_id=<id>` ko'rinishidagi to'g'ridan-to'g'ri havola beriladi (Telegram Desktop'da bosilsa ochiladi).
4. **Hisobot kanali** (`.env`dagi `REPORT_CHANNEL_ID`) — har video uchun TO'LIQ hisobot (video + barcha nishon havolalari) videoning o'zi bilan birga (caption sifatida) shu alohida kanalga post qilinadi; admin bilan bevosita suhbatda esa faqat QISQA xulosa (kod + muvaffaqiyat soni + 1 ta havola) qoladi — ortiqcha batafsillik bilan suhbatni to'ldirmasin uchun.
5. Admin chatidagi kino kodi endi 🔢 Raqamdagi bilan bir xil `<code>` (HTML) formatida — ustiga bosilsa avtomatik nusxa olinadi.

Tekshirildi: har bosqichda `py_compile` xatosiz, `pm2 restart kino-uploader --update-env` xatosiz, jarayon **online** barqaror (restart soni normal ko'tarilib, keyin crash-loop yo'q), Telethon userbot har safar qayta ulandi (`Virus @Best_telegram_2026`), `pm2 save` bilan saqlandi. **Haqiqiy ko'p-videoli avto yuklash oqimi (real ulangan kanal bilan) va hisobot kanaliga yozish huquqi production'da to'liq sinalmagan** — admin tomonidan amalda tekshirish tavsiya etiladi. To'liq tafsilot (barcha kod/format qarorlari, "Ishlab chiqish tarixi" 17–22-bandlar): `workspace/kino-uploader/CLAUDE.md`.

## Holat — 2026-09-06: bosh sahifadan "papkalar" tabi o'chirildi

Foydalanuvchi so'rovi bilan (avval rootweb'da xuddi shu narsa qilingan edi) drawer'dagi **"papkalar"** tabi (istalgan absolyut yo'lni `localStorage`da — `claudewebFolderBookmarks` kaliti — yorliq sifatida saqlab, `browseRoot` orqali fayllar paneliga sakratuvchi funksiya) butunlay olib tashlandi:

- **`index.html`:** `data-tab="folders"` tugmasi va `#panelFolders` paneli (forma + ro'yxat) o'chirildi. **"qurilmalar" (devices) tabi tegilmadi**, o'z holida qoladi.
- **`app.js`:** DOM referenslar (`panelFolders`, `addFolderForm`, `newFolderLabelInput`, `newFolderPathInput`, `folderBookmarkListEl`), `folderBookmarks` massivi, tab-almashtirish ulanishi (`panelFolders.classList...`, `if (tab==='folders')`), va butun funksional blok (`FOLDER_BOOKMARKS_KEY`, `loadFolderBookmarks`, `saveFolderBookmarks`, `renderFolderBookmarkList`, `addFolderForm` submit handler) olib tashlandi.
- **`genId()` funksiyasiga TEGILMADI** — u "qurilmalar" bo'limida ham ishlatiladi (`devicesList.push({id: genId(), ...})`), shuning uchun umumiy, o'chirilmadi.
- **`browseRoot` o'zgaruvchisiga TEGILMADI** — fayllar API chaqiruvlarida bir nechta joyda ishlatiladi (`buildBrowseQuery()`, yuklash/ko'chirish tekshiruvlari), faqat uni "papkalar" yorlig'i bosilganda o'rnatuvchi YAGONA joy o'chirildi — endi bu o'zgaruvchi doim `null` qoladi (claudeweb'da rootweb'dagi kabi "/root'ga o'tish" tugmasi yo'q edi), lekin qolgan kod xavfsiz ishlayveradi.

Tekshirildi: `node -c app.js` (Node22) xatosiz, `index.html`dagi `<div>` ochilish/yopilish soni teng (36/36), fayl egaligi qayta `claudeweb:claudeweb`ga tiklandi. Bu — faqat statik `public/` fayllariga o'zgarish (Express ularni build bosqichisiz to'g'ridan-to'g'ri diskdan xizmat qiladi), **`pm2 restart claude-code-web` shart emas** — brauzerda hard-refresh yetarli.

**Darhol topilgan yon-ta'sir (shu kuni tuzatildi):** "papkalar" tabi o'chirilgandan keyin qolgan 5 ta tab ("loyihalar"/"fayllar"/"botlar"/"limit"/"qurilmalar") drawer'ning eski tor kengligiga (`.drawer { width: min(88vw, 340px) }`) sig'may qoldi — skrinshotda "qurilmalar" so'zi qirqilib, yopish (X) tugmasi va orqadagi chat matni bilan qoplanib ko'rinardi. `style.css`da tuzatildi:
- `.drawer` kengligi `min(88vw, 340px)` → `min(92vw, 420px)`ga oshirildi, `overflow: hidden` qo'shildi (spill-over ehtimolini butunlay yopish uchun).
- `.drawer-tabs`ga `overflow-x: auto` + skrollbar yashirish qo'shildi, `.drawer-tab`ga `white-space: nowrap; flex-shrink: 0` — endi tablar soni/nomi qanchalik uzun bo'lmasin, hech qachon qirqilib/ustma-ust tushib qolmaydi, kerak bo'lsa gorizontal skroll bilan ko'rinadi.

Tekshirildi: CSS qavslar balansi teng (288/288), fayl egaligi tiklandi. Statik fayl — restart shart emas.

## Holat — 2026-09-07: `polat`da chek/admin oqimi ustida bir nechta tuzatish (chop etishdan keyin qaytish, yangi oyna o'rniga modal, kesishma bug)

`polat` (restoran tizimi, `polat/CLAUDE.md`ga qarang) ustida bitta sessiyada ketma-ket bir nechta so'rov bajarildi:

1. **Admin bildirishnomasi tezlashtirildi** — "hisob-kitob qilindi" (chek chop etish) bildirishnomasi poll intervali 10s → 3s (`public/app.js`, `initAdminPrintRequests()`).
2. **Chek chop etilgach admin avtomatik o'z bo'limiga qaytishi** — dastlab `waiter/receipt.js`ga `returnToAdminAfterPrint()` qo'shildi (yangi tabni yopish/referrer'ga qaytarish).
3. **Keyin "har safar yangi oyna ochmasin" so'roviga ko'ra** — chek chop etish butunlay **modal**ga o'tkazildi: QZ Tray mantiqi (`printReceiptView`, `buildEscPosReceipt`, `setupQzSecurity`, lazy `loadQzTray`) umumiy `public/app.js`ga ko'chirildi, admin panelidagi bildirishnoma va Hisobot'dagi "Chek" tugmasi endi hech qanday navigatsiya qilmay, shu sahifaning o'zida `.modal-backdrop` popup'ida chekni ko'rsatadi.
4. **Shu ko'chirish haqiqiy production bug keltirib chiqardi** — "chek yuklanmayapti": `app.js`ga qo'shilgan `let qzSecuritySetUp` va `buildEscPosReceipt` `waiter/receipt.js`dagi xuddi shu nomlar bilan bitta sahifada (`receipt.html` ikkalasini ham ulaydi) to'qnashib, `SyntaxError` bilan butun `receipt.js`ni ishga tushirmay qo'ygan edi (chek "Yuklanmoqda..." holatida abadiy qotib qolardi). `receipt.js`dan duplikat mantiq olib tashlanib, `app.js`dagi yagona manbadan foydalanadigan qilindi — Node `vm` konteksti orqali ikkala skriptni birga yuklab tasdiqlangan.

Barcha bosqichlarda `pm2 restart polat --update-env` xatosiz, jarayon barqaror **online**. To'liq texnik tafsilot (kod, tekshiruv usullari, saboq) — `polat/CLAUDE.md`dagi tegishli "Holat — 2026-09-07" bo'limlarida.

## Holat — 2026-09-08: `polat` admin menyusiga "Tan narxi" maydoni qo'shildi

Foydalanuvchi so'rovi: admin menyuga taom qo'shayotganda/tahrirlayotganda tan narxi (cost price) ham sotuv narxi bilan birga kiritilsin. `menu_items` jadvaliga ixtiyoriy `cost_price` ustuni qo'shildi (xavfsiz `ALTER TABLE` migratsiyasi, avval baza nusxasida, keyin production'da sinaldi), admin taom modalida yangi maydon paydo bo'ldi (ombor bilan bog'langan taomda avtomatik ombordagi tan narxdan to'ldiriladi), ro'yxatda narx yonida foyda ko'rsatkichi chiqadi. **Tan narx faqat admin panelida ko'rinadi** — afitsiant (`/api/waiter/menu`, oldin `SELECT *` edi, endi aniq ustunlarga toraytirildi) va mijoz menyusida chiqmaydi. To'liq end-to-end HTTP oqim (yaratish/yangilash/validatsiya/rol-asosli ko'rinish) sinaldi, `pm2 restart polat --update-env` xatosiz, jarayon barqaror online. To'liq tafsilot: `polat/CLAUDE.md`dagi "Holat — 2026-09-08: admin menyuga taom qo'shganda/tahrirlaganda 'Tan narxi' maydoni qo'shildi" bo'limi.

## Muhim texnik eslatmalar

- VPS tizim Node.js eski (v12.22.9) — shuning uchun `/usr/local/bin/node22` kerak.
- `.env`dagi `PROJECT_DIR=./workspace` papkasi — Claude shu yerda ishlaydi (fayl o'qish/yozish/bajarish). Bu papka yo'q bo'lsa server "native binary failed to launch" degan CHALG'ITUVCHI xato beradi (asl sabab: `cwd` mavjud emas, libc/musl bilan aloqasi yo'q).
- `server/index.js` boshida `CLAUDE_*`/`CLAUDECODE`/`AI_AGENT` env o'zgaruvchilari tozalanadi — agar bu server biror Claude Code sessiyasi ichidan (masalan qo'lda pm2 start) ishga tushirilsa, begona sessiya o'zgaruvchilari ichki `claude` jarayoniga aralashib ketmasin deb.

## Holat — 2026-09-14: kod rootweb bilan birlashtirildi (bitta manba)

Bu loyiha va `rootweb` — ikki xil **shaxs** (alohida repo, alohida tizim
foydalanuvchisi, alohida Claude hisobi, alohida limit), lekin endi **kod
bitta**. Farqlar faqat `.env` va `server/config.js` da to'planadi.

`INSTANCE_MODE=sandbox` bu yerda nimani anglatadi:

| Sozlama | Bu yerda | rootweb'da |
|---|---|---|
| Standart ruxsat rejimi | `manual` | `avto` |
| Favqulodda amallar (unban, UFW) | o'chirilgan (403) | yoqilgan |
| `/savdo` proksisi | yoqilgan (`PROXY_PORT=3212`) | o'chirilgan |
| "Qurilmalar" tabi | yoqilgan | o'chirilgan |
| Bir vaqtdagi sessiyalar | 2 | 3 |

**Nega birlashtirildi:** rootweb'da 20 dan ortiq xavfsizlik va UI tuzatishi
qilingan edi, bu yerda ularning BIRORTASI yo'q edi — chunki har bir tuzatish
ikki marta qilinishi kerak edi va biri unutilardi.

### Shu bilan kelgan xavfsizlik tuzatishlari

Batafsil: `docs/xavfsizlik-tuzatishlari.md`

- **CSWSH** — WebSocket handshake'ida `Origin` tekshiruvi yo'q edi. Siz bu
  saytga login qilgan holda istalgan boshqa saytga kirsangiz, o'sha sayt
  `wss://.../ws` ochib, 8 ta botni to'xtatish/o'zgartirish va ularning
  `.env` tokenlarini o'qish imkoniga ega bo'lardi. ACL buni to'smasdi —
  botlar shu foydalanuvchining o'z domenida.
- **Sessiya tokeni** o'zgarmas edi (`sign('ok')`), muddati serverda
  tekshirilmasdi, "Chiqish" uni bekor qilmasdi — o'g'irlangan cookie
  abadiy ishlardi.
- **Bash siyosati** faqat denylist edi. `echo ... | base64 -d | sh`,
  `find -delete`, `python3 - <<EOF`, `cat x.py | python3` kabi usullar
  bemalol chetlab o'tardi. Endi allowlist + denylist, testlar bilan
  (`npm test` — 17 ta test).
- **XSS** — `auth.html` da jarayonning xom chiqishi `innerHTML` ga
  qo'yilardi; `escapeHtml` atribut kontekstida qo'shtirnoqni o'tkazardi.
- **CSP va clickjacking sarlavhalari** yo'q edi.
- **Login urinishlari** ilovada cheklanmagan edi.
- **highlight.js** cdnjs'dan SRI'siz yuklanardi — CDN buzilsa
  autentifikatsiyalangan origin ichida ixtiyoriy JS.
- Atomik fayl yozuvi, audit-log rotatsiyasi, `permissionMode` tiklanishi.

### Yangi funksiyalar

- **VPS holati** tabi — yuklama, xotira, disk (Claude'siz, 0 token)
- **Botlar panelida 💬 suhbat** — papka PM2'ning `pm_cwd` idan olinadi,
  sessiya to'g'ridan-to'g'ri o'sha papkada toza holda boshlanadi.
  `CLAUDE.md` avtomatik o'qiladi, `OXIRGI-ISH.md` oxirgi amallarni aytadi.
  "Loyihani top" deb yozish shart emas.
- Sozlamalar paneli, chatda qidiruv, tez buyruqlar, tool natijasini
  ko'rish, PWA (offline qobiq), "Tizim" mavzusi.

### ⚠️ Resurs cheklovi hali QO'YILMAGAN

O'lchandi (2026-09-13): 26 root bot ~3.3 GB + 8 bot (bu yerda) ~1.2 GB +
tizim ~0.5 GB. Har ochiq Claude sessiyasi ustiga 300-400 MB. Hech qanday
cheklov yo'q — xotira tugasa OOM killer qurbonni **o'zi tanlaydi**, u
PostgreSQL yoki to'lov boti bo'lishi mumkin.

ACL izolyatsiyasi *nimaga kirishni* cheklaydi, *qancha yeyishni* emas.

`deploy/systemd-limits.conf` tayyor, lekin **o'rnatilmagan**. Avval
`free -h` va `nproc` bilan VPS resursini tekshirib, qiymatlarni moslang.
Batafsil: `docs/resurs-cheklovi.md`.

Kod ichidagi cheklov allaqachon ishlaydi: `MAX_SESSIONS=2`,
`SESSION_IDLE_MINUTES=45`. Suhbat yo'qolmaydi — tarix diskda qoladi,
faqat subprocess bo'shaydi. Ish bajarayotgan sessiya hech qachon yopilmaydi.
