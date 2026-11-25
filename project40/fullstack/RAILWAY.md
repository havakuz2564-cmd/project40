# Пошаговая инструкция: деплой на Railway (без Redis)

Проект: FastAPI + PostgreSQL. Celery работает в синхронном режиме
(`CELERY_TASK_ALWAYS_EAGER=true`) — отдельный Redis и worker НЕ нужны.

---

## Шаг 0. Что уже настроено в репозитории

Эти файлы не нужно менять, просто убедитесь, что они в проекте:

- `Dockerfile` — сборка образа, запуск через `uvicorn` с переменной `$PORT`
- `railway.toml` — конфиг деплоя для Railway (startCommand, healthcheck)
- `.dockerignore` — чтобы в образ не попали `.env`, логи, `__pycache__`
- `celery_app.py` — читает `CELERY_TASK_ALWAYS_EAGER` из окружения
- `.gitignore` — файл `.env` НЕ должен попадать в git

---

## Шаг 1. Залить код на GitHub

В терминале в папке проекта:

```bash
cd fullstack
git init                          # если репозиторий ещё не создан
git add .
git commit -m "Initial commit for Railway"
```

Далее на github.com создайте новый пустой репозиторий (например `notes-api`)
и выполните (URL скопируйте со страницы репозитория):

```bash
git remote add origin https://github.com/ВАШ_ЛОГИН/notes-api.git
git branch -M main
git push -u origin main
```

⚠️ Проверьте, что файла `.env` нет в репозитории (он в .gitignore).
Пароли на Railway задаются только через Variables, не через файлы.

---

## Шаг 2. Создать проект на Railway

1. Откройте https://railway.app → **Login with GitHub**.
2. Нажмите **+ New Project** (внизу слева).
3. Выберите **Deploy from GitHub repo**.
4. Если репозитория нет в списке — нажмите **Configure GitHub App** →
   установите расширение Railway на свой репозиторий.
5. Выберите ваш репозиторий `notes-api`, ветку `main`.
6. Railway сам найдёт `Dockerfile` и начнёт первую сборку.
   Первая сборка займёт ~3–5 минут. Она УПАДЁТ — это нормально,
   мы ещё не задали переменные окружения. Переходим к шагу 3.

---

## Шаг 3. Добавить базу данных PostgreSQL

1. В окне проекта нажмите **+ New** → **Database**.
2. Выберите **PostgreSQL** (Railway создаст сервис с БД автоматически).
3. Дождитесь статуса **Healthy** (зелёная точка у сервиса).

Никаких паролей придумывать не нужно — Railway сам создаст подключение.

---

## Шаг 4. Переменные окружения для приложения

1. Кликните на сервис вашего приложения (не на базу!).
2. Вкладка **Variables** → **New Variable**. Добавьте по одной каждая:

| Key                        | Value (что писать)                                        |
|----------------------------|-----------------------------------------------------------|
| `DATABASE_URL`             | см. ниже — через Reference, НЕ пальцем!                   |
| `SECRET_KEY`               | любая длинная случайная строка, например `my-secret-2026-x9k2` |
| `ALGORITHM`                | `HS256`                                                   |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `30`                                                 |
| `CORS_ORIGINS`             | адрес фронтенда, напр. `http://localhost:5173` или домен сайта |
| `CELERY_TASK_ALWAYS_EAGER` | `true`                                                    |

Как задать `DATABASE_URL` правильно:
1. Нажмите **Add Database URL Variable / Reference** (или создайте переменку
   `DATABASE_URL` и в поле значения нажмите иконку **Reference**).
2. Выберите сервис **PostgreSQL** → переменную **INTERNAL DATABASEURL**
   (или `DATABASE_URL`).
3. Должно получиться значение вида `${{Postgres.DATABASE_URL}}` —
   Railway подставит реальный адрес при деплое.

⚠️ Не используйте `PUBLIC DB URL` — приложение должно ходить в базу
по внутренней сети Railway (переменная `INTERNAL DATABASEURL`).

---

## Шаг 5. Сделать приложение общедоступным (домен)

1. В сервисе приложения → вкладка **Settings** → **Networking**.
2. Нажмите **Generate Domain**.
3. Выберите **Public Networking**.
4. Railway выдаст домен вида `notes-api-upstream-production-xxxx.up.railway.app`.

(Если Railway требует план Hobby для Public Networking — активируйте
план $5/мес с триальными $5; Free-плана больше не существует.)

---

## Шаг 6. Перезапустить деплой

1. Вкладка **Deployments** → три точки справа → **Redeploy**.
2. Дождитесь статуса **SUCCESS** / **Healthy**.

---

## Шаг 7. Проверка работоспособности

Подставьте ваш домен вместо `<APP>`:

```bash
# 1. Корень приложения
curl https://<APP>.up.railway.app/
# ожидаем: {"message":"Привет, FastAPI! Сервис заметок работает."}

# 2. Регистрация
curl -X POST https://<APP>.up.railway.app/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"secret123"}'
# ожидаем: {"id":1,"email":"test@test.com","role":"user"}

# 3. Логин (получаем токен)
TOKEN=$(curl -s -X POST https://<APP>.up.railway.app/auth/login \
  -d "username=test@test.com&password=secret123" | python -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

# 4. Создать категорию
curl -X POST https://<APP>.up.railway.app/categories/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Работа"}'

# 5. Создать заметку
curl -X POST https://<APP>.up.railway.app/notes/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Моя заметка","content":"текст","category_id":1}'

# 6. Список заметок
curl https://<APP>.up.railway.app/notes/ -H "Authorization: Bearer $TOKEN"
```

Также можно открыть в браузере: `https://<APP>.up.railway.app/docs` —
Swagger UI, все эндпоинты проверяются прямо там (кнопки Authorize / Try it out).

---

## Частые проблемы

| Симптом | Причина и решение |
|---|---|
| Crash loop после деплоя | Не задан `DATABASE_URL` или задан публичный адрес БД. Проверьте Reference на `INTERNAL DATABASEURL`. |
| Ошибка bcrypt/passlib | В requirements уже закреплён `bcrypt==4.0.1` — пересоберите образ (Redeploy). |
| 405 Method Not Allowed в /docs | Это rate-limit на `/auth/login` (5 попыток/мин) — подождите минуту. |
| Приложение недоступно по HTTPS | Не сделан Generate Domain (Шаг 5). |
| Пустая база после редиплоя | Так и должно быть: таблицы создаются при старте (`create_all`), данные остаются в объёме Railway Postgres. |

---

## Как обновлять код

Просто пушьте в main:

```bash
git add . && git commit -m "update" && git push
```

Railway автоматически соберёт и задеплоит новую версию.
