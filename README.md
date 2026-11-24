<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ | Бондаренко Вадим Олександрович |
| Група | ЗКБТЗІ-21 |
| Ідентифікатор | se-201-02 |
| Дата виконання | 23.11.2026 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024) |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | `origin.se-201-02.lab` |
| Імена для сертифіката | `www.se-201-02.lab`, `app.se-201-02.lab` |
| Ваш сервер (IP) | `10.53.1.1` |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
[1] 31
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: origin.se-201-02.lab
SNI:
protocol: HTTP/1.1
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
23:06:46.317479 IP 10.53.1.1.60258 > 10.53.0.80.80: Flags [P.], seq 1:84, ack 1, win 502, options [nop,nop,TS val 1366219007 ecr 784308209], length 83: HTTP: GET / HTTP/1.1
Qn......GET / HTTP/1.1
Host: origin.se-201-02.lab
server: web
Host: origin.se-201-02.lab
```

Вивід кроку 1.2:

```text
[1] 37
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: origin.se-201-02.lab
SNI: origin.se-201-02.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` | Видно (`GET / HTTP/1.1`) | Не видно (`0`) |
| заголовок `Host` | Видно (`Host: origin.se-201-02.lab`) | Не видно (`0`) |
| тіло відповіді (`server: web`) | Видно (`server: web`) | Не видно (`0`) |
| ім'я віддаленого сервера (кількість входжень) | `2` (у заголовку `Host` запиту та в тілі відповіді) | `1` (відкритим текстом у розширенні SNI повідомлення `ClientHello`) |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?

Ім'я сервера передається відкритим текстом у першому повідомленні рукостискання **`ClientHello`** (тип запису `Handshake`), у розширенні **SNI** (`Server Name Indication`, `extension_type=server_name`), полі `server_name`. Це необхідно для того, щоб сервер ще до початку шифрування знав, для якого домену клієнт встановлює з'єднання, і міг обрати відповідний сертифікат.

2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?

Звичайний протокол HTTP працює поверх TCP без криптографічного рівня TLS, тому розширення SNI у ньому відсутнє (ім'я хоста передається лише всередині HTTP-заголовка `Host:`). У випадку HTTPS клієнт `curl` під час TLS-рукостискання передає ім'я хоста в розширенні SNI, а тестовий сервер `web` зчитує це значення з параметрів TLS-сесії та виводить його у тілі відповіді.

3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?

Версію прикладного протоколу визначило розширення TLS **ALPN** (`Application-Layer Protocol Negotiation`). Під час `ClientHello` клієнт `curl` пропонує серверу перелік підтримуваних протоколів (`h2`, `http/1.1`), а сервер під час рукостискання обирає `h2` (HTTP/2). У нешифрованому HTTP механізму ALPN немає, тому використовується стандартний `HTTP/1.1`.

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=331
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Certificate, Length=989
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=75
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)
```

Вивід кроку 2.2 (TLS 1.2):

```text
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=226
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=984
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=112
Received Record
  Content Type = Handshake (22)
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
    NewSessionTicket, Length=230
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = ApplicationData (23)
Sent Record
  Content Type = Alert (21)
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)
```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 | К→С | `Handshake (22)` | — | `ClientHello` | `331` |
| 2 | С→К | `Handshake (22)` | — | `ServerHello` | `118` |
| 3 | С→К | `ChangeCipherSpec (20)` | — | — | — |
| 4 | С→К | `ApplicationData (23)` | `Handshake (22)` | `EncryptedExtensions` | `15` |
| 5 | С→К | `ApplicationData (23)` | `Handshake (22)` | `Certificate` | `989` |
| 6 | С→К | `ApplicationData (23)` | `Handshake (22)` | `CertificateVerify` | `75` |
| 7 | С→К | `ApplicationData (23)` | `Handshake (22)` | `Finished` | `48` |
| 8 | К→С | `ChangeCipherSpec (20)` | — | — | — |
| 9 | К→С | `ApplicationData (23)` | `Handshake (22)` | `Finished` | `48` |
| 10 | К→С | `ApplicationData (23)` | `Alert (21)` | `close_notify` (закриття сесії) | — |

