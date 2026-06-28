# ShopAPI — Etapa 6
## Testing completo

> **Objetivo:** Suite de tests organizada por módulo con helpers reutilizables, cobertura de permisos, validaciones, filtros y acciones.
> **Checkpoint final:** `uv run python manage.py test store` pasa sin errores.

---

## 6.1 Configuración de base de datos para tests

Los tests usan una base de datos separada en PostgreSQL.
Django la crea y destruye automáticamente en cada ejecución.

Agregar al `.env`:

```ini
# Test database (Django la crea automáticamente)
TEST_DB_NAME=shopapi_test_db
```

Actualizar `config/settings.py` en la sección `DATABASES`:

```python
DATABASES = {
    'default': {
        'ENGINE':   'django.db.backends.postgresql',
        'NAME':     config('DB_NAME'),
        'USER':     config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST':     config('DB_HOST', default='localhost'),
        'PORT':     config('DB_PORT', default='5432'),
        'TEST': {
            'NAME': config('TEST_DB_NAME', default='shopapi_test_db'),
        },
    }
}
```

El usuario de PostgreSQL debe tener permisos para crear bases de datos:

```sql
-- Ejecutar en psql
ALTER USER shopapi_user CREATEDB;
```

---

## 6.2 Estructura de tests

```
store/tests/
├── __init__.py
├── helpers.py
├── test_auth.py
├── test_users.py
├── test_categories.py
├── test_products.py
└── test_orders.py
```

---

## 6.3 Helpers reutilizables

```python
# store/tests/helpers.py
from django.contrib.auth.models import User
from rest_framework.test import APIClient
from rest_framework_simplejwt.tokens import RefreshToken

from store.models import Category, Product, Order, OrderItem


def create_user(username='user', email=None, password='Pass1234!', **kwargs):
    email = email or f'{username}@test.com'
    return User.objects.create_user(
        username=username, email=email, password=password, **kwargs
    )


def create_staff(username='staff', email=None, password='Admin1234!'):
    email = email or f'{username}@test.com'
    return User.objects.create_user(
        username=username, email=email, password=password, is_staff=True
    )


def get_tokens(user):
    refresh = RefreshToken.for_user(user)
    return str(refresh.access_token), str(refresh)


def auth_client(user):
    client = APIClient()
    access, _ = get_tokens(user)
    client.credentials(HTTP_AUTHORIZATION=f'Bearer {access}')
    return client


def create_category(name='Electronics', slug='electronics', is_active=True):
    return Category.objects.create(name=name, slug=slug, is_active=is_active)


def create_product(name='Laptop', price=850, stock=10, category=None, is_active=True):
    if category is None:
        category = create_category()
    return Product.objects.create(
        name=name, price=price,
        stock=stock, category=category, is_active=is_active,
    )


def create_order(user, status='pending'):
    return Order.objects.create(user=user, status=status)


def add_item(order, product=None, quantity=1):
    if product is None:
        product = create_product()
    return OrderItem.objects.create(
        order=order,
        product=product,
        quantity=quantity,
        unit_price=product.price,
    )
```

---

## 6.4 Tests de Auth

