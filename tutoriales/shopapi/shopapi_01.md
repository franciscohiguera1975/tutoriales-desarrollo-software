# ShopAPI — Etapa 1
## Setup + Estructura de carpetas + Health check

> **Objetivo:** Servidor corriendo con estructura profesional de carpetas separadas por dominio, variables de entorno y base de datos PostgreSQL.
> **Checkpoint final:** `GET /api/health/` retorna `200 OK`.

---

## 1.1 Instalación de uv

`uv` es un gestor de paquetes y entornos virtuales para Python escrito en Rust.
Reemplaza a `pip` + `venv` en un solo comando y es significativamente más rápido.

### Linux / macOS

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh

# Recargar el PATH
source ~/.bashrc        # Linux con bash
source ~/.zshrc         # macOS con zsh

uv --version
```

### Windows (PowerShell)

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Cerrar y volver a abrir PowerShell para recargar el PATH
uv --version
```

> **Error `execution of scripts is disabled` en Windows:**
> Abrir PowerShell como Administrador y ejecutar:
> ```powershell
> Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```

### Python con uv

`uv` gestiona su propio Python — no necesitas tener Python instalado previamente.

```bash
uv python list           # ver versiones disponibles
uv python install 3.12   # instalar versión específica si se requiere
```

---

## 1.2 PostgreSQL

### Instalación

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**macOS:**
```bash
brew install postgresql@16
brew services start postgresql@16
```

**Windows:**
Descargar el instalador desde [https://www.postgresql.org/download/windows](https://www.postgresql.org/download/windows)
e instalarlo con las opciones predeterminadas. El instalador incluye pgAdmin.

### Crear la base de datos y el usuario

```bash
# Acceder a la consola de PostgreSQL
sudo -u postgres psql          # Linux / macOS
# En Windows: abrir pgAdmin o usar la consola psql desde el menú de inicio

# Dentro de la consola psql
CREATE USER shopapi_user WITH PASSWORD 'shopapi_pass';
CREATE DATABASE shopapi_db OWNER shopapi_user;
GRANT ALL PRIVILEGES ON DATABASE shopapi_db TO shopapi_user;
\q
```

---

## 1.3 Crear el proyecto

```bash
mkdir shopapi && cd shopapi

uv init

# Dependencias del proyecto + psycopg2 para conectar con PostgreSQL
uv add django djangorestframework djangorestframework-simplejwt \
       django-filter django-cors-headers \
       psycopg2-binary python-decouple

uv run django-admin startproject config .
uv run python manage.py startapp store
```

> **Windows — mismo flujo, sin el `\` de continuación de línea:**
> ```powershell
> uv add django djangorestframework djangorestframework-simplejwt django-filter django-cors-headers psycopg2-binary python-decouple
> ```

> **Activar el entorno manualmente para no escribir `uv run` en cada comando:**
>
> Linux / macOS: `source .venv/bin/activate`
>
> Windows: `.venv\Scripts\Activate.ps1`

---

## 1.4 Estructura de carpetas objetivo

```
shopapi/
├── manage.py
├── pyproject.toml
├── uv.lock
├── .env                         ← variables de entorno (nunca subir al repositorio)
├── .env.example                 ← plantilla sin valores reales (sí subir al repositorio)
├── .gitignore
├── config/
│   ├── __init__.py
│   ├── settings.py
│   └── urls.py
└── store/
    ├── __init__.py
    ├── admin.py
    ├── apps.py
    ├── migrations/
    │   └── __init__.py
    ├── models/
    │   └── __init__.py
    ├── serializers/
    │   └── __init__.py
    ├── views/
    │   └── __init__.py
    ├── tests/
    │   └── __init__.py
    ├── filters.py
    ├── pagination.py
    ├── permissions.py
    └── urls.py
```

### Crear directorios y archivos vacíos

**Linux / macOS:**
```bash
mkdir -p store/models store/serializers store/views store/tests

touch store/models/__init__.py
touch store/serializers/__init__.py
touch store/views/__init__.py
touch store/tests/__init__.py
touch store/filters.py
touch store/permissions.py
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force store\models, store\serializers, store\views, store\tests

$files = @(
    "store\models\__init__.py",
    "store\serializers\__init__.py",
    "store\views\__init__.py",
    "store\tests\__init__.py",
    "store\filters.py",
    "store\permissions.py"
)
$files | ForEach-Object { New-Item -ItemType File -Force $_ }
```

---

## 1.5 Variables de entorno

### `.env`

Este archivo contiene los valores reales. **No debe subirse al repositorio.**

```ini
# Django
SECRET_KEY=django-insecure-change-this-in-production
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1

# PostgreSQL
DB_NAME=shopapi_db
DB_USER=shopapi_user
DB_PASSWORD=shopapi_pass
DB_HOST=localhost
DB_PORT=5432

# CORS
CORS_ALLOW_ALL_ORIGINS=True
```

### `.env.example`

Plantilla pública sin valores reales — sirve como referencia para otros desarrolladores.

```ini
# Django
SECRET_KEY=
DEBUG=
ALLOWED_HOSTS=

# PostgreSQL
DB_NAME=
DB_USER=
DB_PASSWORD=
DB_HOST=
DB_PORT=

# CORS
CORS_ALLOW_ALL_ORIGINS=
```

### `.gitignore`

```gitignore
.env
.venv/
__pycache__/
*.pyc
db.sqlite3
```

---

## 1.6 Settings

`python-decouple` lee las variables desde el archivo `.env` automáticamente.

```python
# config/settings.py
from datetime import timedelta
from pathlib import Path
from decouple import config, Csv

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY    = config('SECRET_KEY')
DEBUG         = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=Csv())

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'rest_framework_simplejwt',
    'rest_framework_simplejwt.token_blacklist',
    'django_filters',
    'corsheaders',
    'store',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
]

