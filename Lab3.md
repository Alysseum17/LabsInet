# Лабораторна робота №3 — Сервіс електронної пошти (SMTP, POP3, IMAP)

**Варіант 6**

| Параметр | Значення |
|---|---|
| Мережа VNet1 | 192.168.106.0/24 (успадковано з ЛР1/ЛР2) |
| Хост-система | 192.168.106.1 |
| VM1 — Домен 1 | `color.edu`, IP 192.168.106.10, поштовий сервер `mail.color.edu` |
| VM2 — Домен 2 | `zone06.net`, IP 192.168.106.20, поштовий сервер `mail.zone06.net` |
| CA (Issuer/Subject) | `C=UA, ST=Ukraine, L=Kyiv, O=KPI, OU=OT, CN=color CA` |
| Поштові скриньки Домену 1 | red@color.edu, green@color.edu, blue@color.edu |
| Поштові скриньки Домену 2 | black@zone06.net, white@zone06.net |

Використовуємо: **Postfix** (MTA), **Dovecot** (POP3/IMAP), **OpenSSL** (CA і сертифікати), для тестів — **telnet**, **openssl s_client**, **swaks**.

---

# ЧАСТИНА 1. ТЕОРІЯ

## 1.1. Архітектура електронної пошти

Електронна пошта — це **асинхронна** служба обміну повідомленнями (на відміну від, скажімо, чату — відправник і отримувач не мусять бути онлайн одночасно). Стек складається з кількох ролей, кожна з яких — окрема програма:

```
Відправник                                                    Отримувач
   │                                                               │
   ▼                                                               ▲
 MUA (Mail User Agent)                                    MUA (Thunderbird, Outlook...)
   │  SMTP (порт 587, submission)                                  │  POP3/IMAP (110/143 або TLS: 995/993)
   ▼                                                               │
 MSA (Mail Submission Agent)                               MDA (Mail Delivery Agent)
   │  перевіряє автентифікацію                                     ▲
   ▼                                                               │
 MTA (Mail Transfer Agent) ── SMTP (порт 25) ──► MTA (Mail Transfer Agent)
   відправника                релеїнг через Інтернет      отримувача (кінцевий, за MX-записом)
```

* **MUA (Mail User Agent)** — поштовий клієнт, яким користується людина: Thunderbird, Outlook, веб-інтерфейс, `mutt`. Відправляє через SMTP, забирає через POP3/IMAP.
* **MSA (Mail Submission Agent)** — приймає лист від MUA (зазвичай порт **587**), обов'язково **автентифікує** користувача — це і є та сама "обов'язкова автентифікація при відправленні", про яку йдеться в завданні. Фактично на практиці MSA і MTA — та сама програма (Postfix), просто по-різному налаштована на різних портах.
* **MTA (Mail Transfer Agent)** — програма-поштовик, яка пересилає лист від сервера до сервера по протоколу SMTP, порт **25**. Кожен домен має свій MTA, який є "кінцевим отримувачем" для листів на цей домен (саме це вимагає п. 2.3 завдання).
* **MDA (Mail Delivery Agent)** — кладе прийнятий лист у поштову скриньку конкретного користувача на диску (у нашому випадку ця роль вбудована в Dovecot/Postfix через `lmtp` або локальну доставку).
* **MAA / POP3-IMAP сервер** — надає МUA доступ до вже прийнятих листів (Dovecot).

**Важлива відмінність, яку часто плутають:** SMTP — це протокол **тільки для відправки/пересилки** пошти (від клієнта до сервера і між серверами). Для **забирання** пошти клієнтом використовуються **інші** протоколи — POP3 або IMAP. SMTP ніколи не "видає" листи клієнту.

## 1.2. Протокол SMTP (RFC 5321)

**SMTP (Simple Mail Transfer Protocol)** — текстовий протокол поверх TCP, за замовчуванням порт **25** (сервер-сервер) або **587** (клієнт-сервер, submission, з обов'язковою автентифікацією). Раніше існував ще "SMTPS" на порту **465** (TLS одразу з початку з'єднання, без STARTTLS) — зараз теж активно використовується.

### Принцип роботи (клієнт-серверна модель діалогу)

1. Клієнт встановлює TCP-з'єднання з сервером.
2. Сервер першим надсилає **вітальне** повідомлення з кодом `220`.
3. Клієнт і сервер обмінюються командами й відповідями в текстовому вигляді, по одній команді за раз — клієнт командує, сервер відповідає кодом.
4. Після передачі даних листа з'єднання закривається (`QUIT`).

### Основні команди SMTP

| Команда | Призначення |
|---|---|
| `EHLO домен` (або застаріла `HELO`) | привітання, клієнт представляється своїм іменем; `EHLO` (Extended HELLO) — сучасна версія, у відповідь сервер перелічує підтримувані розширення (`STARTTLS`, `AUTH`, `SIZE` тощо) |
| `MAIL FROM:<адреса>` | вказати **адресу відправника конверта** (envelope sender) — це не те саме, що заголовок `From:` усередині листа! |
| `RCPT TO:<адреса>` | вказати **адресу отримувача конверта**. Можна повторювати кілька разів, якщо отримувачів декілька |
| `DATA` | почати передачу самого тексту листа (заголовки + тіло). Завершується рядком з однією крапкою `.` на окремому рядку |
| `STARTTLS` | перевести вже встановлене з'єднання в зашифрований режим (TLS) без розриву з'єднання |
| `AUTH PLAIN` / `AUTH LOGIN` | автентифікація клієнта (логін/пароль у Base64) — обов'язково після `STARTTLS`, щоб пароль не йшов у відкритому вигляді |
| `RSET` | скасувати поточну транзакцію (відправник/отримувач/дані), почати заново |
| `VRFY адреса` | перевірити, чи існує скринька (у сучасних серверах зазвичай вимкнено з міркувань безпеки — щоб не давати можливість перебирати адреси) |
| `NOOP` | "нічого не роблю", просто перевірка зв'язку |
| `QUIT` | завершити сесію |

