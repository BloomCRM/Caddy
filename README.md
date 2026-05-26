# caddy — shared reverse proxy

Один Caddy на сервері для всіх проектів. TLS Let's Encrypt автоматично.

| Домен | Контейнер |
|---|---|
| `valentynah.com` | `valentyna_app:8080` |
| `crm.valentynah.com` | `bloom_admin:8080` |
| `api.valentynah.com` | `bloom_api:8080` |
| `platform.valentynah.com` | `bloom_platformadmin:8080` |

## Перший запуск на сервері

```bash
docker network create web          # один раз на весь сервер
cd /opt/caddy
docker compose up -d
docker compose logs -f caddy       # дивитися як видаються TLS-сертифікати
```

## Додати новий домен

1. Додай блок у `Caddyfile`
2. `docker compose restart caddy`  ← НЕ `down/up` (інакше сертифікати скидаються)

## Оновити конфіг

```bash
cd /opt/caddy
git pull
docker compose restart caddy
```

## Архітектура

Зовнішня Docker-мережа `web` — всі проекти підключаються до неї як `external: true`.
Caddy бачить контейнери за іменами (`valentyna_app`, `bloom_api`, тощо).
Postgres, Seq та інші internal-сервіси лишаються у власних private-мережах.
