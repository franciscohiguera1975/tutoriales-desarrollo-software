# ShopAPI — Etapa 7
## Imágenes: foto de producto y avatar de usuario

> **Objetivo:** Agregar un campo `image` al modelo `Product` y un modelo `UserProfile` con campo `avatar` vinculado a `User`. Configurar almacenamiento de archivos, serializers para carga/lectura de imágenes y señales para crear el perfil automáticamente.
> **Estructura:** Dos rodajas verticales independientes. Cada una termina con una parada de prueba antes de continuar.

| Parada | Qué se prueba |
|--------|--------------|
| 🚦 1 | Subir avatar de usuario y ver URL absoluta en la respuesta |
| 🚦 2 | Subir imagen de producto y ver URL absoluta en la respuesta |

---

## 6.1 Instalar Pillow

Django requiere Pillow para usar `ImageField`.

```bash
uv add Pillow
```

---

## 6.2 Configurar MEDIA — actualizar `config/settings.py`

Agregar al final del archivo, junto a `STATIC_URL`:

```python
# config/settings.py  (agregar estas líneas)
MEDIA_URL  = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

---

## 6.3 Servir archivos en desarrollo — actualizar `config/urls.py`

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('store.urls')),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

---

## ── Parte 1 · Avatar de usuario ─────────────────────────────────────────────

## 6.4 Modelo `UserProfile` — nuevo archivo `store/models/profile.py`

```python
# store/models/profile.py
import uuid
from pathlib import Path
from django.db import models
from django.contrib.auth.models import User


def avatar_upload_path(instance, filename):
    ext = Path(filename).suffix.lower()
    return f'avatars/user_{instance.user_id}/{uuid.uuid4()}{ext}'


class UserProfile(models.Model):
    user   = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    avatar = models.ImageField(upload_to=avatar_upload_path, blank=True, null=True)

    def __str__(self):
        return f'Profile of {self.user.username}'
```

---

## 6.5 `store/models/__init__.py` — actualizar

```python
# store/models/__init__.py
from .category import Category
from .product  import Product
from .order    import Order, OrderItem
from .profile  import UserProfile

__all__ = ['Category', 'Product', 'Order', 'OrderItem', 'UserProfile']
```

---

## 6.6 Señales — nuevo archivo `store/signals.py`

Crea automáticamente un `UserProfile` cada vez que se registra un nuevo usuario.

```python
# store/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User
from store.models.profile import UserProfile


@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        UserProfile.objects.create(user=instance)
```

---

## 6.7 Registrar señales — actualizar `store/apps.py`

```python
# store/apps.py
from django.apps import AppConfig


class StoreConfig(AppConfig):
    name = 'store'

    def ready(self):
        import store.signals  # noqa: F401
```

---

## 6.8 Migración — UserProfile

```bash
uv run python manage.py makemigrations store
uv run python manage.py migrate
```

> **Usuarios existentes en la base de datos** no tendrán `UserProfile` todavía. Si necesitas crearlos, hazlo desde el admin de Django: cada usuario que edites en `/admin/` ya mostrará el inline de perfil después de la siguiente sección.

---

## 6.9 Serializers de usuario — actualizar `store/serializers/user.py`

Solo hay que actualizar `UserSerializer` (agrega `avatar_url`) y `UserProfileSerializer` (agrega soporte para subir y leer el avatar). Los demás serializers no cambian.

```python
# store/serializers/user.py
from rest_framework import serializers
from django.contrib.auth.models import User


class RegisterSerializer(serializers.Serializer):
    username  = serializers.CharField(max_length=150)
    email     = serializers.EmailField()
    password  = serializers.CharField(min_length=8, write_only=True)
    password2 = serializers.CharField(write_only=True)

    def validate_username(self, value):
        if User.objects.filter(username=value).exists():
            raise serializers.ValidationError('This username is already taken.')
        return value

    def validate_email(self, value):
        if User.objects.filter(email=value).exists():
            raise serializers.ValidationError('This email is already registered.')
        return value

    def validate(self, data):
        if data['password'] != data['password2']:
            raise serializers.ValidationError({'password2': 'Passwords do not match.'})
        return data

    def create(self, validated_data):
        validated_data.pop('password2')
        return User.objects.create_user(**validated_data)