### Коди відповідей SMTP

Перша цифра коду визначає загальний зміст:

| Клас | Значення |
|---|---|
| `2xx` | Успіх (`250 OK`) |
| `3xx` | Проміжний стан, очікується продовження (`354 Start mail input`) |
| `4xx` | Тимчасова помилка — спробуй пізніше (`450 Mailbox busy`) |
| `5xx` | Постійна помилка — не намагайся повторити так само (`550 No such user`) |

### Приклад повного діалогу (те, що ти побачиш і зробиш вручну через telnet)

```
S: 220 mail.color.edu ESMTP Postfix
C: EHLO client.example
S: 250-mail.color.edu
S: 250-STARTTLS
S: 250 AUTH PLAIN LOGIN
C: MAIL FROM:<red@color.edu>
S: 250 2.1.0 Ok
C: RCPT TO:<black@zone06.net>
S: 250 2.1.5 Ok
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: Subject: Test
C:
C: Hello!
C: .
S: 250 2.0.0 Ok: queued as A1B2C3
C: QUIT
S: 221 2.0.0 Bye
```

## 1.3. Формат поштового повідомлення (RFC 5322 + MIME)

Лист складається з двох частин, розділених **порожнім рядком**:

```
Заголовки (headers)
                            ← порожній рядок-роздільник
Тіло листа (body)
```

### Ключові заголовки

| Заголовок | Призначення |
|---|---|
| `From:` | хто "видимо" відправив лист (те, що бачить людина у клієнті) |
| `To:` | кому "видимо" адресовано |
| `Subject:` | тема |
| `Date:` | дата й час відправлення |
| `Message-ID:` | унікальний ідентифікатор листа (для відстеження дублікатів, відповідей) |
| `Received:` | **найважливіший для аналізу заголовок**. Кожен MTA, через який пройшов лист, **додає власний** рядок `Received:` **зверху**. Тому, читаючи знизу вгору, можна відновити весь шлях листа через сервери — це і є "аналіз заголовків" з п. 2.6 завдання |
| `Return-Path:` | адреса, куди повертати недоставлені повідомлення (bounce), встановлюється кінцевим MTA з envelope-sender |

### Конверт vs заголовки — принципова різниця

* **Envelope (конверт)** — адреси з команд `MAIL FROM` / `RCPT TO`. Саме за ними MTA **фактично маршрутизує** лист.
* **Заголовки `From:`/`To:`** — те, що показує клієнт користувачу. Вони можуть **не збігатися** з конвертом (класичний механізм списків розсилки, BCC тощо).

### MIME

Базовий SMTP/RFC 5322 підтримує лише 7-бітний ASCII текст. **MIME (Multipurpose Internet Mail Extensions)** розширює формат: дозволяє вкладення, не-ASCII символи, кілька частин листа (текст + HTML + файли) через заголовки `Content-Type`, `Content-Transfer-Encoding`, `MIME-Version`.

## 1.4. Маршрутизація повідомлень електронної пошти

Коли MTA відправника має доставити лист на `black@zone06.net`, він:

1. Бере частину адреси після `@` — домен `zone06.net`.
2. Робить DNS-запит **типу MX** для цього домену.
3. Отримує список поштових серверів з пріоритетами:

```
zone06.net.  MX  10  mail.zone06.net.
```

4. Якщо серверів декілька — **менше число пріоритету = вищий пріоритет** (спочатку пробують сервер з найменшим числом).
5. Резолвить `mail.zone06.net` у IP-адресу (звичайний A-запис).
6. З'єднується з цим сервером по SMTP (порт 25) і передає лист.
7. Якщо основний сервер недоступний — пробує наступний за пріоритетом (це і забезпечує відмовостійкість пошти, аналогічно до primary/secondary DNS з ЛР2).

Якщо MX-запису немає — деякі MTA як fallback використовують A-запис самого домену, але покладатись на це не варто — **MX обов'язковий**, це прямо вимагає завдання (п. 2.4).

**Relay (реле)** — MTA, що пересилає лист не для себе, а далі, до кінцевого отримувача. Щоб уникнути **open relay** (коли будь-хто в Інтернеті може використати твій сервер для розсилки спаму через чужі домени), сервер повинен пересилати листи *для зовнішніх доменів* **тільки від автентифікованих** користувачів — саме тому в завданні написано "обов'язкова автентифікація користувачів при відправленні повідомлень на зовнішні MTA".

## 1.5. POP3 (RFC 1939)

**POP3 (Post Office Protocol v3)** — простий протокол для **завантаження** листів з сервера на клієнт, порт **110** (незашифровано) / **995** (POP3S, TLS одразу).

Модель: **"завантажив і видалив"** (download-and-delete) — за замовчуванням після завантаження лист прибирається з сервера (хоча є опція "залишити копію"). Це означає, що листи фізично зберігаються **на пристрої клієнта**, а не на сервері — незручно, якщо перевіряєш пошту з кількох пристроїв.

### Стани сесії POP3

1. **AUTHORIZATION** — вхід (логін/пароль)
2. **TRANSACTION** — робота зі списком листів (перегляд, завантаження, позначення на видалення)
3. **UPDATE** — фактичне видалення позначених листів при виході

### Основні команди

| Команда | Призначення |
|---|---|
| `USER ім'я` | вказати ім'я користувача |
| `PASS пароль` | вказати пароль |
| `STAT` | кількість листів і загальний розмір скриньки |
| `LIST` | список листів з номерами й розмірами |
| `RETR n` | завантажити лист номер `n` |
| `DELE n` | позначити лист `n` на видалення (видалиться фактично лише після `QUIT`) |
| `RSET` | скасувати всі позначки на видалення в цій сесії |
| `QUIT` | завершити сесію (і застосувати видалення) |