ROOT_URLCONF = 'config.urls'

TEMPLATES = [{
    'BACKEND': 'django.template.backends.django.DjangoTemplates',
    'DIRS': [],
    'APP_DIRS': True,
    'OPTIONS': {'context_processors': [
        'django.template.context_processors.debug',
        'django.template.context_processors.request',
        'django.contrib.auth.context_processors.auth',
        'django.contrib.messages.context_processors.messages',
    ]},
}]

DATABASES = {
    'default': {
        'ENGINE':   'django.db.backends.postgresql',
        'NAME':     config('DB_NAME'),
        'USER':     config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST':     config('DB_HOST', default='localhost'),
        'PORT':     config('DB_PORT', default='5432'),
    }
}

LANGUAGE_CODE      = 'en-us'
TIME_ZONE          = 'America/Guayaquil'
USE_I18N           = True
USE_TZ             = True
STATIC_URL         = '/static/'
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    'DEFAULT_PAGINATION_CLASS': 'store.pagination.StandardPagination',
    'PAGE_SIZE': 10,
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME':    timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME':   timedelta(days=1),
    'ROTATE_REFRESH_TOKENS':    True,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM':                'HS256',
    'AUTH_HEADER_TYPES':        ('Bearer',),
    'USER_ID_FIELD':            'id',
    'USER_ID_CLAIM':            'user_id',
}

CORS_ALLOW_ALL_ORIGINS = config('CORS_ALLOW_ALL_ORIGINS', default=False, cast=bool)
```

---

## 1.7 Paginación

```python
# store/pagination.py
from rest_framework.pagination import PageNumberPagination


class StandardPagination(PageNumberPagination):
    page_size             = 10
    page_size_query_param = 'page_size'
    max_page_size         = 100
```

---

## 1.8 Health check

```python
# store/views/health.py
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response


@api_view(['GET'])
@permission_classes([AllowAny])
def health_check(request):
    return Response({'status': 'ok', 'version': '1.0'})
```

```python
# store/views/__init__.py
```

---

## 1.9 URLs

```python
# store/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from store.views.health import health_check

router = DefaultRouter()

urlpatterns = [
    path('health/', health_check),
    path('', include(router.urls)),
]
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('store.urls')),
]
```

---

## 1.10 Primera migración y arranque

```bash
# Verifica la conexión con PostgreSQL y aplica las migraciones
uv run python manage.py migrate

# Verifica que no hay errores de configuración
uv run python manage.py check

# Crea el superusuario para acceder al admin
uv run python manage.py createsuperuser

# Inicia el servidor
uv run python manage.py runserver
```

---

## ✅ Checkpoint Etapa 1

**Linux / macOS:**
```bash
curl http://localhost:8000/api/health/
```

**Windows (PowerShell):**
```powershell
Invoke-RestMethod http://localhost:8000/api/health/
```

Respuesta esperada:
```json
{
  "status": "ok",
  "version": "1.0"
}
```

- [ ] `migrate` sin errores — confirma que PostgreSQL está conectado
- [ ] `check` sin warnings
- [ ] `GET /api/health/` retorna `200 OK`
- [ ] Panel admin accesible en `http://localhost:8000/admin/`

---

## 📦 Colección Postman — Etapa 1

Crear un **Environment** en Postman antes de importar cualquier colección:

| Variable   | Valor inicial               |
|------------|-----------------------------|
| `base_url` | `http://localhost:8000/api` |
| `access`   | *(vacío — se llena en Etapa 2)* |
| `refresh`  | *(vacío — se llena en Etapa 2)* |

```json
{
  "info": {
    "name": "ShopAPI — Stage 1: Setup",
    "_postman_id": "shopapi-stage-01",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Health Check",
      "event": [{
        "listen": "test",
        "script": { "exec": [
          "pm.test('Status 200', () => pm.response.to.have.status(200));",
          "pm.test('status is ok', () => {",
          "  pm.expect(pm.response.json().status).to.eql('ok');",
          "});"
        ]}
      }],
      "request": {
        "method": "GET",
        "header": [],
        "url": "{{base_url}}/health/"
      }
    }
  ],
  "variable": [
    { "key": "base_url", "value": "http://localhost:8000/api" },
    { "key": "access",   "value": "" },
    { "key": "refresh",  "value": "" }
  ]
}
```

> **Cómo importar:** Postman → Import → Raw text → pegar el JSON → Import

---

## Resumen

| Elemento | Estado |
|---|---|
| uv instalado (Linux, macOS y Windows) | ✅ |
| PostgreSQL instalado y configurado | ✅ |
| Variables de entorno con `python-decouple` | ✅ |
| `.env.example` y `.gitignore` creados | ✅ |
| Proyecto Django + uv | ✅ |
| Estructura de carpetas por dominio | ✅ |
| Settings leyendo desde `.env` | ✅ |
| Paginación global configurada | ✅ |
| `GET /api/health/` funcionando | ✅ |

**Siguiente etapa →** Auth JWT + CRUD de Usuarios