Вивід кроків 2.3 і 2.4:

```text
extension_type=server_name(0), length=25
          0000 - 00 17 00 00 14 6f 72 69-67 69 6e 2e 73 65 2d   .....origin.se-
          000f - 32 30 31 2d 30 32 2e 6c-61 62                  201-02.lab
        extension_type=ec_point_formats(11), length=4
[1] 64
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 70
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.2: 2
```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт | `331` | `226` |
| Номер першого запису, тип якого спостерігач не бачить | `4` (`EncryptedExtensions`, обгорнутий у `ApplicationData (23)`) | `8` (`Finished` клієнта після `ChangeCipherSpec`) |
| Кількість RTT від ClientHello до першого HTTP-запиту | `1` | `2` |
| Чи видно сертифікат у дампі (результат 2.4) | Ні (`TLS 1.3: 0`) | Так (`TLS 1.2: 2`) |
| Обраний протокол ALPN | `h2` | `h2` |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?

Шифрування починається з четвертого запису (`EncryptedExtensions`), де з'являються рядки `Content Type = ApplicationData (23)` та `Inner Content Type = Handshake (22)`. Зовнішній заголовок запису для пасивного спостерігача завжди має тип `ApplicationData (23)`, тому спостерігач бачить лише зашифровані прикладні дані і не може визначити реальний внутрішній тип повідомлень (`Handshake` або `Alert`).

2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.

У TLS 1.3 клієнт не чекає, доки сервер обере параметри шифрування, а одразу генерує та передає свій ефемерний відкритий ключ обміну всередині `ClientHello` у розширенні **`key_share`** (`extension_type=key_share`). Це збільшує розмір `ClientHello` (331 байт проти 226 байт у TLS 1.2), але дозволяє обом сторонам обчислити спільний сесійний ключ уже після першого обміну повідомленнями та скоротити рукостискання на 1 RTT.

3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?

Довжина даних розширення `server_name` складається з трьох службових полів та самого імені хоста: **2 байти** — загальна довжина списку імен (`Server Name List Length`, у гекс-дампі `00 17` = 23 байти), **1 байт** — тип імені (`Name Type = 0x00`, `host_name`), **2 байти** — довжина імені хоста (`Host Name Length`, у гекс-дампі `00 14` = 20 байт) і **$N$ байтів** — ASCII-символи доменного імені ($2 + 1 + 2 + N = 5 + N$). Для імені `origin.se-201-02.lab` (20 символів) загальна довжина розширення становить $5 + 20.

4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?