class UserSerializer(serializers.ModelSerializer):
    num_orders = serializers.SerializerMethodField()
    avatar_url = serializers.SerializerMethodField()   # ← nuevo

    class Meta:
        model  = User
        fields = [
            'id', 'username', 'email', 'first_name', 'last_name',
            'is_staff', 'is_active', 'date_joined', 'num_orders', 'avatar_url',
        ]
        read_only_fields = ['id', 'date_joined']

    def get_num_orders(self, obj):
        return obj.orders.count()

    def get_avatar_url(self, obj):                     # ← nuevo
        request = self.context.get('request')
        try:
            avatar = obj.profile.avatar
            if avatar:
                return request.build_absolute_uri(avatar.url) if request else avatar.url
        except Exception:
            pass
        return None


class UserProfileSerializer(serializers.ModelSerializer):
    avatar     = serializers.ImageField(              # ← nuevo
        source='profile.avatar', required=False, allow_null=True
    )
    avatar_url = serializers.SerializerMethodField()  # ← nuevo

    class Meta:
        model  = User
        fields = ['id', 'username', 'email', 'first_name', 'last_name', 'avatar', 'avatar_url']
        read_only_fields = ['id']
        extra_kwargs = {'avatar': {'write_only': True}}

    def get_avatar_url(self, obj):
        request = self.context.get('request')
        try:
            avatar = obj.profile.avatar
            if avatar:
                return request.build_absolute_uri(avatar.url) if request else avatar.url
        except Exception:
            pass
        return None

    def validate_email(self, value):
        request = self.context.get('request')
        if User.objects.filter(email=value).exclude(pk=request.user.pk).exists():
            raise serializers.ValidationError('This email is already in use.')
        return value

    def validate_avatar(self, value):
        max_size    = 2 * 1024 * 1024  # 2 MB
        valid_types = ['image/jpeg', 'image/png', 'image/webp']
        if value and value.size > max_size:
            raise serializers.ValidationError('Image size must not exceed 2 MB.')
        if value and value.content_type not in valid_types:
            raise serializers.ValidationError('Only JPEG, PNG, and WebP images are allowed.')
        return value

    def update(self, instance, validated_data):
        from store.models.profile import UserProfile
        profile_data = validated_data.pop('profile', {})
        instance     = super().update(instance, validated_data)
        if profile_data:
            profile, _ = UserProfile.objects.get_or_create(user=instance)
            for attr, value in profile_data.items():
                setattr(profile, attr, value)
            profile.save()
        return instance


class ChangePasswordSerializer(serializers.Serializer):
    current_password = serializers.CharField(write_only=True)
    new_password     = serializers.CharField(min_length=8, write_only=True)
    new_password2    = serializers.CharField(write_only=True)

    def validate_current_password(self, value):
        if not self.context['request'].user.check_password(value):
            raise serializers.ValidationError('Current password is incorrect.')
        return value

    def validate(self, data):
        if data['new_password'] != data['new_password2']:
            raise serializers.ValidationError({'new_password2': 'Passwords do not match.'})
        return data
```

---

## 🚦 PARADA 1 — Avatar de usuario

> **Qué está listo:** modelo `UserProfile`, señal que crea el perfil, migración aplicada, serializer con soporte de avatar.
> **Qué vas a probar:** que al registrarte el perfil existe, que `avatar_url` aparece como `null`, que puedes subir un avatar y obtener la URL absoluta.

**1 — Registrar un usuario nuevo**

```
POST /api/auth/register/
Content-Type: application/json

{
  "username": "ana",
  "email": "ana@test.com",
  "password": "test1234!!",
  "password2": "test1234!!"
}
```

Respuesta esperada: `201 Created`. El perfil se crea automáticamente por la señal.

---

**2 — Ver perfil sin avatar**

```
GET /api/users/profile/
Authorization: Bearer <access_token>
```

Respuesta esperada:
```json
{
  "id": 1,
  "username": "ana",
  "avatar_url": null
}
```

Si `avatar_url` no aparece en el JSON, el serializer no se guardó correctamente o el servidor no fue reiniciado.

---

**3 — Subir avatar**

```
PATCH /api/users/profile/
Authorization: Bearer <access_token>
Content-Type: multipart/form-data