## 1.6. IMAP (RFC 3501)

**IMAP (Internet Message Access Protocol)** — сучасніший протокол, порт **143** (незашифровано) / **993** (IMAPS, TLS одразу).

Ключова відмінність від POP3: листи **залишаються і керуються на сервері**. Клієнт бачить структуру папок (Inbox, Sent, Trash, довільні власні папки), може позначати листи прочитаними/непрочитаними, переміщати між папками — і все це синхронізується між усіма пристроями, бо стан зберігається централізовано на сервері.

### Основні команди (у IMAP кожна команда клієнта має тег-ідентифікатор, наприклад `a1`)

| Команда | Призначення |
|---|---|
| `a1 LOGIN user pass` | автентифікація |
| `a2 SELECT INBOX` | відкрити (вибрати) конкретну папку для роботи |
| `a3 FETCH 1 BODY[]` | завантажити вміст конкретного листа |
| `a4 STORE 1 +FLAGS \Seen` | змінити прапорець листа (наприклад, позначити прочитаним) |
| `a5 LIST "" "*"` | отримати список усіх папок |
| `a6 LOGOUT` | завершити сесію |

**Порівняння POP3 vs IMAP (контрольне питання 4.5 по суті):** POP3 простіший, орієнтований на один пристрій, "тягне листи собі й видаляє з сервера". IMAP складніший, тримає стан на сервері, підтримує кілька папок і синхронізацію між багатьма пристроями — тому саме IMAP є стандартом де-факто для сучасних поштових клієнтів.

## 1.7. TLS у пошті

Є два різних способи "додати шифрування" — і в лабораторній треба реалізувати саме перший (STARTTLS), бо саме so вимагають стандартні порти SMTP submission/POP3/IMAP:

* **STARTTLS (opportunistic/explicit TLS)** — з'єднання **починається як звичайний нешифрований** текстовий діалог на стандартному порту (25 для SMTP, 110 для POP3, 143 для IMAP), і лише після команди `STARTTLS` з'єднання "перемикається" на шифрування, продовжуючи той самий TCP-канал.
* **Implicit TLS (SMTPS/POP3S/IMAPS)** — шифрування вмикається **одразу**, ще до будь-якого тексту протоколу, на окремих виділених портах: **465** (SMTPS), **995** (POP3S), **993** (IMAPS).