```python
# store/tests/test_auth.py
from django.test import TestCase
from rest_framework.test import APIClient
from rest_framework import status

from .helpers import create_user, get_tokens


class RegisterTests(TestCase):

    def setUp(self):
        self.client = APIClient()
        self.url    = '/api/auth/register/'
        self.data   = {
            'username':  'john',
            'email':     'john@test.com',
            'password':  'Pass1234!',
            'password2': 'Pass1234!',
        }

    def test_register_returns_jwt(self):
        resp = self.client.post(self.url, self.data)
        self.assertEqual(resp.status_code, status.HTTP_201_CREATED)
        self.assertIn('access',   resp.data)
        self.assertIn('refresh',  resp.data)
        self.assertIn('is_staff', resp.data)
        self.assertFalse(resp.data['is_staff'])

    def test_register_passwords_do_not_match(self):
        self.data['password2'] = 'Different!'
        resp = self.client.post(self.url, self.data)
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)

    def test_register_duplicate_username(self):
        create_user('john')
        resp = self.client.post(self.url, self.data)
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)

    def test_register_duplicate_email(self):
        create_user('other', email='john@test.com')
        resp = self.client.post(self.url, self.data)
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)

    def test_register_short_password(self):
        self.data['password'] = self.data['password2'] = '123'
        resp = self.client.post(self.url, self.data)
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)


class LoginTests(TestCase):

    def setUp(self):
        self.client = APIClient()
        self.user   = create_user('ana', password='Pass1234!')

    def test_login_returns_tokens(self):
        resp = self.client.post('/api/auth/login/', {
            'username': 'ana', 'password': 'Pass1234!'
        })
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertIn('access',   resp.data)
        self.assertIn('refresh',  resp.data)
        self.assertIn('username', resp.data)
        self.assertIn('is_staff', resp.data)

    def test_login_invalid_credentials(self):
        resp = self.client.post('/api/auth/login/', {
            'username': 'ana', 'password': 'wrong'
        })
        self.assertEqual(resp.status_code, status.HTTP_401_UNAUTHORIZED)


class RefreshLogoutTests(TestCase):

    def setUp(self):
        self.client  = APIClient()
        self.user    = create_user('bob')
        self.access, self.refresh = get_tokens(self.user)

    def test_refresh_returns_new_access(self):
        resp = self.client.post('/api/auth/token/refresh/', {'refresh': self.refresh})
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertIn('access', resp.data)

    def test_logout_blacklists_refresh(self):
        self.client.credentials(HTTP_AUTHORIZATION=f'Bearer {self.access}')
        resp = self.client.post('/api/auth/logout/', {'refresh': self.refresh})
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        resp2 = self.client.post('/api/auth/token/refresh/', {'refresh': self.refresh})
        self.assertEqual(resp2.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_logout_without_refresh_returns_400(self):
        self.client.credentials(HTTP_AUTHORIZATION=f'Bearer {self.access}')
        resp = self.client.post('/api/auth/logout/', {})
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)
```

---

## 6.5 Tests de Usuarios

```python
# store/tests/test_users.py
from django.test import TestCase
from rest_framework import status

from .helpers import create_user, create_staff, auth_client


class ProfileTests(TestCase):

    def setUp(self):
        self.user   = create_user('carlos')
        self.client = auth_client(self.user)

    def test_get_own_profile(self):
        resp = self.client.get('/api/users/profile/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(resp.data['username'], 'carlos')

    def test_update_own_profile(self):
        resp = self.client.patch('/api/users/profile/', {'first_name': 'Carlos'})
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(resp.data['first_name'], 'Carlos')

    def test_change_password_success(self):
        resp = self.client.post('/api/users/change-password/', {
            'current_password': 'Pass1234!',
            'new_password':     'New5678!',
            'new_password2':    'New5678!',
        })
        self.assertEqual(resp.status_code, status.HTTP_200_OK)

    def test_change_password_wrong_current(self):
        resp = self.client.post('/api/users/change-password/', {
            'current_password': 'Wrong!',
            'new_password':     'New5678!',
            'new_password2':    'New5678!',
        })
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)


class UserStaffTests(TestCase):

    def setUp(self):
        self.staff  = create_staff()
        self.user   = create_user('diana')
        self.client = auth_client(self.staff)

    def test_staff_can_list_users(self):
        resp = self.client.get('/api/users/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertIn('results', resp.data)

    def test_regular_user_cannot_list(self):
        resp = auth_client(self.user).get('/api/users/')
        self.assertEqual(resp.status_code, status.HTTP_403_FORBIDDEN)

    def test_staff_can_toggle_active(self):
        resp = self.client.post(f'/api/users/{self.user.id}/toggle-active/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertIn('is_active', resp.data)

    def test_staff_can_get_stats(self):
        resp = self.client.get('/api/users/stats/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        for field in ['total', 'active', 'inactive', 'staff']:
            self.assertIn(field, resp.data)

    def test_filter_by_is_staff(self):
        resp = self.client.get('/api/users/?is_staff=true')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        for u in resp.data['results']:
            self.assertTrue(u['is_staff'])
```