avatar: <seleccionar archivo .jpg o .png>
```

Respuesta esperada:
```json
{
  "avatar_url": "http://localhost:8000/media/avatars/user_1/<uuid>.jpg"
}
```

**4 — Verificar en el navegador**

Abrir la URL de `avatar_url` directamente en el navegador → debe mostrarse la imagen.

> **Si retorna 404:** verificar que `config/urls.py` tiene el bloque `if settings.DEBUG: urlpatterns += static(...)` y que `DEBUG=True` en `.env`.
>
> **Si retorna 400 al subir:** revisar el tipo de archivo (solo JPEG, PNG, WebP) y que el tamaño sea menor a 2 MB.

---

## ── Parte 2 · Imagen de producto ────────────────────────────────────────────

## 6.10 Campo `image` en `store/models/product.py`

```python
# store/models/product.py
import uuid
from pathlib import Path
from django.db import models
from .category import Category


def product_image_path(instance, filename):
    ext = Path(filename).suffix.lower()
    return f'products/{uuid.uuid4()}{ext}'


class Product(models.Model):
    name        = models.CharField(max_length=200)
    description = models.TextField(blank=True, default='')
    price       = models.DecimalField(max_digits=10, decimal_places=2)
    stock       = models.PositiveIntegerField(default=0)
    is_active   = models.BooleanField(default=True)
    image       = models.ImageField(upload_to=product_image_path, blank=True, null=True)   # ← nuevo
    category    = models.ForeignKey(
        Category,
        on_delete=models.PROTECT,
        related_name='products',
    )
    created_at  = models.DateTimeField(auto_now_add=True)
    updated_at  = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['name']

    def __str__(self):
        return self.name

    @property
    def price_with_tax(self):
        return round(float(self.price) * 1.15, 2)

    @property
    def in_stock(self):
        return self.stock > 0
```

---

## 6.11 Migración — campo `image` en Product

```bash
uv run python manage.py makemigrations store
uv run python manage.py migrate
```

> Esto genera una segunda migración (solo agrega la columna `image` a la tabla de productos). No afecta los datos existentes.

---

## 6.12 Serializers de producto — actualizar `store/serializers/product.py`

```python
# store/serializers/product.py
from rest_framework import serializers
from store.models import Product
from store.serializers.category import CategorySerializer


class ProductSummarySerializer(serializers.ModelSerializer):
    image_url = serializers.SerializerMethodField()   # ← nuevo

    class Meta:
        model  = Product
        fields = ['id', 'name', 'price', 'stock', 'is_active', 'image_url']

    def get_image_url(self, obj):
        request = self.context.get('request')
        if obj.image:
            return request.build_absolute_uri(obj.image.url) if request else obj.image.url
        return None


class ProductSerializer(serializers.ModelSerializer):
    category       = CategorySerializer(read_only=True)
    category_id    = serializers.PrimaryKeyRelatedField(
        source='category',
        write_only=True,
        queryset=Product.objects.none(),
    )
    price_with_tax = serializers.SerializerMethodField()
    in_stock       = serializers.SerializerMethodField()
    image_url      = serializers.SerializerMethodField()   # ← nuevo

    class Meta:
        model  = Product
        fields = [
            'id', 'name', 'description',
            'price', 'price_with_tax',
            'stock', 'in_stock', 'is_active',
            'category', 'category_id',
            'image', 'image_url',             # ← nuevo
            'created_at', 'updated_at',
        ]
        read_only_fields = ['id', 'created_at', 'updated_at']
        extra_kwargs = {'image': {'required': False, 'allow_null': True}}

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        from store.models import Category
        self.fields['category_id'].queryset = Category.objects.filter(is_active=True)

    def get_price_with_tax(self, obj):
        return obj.price_with_tax

    def get_in_stock(self, obj):
        return obj.in_stock

    def get_image_url(self, obj):
        request = self.context.get('request')
        if obj.image:
            return request.build_absolute_uri(obj.image.url) if request else obj.image.url
        return None

    def validate_price(self, value):
        if value <= 0:
            raise serializers.ValidationError('Price must be greater than 0.')
        return value

    def validate_stock(self, value):
        if value < 0:
            raise serializers.ValidationError('Stock cannot be negative.')
        return value

    def validate_image(self, value):
        max_size    = 2 * 1024 * 1024  # 2 MB
        valid_types = ['image/jpeg', 'image/png', 'image/webp']
        if value and value.size > max_size:
            raise serializers.ValidationError('Image size must not exceed 2 MB.')
        if value and value.content_type not in valid_types:
            raise serializers.ValidationError('Only JPEG, PNG, and WebP images are allowed.')
        return value
