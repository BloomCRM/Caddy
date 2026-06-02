# caddy — shared reverse proxy

Один Caddy на сервері для всіх проектів. Термінує TLS (Let's Encrypt автоматично) і
проксує запити на контейнери за **іменем хоста (домен)**. Прод-сервіси не мають публічних
портів — єдина точка входу ззовні це Caddy.

---

## Як працює роутинг

```
Браузер ──HTTPS──▶  Caddy  ──HTTP :8080──▶  контейнер у мережі "web"
 (домен)          (TLS тут)                 (за іменем контейнера)
```

1. Запит приходить на `:443` за доменом (напр. `api.valentynah.com`).
2. Caddy матчить **site-блок** із цим доменом у `Caddyfile`.
3. `reverse_proxy <container>:8080` пересилає запит контейнеру за його **іменем** у Docker-мережі `web`.
4. Caddy додає security-заголовки, gzip/zstd, пише JSON-лог.

Маршрут визначається **доменом (Host header)**, не шляхом. Один домен → один контейнер.

| Домен | Контейнер | Проект |
|---|---|---|
| `valentynah.com`, `www.…` | `valentyna_app:8080` | публічний сайт (www → apex redirect) |
| `crm.valentynah.com` | `bloom_admin:8080` | Bloom Admin (tenant) |
| `api.valentynah.com` | `bloom_api:8080` | Bloom API |
| `platform.valentynah.com` | `bloom_platformadmin:8080` | Bloom Platform Admin |

Кожен Bloom-контейнер слухає `:8080` всередині (`ASPNETCORE_URLS=http://+:8080`), назовні портів не відкриває.

---

## Мережі (важливо)

Дві окремі Docker-мережі:

| Мережа | Тип | Хто в ній | Призначення |
|---|---|---|---|
| **`web`** | зовнішня, спільна (`external: true`) | Caddy + усі **публічні** контейнери (`bloom_api`, `bloom_admin`, `bloom_platformadmin`, `valentyna_app`) | вхідний трафік ззовні |
| **`bloom_internal`** | приватна (у `docker-compose.yml` Bloom) | `postgres`, `seq`, `bloom_outboxworker` + знову ж публічні сервіси | внутрішня комунікація, **не** доступна ззовні |

Тобто **Caddy бачить лише те, що в мережі `web`**. Postgres, Seq і **outbox-worker** туди не входять —
вони доступні тільки всередині `bloom_internal` (БД/логи закриті, worker — фоновий, без HTTP).

> Щоб контейнер був доступний через Caddy: він має (а) бути в мережі `web` і (б) мати site-блок у `Caddyfile` з його іменем.

---

## Прод (Hetzner, `/opt/caddy/`)

### Перший запуск

```bash
docker network create web          # один раз на весь сервер (спільна мережа)
cd /opt/caddy
docker compose up -d
docker compose logs -f caddy       # дивитися, як видаються TLS-сертифікати
```

### Що дає кожен site-блок (`Caddyfile`)

- `reverse_proxy <container>:8080` — проксі на контейнер;
- `encode gzip zstd` — стиснення;
- security-заголовки: `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`
  (`SAMEORIGIN` для CRM, **`DENY`** для Platform Admin — анти-clickjacking), `Referrer-Policy`,
  `Permissions-Policy`, прибраний `Server`;
- JSON-лог із ротацією у `/data/logs/<service>.log` (`roll_size 10mb`, `roll_keep 5`).

### Додати новий сервіс/домен

1. У `docker-compose.yml` проєкту переконайся, що контейнер у мережі `web`.
2. Додай site-блок у `Caddyfile` (скопіюй найближчий за змістом, заміни домен і `reverse_proxy <container>:8080`).
3. Застосуй: `docker compose restart caddy` — **НЕ** `down/up` (інакше скинеться стан TLS).
4. DNS: A-запис нового домену → IP сервера (інакше Let's Encrypt не видасть сертифікат).

### Оновити конфіг

```bash
cd /opt/caddy
git pull
docker compose restart caddy
docker compose logs -f caddy       # перевірити сертифікати/помилки
```

### Перевірити TLS

```bash
docker exec shared_caddy ls /data/caddy/certificates/      # видані сертифікати
curl -vI https://crm.valentynah.com 2>&1 | grep -E "SSL|issuer|expire"
```

---

## Локальний Caddy (`*.localhost`, без TLS)

Для локального стека використовується **окремий** конфіг — `Bloom/Caddyfile.local`
(його монтує `Bloom/docker-compose.local.yml`), **не** цей `Caddyfile`.

Відмінності від прода:
- `auto_https off` — лише HTTP (без сертифікатів);
- домени `*.localhost` (Chrome/Edge резолвлять у `127.0.0.1` без правок `hosts`);
- без security-заголовків і логів (вони не потрібні локально).

```
http://crm.localhost        → bloom_admin:8080
http://api.localhost        → bloom_api:8080
http://platform.localhost   → bloom_platformadmin:8080
http://valentynah.localhost → valentyna_app:8080
```

Деталі запуску локального стека — `Bloom/docs/local-dev.md`.

---

## Що НЕ проходить через Caddy

- **Postgres** і **Seq** — лише `bloom_internal`; доступ через SSH-тунель (див. `Bloom/docs/ops.md`).
- **`bloom_outboxworker`** — фоновий процес (outbox dispatch + Hangfire server), HTTP не слухає,
  у мережу `web` не входить, домену не має.
- **Hangfire dashboard** — це маршрут `/hangfire` **усередині** `bloom_platformadmin`, тож він
  доступний через `platform.valentynah.com/hangfire` (Caddy проксує як частину Platform Admin),
  захищений claim `is_platform_admin`.

---

## Troubleshooting

| Симптом | Причина / фікс |
|---|---|
| **502 Bad Gateway** | Контейнер не в мережі `web` або змінив ім'я. Перевір: `docker network inspect web \| grep <container>` і що ім'я в `Caddyfile` збігається з `container_name`. |
| Сертифікат не видається | DNS A-запис домену не вказує на сервер, або порт 80/443 зайнятий/закритий фаєрволом. `docker compose logs -f caddy`. |
| Після `git pull` зміни не застосувались | `docker compose restart caddy`. |
| Втратив сертифікати після рестарту | Робив `down/up` замість `restart`. Сертифікати у volume `/data` — не видаляй volume. |
| `crm.localhost` не відкривається | Це локальний стек — потрібен Chrome/Edge (резолвлять `*.localhost`) і піднятий `bloomlocal_caddy`. |

---

## Архітектура (підсумок)

Зовнішня мережа `web` — спільна шина для вхідного трафіку всіх проектів (`external: true` у кожному compose).
Caddy — єдина точка термінації TLS і маршрутизації за доменом. Внутрішні сервіси (БД, логи, воркери)
живуть у приватних мережах і назовні не виставляються.