---

## 6.6 Tests de Categorías

```python
# store/tests/test_categories.py
from django.test import TestCase
from rest_framework import status

from .helpers import create_user, create_staff, auth_client, create_category


class CategoryPermissionTests(TestCase):

    def setUp(self):
        self.user     = create_user('eve')
        self.staff    = create_staff()
        self.category = create_category()

    def test_authenticated_user_can_list(self):
        resp = auth_client(self.user).get('/api/categories/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)

    def test_unauthenticated_returns_401(self):
        from rest_framework.test import APIClient
        resp = APIClient().get('/api/categories/')
        self.assertEqual(resp.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_regular_user_cannot_create(self):
        resp = auth_client(self.user).post('/api/categories/', {
            'name': 'Test', 'slug': 'test'
        })
        self.assertEqual(resp.status_code, status.HTTP_403_FORBIDDEN)

    def test_staff_can_create(self):
        resp = auth_client(self.staff).post('/api/categories/', {
            'name': 'Home', 'slug': 'home', 'is_active': True
        })
        self.assertEqual(resp.status_code, status.HTTP_201_CREATED)

    def test_staff_can_delete(self):
        resp = auth_client(self.staff).delete(f'/api/categories/{self.category.id}/')
        self.assertEqual(resp.status_code, status.HTTP_204_NO_CONTENT)


class CategoryFilterTests(TestCase):

    def setUp(self):
        self.client = auth_client(create_user('filters'))
        create_category('Electronics', 'electronics', is_active=True)
        create_category('Clothing',    'clothing',    is_active=False)

    def test_filter_by_active(self):
        resp = self.client.get('/api/categories/?is_active=true')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(resp.data['count'], 1)
        self.assertEqual(resp.data['results'][0]['name'], 'Electronics')

    def test_search_by_name(self):
        resp = self.client.get('/api/categories/?search=electro')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(resp.data['count'], 1)

    def test_stats_returns_expected_fields(self):
        resp = self.client.get('/api/categories/stats/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        for field in ['total', 'active', 'inactive', 'detail']:
            self.assertIn(field, resp.data)
```

---

## 6.7 Tests de Productos

```python
# store/tests/test_products.py
from django.test import TestCase
from rest_framework import status
from rest_framework.test import APIClient

from .helpers import create_user, create_staff, auth_client, create_category, create_product


class ProductPermissionTests(TestCase):

    def setUp(self):
        self.user    = create_user('frank')
        self.staff   = create_staff()
        self.cat     = create_category()
        self.product = create_product(category=self.cat)

    def test_authenticated_can_list(self):
        resp = auth_client(self.user).get('/api/products/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertIn('results', resp.data)

    def test_unauthenticated_returns_401(self):
        resp = APIClient().get('/api/products/')
        self.assertEqual(resp.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_regular_user_cannot_create(self):
        resp = auth_client(self.user).post('/api/products/', {
            'name': 'Test', 'price': '10.00',
            'stock': 5, 'category_id': self.cat.id,
        })
        self.assertEqual(resp.status_code, status.HTTP_403_FORBIDDEN)

    def test_staff_can_create(self):
        resp = auth_client(self.staff).post('/api/products/', {
            'name': 'Keyboard', 'price': '79.00',
            'stock': 12, 'category_id': self.cat.id,
        })
        self.assertEqual(resp.status_code, status.HTTP_201_CREATED)

    def test_price_with_tax_is_15_percent(self):
        resp = auth_client(self.user).get(f'/api/products/{self.product.id}/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        expected = round(float(self.product.price) * 1.15, 2)
        self.assertEqual(float(resp.data['price_with_tax']), expected)


class ProductFilterTests(TestCase):

    def setUp(self):
        self.client = auth_client(create_user('gina'))
        cat = create_category()
        create_product('Laptop',   price=850, stock=5,  category=cat)
        create_product('Cheap',    price=20,  stock=0,  category=cat)
        create_product('Inactive', price=50,  stock=10, category=cat, is_active=False)

    def test_filter_by_max_price(self):
        resp = self.client.get('/api/products/?price_max=100')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(resp.data['count'], 1)
        self.assertEqual(resp.data['results'][0]['name'], 'Cheap')

    def test_filter_by_min_stock(self):
        resp = self.client.get('/api/products/?stock_min=1')
        names = [p['name'] for p in resp.data['results']]
        self.assertIn('Laptop', names)
        self.assertNotIn('Cheap', names)

    def test_search_by_name(self):
        resp = self.client.get('/api/products/?search=lapt')
        self.assertEqual(resp.data['count'], 1)

    def test_available_is_public(self):
        resp = APIClient().get('/api/products/available/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        names = [p['name'] for p in resp.data['results']]
        self.assertIn('Laptop', names)
        self.assertNotIn('Cheap', names)


class ProductActionTests(TestCase):

    def setUp(self):
        self.staff   = create_staff()
        self.user    = create_user('henry')
        self.product = create_product(stock=10)

    def test_restock_adds_stock(self):
        resp = auth_client(self.staff).post(
            f'/api/products/{self.product.id}/restock/',
            {'quantity': 5}
        )
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(resp.data['new_stock'], 15)

    def test_restock_regular_user_returns_403(self):
        resp = auth_client(self.user).post(
            f'/api/products/{self.product.id}/restock/',
            {'quantity': 5}
        )
        self.assertEqual(resp.status_code, status.HTTP_403_FORBIDDEN)

    def test_restock_invalid_quantity(self):
        resp = auth_client(self.staff).post(
            f'/api/products/{self.product.id}/restock/',
            {'quantity': -1}
        )
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)

    def test_stats_returns_expected_fields(self):
        resp = auth_client(self.user).get('/api/products/stats/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        for field in ['total_active', 'avg_price', 'total_stock', 'out_of_stock']:
            self.assertIn(field, resp.data)
```

