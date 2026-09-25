# EduCheck AI — инструкция по развёртыванию (Render + Neon + Cloudflare R2)

Документ описывает полный запуск системы с нуля: база данных, бэкенд, фронтенд,
файловое хранилище и (опционально) AI-проверка через OpenAI.
Все шаги выполняются один раз; далее сервисы обновляются автоматически при пуше в GitHub.

---

## 0. Что понадобится

| Сервис | Зачем | Тариф |
|---|---|---|
| GitHub | исходный код проекта | бесплатно |
| Neon (neon.tech) | PostgreSQL-база данных | бесплатно |
| Render (render.com) | хостинг бэкенда и фронтенда | бесплатно |
| Cloudflare R2 | хранение файлов (работы, методички) | бесплатно (10 ГБ) |
| OpenAI (опционально) | AI-проверка работ | платно по токенам |

Без OpenAI система работает в режиме `mock` — AI ставит детерминированную
демонстрационную оценку, ничего не оплачивая.

---

## 1. Архитектура в трёх строках

- `backend/` — FastAPI (Python 3.11+), API по адресу `https://<BACKEND>/api/v1`.
- `frontend/` — статический сайт (чистый JS, без сборки),hash-роутинг `#/...`.
- База и файлы НЕ хранятся на Render: БД в Neon, файлы в R2 (браузер грузит их напрямую по presigned-URL).

Важно: таблицы в БД создаются САМИ при первом старте бэкенда (и сами добавляют
новые колонки при обновлениях). Ручные миграции не нужны.

---

## 2. Шаг 1 — база данных Neon

1. Зарегистрируйтесь на neon.tech → **New Project** (имя любое, регион — Europe).
2. В панели проекта откроется строка подключения вида:
   postgresql://user:PASSWORD@ep-xxxx-yy.eu-central-1.aws.neon.tech/neondb?sslmode=require
3. Скопируйте её целиком — это значение переменной `DATABASE_URL` (шаг 4).
   Ничего в ней менять не нужно: бэкенд сам поправит схему и SSL-параметры.

---

## 3. Шаг 2 — хранилище Cloudflare R2 (УСЛОВИЕ 1 из 2: ключи для бэкенда)

1. dash.cloudflare.com → **R2 Object Storage** → **Create bucket**.
   Имя бакета, например: `educheck-files`. Регион любой.
2. R2 → **Manage R2 API Tokens** → **Create API Token**:
   - Permissions: **Object Read & Write**;
   - Apply to: **только ваш бакет**;
   - Создайте и СОХРАНИТЕ: `Access Key ID` и `Secret Access Key` (показываются один раз).
3. Там же (или на странице бакета) скопируйте **Endpoint URL** вида:
   https://<ACCOUNT_ID>.r2.cloudflarestorage.com
4. Эти три значения + имя бакета уйдут в env бэкенда (шаг 4):
   S3_ENDPOINT, S3_ACCESS_KEY, S3_SECRET_KEY, S3_BUCKET.

> Второе условие (CORS бакета) выполняется ПОСЛЕ деплоя фронтенда — шаг 6.
> Без него браузер не сможет загрузить файл напрямую в R2.

---

## 4. Шаг 3 — деплой бэкенда на Render

1. Render → **New → Web Service** → подключите репозиторий с проектом.
2. Настройки сервиса:
   - **Name**: например `educheck-backend` (запомните итоговый URL);
   - **Root Directory**: `backend`
   - **Runtime**: Python 3;
   - **Build Command**: `pip install -r requirements.txt`
   - **Start Command**: `uvicorn main:app --host 0.0.0.0 --port $PORT`
   - **Instance**: Free (или Starter).
3. **Environment** → добавьте переменные (см. Приложение A):
   DATABASE_URL, JWT_SECRET, CORS_ORIGINS, S3_*, AI_PROVIDER и др.
   - `JWT_SECRET` сгенерируйте: в терминале `openssl rand -hex 32`
     (или любая длинная случайная строка).
   - `CORS_ORIGINS` пока поставьте-заглушку:
     `http://localhost:5173,http://localhost:3000`
     (после деплоя фронтенда допишем реальный адрес — шаг 5).
4. Deploy. Первый старт занимает 2–5 минут. В логах должно появиться:
   `DB initialized`.