```

---

## 6.13 Admin — actualizar `store/admin.py`

```python
# store/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
from django.contrib.auth.models import User
from store.models import Category, Product, Order, OrderItem, UserProfile


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display        = ['id', 'name', 'slug', 'is_active', 'created_at']
    list_filter         = ['is_active']
    search_fields       = ['name']
    prepopulated_fields = {'slug': ('name',)}


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display  = ['id', 'name', 'price', 'stock', 'is_active', 'category']
    list_filter   = ['is_active', 'category']
    search_fields = ['name', 'description']
    list_editable = ['price', 'stock', 'is_active']


class OrderItemInline(admin.TabularInline):
    model  = OrderItem
    extra  = 0
    fields = ['product', 'quantity', 'unit_price']


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display    = ['id', 'user', 'status', 'total', 'created_at']
    list_filter     = ['status']
    search_fields   = ['user__username']
    inlines         = [OrderItemInline]
    readonly_fields = ['total', 'created_at', 'updated_at']


class UserProfileInline(admin.StackedInline):
    model               = UserProfile
    can_delete          = False
    verbose_name_plural = 'Profile'
    fields              = ['avatar']


class UserAdmin(BaseUserAdmin):
    inlines = [UserProfileInline]


admin.site.unregister(User)
admin.site.register(User, UserAdmin)
```

---

## 🚦 PARADA 2 — Imagen de producto

> **Qué está listo:** campo `image` en `Product`, migración aplicada, serializer con `image_url`, admin con inline de perfil.
> **Qué vas a probar:** que `image_url` aparece como `null` en productos, que puedes subir una imagen como staff y obtener la URL absoluta.

**1 — Ver producto sin imagen**

```
GET /api/products/{id}/
```

Respuesta esperada:
```json
{
  "id": 1,
  "name": "Laptop Pro",
  "image_url": null
}
```

---

**2 — Subir imagen de producto (requiere staff)**

```
PATCH /api/products/{id}/
Authorization: Bearer <access_token_staff>
Content-Type: multipart/form-data

image: <seleccionar archivo .jpg o .png>
```

Respuesta esperada:
```json
{
  "image_url": "http://localhost:8000/media/products/<uuid>.jpg"
}
```

---

**3 — Verificar en el navegador**

Abrir la URL de `image_url` directamente → debe mostrarse la imagen.

---

**4 — Verificar inline en el admin**

Ir a `/admin/` → **Users** → seleccionar cualquier usuario → confirmar que aparece el bloque **Profile** con el campo avatar debajo de los datos del usuario.

---

**5 — Probar validación (imagen inválida)**

```
PATCH /api/products/{id}/
Authorization: Bearer <access_token_staff>
Content-Type: multipart/form-data

