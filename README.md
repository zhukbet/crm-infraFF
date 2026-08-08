# support-crm-infra

Оркестрація локальної розробки й деплою для саппорт-CRM поверх Telegram. Коду продукту тут
немає — лише зведення `crm-backendFF` і `crm-frontendFF` в один робочий стек. Специфікація
хостингу і масштабування — розділи 3.3, 3.4, 3.7 `tz-support-telegram-helpdesk.md` (додається
окремо, не в цьому репозиторії).

Бекенд і фронтенд підключені як **git submodules** (`backend/`, `frontend/`), а не окремо
клоновані поруч — так весь стек піднімається з одного `git clone --recurse-submodules`.

## Статус — каркас, стек поки не піднімається до кінця

Зроблено: `docker-compose.yml` (postgres, redis, api, workers, web, reverse-proxy), `Caddyfile`
(один домен: `/api/*` і `/ws/*` → backend, решта → frontend), корньовий `.env.example` з усіма
змінними для обох сервісів, GitHub Actions smoke-тест (підняти стек + перевірити `/api/health`
і завантаження фронтенду).

**Відомо, що зараз не запуститься:** `crm-backendFF` ще не має `main.ts`/`worker.ts`/кореневого
`AppModule` (див. його README, розділ «Відомі прогалини») — контейнери `api`/`workers` збудуються,
але впадуть одразу при старті, бо `dist/main.js` нема що запускати. Також бекенд ще не має
`GET /health` і глобального префіксу `/api` (`app.setGlobalPrefix('api')`) — обидва потрібні,
щоб `Caddyfile` і healthcheck у `docker-compose.yml` запрацювали. Це не проблема інфра-репозиторію
— тут усе за специфікацією, просто бекенду ще бракує цих кількох рядків у `main.ts`.

## Запуск

```bash
git clone --recurse-submodules https://github.com/zhukbet/support-crm-infra.git
cd support-crm-infra
cp .env.example .env   # заповнити BOT_TOKEN, JWT_SECRET
docker compose up -d --build
```