Запис `ChangeCipherSpec (20)` передається виключно для зворотної сумісності зі старими проміжними мережевими пристроями (middleboxes — фаєрволами, балансувальниками, проксі), які очікують побачити класичний потік записів TLS 1.2 (`ClientHello` → `ServerHello` → `ChangeCipherSpec` → `ApplicationData`) і можуть розірвати з'єднання, якщо зашифровані записи `ApplicationData` з'являться одразу після `ServerHello`.

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 725b16
notBefore=Nov 23 23:51:48 2026 GMT
notAfter=Nov 23 23:51:48 2056 GMT
curl: (60) SSL certificate problem: self-signed certificate
More details here: [https://curl.se/docs/sslcerts.html](https://curl.se/docs/sslcerts.html)

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
2026/11/24 00:01:52 [INFO] [www.se-201-02.lab, app.se-201-02.lab] acme: Obtaining SAN certificate
2026/11/24 00:01:53 [INFO] [app.se-201-02.lab] AuthURL: [https://ca.lab:14000/authZ/frn2kXTTHp25COWvHPNk6zW1NRRSz2M3SUdzF2lzqoY](https://ca.lab:14000/authZ/frn2kXTTHp25COWvHPNk6zW1NRRSz2M3SUdzF2lzqoY)
2026/11/24 00:01:53 [INFO] [www.se-201-02.lab] AuthURL: [https://ca.lab:14000/authZ/s1CHIz9BYv8d3SKC8Lufhfxr-2e9ppnkN280OH3qrwo](https://ca.lab:14000/authZ/s1CHIz9BYv8d3SKC8Lufhfxr-2e9ppnkN280OH3qrwo)
2026/11/24 00:01:53 [INFO] [www.se-201-02.lab] acme: use http-01 solver
2026/11/24 00:01:53 [INFO] [app.se-201-02.lab] acme: use http-01 solver
2026/11/24 00:01:53 [INFO] [www.se-201-02.lab] acme: Trying to solve HTTP-01
2026/11/24 00:01:56 [INFO] [www.se-201-02.lab] The server validated our request
2026/11/24 00:01:56 [INFO] [app.se-201-02.lab] acme: Trying to solve HTTP-01
2026/11/24 00:02:03 [INFO] [app.se-201-02.lab] The server validated our request
2026/11/24 00:02:03 [INFO] [www.se-201-02.lab, app.se-201-02.lab] acme: Validations succeeded; requesting certificates
2026/11/24 00:02:03 [INFO] [www.se-201-02.lab] Server responded with a certificate.
10.53.0.10 - - [24/Nov/2026:00:01:53 +0000] "GET /.well-known/acme-challenge/iWZyO_N_9IxzD9EfCXkNeZHy58Fa4BHV9X-tktyJFTs HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [24/Nov/2026:00:01:56 +0000] "GET /.well-known/acme-challenge/N3DxAWCxrfNTjlr5Ajrlsgcie--ko93oacPl_81iLQA HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```

Вивід кроку 3.4:

```text
subject=CN = www.se-201-02.lab
issuer=CN = Pebble Intermediate CA 54c6f7
notBefore=Nov 24 00:02:03 2026 GMT
notAfter=Nov 24 00:02:02 2031 GMT
serial=7C1D9D405D2B1908
X509v3 Subject Alternative Name: 
    DNS:www.se-201-02.lab, DNS:app.se-201-02.lab
subject=CN = Pebble Intermediate CA 54c6f7
issuer=CN = Pebble Root CA 725b16
```

Вивід кроку 3.5:

```text
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/24 00:03:17 [notice] 93#93: signal process started
curl: (60) SSL certificate problem: self-signed certificate
More details here: [https://curl.se/docs/sslcerts.html](https://curl.se/docs/sslcerts.html)

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.se-201-02.lab
   i:CN = Pebble Intermediate CA 54c6f7
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 24 00:02:03 2026 GMT; NotAfter: Nov 24 00:02:02 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```text
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/24 00:03:17 [notice] 101#101: signal process started
server: se-201-02 (https)
Host: www.se-201-02.lab
SNI: www.se-201-02.lab
protocol: TLSv1.3
server: se-201-02 (https)
Host: app.se-201-02.lab
SNI: app.se-201-02.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.se-201-02.lab
   i:CN = Pebble Intermediate CA 54c6f7
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 24 00:02:03 2026 GMT; NotAfter: Nov 24 00:02:02 2031 GMT
 1 s:CN = Pebble Intermediate CA 54c6f7
   i:CN = Pebble Root CA 725b16
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 23 23:51:48 2026 GMT; NotAfter: Nov 23 23:51:48 2056 GMT
---
Verify return code: 0 (ok)  
```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | `CN = Pebble Intermediate CA 54c6f7` |
| Видавець сертифіката видавця | `CN = Pebble Root CA 725b16` |
| Імена в SAN | `DNS:www.se-201-02.lab, DNS:app.se-201-02.lab` |
| Строк дії (від – до) | `Nov 24 00:02:03 2026 GMT` – `Nov 24 00:02:02 2031 GMT` |
| Тип ключа сервера і алгоритм підпису | `id-ecPublicKey, 256 (bit)` і `RSA-SHA256` |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | `1` у кроці 3.5; `2` у кроці 3.6 |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.

Перевірку виконав центр сертифікації ACME (**Pebble**, адреса `10.53.0.10`) методом **HTTP-01**. Клієнт `lego` розмістив унікальні токени перевірки в каталозі `/var/www/acme/.well-known/acme-challenge/`, після чого Pebble зробив HTTP-запити (на порт 80) до обох доменів `www.se-201-02.lab` та `app.se-201-02.lab` і підтвердив їх. Це підтверджують рядки журналу `lego`: `[www.se-201-02.lab] The server validated our request`, `[app.se-201-02.lab] The server validated our request`, а також записи журналу `nginx`: `"GET /.well-known/acme-challenge/iWZyO_N_9IxzD9EfCXkNeZHy58Fa4BHV9X-tktyJFTs HTTP/1.1" 200` та `"GET /.well-known/acme-challenge/N3DxAWCxrfNTjlr5Ajrlsgcie--ko93oacPl_81iLQA HTTP/1.1" 200`.

2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?

Сертифікат сервера підписаний проміжним центром сертифікації (`Pebble Intermediate CA 54c6f7`), а не напряму кореневим (`Pebble Root CA 725b16`). У кроці 3.5 сервер надсилав лише один свій сертифікат без проміжного сертифіката видавця (`Verify return code: 21 (unable to verify the first certificate)`), тому клієнт, маючи в довірених лише кореневий сертифікат `acme-root.crt`, не зміг побудувати безперервний ланцюжок довіри до кореня.

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?

`id-ecPublicKey` означає алгоритм власного відкритого ключа сервера (еліптична криптографія ECDSA з довжиною ключа 256 біт), згенерованого клієнтом `lego`. `RSA-SHA256` означає криптографічний алгоритм, яким проміжний центр сертифікації (`Pebble Intermediate CA`) підписав цей сертифікат своїм закритим ключем RSA. Вони не суперечать одне одному, оскільки стосуються різних ключів: ключ сервера використовує еліптичну криву, а ключ видавця — RSA.

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?

Після перезапуску стенда перевірка сертифіката перестане проходити, оскільки тестовий центр сертифікації Pebble під час кожного запуску генерує нову пару кореневого та проміжного ключів, а старий сертифікат сервера залишається підписаним попереднім проміжним ключем. У реальній інфраструктурі цьому відповідає компрометація або планова ротація ключів центру сертифікації (відкликання старого проміжного чи кореневого сертифіката та перехід на новий ЦС), після чого всі раніше видані сертифікати потребують перевипуску.

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
=== f1.se-201-02.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f1.se-201-02.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: May  7 21:01:18 2026 GMT; NotAfter: Nov  3 21:01:18 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 23 21:01:17 2026 GMT; NotAfter: Nov 22 21:01:17 2031 GMT
---
Verify return code: 10 (certificate has expired)
=== f2.se-201-02.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f2.se-201-02.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 23 21:01:18 2026 GMT; NotAfter: Feb 21 21:01:18 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
=== f3.se-201-02.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f3.se-201-02.lab'
Certificate chain
 0 s:CN = staging.se-201-02.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 23 21:01:18 2026 GMT; NotAfter: Feb 21 21:01:18 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 23 21:01:17 2026 GMT; NotAfter: Nov 22 21:01:17 2031 GMT
---
Verify return code: 0 (ok)
=== f4.se-201-02.lab
server: web, f4
Certificate chain
 0 s:CN = f4.se-201-02.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 23 21:01:18 2026 GMT; NotAfter: Feb 21 21:01:18 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 23 21:01:17 2026 GMT; NotAfter: Nov 22 21:01:17 2031 GMT
---
Verify return code: 0 (ok)
=== f5.se-201-02.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f5.se-201-02.lab
   i:CN = f5.se-201-02.lab
   v:NotBefore: Nov 23 21:01:18 2026 GMT; NotAfter: Feb 21 21:01:18 2027 GMT
---
Verify return code: 18 (self-signed certificate)
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| `f1` | `SSL certificate problem: certificate has expired` | `10 (certificate has expired)` | Перевірка строку дії сертифіката (сертифікат прострочений) | `v:NotBefore: May 7 21:01:18 2026 GMT; NotAfter: Nov 3 21:01:18 2026 GMT` | Перевипустити (оновити) сертифікат сервера в центрі сертифікації та перезавантажити конфігурацію вебсервера |
| `f2` | `SSL certificate problem: unable to get local issuer certificate` | `21 (unable to verify the first certificate)` | Побудова ланцюжка довіри до кореневого ЦС (відсутній проміжний сертифікат видавця) | `0 s:CN = f2.se-201-02.lab` / `i:O = OPISM Lab, CN = OPISM Stand Intermediate CA` (запис `1` у `Certificate chain` відсутній) | Об'єднати сертифікат сервера та проміжний сертифікат видавця в один файл (`fullchain`) і вказати його в конфігурації вебсервера |
| `f3` | `SSL: no alternative certificate subject name matches target host name 'f3.se-201-02.lab'` | `0 (ok)` | Перевірка відповідності імені вузла (`hostname verification` / SAN) | `0 s:CN = staging.se-201-02.lab` | Випустити новий сертифікат, у полі SAN (`Subject Alternative Name`) якого буде вказано домен `f3.se-201-02.lab` |
| `f4` | Помилок немає (`server: web, f4`) | `0 (ok)` | Усі перевірки пройшли успішно | `server: web, f4` та `Verify return code: 0 (ok)` | Виправлення не потрібні |
| `f5` | `SSL certificate problem: self-signed certificate` | `18 (self-signed certificate)` | Перевірка довіри до центру сертифікації (сертифікат самопідписаний) | `0 s:CN = f5.se-201-02.lab` / `i:CN = f5.se-201-02.lab` | Замінити самопідписаний сертифікат на сертифікат, підписаний довіреним центром сертифікації (`OPISM Stand Intermediate CA`) з повним ланцюжком |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?

На сервері `f2.se-201-02.lab` це випадок, коли **сервер не надсилає проміжний сертифікат** (`OPISM Stand Intermediate CA`), хоча сам видавець входить до довіреної інфраструктури стенда.

У виводі `openssl s_client` ці дві несправності відрізняються складом блоку `Certificate chain` і кодом помилки:
* **Неповний ланцюжок (`f2`):** у `Certificate chain` є лише один сертифікат (`0 s:CN = f2...`, `i:O = OPISM Lab, CN = OPISM Stand Intermediate CA`), проміжний сертифікат відсутній, а код перевірки — `Verify return code: 21 (unable to verify the first certificate)`.
* **Недовірений центр сертифікації (`f5` або сторонній ЦС):** у полі `i:` (Issuer) вказано невідомий або самопідписаний центр, який не зводиться до довіреного кореня клієнта; для самопідписаного сертифіката (`f5`) код помилки — `Verify return code: 18 (self-signed certificate)`, а для стороннього кореневого ЦС навіть за наявності повного ланцюжка — `Verify return code: 19 (self-signed certificate in certificate chain)` або `20 (unable to get local issuer certificate)`.

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.

На сервері `f3.se-201-02.lab` сертифікат виданий для `staging.se-201-02.lab`. Команда `openssl s_client` без прапорця `-verify_hostname` перевіряє лише криптографічні підписи, довіру до кореневого ЦС і строк дії сертифіката — оскільки сертифікат `staging.se-201-02.lab` підписаний довіреним ЦС стенда і чинний за часом, `openssl` повертає `Verify return code: 0 (ok)`. Натомість `curl` обов'язково виконує перевірку імені вузла (`hostname verification`), порівнює запитуваний домен `f3.se-201-02.lab` зі списком імен у розширенні `Subject Alternative Name` (SAN), виявляє невідповідність і розриває з'єднання.

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?

На запит `curl -k` відповіли б **усі п'ять серверів (`f1`, `f2`, `f3`, `f4`, `f5`)**, оскільки прапорець `-k` (`--insecure`) повністю вимикає перевірку сертифіката сервера (ігноруються прострочений строк дії у `f1`, неповний ланцюжок у `f2`, невідповідність імені у `f3` та самопідписаний сертифікат у `f5`).

Порівняно з `--cacert` при використанні `-k` повністю **втрачається автентифікація сервера**: з'єднання залишається зашифрованим, але клієнт не має жодної гарантії, що спілкується саме зі справжнім сервером, а не з атакуючим «людиною посередині» (MITM), який підставив власний сертифікат.

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | `2` RTT (`1` DNS + `1` TCP) | `2` RTT (`1` DNS + `1` TCP) | `0` RTT |
| `time_appconnect` | `3` RTT (`1` DNS + `1` TCP + `1` TLS 1.3) | `4` RTT (`1` DNS + `1` TCP + `2` TLS 1.2) | `0` RTT |
| `time_starttransfer` | `4` RTT (`3` RTT + `1` HTTP) | `5` RTT (`4` RTT + `1` HTTP) | `1` RTT (лише HTTP-запит і відповідь) |

Вивід кроків 5.2 і 5.3:

```text
=== 5.2 TLS 1.3 (3 спроби)
namelookup=0.050931 connect=0.076075 appconnect=0.104834 starttransfer=0.130088 total=0.130243 connects=1
namelookup=0.025421 connect=0.050557 appconnect=0.078688 starttransfer=0.104437 total=0.104487 connects=1
namelookup=0.025421 connect=0.050546 appconnect=0.078684 starttransfer=0.104011 total=0.104069 connects=1
=== 5.2 TLS 1.2 (3 спроби)
namelookup=0.025488 connect=0.050620 appconnect=0.104137 starttransfer=0.129431 total=0.129493 connects=1
namelookup=0.025410 connect=0.050576 appconnect=0.105081 starttransfer=0.130466 total=0.130530 connects=1
namelookup=0.025424 connect=0.050558 appconnect=0.105159 starttransfer=0.130471 total=0.130529 connects=1
=== 5.3 Повторне використання з'єднання (3 запити)
namelookup=0.025436 connect=0.050588 appconnect=0.078881 starttransfer=0.104171 total=0.104230 connects=1
server: web
Host: origin.se-201-02.lab
SNI: origin.se-201-02.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
namelookup=0.000055 connect=0.000000 appconnect=0.000000 starttransfer=0.025436 total=0.025473 connects=0
server: web
Host: origin.se-201-02.lab
SNI: origin.se-201-02.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
namelookup=0.000018 connect=0.000000 appconnect=0.000000 starttransfer=0.026179 total=0.026214 connects=0
```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | `25.42` | `50.56` | `78.69` | `104.44` | `104.49` | `1` |
| TLS 1.2, нове | `25.42` | `50.58` | `105.08` | `130.47` | `130.53` | `1` |
| 5.3, перший запит | `25.44` | `50.59` | `78.88` | `104.17` | `104.23` | `1` |
| 5.3, другий запит | `0.06` | `0.00` | `0.00` | `25.44` | `25.47` | `0` |
| 5.3, третій запит | `0.02` | `0.00` | `0.00` | `26.18` | `26.21` | `0` |

RTT, визначений вимірюванням, і з якого значення його визначено:
**1 RTT ≈ 25 мс** Це значення найчистіше визначається за етапом `starttransfer` при повторному використанні з'єднання (`25.44 мс` у другому запиті кроку 5.3, де немає ні DNS, ні TCP, ні TLS, а виконується рівно один обмін HTTP-запит/відповідь), а також за різницею між сусідніми етапами нового з'єднання: `connect − namelookup = 50.56 − 25.42 = 25.14 мс` (1 RTT на TCP-рукостискання) і `starttransfer − appconnect = 104.44 − 78.69 = 25.75 мс` (1 RTT на HTTP-запит).

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?

Вираження медіанних значень через $\text{RTT} \approx 25.4\text{ мс}$:
* **TLS 1.3, нове з'єднання:** `namelookup` $= 1.00\text{ RTT}$ (`25.42 мс`), `connect` $= 1.99\text{ RTT} \approx 2\text{ RTT}$ (`50.56 мс`), `appconnect` $= 3.10\text{ RTT}$ (`78.69 мс`), `starttransfer` $= 4.11\text{ RTT}$ (`104.44 мс`).
* **TLS 1.2, нове з'єднання:** `namelookup` $= 1.00\text{ RTT}$ (`25.42 мс`), `connect` $= 1.99\text{ RTT} \approx 2\text{ RTT}$ (`50.58 мс`), `appconnect` $= 4.14\text{ RTT}$ (`105.08 мс`), `starttransfer` $= 5.14\text{ RTT}$ (`130.47 мс`).
* **Повторне використання з'єднання (5.3, 2-й та 3-й запити):** `namelookup` $\approx 0\text{ RTT}$, `connect` $= 0\text{ RTT}$, `appconnect` $= 0\text{ RTT}$, `starttransfer` $= 1.00\text{–}1.03\text{ RTT} \approx 1\text{ RTT}$ (`25.44–26.18 мс`).

Вимірювання відрізняється від цілого числа RTT на етапі **`appconnect`** (та відповідно накопичується у `starttransfer` і `total`), де додається **$\approx 0.10\text{–}0.14\text{ RTT}$ ($3\text{–}3.8\text{ мс}$)**. Це пояснюється обчислювальними витратами (CPU) на криптографічні операції під час TLS-рукостискання: генерацію ефемерних ключів ECDHE, обчислення спільного секрету, перевірку цифрового підпису сертифіката та вироблення сесійних ключів.

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?

Локальний клієнт `curl` у нашому контейнері (`10.53.1.1`) не має власного системного кешу DNS, тому при кожному новому запуску процесу він надсилає мережевий UDP-запит до рекурсивного резолвера стенда (`10.53.0.53`). Оскільки канал до резолвера також має мережеву затримку 1 RTT ($\approx 25.4\text{ мс}$), навіть миттєва відповідь із кешу резолвера займає `1 RTT` (`25.42 мс`). Значення `time_namelookup` було б приблизно вдвічі більшим ($\approx 2\text{ RTT} \approx 51\text{ мс}$, як у найпершій спробі кроку 5.2 — `50.93 мс`), якби кеш резолвера був порожнім (холодний кеш) і резолверу довелося додатково звертатися по мережі до авторитативного DNS-сервера зони `se-201-02.lab` або якщо `curl` виконує два послідовні запити `A` та `AAAA` (IPv4 та IPv6) замість паралельних.

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.

Приймемо чистий мережевий обмін без локального DNS-кешування (або враховуючи виміряний `starttransfer`, де $1\text{ RTT} = 25\text{ мс}$, а з урахуванням криптографії згідно з вимірюваннями: нове з'єднання TLS 1.3 $\approx 104.4\text{ мс}$, TLS 1.2 $\approx 130.5\text{ мс}$, повторний запит $\approx 25.4\text{ мс}$):

* **Нове з'єднання TLS 1.3 на кожен запит** ($4\text{ RTT}$ на запит $= 200 \times 4 = 800\text{ RTT}$):
  * За теоретичними RTT ($1\text{ RTT} = 25\text{ мс}$): $800 \times 25\text{ мс} = \mathbf{20\,000\text{ мс} = 20.0\text{ с}}$ (або за фактичним `starttransfer` $104.44\text{ мс}$: $200 \times 104.44\text{ мс} \approx \mathbf{20.89\text{ с}}$; без DNS по $3\text{ RTT}$ — **$15.0\text{ с}$**).
* **Нове з'єднання TLS 1.2 на кожен запит** ($5\text{ RTT}$ на запит $= 200 \times 5 = 1000\text{ RTT}$):
  * За теоретичними RTT ($1\text{ RTT} = 25\text{ мс}$): $1000 \times 25\text{ мс} = \mathbf{25\,000\text{ мс} = 25.0\text{ с}}$ (або за фактичним `starttransfer` $130.47\text{ мс}$: $200 \times 130.47\text{ мс} \approx \mathbf{26.09\text{ с}}$; без DNS по $4\text{ RTT}$ — **$20.0\text{ с}$**).
* **Усі 200 запитів через одне з'єднання (TLS 1.3)** ($4\text{ RTT}$ на перший запит + $199 \times 1\text{ RTT}$ на решту $= 203\text{ RTT}$):
  * За теоретичними RTT ($1\text{ RTT} = 25\text{ мс}$): $203 \times 25\text{ мс} = \mathbf{5\,075\text{ мс} \approx 5.08\text{ с}}$ (або за фактичними вимірюваннями: $104.17 + 199 \times 25.44 \approx \mathbf{5\,167\text{ мс} \approx 5.17\text{ с}}$).

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:
У завданні 5 очікувалося, що `time_namelookup` буде близьким до нуля (оскільки домен `origin.se-201-02.lab` уже неодноразово запитувався у попередніх завданнях і мав бути закешованим), а також що в найпершій спробі кроку 5.2 воно дорівнюватиме 1 RTT, як і в наступних. Натомість у першому рядку виводу кроку 5.2 (`namelookup=0.050931 connect=0.076075 appconnect=0.104834 starttransfer=0.130088`) `time_namelookup` склало 50.9 мс (2 RTT), а в усіх наступних запусках нового процесу `curl` стабільно трималося на рівні 25.4 мс (1 RTT), і лише при повторному використанні з'єднання в одній команді (`namelookup=0.000055`) впало майже до нуля. Це показало, що `curl` кешує DNS лише в межах пам'яті одного процесу, а кожен новий запуск звертається по мережі до резолвера стенда.

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

У кроці 3.6 під час налаштування власного HTTPS-сервера після додавання проміжного сертифіката (`cat www.se-201-02.lab.crt www.se-201-02.lab.issuer.crt > /lab/tls/server.crt`) і виконання `web-reload` перший запит до `https://www.se-201-02.lab/` одразу повернув коректну відповідь (`server: se-201-02 (https)`), тоді як виконаний у ту саму секунду запит до `https://app.se-201-02.lab/` повернув помилку `curl: (60) SSL certificate problem: unable to get local issuer certificate`, хоча вже за мить повторний запит до `app.se-201-02.lab` пройшов успішно, а `openssl s_client` показав `Verify return code: 0 (ok)`. Перевірено вміст `/lab/tls/server.crt` та SAN обох імен — сертифікат містив обидва домени та повний ланцюжок, тож найімовірніше під час асинхронного перезавантаження `nginx -s reload` запит встиг обробити старий робочий процес до завершення перечитування конфігурації.

## Декларація використання ШІ

Використовувалися інструменти ШІ: Так

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | Gemini | Надай відповіді на контрольні запитання в кінці кожного завдання, та поясни їх| Теоретичні пояснення розрахунків у завданнях 1-5|
| 2 | Gemini| Напиши висновок до роботи по темі без води, суто по темі| Оформлення підсумків лабораторної роботи |