image: <archivo .pdf o archivo > 2 MB>
```

Respuesta esperada: `400 Bad Request` con el mensaje de validación.

---

## ✅ Checkpoint Etapa 7

| # | Acción | Endpoint | Método | Auth | Content-Type | Esperado |
|---|--------|----------|--------|------|--------------|----------|
| 1 | Registrar usuario | `/api/auth/register/` | POST | — | JSON | 201, `UserProfile` creado |
| 2 | Ver perfil (sin avatar) | `/api/users/profile/` | GET | Bearer | — | 200, `avatar_url: null` |
| 3 | Subir avatar | `/api/users/profile/` | PATCH | Bearer | multipart | 200, `avatar_url` con URL |
| 4 | Ver archivo en navegador | URL de `avatar_url` | GET | — | — | 200, imagen visible |
| 5 | Ver producto (sin imagen) | `/api/products/{id}/` | GET | — | — | 200, `image_url: null` |
| 6 | Subir imagen de producto | `/api/products/{id}/` | PATCH | Bearer (staff) | multipart | 200, `image_url` con URL |
| 7 | Ver archivo en navegador | URL de `image_url` | GET | — | — | 200, imagen visible |
| 8 | Subir archivo > 2 MB | `/api/products/{id}/` | PATCH | Bearer (staff) | multipart | 400, error de validación |

---

## 📦 Colección Postman — Etapa 7

```json
{
  "info": {
    "name": "ShopAPI — Stage 7: Images",
    "_postman_id": "shopapi-stage-07",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "── PARADA 1: Avatar ──",
      "item": [
        {
          "name": "Register user",
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"username\": \"ana\",\n  \"email\": \"ana@test.com\",\n  \"password\": \"test1234!!\",\n  \"password2\": \"test1234!!\"\n}"
            },
            "url": "{{base_url}}/auth/register/"
          }
        },
        {
          "name": "Get profile (avatar_url: null)",
          "request": {
            "method": "GET",
            "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
            "url": "{{base_url}}/users/profile/"
          }
        },
        {
          "name": "Upload avatar",
          "request": {
            "method": "PATCH",
            "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
            "body": {
              "mode": "formdata",
              "formdata": [
                { "key": "avatar", "type": "file", "src": "/path/to/avatar.jpg" }
              ]
            },
            "url": "{{base_url}}/users/profile/"
          }
        }
      ]
    },
    {
      "name": "── PARADA 2: Product image ──",
      "item": [
        {
          "name": "Get product (image_url: null)",
          "request": {
            "method": "GET",
            "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
            "url": "{{base_url}}/products/{{product_id}}/"
          }
        },
        {
          "name": "Upload product image (staff)",
          "request": {
            "method": "PATCH",
            "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
            "body": {
              "mode": "formdata",
              "formdata": [
                { "key": "image", "type": "file", "src": "/path/to/product.jpg" }
              ]
            },
            "url": "{{base_url}}/products/{{product_id}}/"
          }
        },
        {
          "name": "Upload oversized image (expect 400)",
          "request": {
            "method": "PATCH",
            "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
            "body": {
              "mode": "formdata",
              "formdata": [
                { "key": "image", "type": "file", "src": "/path/to/large_file.jpg" }
              ]
            },
            "url": "{{base_url}}/products/{{product_id}}/"
          }
        }
      ]
    }
  ],
  "variable": [
    { "key": "base_url",    "value": "http://localhost:8000/api" },
    { "key": "access",      "value": "" },
    { "key": "product_id",  "value": "" }
  ]
}
```

---

## Resumen

| Elemento | Parada |
|----------|--------|
| Pillow instalado | base |
| `MEDIA_URL` / `MEDIA_ROOT` en settings | base |
| Archivos servidos en desarrollo con `static()` | base |
| Modelo `UserProfile` con `avatar_upload_path` | 🚦 1 |
| Señal `post_save` que crea perfil en cada registro | 🚦 1 |
| `StoreConfig.ready()` registra señales | 🚦 1 |
| Migración de `UserProfile` | 🚦 1 |
| `avatar_url` absoluta en `UserSerializer` y `UserProfileSerializer` | 🚦 1 |
| Validación de tamaño (2 MB) y tipo (JPEG/PNG/WebP) en avatar | 🚦 1 |
| Campo `image` en `Product` con `product_image_path` | 🚦 2 |
| Migración de campo `image` | 🚦 2 |
| `image_url` absoluta en `ProductSerializer` y `ProductSummarySerializer` | 🚦 2 |
| Validación de tamaño y tipo en imagen de producto | 🚦 2 |
| Inline `UserProfileInline` en Admin | 🚦 2 |

**Siguiente etapa →** Servicio de correo electrónico (bienvenida, confirmación de orden, recuperación de contraseña)