Якщо репозиторій вже клонований без `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

Підтягнути новий код у submodule (наприклад, після мержу PR у `crm-backendFF`):

```bash
git submodule update --remote backend    # або frontend
git add backend                          # закомітити новий pinned-коміт submodule
git commit -m "Оновити backend submodule"
```

## Налаштування Telegram-бота (перед першим запуском)

1. У Telegram написати [@BotFather](https://t.me/BotFather), `/newbot`, отримати `BOT_TOKEN`.
2. Вимкнути privacy mode: `@BotFather` → обрати бота → `/setprivacy` → **Disable** (інакше бот
   не бачитиме звичайні повідомлення в групах, тільки команди й reply на себе).
3. Додати бота в тестову Telegram-групу (звичайний учасник — з вимкненим privacy mode цього
   достатньо; права адміна — альтернатива, якщо privacy mode вимкнути не вдається).
4. Вписати `BOT_TOKEN` у `.env`.
5. Підняти стек (`docker compose up -d --build`) і виставити публічний HTTPS-домен (тунель типу
   ngrok/Cloudflare Tunnel для локальної розробки, або реальний домен на проді).
6. Налаштувати webhook на цей домен:
   ```bash
   curl -F "url=https://<ваш-домен>/api/telegram/webhook" \
        -F "secret_token=<той самий, що TELEGRAM_WEBHOOK_SECRET у .env>" \
        "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook"
   ```
7. Написати щось у тестовій групі — повідомлення має зʼявитись як новий тред у інтерфейсі агента.

## Деплой (розділ 3.7 ТЗ)

- **Старт (MVP):** один VPS (Hetzner/DigitalOcean) з Docker Compose — той самий `docker-compose.yml`,
  або PaaS (Railway/Render/Fly.io) з окремим сервісом на кожен `target`/build-context.
- **БД і черга — одразу managed**, а не в контейнерах поруч: Postgres → Neon/Supabase/RDS,
  Redis → Upstash. Замінити `DATABASE_URL`/`REDIS_URL` в `.env` на managed-URL і прибрати сервіси
  `postgres`/`redis` із `docker-compose.yml` на проді.
- **Ріст:** перехід на Kubernetes — ті самі Docker-образи (`backend` цілі `api`/`workers`,
  `frontend`), лише оркестрація змінюється, код і `Dockerfile` — ні.
- Головний важіль масштабування — більше реплік `workers` і чергами (BullMQ/Redis), не зміна
  платформи.

## Git-workflow для команди

Той самий підхід, що й у `crm-backendFF`/`crm-frontendFF`: гілка `main` — завжди робочий стан,
у неї напряму не пушимо, кожен працює у власній гілці зі своїм іменем.

### Основні поняття (коротко)

- **repo (репозиторій)** — папка проєкту з історією змін (`.git` всередині).
- **remote / origin** — копія репозиторію на GitHub. `origin` — стандартна назва цієї копії.
- **branch (гілка)** — незалежна лінія розробки. `main` — спільна/робоча, інші — особисті.
- **commit (коміт)** — знімок змін із повідомленням, що і навіщо змінили.
- **clone** — перше завантаження репозиторію з GitHub собі на комп'ютер (робиться один раз).
- **pull** — забрати нові коміти з GitHub і накласти на свою поточну гілку (`fetch` + `merge`).
- **push** — відправити свої локальні коміти на GitHub.
- **submodule** — посилання на конкретний коміт іншого репозиторію (тут: `backend/`, `frontend/`)
  всередині цього репозиторію. Оновлюється окремою командою (`git submodule update --remote`),
  не автоматично при звичайному `git pull`.

### 0. Перше клонування (робиться один раз на комп'ютері)

```bash
git clone --recurse-submodules https://github.com/zhukbet/support-crm-infra.git
cd support-crm-infra
```

### 1. Створити свою гілку

```bash
git checkout main
git pull origin main
git checkout -b <своє_ім'я>   # своя гілка від актуального main, напр. Dima
```

### 2. Щоденний цикл роботи

```bash
git status                     # що змінено
git diff                       # самі зміни рядок-в-рядок
git add <файл1> <файл2>        # НЕ git add . про запас
git commit -m "що і навіщо"
git push origin <своє_ім'я>
```

### 3. Стягнути оновлення (pull)

```bash
git checkout main
git pull origin main
git checkout <своє_ім'я>
git merge main                 # або: git rebase main
```

Якщо конфлікт — git покаже файли з конфліктом; відкрити їх, прибрати маркери `<<<<<<<`,
`=======`, `>>>>>>>`, вибравши правильний варіант, тоді:

```bash
git add <виправлені файли>
git merge --continue            # для rebase: git rebase --continue
```

### 4. Запушити свою роботу (push) і відкрити PR

```bash
git push -u origin <своє_ім'я>   # -u лише для першого push цієї гілки, далі просто git push
```

Далі — Pull Request зі своєї гілки в `main` на GitHub. Не пушимо напряму в `main`. Після мержу:

```bash
git checkout main
git pull origin main
```

### Шпаргалка команд

| Команда | Що робить |
|---|---|
| `git status` | що змінено, на якій гілці |
| `git diff` | детальні зміни рядок-в-рядок (ще не в коміті) |
| `git log --oneline` | коротка історія комітів |
| `git branch -a` | список усіх гілок (локальних і на GitHub) |
| `git submodule update --remote backend` | підтягнути свіжий коміт submodule `backend` |
| `git stash` / `git stash pop` | тимчасово відкласти незакомічені зміни й повернути назад |

### Типові проблеми

- **Закомітив, але ще не запушив, і хочу виправити повідомлення коміту:**
  `git commit --amend -m "нове повідомлення"`.
- **`git push` каже "rejected"** (хтось інший уже пушив у цю гілку): `git pull origin <гілка>`,
  вирішити конфлікти якщо є, потім `git push` знову.
- **Забув `--recurse-submodules` при клонуванні, папки `backend`/`frontend` порожні:**
  `git submodule update --init --recursive`.

### Правила

- Не комітити `.env` (він і так у `.gitignore`) — там `BOT_TOKEN`, `JWT_SECRET`.
- Один коміт — одна логічна зміна, повідомлення — що і навіщо.
- Зміни в самому коді бекенду/фронтенду робити в їхніх репозиторіях, не тут — сюди підтягувати
  вже готовий коміт через `git submodule update --remote`.