5. Проверка: откройте `https://<BACKEND>/health` → `{"status":"ok",...,"db":"ok"}`
   и `https://<BACKEND>/docs` → Swagger с перечнем API.

---

## 5. Шаг 4 — деплой фронтенда на Render

1. Render → **New → Static Site** → тот же репозиторий.
2. Настройки:
   - **Name**: например `educheck-frontend`;
   - **Root Directory**: `frontend`
   - **Build Command**:
     echo "window.__EDUCHECK_API__ = '${API_BASE}';" > js/runtime-config.js
   - **Publish Directory**: `.`
   - **Environment** → переменная:
     API_BASE = https://<BACKEND>.onrender.com   (URL из шага 3)
3. Deploy. Получите адрес вида `https://educheck-frontend.onrender.com`.
   Эта команда сборки «вшивает» адрес бэкенда в сайт; локальная разработка
   при этом продолжает работать на `localhost:8000` автоматически.

> План Б: если сборочную команду задать забыли, откройте `frontend/js/config.js`
> и впишите адрес бэкенда в строку `API_BASE:` вручную, затем закоммитьте.

---

## 6. Шаг 5 — финальный CORS бэкенда

Render → бэкенд-сервис → **Environment** → измените:

CORS_ORIGINS = https://educheck-frontend.onrender.com,http://localhost:5173,http://localhost:3000

(через запятую, БЕЗ пробелов; подставьте свой адрес фронтенда).
Render автоматически перезадеплоит сервис.

---

## 7. Шаг 6 — CORS бакета R2 (УСЛОВИЕ 2 из 2)

1. Cloudflare → R2 → ваш бакет → **Settings** → **CORS Policy** → вставьте:

[
  {
    "AllowedOrigins": [
      "https://educheck-frontend.onrender.com",
      "http://localhost:5173",
      "http://localhost:3000"
    ],
    "AllowedMethods": ["PUT", "GET", "HEAD"],
    "AllowedHeaders": ["*"],
    "MaxAgeSeconds": 3600
  }
]

2. Замените домен на свой адрес фронтенда. Сохраните.

Зачем: файлы загружаются браузером НАПРЯМУЮ в R2 по подписанному URL (минуя бэкенд).
Без этого правила браузер блокирует PUT-запрос, и загрузка файла падает с
«сетевой ошибкой», хотя бэкенд полностью исправен.

---

## 8. Шаг 7 — AI-проверка (опционально)

