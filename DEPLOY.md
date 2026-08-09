# Прод-деплой (Seq 55, §3.7 ТЗ)

Немає реальних хостинг-креденшелів для живого деплою — цей документ дає
конкретний, відтворюваний покроковий шлях для того, хто їх матиме. Усе, що
можна перевірити без хостингу (образи, healthcheck, graceful shutdown,
docker-compose smoke-test), вже перевірено — див. `crm-backendFF/DOD.md` і
CI-воркфлоу `smoke-test.yml` у цьому репо.

## Обраний шлях: один VPS + Docker Compose

§3.7 ТЗ прямо рекомендує це для MVP-масштабу ("невеликий DevOps-ресурс"), і саме
під це вже написані `docker-compose.yml` і `Caddyfile` у цьому репо — переходити
на PaaS (Railway/Render/Fly.io) чи managed Postgres/Redis (Neon/Supabase/Upstash)
можна пізніше суто конфігураційно, без зміни коду (це прямо обумовлено в ТЗ).

### 1. Сервер

- VPS (Hetzner CX22 / DigitalOcean $12 і вище) — 2 vCPU / 4 GB RAM з запасом
  вистачить на api + workers + web + postgres + redis + caddy для MVP-обсягу.
- Ubuntu 22.04/24.04, Docker Engine + Docker Compose plugin.
- DNS: A-запис домену на IP сервера (Caddy сам візьме TLS-сертифікат через Let's
  Encrypt при запуску — треба лише 80/443 відкриті).

### 2. Клонування й секрети

```bash
git clone --recurse-submodules https://github.com/zhukbet/crm-infraFF.git
cd crm-infraFF
cp .env.example .env
```

Заповнити в `.env` (мінімум для живого запуску):

- `BOT_TOKEN` — токен від @BotFather.
- `TELEGRAM_WEBHOOK_SECRET`, `JWT_SECRET` — випадкові рядки (`openssl rand -hex 32`).
- `TELEGRAM_WEBHOOK_URL` — `https://<домен>/api/telegram/webhook`.
- `PUBLIC_ORIGIN` / `PUBLIC_WS_ORIGIN` — `https://<домен>` / `wss://<домен>`.
- `TELEGRAM_BOT_USERNAME` — `@username` того ж бота (для Login Widget на фронті).
- `HTTP_PORT` — лишити 80, якщо Caddy сам термінує TLS на 443 (див. Caddyfile).

### 3. Перший запуск

```bash
docker compose up -d --build
docker compose ps        # усі 6 сервісів мають бути healthy/running
curl -s https://<домен>/api/health   # {"status":"ok"}
```

### 4. Прив'язка вебхука Telegram

```bash
curl -F "url=https://<домен>/api/telegram/webhook" \
     -F "secret_token=$TELEGRAM_WEBHOOK_SECRET" \
     "https://api.telegram.org/bot$BOT_TOKEN/setWebhook"
```

### 5. Перший агент (allowlist)

Прямого UI для першого адміна немає (allowlist — це таблиця `agents`), тому
перший запис створюється вручну:

```bash
docker compose exec postgres psql -U postgres -d support_crm -c \
  "INSERT INTO agents (id, telegram_user_id, name, username, role, is_active) \
   VALUES (gen_random_uuid(), <твій_telegram_user_id>, 'Admin', 'admin', 'admin', true);"
```

Далі всі наступні агенти додаються самим адміном через `POST /agents` (UI —
розділ 15 ТЗ, "Керування командою").

### 6. Оновлення (CD)

Ручний деплой оновлення (поки без окремого CD-воркфлоу):

```bash
git submodule update --remote --merge   # підтягнути нові коміти backend/frontend
docker compose up -d --build
```

## Що не зроблено в цій сесії й чому

- **Живого сервера немає** — ця сесія працює без хостинг-креденшелів, тож кроки
  1–6 не виконані наживо, лише перевірені компонентно (docker-compose
  smoke-test у CI піднімає весь стек і білдить ті самі образи, `/api/health`
  і фронт відповідають).
- **CD-воркфлоу (авто-паблиш образів у GHCR при пуші в main)** свідомо не
  додавав без явного запиту — це створило б реальний side-effect (публічні
  Docker-образи під акаунтом користувача) без підтвердження. Якщо потрібно —
  скажи, і додам `.github/workflows/publish.yml` з `docker/build-push-action`
  у GHCR, тегованим по SHA й `latest`.