---

## 6.8 Tests de Pedidos

```python
# store/tests/test_orders.py
from django.test import TestCase
from rest_framework import status

from .helpers import (
    create_user, create_staff, auth_client,
    create_product, create_order, add_item,
)


class OrderCRUDTests(TestCase):

    def setUp(self):
        self.user    = create_user('ivan')
        self.client  = auth_client(self.user)
        self.product = create_product(stock=20)

    def test_create_empty_order(self):
        resp = self.client.post('/api/orders/', {})
        self.assertEqual(resp.status_code, status.HTTP_201_CREATED)
        self.assertEqual(resp.data['status'], 'pending')
        self.assertEqual(resp.data['num_items'], 0)

    def test_add_item_reduces_stock(self):
        order       = create_order(self.user)
        stock_before = self.product.stock
        resp = self.client.post(
            f'/api/orders/{order.id}/add-item/',
            {'product_id': self.product.id, 'quantity': 3}
        )
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.product.refresh_from_db()
        self.assertEqual(self.product.stock, stock_before - 3)

    def test_add_same_item_increments_quantity(self):
        order = create_order(self.user)
        self.client.post(
            f'/api/orders/{order.id}/add-item/',
            {'product_id': self.product.id, 'quantity': 2}
        )
        resp = self.client.post(
            f'/api/orders/{order.id}/add-item/',
            {'product_id': self.product.id, 'quantity': 3}
        )
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(resp.data['num_items'], 1)

    def test_insufficient_stock_returns_400(self):
        order = create_order(self.user)
        resp  = self.client.post(
            f'/api/orders/{order.id}/add-item/',
            {'product_id': self.product.id, 'quantity': 999}
        )
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)

    def test_confirm_order_with_items(self):
        order = create_order(self.user)
        add_item(order, self.product)
        resp  = self.client.post(f'/api/orders/{order.id}/confirm/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(resp.data['status'], 'confirmed')

    def test_confirm_empty_order_returns_400(self):
        order = create_order(self.user)
        resp  = self.client.post(f'/api/orders/{order.id}/confirm/')
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)

    def test_cannot_add_item_to_confirmed_order(self):
        order = create_order(self.user, status='confirmed')
        resp  = self.client.post(
            f'/api/orders/{order.id}/add-item/',
            {'product_id': self.product.id, 'quantity': 1}
        )
        self.assertEqual(resp.status_code, status.HTTP_400_BAD_REQUEST)


class OrderPermissionTests(TestCase):

    def setUp(self):
        self.user1   = create_user('julia')
        self.user2   = create_user('kevin')
        self.staff   = create_staff()
        self.product = create_product(stock=20)
        self.order   = create_order(self.user1)

    def test_user_cannot_see_other_users_order(self):
        resp = auth_client(self.user2).get(f'/api/orders/{self.order.id}/')
        self.assertEqual(resp.status_code, status.HTTP_404_NOT_FOUND)

    def test_staff_can_see_any_order(self):
        resp = auth_client(self.staff).get(f'/api/orders/{self.order.id}/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)

    def test_staff_can_update_status(self):
        add_item(self.order, self.product)
        self.order.status = 'confirmed'
        self.order.save()
        resp = auth_client(self.staff).post(
            f'/api/orders/{self.order.id}/update-status/',
            {'status': 'shipped'}
        )
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        self.assertEqual(resp.data['status'], 'shipped')

    def test_regular_user_cannot_update_status(self):
        resp = auth_client(self.user1).post(
            f'/api/orders/{self.order.id}/update-status/',
            {'status': 'shipped'}
        )
        self.assertEqual(resp.status_code, status.HTTP_403_FORBIDDEN)


class OrderFilterTests(TestCase):

    def setUp(self):
        self.staff  = create_staff()
        self.client = auth_client(self.staff)
        user = create_user('laura')
        create_order(user, status='pending')
        create_order(user, status='confirmed')
        create_order(user, status='shipped')

    def test_filter_by_status(self):
        resp = self.client.get('/api/orders/?status=confirmed')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        for order in resp.data['results']:
            self.assertEqual(order['status'], 'confirmed')

    def test_stats_staff_only(self):
        resp = self.client.get('/api/orders/stats/')
        self.assertEqual(resp.status_code, status.HTTP_200_OK)
        for field in ['total_orders', 'total_revenue', 'by_status']:
            self.assertIn(field, resp.data)

    def test_stats_regular_user_returns_403(self):
        resp = auth_client(create_user('mario')).get('/api/orders/stats/')
        self.assertEqual(resp.status_code, status.HTTP_403_FORBIDDEN)
```