- Хотите реальную нейросеть: создайте ключ на platform.openai.com и задайте env:
  AI_PROVIDER=openai, OPENAI_API_KEY=sk-..., OPENAI_MODEL=gpt-4o-mini
  (модель может быть любой OpenAI-совместимой; если используете сторонний
  OpenAI-совместимый эндпоинт — добавьте OPENAI_BASE_URL=https://.../v1).
- Хотите демо без расходов: `AI_PROVIDER=mock` (или просто не задавайте переменные).
  Mock ставит детерминированную оценку 40–100 по тексту работы.
- Ошибка AI больше никогда не роняет страницу: текст причины показывается
  в карточке работы и в логах бэкенда.

---

## 9. Шаг 8 — приёмочный тест (5 минут)

1. Откройте фронтенд → **Регистрация** → создайте преподавателя (role: teacher).
2. Войдите → **Группы → Создать группу** (название + описание) →
   **Добавить студента** (вкладка «Создать новый» — аккаунт студента создастся сам).
3. **Задания → Новое задание**: выберите группу, дедлайн, лимит попыток (0 = ∞),
   прикрепите PDF-файл задания → Создать. На карточке появится бейдж 📎.
4. Выйдите → войдите студентом → **Мои задания → открыть задание** →
   скачайте файл задания → **Сдать работу** (текст + файл).
5. Войдите преподавателем → **На проверку** → карточка работы →
   **🤖 Запустить AI-проверку** → дождитесь балла → поставьте итоговую оценку
   и комментарий → Сохранить.
6. Войдите студентом → **Мои работы → работа** → видны итоговая оценка
   и комментарий преподавателя.

Если всё прошло — система готова к эксплуатации.

---

## Приложение A — переменные окружения бэкенда (полный список)

| Переменная | Пример | Обязательна |
|---|---|---|
| DATABASE_URL | postgresql://user:pass@ep-...neon.tech/neondb?sslmode=require | да |
| JWT_SECRET | <64 hex-символа> | да |
| CORS_ORIGINS | https://FRONT.onrender.com,http://localhost:5173 | да |
| S3_ENDPOINT | https://<ACC>.r2.cloudflarestorage.com | да (файлы) |
| S3_REGION | auto | да (файлы) |
| S3_ACCESS_KEY | <Access Key ID> | да (файлы) |
| S3_SECRET_KEY | <Secret Access Key> | да (файлы) |
| S3_BUCKET | educheck-files | да (файлы) |
| MAX_FILE_SIZE_MB | 20 | нет |
| AI_PROVIDER | mock или openai | нет |
| OPENAI_API_KEY | sk-... | для openai |
| OPENAI_MODEL | gpt-4o-mini | для openai |
| OPENAI_BASE_URL | https://.../v1 | только для сторонних эндпоинтов |
| DEBUG | false | нет |

Переменные фронтенда (Static Site): `API_BASE` = URL бэкенда.

---

## Приложение B — полная очистка данных (кроме двух пользователей)

Выполнять в Neon → SQL Editor. Замените 9 и 10 на ID оставляемых препода и студента
(посмотреть: SELECT id, email, role FROM users;).

BEGIN;
DELETE FROM comments;
DELETE FROM ai_checks;
DELETE FROM files;
DELETE FROM submissions;
DELETE FROM task_groups;
DELETE FROM tasks;
DELETE FROM group_members;
DELETE FROM groups;
DELETE FROM users WHERE id NOT IN (9, 10);
ALTER SEQUENCE comments_id_seq      RESTART WITH 1;
ALTER SEQUENCE ai_checks_id_seq     RESTART WITH 1;
ALTER SEQUENCE files_id_seq         RESTART WITH 1;
ALTER SEQUENCE submissions_id_seq   RESTART WITH 1;
ALTER SEQUENCE task_groups_id_seq   RESTART WITH 1;
ALTER SEQUENCE tasks_id_seq         RESTART WITH 1;
ALTER SEQUENCE group_members_id_seq RESTART WITH 1;
ALTER SEQUENCE groups_id_seq        RESTART WITH 1;
COMMIT;

Файлы в бакете R2 при этом удаляются вручную (папка files/), если нужны «чистые»
данные полностью.

---

## Приложение C — диагностика

| Симптом | Причина | Лечение |
|---|---|---|
| Все запросы с фронта: «blocked by CORS policy» | Неверный/неполный CORS_ORIGINS | Шаг 5 |
| «Сетевая ошибка» ТОЛЬКО при загрузке файла | Не настроен CORS бакета R2 | Шаг 6 |
| Тост «Файловое хранилище не настроено (не хватает: S3_...)» | Не заданы env S3 | Приложение A |
| 500 при старте, в логах про DATABASE_URL | Неверная строка подключения | Шаг 1 |
| В панели работы красный текст «Ошибка AI: ...» | Проблема ключа/модели OpenAI | Шаг 7 (или mock) |
| Сайт открывается, но данные не грузятся, в консоли 404 на /api | Не записан API_BASE при сборке | Шаг 4 (Build Command) или План Б |
| Первый запрос после простоя длится ~40 сек | Free-тариф Render усыпляет сервис | норма, подождать |

Логи бэкенда: Render → сервис → **Logs**. Все необработанные ошибки пишутся
туда полным traceback'ом.

---

## Приложение D — локальный запуск (для разработки)

1. Бэкенд: cd backend; python -m venv .v; source .v/bin/activate;
   pip install -r requirements.txt; создайте .env с теми же переменными
   (DATABASE_URL можно указать локальный postgres); uvicorn main:app --reload --port 8000
2. Фронтенд: cd frontend; python -m http.server 5173 → откройте http://localhost:5173
   (фронт сам поймёт, что надо стучаться на localhost:8000).

---

## Памятка администратору

- Схема БД создаётся/дообновляется автоматически при каждом старте бэкенда.
- Смена любых env-переменных на Render = автоматический redeploy.
- Не коммитьте `.env` и ключи в GitHub (в репозитории есть `.gitignore`).
- Регулярно меняйте OPENAI_API_KEY и S3-ключи при подозрении на утечку.