Для того, щоб клієнт довіряв сертифікату сервера, сертифікат має бути підписаний якимось **центром сертифікації (CA)**, якому клієнт довіряє. У реальному Інтернеті це публічні CA (Let's Encrypt тощо); у лабораторії ми **самі створюємо свій CA** і самі ж додаємо його сертифікат у список довірених на всіх машинах (п. 2.2 завдання) — тоді наші власні, "саморобні" сертифікати серверів будуть прийматись без попереджень.

**Ланцюжок довіри:**
```
CA (self-signed, CN=color CA)
  │  підписує своїм приватним ключем
  ▼
Сертифікат mail.color.edu  ──┐
Сертифікат mail.zone06.net ──┴── обидва підписані одним CA
```
Клієнт, який довіряє CA, автоматично довіряє будь-якому сертифікату, підписаному цим CA — саме тому досить один раз додати сертифікат CA у довірені, а не кожен серверний сертифікат окремо.

## 1.8. Автентифікація (SMTP AUTH / SASL)

**SASL (Simple Authentication and Security Layer)** — загальний фреймворк автентифікації, який SMTP/POP3/IMAP використовують через команду `AUTH`. Найпростіші механізми:

* `AUTH PLAIN` — логін і пароль передаються разом одним Base64-рядком (`\0user\0password`, закодовано).
* `AUTH LOGIN` — логін і пароль запитуються окремими кроками, кожен теж у Base64.

**Важливо:** Base64 — це **не шифрування**, а лише кодування. Тому `AUTH` без попереднього `STARTTLS` передає пароль практично у відкритому вигляді — саме тому автентифікацію дозволяють лише **після** встановлення TLS.

На практиці в Postfix автентифікацію реалізує Dovecot через **`dovecot SASL`** — тобто Postfix не перевіряє паролі сам, а звертається до Dovecot через сокет, а Dovecot вже перевіряє пароль системного (або віртуального) користувача. Це та сама технічна причина, чому Dovecot потрібно налаштувати **до або одночасно** з автентифікацією в Postfix.

---

# ЧАСТИНА 2. ПОКРОКОВЕ ВИКОНАННЯ (варіант 6)

## Крок 0. План дій

1. На VM1 створити свій CA і сертифікати для `mail.color.edu` і `mail.zone06.net`.
2. Додати CA у довірені на VM1 та VM2.
3. Додати MX/A-записи в DNS-зони з ЛР2 (`color.edu`, `zone06.net`) — це робимо на VM1 (master), бо саме там лежать файли зон.
4. Встановити й налаштувати Postfix на VM1 (домен `color.edu`) і на VM2 (домен `zone06.net`): прийом пошти, TLS, обов'язкова автентифікація при відправленні.
5. Створити поштові скриньки (системних користувачів) red/green/blue на VM1, black/white на VM2.
6. Встановити й налаштувати Dovecot на обох — POP3S/IMAPS + SASL для Postfix.
7. Перевірити все: telnet (звичайний SMTP-діалог), openssl s_client (STARTTLS, сертифікат), swaks (лист з автентифікацією), забір листа по POP3/IMAP, аналіз заголовків, tcpdump.

---

## Крок 1. Створення CA та сертифікатів (на VM1)

### 1.1. Каталог та приватний ключ CA

```bash
sudo mkdir -p /etc/ssl/colorCA/{private,certs}
cd /etc/ssl/colorCA
sudo openssl genrsa -out private/ca.key.pem 4096
sudo chmod 400 private/ca.key.pem
```

| Частина команди | Що означає |
|---|---|
| `genrsa` | згенерувати приватний RSA-ключ |
| `-out private/ca.key.pem` | куди зберегти ключ |
| `4096` | довжина ключа в бітах (більше = надійніше, але повільніше; 4096 — стандартно для CA) |
| `chmod 400` | лише власник може читати файл, ніхто більше — приватний ключ CA критично важливий |

### 1.2. Самопідписаний сертифікат CA

```bash
sudo openssl req -x509 -new -nodes -key private/ca.key.pem -sha256 -days 3650 \
  -subj "/C=UA/ST=Ukraine/L=Kyiv/O=KPI/OU=OT/CN=color CA" \
  -out certs/ca.cert.pem
```

| Прапорець | Що означає |
|---|---|
| `req` | режим "запит на сертифікат" (тут одразу створюємо і сертифікат) |
| `-x509` | замість звичайного CSR одразу згенерувати **готовий сертифікат** (бо CA підписує сам себе) |
| `-new` | новий запит |
| `-nodes` | **no DES** — не шифрувати приватний ключ паролем (спрощує автоматизацію в лабораторній; у продакшені так не роблять) |
| `-key private/ca.key.pem` | яким ключем підписувати |
| `-sha256` | алгоритм хешування для підпису |
| `-days 3650` | термін дії сертифіката (10 років — це CA, довгостроковий) |
| `-subj "..."` | Subject/Issuer сертифіката — **точно за варіантом**: C=UA, ST=Ukraine, L=Kyiv, O=KPI, OU=OT, CN=color CA. Для самопідписаного CA Issuer завжди дорівнює Subject |
| `-out certs/ca.cert.pem` | готовий файл сертифіката CA (це саме той файл, який далі додаємо у довірені) |

### 1.3. Сертифікат для `mail.color.edu`

```bash
sudo mkdir -p /etc/ssl/colorCA/server-color
cd /etc/ssl/colorCA/server-color

# ключ сервера
sudo openssl genrsa -out mail.color.edu.key.pem 2048

# запит на підпис (CSR)
sudo openssl req -new -key mail.color.edu.key.pem \
  -subj "/C=UA/ST=Ukraine/L=Kyiv/O=KPI/OU=OT/CN=mail.color.edu" \
  -out mail.color.edu.csr.pem

# підписуємо своїм CA
sudo openssl x509 -req -in mail.color.edu.csr.pem \
  -CA /etc/ssl/colorCA/certs/ca.cert.pem -CAkey /etc/ssl/colorCA/private/ca.key.pem \
  -CAcreateserial -sha256 -days 825 \
  -out mail.color.edu.cert.pem
```

| Прапорець | Що означає |
|---|---|
| `-CA ...ca.cert.pem` | яким сертифікатом підписуємо (нашого CA) |
| `-CAkey ...ca.key.pem` | яким приватним ключем CA підписуємо |
| `-CAcreateserial` | створити файл із серійними номерами виданих сертифікатів (щоб кожен наступний сертифікат мав унікальний серійник) |
| `-days 825` | термін дії серверного сертифіката (типове практичне обмеження — до ~2 років) |

Важливо: **CN серверного сертифіката має точно збігатись з іменем, за яким до сервера звертаються** (`mail.color.edu`), інакше клієнти видаватимуть попередження про невідповідність імені.

### 1.4. Сертифікат для `mail.zone06.net` (теж на VM1, тому що тут CA)

```bash
sudo mkdir -p /etc/ssl/colorCA/server-zone06
cd /etc/ssl/colorCA/server-zone06

sudo openssl genrsa -out mail.zone06.net.key.pem 2048

sudo openssl req -new -key mail.zone06.net.key.pem \
  -subj "/C=UA/ST=Ukraine/L=Kyiv/O=KPI/OU=OT/CN=mail.zone06.net" \
  -out mail.zone06.net.csr.pem

sudo openssl x509 -req -in mail.zone06.net.csr.pem \
  -CA /etc/ssl/colorCA/certs/ca.cert.pem -CAkey /etc/ssl/colorCA/private/ca.key.pem \
  -CAcreateserial -sha256 -days 825 \
  -out mail.zone06.net.cert.pem
```

### 1.5. Перевірка сертифіката

```bash
openssl x509 -in mail.color.edu.cert.pem -noout -text | head -20
openssl verify -CAfile /etc/ssl/colorCA/certs/ca.cert.pem mail.color.edu.cert.pem
```

Очікується: `mail.color.edu.cert.pem: OK`.

### 1.6. Перенести сертифікат зони2 та CA на VM2

```bash
# на VM1
scp /etc/ssl/colorCA/certs/ca.cert.pem \
    /etc/ssl/colorCA/server-zone06/mail.zone06.net.cert.pem \
    /etc/ssl/colorCA/server-zone06/mail.zone06.net.key.pem \
    student@192.168.106.20:/tmp/
```

На VM2:

```bash
sudo mkdir -p /etc/ssl/colorCA/certs /etc/ssl/mail
sudo mv /tmp/ca.cert.pem /etc/ssl/colorCA/certs/
sudo mv /tmp/mail.zone06.net.cert.pem /etc/ssl/mail/
sudo mv /tmp/mail.zone06.net.key.pem  /etc/ssl/mail/
sudo chmod 400 /etc/ssl/mail/mail.zone06.net.key.pem
```

### 1.7. Додати CA у довірені на обох ВМ

```bash
sudo cp /etc/ssl/colorCA/certs/ca.cert.pem /usr/local/share/ca-certificates/colorCA.crt
sudo update-ca-certificates
```

`update-ca-certificates` перечитує всі файли з `/usr/local/share/ca-certificates/` і додає їх у загальносистемне сховище довірених CA (`/etc/ssl/certs/ca-certificates.crt`) — після цього і `openssl`, і Postfix, і Dovecot довірятимуть нашим сертифікатам без попереджень.

---

## Крок 2. Оновлення DNS-зон (на VM1, master з ЛР2)

Додаємо A-запис на `mail` та MX-запис у файл `color.edu` (**обов'язково збільшуємо Serial**, інакше VM2 як DNS-slave не підхопить зміни):

```bash
sudo nano /etc/bind/zones/db.color.edu
```

```dns
$TTL    3600
$ORIGIN color.edu.
@       IN      SOA     ns1.color.edu. admin.color.edu. (
                        2026093001      ; Serial (збільшено!)
                        3600 900 604800 86400 )
@       IN      NS      ns1.color.edu.
@       IN      NS      ns2.color.edu.
ns1     IN      A       192.168.106.10
ns2     IN      A       192.168.106.20
;
; ----- Поштовий сервер -----
@       IN      MX      10      mail.color.edu.
mail    IN      A       192.168.106.10
;
; ----- Ресурси зони (варіант 6, з ЛР2) -----
red     IN      A       192.168.22.10
green   IN      A       192.168.22.20
blue    IN      A       192.168.22.30
black   IN      A       192.168.22.40
white   IN      A       192.168.22.50
ws1     IN      CNAME   red
ws2     IN      CNAME   green
ws3     IN      CNAME   blue
ws4     IN      CNAME   black
ws5     IN      CNAME   white
```

Пояснення нових рядків:
* `@ IN MX 10 mail.color.edu.` — "для домену `color.edu` поштовий сервер — `mail.color.edu`, пріоритет 10". `@` тут означає сам домен `color.edu` (тобто листи саме на адреси `xxx@color.edu`).
* `mail IN A 192.168.106.10` — прив'язуємо ім'я `mail.color.edu` до реальної IP-адреси сервера (це та адреса, куди інші MTA фактично підключатимуться по SMTP).

Аналогічно редагуємо другу зону:

```bash
sudo nano /etc/bind/zones/db.zone06.net
```

```dns
$TTL    3600
$ORIGIN zone06.net.
@       IN      SOA     ns1.zone06.net. admin.zone06.net. (
                        2026093001      ; Serial (збільшено!)
                        3600 900 604800 86400 )
@       IN      NS      ns1.zone06.net.
@       IN      NS      ns2.zone06.net.
ns1     IN      A       192.168.106.10
ns2     IN      A       192.168.106.20
;
@       IN      MX      10      mail.zone06.net.
mail    IN      A       192.168.106.20
;
red     IN      A       192.168.22.10
green   IN      A       192.168.22.20
blue    IN      A       192.168.22.30
black   IN      A       192.168.22.40
white   IN      A       192.168.22.50
ws1     IN      CNAME   red
ws2     IN      CNAME   green
ws3     IN      CNAME   blue
ws4     IN      CNAME   black
ws5     IN      CNAME   white
```

Перевірка й застосування:

```bash
sudo named-checkzone color.edu   /etc/bind/zones/db.color.edu
sudo named-checkzone zone06.net  /etc/bind/zones/db.zone06.net
sudo rndc reload

# перевірка з будь-якої машини
dig @192.168.106.10 color.edu   MX +short     # 10 mail.color.edu.
dig @192.168.106.10 mail.color.edu A +short   # 192.168.106.10
dig @192.168.106.20 zone06.net  MX +short     # має підхопитись через slave-transfer
```

---

## Крок 3. Встановлення Postfix та Dovecot (на обох ВМ)

```bash
sudo apt update
sudo apt install -y postfix dovecot-core dovecot-imapd dovecot-pop3d mailutils
```

Під час встановлення Postfix запитає тип конфігурації — вибрати **"Internet Site"**, а "System mail name" вказати відповідно до машини: на VM1 — `color.edu`, на VM2 — `zone06.net`.

---

## Крок 4. Налаштування Postfix на VM1 (домен `color.edu`)

### 4.1. Основний файл `main.cf`

```bash
sudo nano /etc/postfix/main.cf
```

Ключові параметри (додати/змінити):

```conf
# ---------- Базова ідентифікація ----------
myhostname = mail.color.edu
mydomain = color.edu
myorigin = $mydomain
mydestination = $myhostname, color.edu, localhost

# ---------- TLS для вхідних з'єднань (SMTP-сервер) ----------
smtpd_tls_cert_file = /etc/ssl/colorCA/server-color/mail.color.edu.cert.pem
smtpd_tls_key_file  = /etc/ssl/colorCA/server-color/mail.color.edu.key.pem
smtpd_tls_CAfile    = /etc/ssl/colorCA/certs/ca.cert.pem
smtpd_use_tls = yes
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes

# ---------- TLS для вихідних з'єднань (SMTP-клієнт, коли ми пересилаємо далі) ----------
smtp_tls_CAfile = /etc/ssl/colorCA/certs/ca.cert.pem
smtp_tls_security_level = may

# ---------- Автентифікація (SASL через Dovecot) ----------
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes

# ---------- Правила прийому пошти ----------
smtpd_relay_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

Розбір кожного параметра:

| Параметр | Що означає |
|---|---|
| `myhostname` | повне ім'я цього поштового сервера (те саме, що ми поставили в сертифікат і в MX) |
| `mydomain` | домен, за який відповідає цей сервер |
| `myorigin` | який домен підставляти в адресу відправника, якщо лист відправлено без повного домену |
| `mydestination` | **список доменів, для яких цей сервер є кінцевим отримувачем** — саме сюди приходять локальні листи (наприклад, на `red@color.edu`), а не пересилаються далі |
| `smtpd_tls_cert_file` / `smtpd_tls_key_file` | сертифікат і приватний ключ, які сервер показує клієнтам при `STARTTLS` |
| `smtpd_tls_CAfile` | сертифікат CA, яким довіряти для перевірки клієнтських сертифікатів (якщо застосовується) |
| `smtpd_use_tls = yes` | увімкнути підтримку TLS взагалі |
| `smtpd_tls_security_level = may` | TLS **доступний, але не обов'язковий** для звичайного вхідного з'єднання (щоб зовнішні MTA без TLS теж могли достукатись — типова практика для порту 25); для власного submission (587) TLS зробимо обов'язковим окремо (крок 4.2) |
| `smtpd_tls_auth_only = yes` | дозволяти команду `AUTH` **тільки після** `STARTTLS` — критично важливо, інакше пароль піде у відкритому вигляді |
| `smtpd_sasl_type = dovecot` | автентифікацію робити через Dovecot (а не через власний механізм Postfix) |
| `smtpd_sasl_path = private/auth` | шлях до Unix-сокета, через який Postfix спілкується з Dovecot SASL (відносно `queue_directory`, реально це `/var/spool/postfix/private/auth`) |
| `smtpd_sasl_auth_enable = yes` | увімкнути можливість автентифікації взагалі |
| `smtpd_relay_restrictions` | **найважливіше правило безпеки**: дивись нижче |

**Пояснення `smtpd_relay_restrictions` по порядку (правила перевіряються зверху вниз, перше співпадіння вирішує):**

1. `permit_mynetworks` — дозволити реле без обмежень, якщо запит іде з "довіреної" локальної мережі (`mynetworks`).
2. `permit_sasl_authenticated` — дозволити реле, якщо клієнт **успішно автентифікувався**.
3. `reject_unauth_destination` — **все інше відхилити**. Саме цей рядок і перетворює сервер із "відкритого релею" (open relay) на безпечний — чужі неавтентифіковані клієнти не зможуть слати через нас пошту на зовнішні домени.

### 4.2. Окремий порт 587 (submission) з обов'язковим TLS+AUTH

```bash
sudo nano /etc/postfix/master.cf
```

Знайти й розкоментувати (або додати) блок `submission`:

```conf
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
```

| Параметр | Що означає |
|---|---|
| `submission inet ...` | новий "сервіс" Postfix, що слухає порт 587 (ім'я `submission` зіставляється з портом через `/etc/services`) |
| `smtpd_tls_security_level=encrypt` | тут TLS **обов'язковий** (на відміну від `may` на 25-му порту) — саме тут виконується вимога "TLS для передачі повідомлень" від MUA |
| `smtpd_relay_restrictions=permit_sasl_authenticated,reject` | на цьому порту приймаємо пошту **тільки** від автентифікованих користувачів, все інше — відмова. Це і є "обов'язкова автентифікація при відправленні" з п. 2.3 |

### 4.3. Створення поштових скриньок (системні користувачі)

```bash
sudo useradd -m -s /usr/sbin/nologin red
sudo useradd -m -s /usr/sbin/nologin green
sudo useradd -m -s /usr/sbin/nologin blue
sudo passwd red      # задати пароль
sudo passwd green
sudo passwd blue
```

| Прапорець | Що означає |
|---|---|
| `-m` | створити домашню директорію користувача (там за замовчуванням Dovecot зберігатиме листи, `~/Maildir`) |
| `-s /usr/sbin/nologin` | заборонити цьому користувачу **логінитись у систему через SSH/консоль** — це ж лише поштова скринька, не потрібен повноцінний shell-доступ |

### 4.4. Перезапуск

```bash
sudo postfix check          # перевірка синтаксису конфігурації
sudo systemctl restart postfix
sudo systemctl status postfix
```

`postfix check` — аналог `named-checkconf` для Postfix: перевіряє права доступу до файлів і синтаксис `main.cf`/`master.cf`, нічого не виводить при успіху.

---

## Крок 5. Налаштування Postfix на VM2 (домен `zone06.net`)

Аналогічно, з іншими значеннями:

```conf
myhostname = mail.zone06.net
mydomain = zone06.net
myorigin = $mydomain
mydestination = $myhostname, zone06.net, localhost

smtpd_tls_cert_file = /etc/ssl/mail/mail.zone06.net.cert.pem
smtpd_tls_key_file  = /etc/ssl/mail/mail.zone06.net.key.pem
smtpd_tls_CAfile    = /etc/ssl/colorCA/certs/ca.cert.pem
smtpd_use_tls = yes
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes

smtp_tls_CAfile = /etc/ssl/colorCA/certs/ca.cert.pem
smtp_tls_security_level = may

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes

smtpd_relay_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

`master.cf` — той самий блок `submission`, що й на VM1.

Користувачі:

```bash
sudo useradd -m -s /usr/sbin/nologin black
sudo useradd -m -s /usr/sbin/nologin white
sudo passwd black
sudo passwd white
```

```bash
sudo postfix check
sudo systemctl restart postfix
```

---

## Крок 6. Налаштування Dovecot (на обох ВМ, значення нижче — для VM1; на VM2 підставити свої шляхи до сертифіката)

### 6.1. Протоколи та формат зберігання листів

```bash
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

```conf
mail_location = maildir:~/Maildir
```

`maildir` — формат зберігання, де кожен лист — окремий файл (на відміну від `mbox`, де всі листи в одному файлі). Maildir надійніший при паралельному доступі й саме він за замовчуванням для сучасних систем.

### 6.2. Список активних протоколів

```bash
sudo nano /etc/dovecot/dovecot.conf
```

```conf
protocols = imap pop3
```

### 6.3. TLS

```bash
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

```conf
ssl = required
ssl_cert = </etc/ssl/colorCA/server-color/mail.color.edu.cert.pem
ssl_key  = </etc/ssl/colorCA/server-color/mail.color.edu.key.pem
ssl_ca   = </etc/ssl/colorCA/certs/ca.cert.pem
```

| Параметр | Що означає |
|---|---|
| `ssl = required` | TLS обов'язковий для будь-якого логіну (і implicit-TLS порти 993/995, і STARTTLS на 143/110 без шифрування логін не пропустять) |
| `<` перед шляхом | синтаксис Dovecot — означає "прочитати вміст файлу" (а не просто рядок-шлях) |

### 6.4. Порти POP3/IMAP

```bash
sudo nano /etc/dovecot/conf.d/10-master.conf
```

Переконатись, що є (зазвичай увімкнено за замовчуванням):

```conf
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

service pop3-login {
  inet_listener pop3 {
    port = 110
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}
```

### 6.5. SASL-сокет для Postfix (щоб Postfix міг перевіряти паролі через Dovecot)

```bash
sudo nano /etc/dovecot/conf.d/10-master.conf
```

Додати всередину блоку `service auth { ... }`:

```conf
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
```

| Параметр | Що означає |
|---|---|
| `unix_listener /var/spool/postfix/private/auth` | створити Unix-сокет саме за цим шляхом — це той самий `smtpd_sasl_path = private/auth`, який ми вказали в `main.cf` Postfix |
| `mode = 0660`, `user/group = postfix` | права доступу — щоб саме процес Postfix міг писати/читати цей сокет |

### 6.6. Метод автентифікації (перевірка через системних користувачів Linux)

```bash
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

```conf
disable_plaintext_auth = yes
auth_mechanisms = plain login
!include auth-system.conf.ext
```

`disable_plaintext_auth = yes` — забороняє передавати логін/пароль відкритим текстом без TLS (узгоджено з `ssl = required` вище). `auth-system.conf.ext` — стандартний модуль, що перевіряє логін/пароль через `/etc/passwd` + `/etc/shadow` (тобто ті самі паролі, що ми задали через `passwd red` тощо).

### 6.7. Перезапуск і перевірка

```bash
sudo doveconf -n > /dev/null    # перевірка синтаксису, виводить лише активні параметри
sudo systemctl restart dovecot
sudo systemctl status dovecot
sudo ss -lnptu | grep -E ':143|:993|:110|:995'
```

---

## Крок 7. Перевірка роботи

### 7.1. Звичайний SMTP-діалог вручну через telnet (без TLS, порт 25)

```bash
telnet mail.color.edu 25
```

```
EHLO test.local
MAIL FROM:<red@color.edu>
RCPT TO:<green@color.edu>
DATA
Subject: Привіт

Це тестовий лист.
.
QUIT
```

Так наочно видно "сирий" протокол — саме це і є матеріал для п. 2.6 завдання (аналіз протокольного обміну).

### 7.2. STARTTLS вручну через openssl (перевірка сертифіката й шифрування)

```bash
openssl s_client -starttls smtp -connect mail.color.edu:587 -CAfile /etc/ssl/colorCA/certs/ca.cert.pem
```

| Прапорець | Що означає |
|---|---|
| `-starttls smtp` | сказати openssl спочатку провести звичайний SMTP-діалог до команди `STARTTLS`, а потім переключитись на TLS (без цього прапорця openssl спробує TLS одразу — і зв'язок не встановиться, бо порт очікує спочатку звичайний текст) |
| `-connect mail.color.edu:587` | адреса й порт сервера |
| `-CAfile ...` | яким CA перевіряти сертифікат сервера — якщо все правильно налаштовано, у виводі буде `Verify return code: 0 (ok)` |

Після встановлення TLS-сесії можна вручну ввести:

```
EHLO test.local
AUTH LOGIN
<base64 логіна>
<base64 пароля>
MAIL FROM:<red@color.edu>
RCPT TO:<black@zone06.net>
DATA
...
```

Base64 логіна/пароля отримати так: `echo -n "red" | base64`.

### 7.3. Швидка перевірка через swaks (одна команда замість ручного діалогу)

```bash
swaks --to green@color.edu --from red@color.edu \
  --server mail.color.edu --port 587 \
  --auth LOGIN --auth-user red --auth-password 'ПАРОЛЬ' \
  --tls
```

| Прапорець | Що означає |
|---|---|
| `--to` / `--from` | envelope-отримувач / envelope-відправник |
| `--server` / `--port` | куди підключатись |
| `--auth LOGIN` | механізм автентифікації |
| `--auth-user` / `--auth-password` | облікові дані |
| `--tls` | вимагати STARTTLS перед відправкою (аналог ручного кроку 7.2, тільки автоматично) |

`swaks` сам покаже повний протокольний обмін (весь SMTP-діалог із кодами) — це зручно вставити прямо у звіт як доказ роботи.

### 7.4. Перевірка міждоменної пересилки (VM1 → VM2)

Відправ лист з `red@color.edu` на `black@zone06.net` (як у прикладі вище) і подивись логи на обох машинах:

```bash
# на VM1 (відправник)
sudo tail -f /var/log/mail.log
```

Побачиш рядки на кшталт `status=sent (250 2.0.0 Ok: queued as ...)` — підтвердження, що лист прийнято сервером VM2.

```bash
# на VM2 (отримувач)
sudo tail -f /var/log/mail.log
ls -la /home/black/Maildir/new/     # тут з'явиться файл нового листа
```

### 7.5. Забір листа по POP3 (порт 995, TLS)

```bash
openssl s_client -connect mail.zone06.net:995 -CAfile /etc/ssl/colorCA/certs/ca.cert.pem -quiet
```

Далі вручну:

```
USER black
PASS ПАРОЛЬ
STAT
LIST
RETR 1
QUIT
```

### 7.6. Забір листа по IMAP (порт 993, TLS)

```bash
openssl s_client -connect mail.zone06.net:993 -CAfile /etc/ssl/colorCA/certs/ca.cert.pem -quiet
```

```
a1 LOGIN black ПАРОЛЬ
a2 SELECT INBOX
a3 FETCH 1 BODY[]
a4 LOGOUT
```

### 7.7. Аналіз заголовків отриманого листа

```bash
cat /home/black/Maildir/new/*
```

Знайти рядки `Received:` — їх буде **два**: один додав Postfix на VM1 (передача від MUA до MTA1), другий — Postfix на VM2 (прийом від MTA1). Читаючи знизу вгору, відновлюється шлях: клієнт → mail.color.edu → mail.zone06.net → скринька black.

### 7.8. Перехоплення трафіку (п. 2.6–2.7 завдання)

```bash
sudo tcpdump -i enp0s8 -n port 25 -A     # звичайний SMTP, видно текст команд
sudo tcpdump -i enp0s8 -n port 587       # submission, видно момент переходу в шифр після STARTTLS
sudo tcpdump -i enp0s8 -n port 993       # IMAP TLS — весь трафік вже зашифрований, видно лише TLS handshake
```

`-A` — виводити вміст пакетів у ASCII, щоб бачити самі SMTP-команди в незашифрованому трафіку. На порту 993/995/зашифрованому 587-після-STARTTLS текст команд побачити вже не вдасться — і це саме та демонстрація, яку варто показати в звіті як доказ, що TLS справді працює.

---

## Крок 8. Типові проблеми

| Симптом | Причина |
|---|---|
| `Relay access denied` | лист адресовано на зовнішній домен, але клієнт не пройшов `permit_sasl_authenticated`/`permit_mynetworks` — перевір `smtpd_relay_restrictions` |
| `Certificate verify failed` в openssl | не виконано `update-ca-certificates`, або CN сертифіката не збігається з іменем, до якого звертаєшся |
| Пошта не йде на інший домен | немає MX-запису, або він не оновився (не збільшено Serial / не пройшов zone transfer) |
| `SASL authentication failed` | Postfix не бачить сокет Dovecot — перевір `smtpd_sasl_path`, права на `/var/spool/postfix/private/auth`, чи Postfix "бачить" chroot (сокет має бути всередині `/var/spool/postfix`, бо Postfix може працювати в chroot) |
| Лист "зависає" в черзі | `mailq` — подивитись чергу; `postfix flush` — примусово спробувати доставити |
| Dovecot не стартує | `doveconf -n` покаже помилку конфігурації; перевір шляхи до сертифікатів |

---

# ЧАСТИНА 3. ВІДПОВІДІ НА КОНТРОЛЬНІ ЗАПИТАННЯ

**4.1. Архітектура електронної пошти.**
Складається з ролей: MUA (клієнт користувача) → MSA (приймає лист від MUA, автентифікує, зазвичай порт 587) → MTA (пересилає лист по SMTP між серверами, порт 25, кінцевий MTA — той, чий домен вказано в адресі отримувача) → MDA (кладе лист у скриньку конкретного користувача) → POP3/IMAP-сервер (надає клієнту доступ до вже прийнятих листів). SMTP використовується лише для відправки/пересилки; забір пошти клієнтом — окремі протоколи POP3/IMAP.

**4.2. Призначення та принципи роботи протоколу SMTP.**
SMTP (RFC 5321) — текстовий протокол поверх TCP для передачі листів від клієнта до сервера і між серверами (порти 25/587/465). Працює за принципом команда-відповідь: клієнт надсилає команди (`EHLO`, `MAIL FROM`, `RCPT TO`, `DATA`, `QUIT`), сервер відповідає числовими кодами (2xx успіх, 4xx тимчасова помилка, 5xx постійна помилка). Підтримує розширення через `EHLO` (STARTTLS, AUTH, SIZE тощо).

**4.3. Формат поштового повідомлення.**
Лист (RFC 5322) складається з блоку заголовків і тіла, розділених порожнім рядком. Заголовки: `From`, `To`, `Subject`, `Date`, `Message-ID`, `Received` (додається кожним MTA на шляху листа, дозволяє відстежити маршрут). Розширення MIME додає підтримку не-ASCII вмісту, вкладень і багаточастинних повідомлень (текст+HTML+файли) через `Content-Type`/`Content-Transfer-Encoding`.

**4.4. Маршрутизація повідомлень електронної пошти.**
Визначається DNS-записами **MX**: MTA-відправник запитує MX для домену отримувача, отримує список поштових серверів з пріоритетами (менше число — вищий пріоритет), резолвить їхні A-записи і встановлює SMTP-з'єднання з сервером із найвищим доступним пріоритетом. Проміжні MTA можуть виступати релеєм, пересилаючи лист далі; щоб уникнути відкритого релею (open relay) для спаму, пересилку на зовнішні домени дозволяють лише автентифікованим користувачам.

**4.5. Призначення та принципи роботи протоколів POP3 та IMAP.**
Обидва призначені для забору вже прийнятих листів клієнтом, на відміну від SMTP (лише відправка). POP3 (RFC 1939, порт 110/995) — проста модель "завантажив і видалив", листи зберігаються локально у клієнта, стани сесії: AUTHORIZATION → TRANSACTION → UPDATE. IMAP (RFC 3501, порт 143/993) — листи й папки зберігаються і керуються на сервері, стан синхронізується між усіма пристроями клієнта, підтримує множинні папки та прапорці повідомлень (прочитано/непрочитано тощо). Тому IMAP підходить для доступу з кількох пристроїв, POP3 — для простого одноразового завантаження.