---

## 6.9 Ejecutar los tests

```bash
# Suite completa
uv run python manage.py test store

# Por módulo
uv run python manage.py test store.tests.test_auth
uv run python manage.py test store.tests.test_users
uv run python manage.py test store.tests.test_categories
uv run python manage.py test store.tests.test_products
uv run python manage.py test store.tests.test_orders

# Un test específico
uv run python manage.py test store.tests.test_orders.OrderCRUDTests.test_create_empty_order

# Con detalle de cada test
uv run python manage.py test store --verbosity=2
```

---

## ✅ Checkpoint Etapa 6

```
......................................................
------------------------------------------------------
Ran 42 tests in 4.1s

OK
```

---

## Resumen del curso completo

| Etapa | Contenido | Checkpoint |
|-------|-----------|------------|
| 1 | Setup + PostgreSQL + variables de entorno | `GET /api/health/` → 200 |
| 2 | Auth JWT + CRUD Usuarios | Login, registro, perfil, cambio de contraseña |
| 3 | CRUD Categorías | Filtros, búsqueda, stats |
| 4 | CRUD Productos | Rangos precio/stock, restock, available |
| 5 | CRUD Pedidos | Ciclo de vida, permisos por propietario |
| 6 | Testing completo | 42 tests pasando |

---

## Próximos pasos

| Área | Herramienta |
|------|-------------|
| Documentación automática | `drf-spectacular` (OpenAPI / Swagger) |
| Tests avanzados | `pytest-django`, `factory_boy` |
| Despliegue | `gunicorn` + `nginx` + PostgreSQL en producción |
| Cache | `redis` + `django-cache-machine` |
| Tareas asíncronas | `celery` + `redis` |
| WebSockets | `django-channels` |