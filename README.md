# Kittygram

Пользователи публикуют котов, присваивают им 
достижения и загружают фотографии.

## О проекте

Full-stack приложение Kittygram. Backend реализует REST API для работы с 
котами и достижениями, frontend — SPA на React, взаимодействующий с API.

Проект демонстрирует:
- проектирование REST API на Django REST Framework;
- токен-аутентификацию через Djoser;
- кастомные сериализаторы (base64-изображения, hex-цвета, вычисляемые поля);
- работу с M2M-связями через промежуточную модель;
- контейнеризацию full-stack приложения (Django + React + PostgreSQL + Nginx);
- раздачу статики и медиа через Nginx.

## Стек

**Backend:**
- Python 3.9, Django 4.2 LTS, Django REST Framework
- Djoser (аутентификация), TokenAuthentication
- PostgreSQL 13
- Pillow, webcolors, WhiteNoise
- Gunicorn

**Frontend:**
- React 17, React Router
- Create React App

**Инфраструктура:**
- Docker, Docker Compose
- Nginx (reverse proxy)

## Архитектура  
┌─────────────────────────────────────────────┐  
│               Nginx (:9000)                 │  
│           /api/, /admin/ → backend          │  
│          /media/, /static/ → volumes        │  
│                   / → frontend              │  
└──────────┬──────────────────────┬───────────┘  
           │                      │                 
    ┌──────▼──────┐        ┌──────▼──────┐  
    │ backend     │        │ frontend    │  
    │ (Django)    │        │ (React)     │  
    │ :8000       │        │ :8000       │  
    └──────┬──────┘        └─────────────┘  
           │                  
    ┌──────▼──────┐  
    │      db     │  
    │ (PostgreSQL)│  
    └─────────────┘  


4 сервиса в Docker Compose:  
- **db** — PostgreSQL 13  
- **backend** — Django + Gunicorn  
- **frontend** — React + http-server  
- **gateway** — Nginx (reverse proxy, порт 9000)  

## Запуск

### Через Docker (рекомендуется)

1. Клонировать репозиторий:
git clone https://github.com/endlesslessness/kittygram.git
cd kittygram

2. Создать .env в корне проекта:
env
POSTGRES_DB=kittygram
POSTGRES_USER=kittygram_user
POSTGRES_PASSWORD=kittygram_password
DB_HOST=db
DB_PORT=5432

DJANGO_SECRET_KEY=your-secret-key-here
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
CSRF_TRUSTED_ORIGINS=http://localhost:9000,http://127.0.0.1:9000

3. Запустить:
docker-compose up --build
Приложение будет доступно по адресу: http://localhost:9000/

### Локально (без Docker)
Backend:

cd backend
python3 -m venv venv
source venv/bin/activate       # Linux/macOS
# venv\Scripts\activate        # Windows

pip install -r requirements.txt
python manage.py migrate
python manage.py runserver

Frontend:
cd frontend
npm ci
npm run start


## Основные эндпоинты  
Метод	Эндпоинт	Описание	Доступ  
POST	/api/users/	Регистрация	Все  
POST	/api/token/login/	Получить токен	Все  
POST	/api/token/logout/	Выйти	Авторизованные  
GET	/api/users/me/	Текущий пользователь	Авторизованные  
GET	/api/cats/	Список котов (пагинация 10)	Авторизованные  
POST	/api/cats/	Добавить кота	Авторизованные  
GET	/api/cats/{id}/	Детали кота	Авторизованные  
PATCH	/api/cats/{id}/	Обновить кота	Владелец  
DELETE	/api/cats/{id}/	Удалить кота	Владелец  
GET	/api/achievements/	Список достижений	Авторизованные  

## Примеры запросов  
Регистрация  
curl -X POST http://localhost:9000/api/users/ \  
  -H "Content-Type: application/json" \  
  -d '{"username": "user", "password": "pass12345"}'  

Получение токена  
curl -X POST http://localhost:9000/api/token/login/ \  
  -H "Content-Type: application/json" \  
  -d '{"username": "user", "password": "pass12345"}'  

Ответ:  
{"auth_token": "abc123def456..."}  

Создание кота с изображением в base64  
curl -X POST http://localhost:9000/api/cats/ \  
  -H "Authorization: Token <auth_token>" \  
  -H "Content-Type: application/json" \  
  -d '{  
    "name": "Барсик",  
    "color": "#FF0000",  
    "birth_year": 2020,  
    "achievements": [{"achievement_name": "Поймал мышь"}],  
    "image": "data:image/png;base64,iVBORw0KGgoAAAANS..."  
  }'  

## Архитектурные решения  
Nginx как reverse proxy  
Nginx принимает все запросы на порту 9000 и распределяет их:  

/api/ и /admin/ → backend (Django);  
  
/media/ и /static/ → volumes;  
  
/ → frontend (React).  
  
Это позволяет frontend и backend работать на одном домене — без CORS.  

Кастомный Base64ImageField  
Клиент отправляет изображение прямо в JSON (base64), без multipart-формы.
Сериализатор декодирует строку и создаёт ContentFile — это упрощает
работу фронтенда и позволяет отправлять изображение вместе с остальными
данными одним запросом.
  
Кастомный Hex2NameColor  
Цвет принимается в hex-формате (#FF0000), но валидируется через
библиотеку webcolors и сохраняется как имя (red). Это защищает от
произвольных значений и делает данные читаемыми.

M2M через промежуточную модель AchievementCat  
Связь «кот — достижение» реализована через явную промежуточную модель.
Это позволяет в будущем добавить метаданные к связи, не меняя схему БД.
  
Токен-аутентификация через Djoser  
Стандартная схема: POST /api/token/login/ → auth_token → заголовок
Authorization: Token <token> в последующих запросах.

Пагинация  
Список котов отдаётся постранично (10 записей), список достижений — без
пагинации.

## Скриншоты  
Главная страница  
https://screenshots/01-main-page.png
  
Список котов (API)  
https://screenshots/02-api-cats.png
  
Админка Django  
https://screenshots/03-admin.png
  
API Root  
https://screenshots/04-api-root.png
  
## Структура проекта  
kittygram/  
├── backend/                     # Django-приложение  
│   ├── cats/                    # Приложение с моделями Cat, Achievement  
│   │   ├── migrations/  
│   │   ├── models.py  
│   │   ├── serializers.py  
│   │   ├── views.py  
│   │   └── tests.py  
│   ├── kittygram_backend/       # Настройки проекта  
│   │   ├── settings.py  
│   │   ├── urls.py  
│   │   └── wsgi.py  
│   ├── Dockerfile  
│   ├── manage.py  
│   └── requirements.txt 
├── frontend/                    # React-приложение  
│   ├── public/  
│   ├── src/  
│   │   ├── components/  
│   │   ├── utils/  
│   │   └── index.js  
│   ├── Dockerfile  
│   └── package.json  
├── nginx/                       # Nginx (reverse proxy)  
│   ├── Dockerfile  
│   └── nginx.conf  
├── screenshots/                 # Скриншоты для README  
├── .env.example  
├── .gitignore  
├── docker-compose.yml  
└── README.md  

## Что можно улучшить  
- Покрыть backend тестами (сейчас tests.py пустой)  
- Настроить CI/CD (GitHub Actions: линтер + тесты)  
- Добавить Swagger/OpenAPI-документацию (drf-spectacular)  
- Настроить HTTPS (Let's Encrypt)  
- Добавить throttling для защиты от брутфорса  

## Контакты
GitHub: @endlesslessness  
tg: @endlesslessness  
Email: yan.lejn@mail.ru  
