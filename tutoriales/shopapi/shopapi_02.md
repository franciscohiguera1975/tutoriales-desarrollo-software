# ShopAPI — Etapa 2
## Auth JWT + CRUD de Usuarios

> **Objetivo:** Sistema de autenticación JWT completo + CRUD de usuarios con paginación, filtros, búsqueda, ordenamiento y acciones extra.
> **Checkpoint final:** registro, login, refresh, logout, perfil propio y gestión de usuarios por staff funcionando.

---

## 2.1 Modelo

En esta etapa usamos el modelo `User` nativo de Django — no hay que crear nada en `models/`.
Django ya provee `username`, `email`, `password`, `is_staff` e `is_active`.

---

## 2.2 Serializers de Auth

```python
# store/serializers/auth.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView


class CustomTokenSerializer(TokenObtainPairSerializer):

    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token['username'] = user.username
        token['email']    = user.email
        token['is_staff'] = user.is_staff
        return token

    def validate(self, attrs):
        data = super().validate(attrs)
        data['user_id']  = self.user.id
        data['username'] = self.user.username
        data['email']    = self.user.email
        data['is_staff'] = self.user.is_staff
        return data


class CustomTokenView(TokenObtainPairView):
    serializer_class = CustomTokenSerializer
```

---

## 2.3 Serializers de Usuario

> **Nota:** El campo `num_orders` se agrega en la **Etapa 5** cuando el modelo `Order`
> ya existe. Incluirlo antes provoca `AttributeError: 'User' object has no attribute 'orders'`.

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
    """
    Vista completa del usuario para staff.
    El campo num_orders se incorpora en la Etapa 5
    una vez que el modelo Order esté disponible.
    """
    class Meta:
        model  = User
        fields = [
            'id', 'username', 'email', 'first_name', 'last_name',
            'is_staff', 'is_active', 'date_joined',
        ]
        read_only_fields = ['id', 'date_joined']


class UserProfileSerializer(serializers.ModelSerializer):

    class Meta:
        model  = User
        fields = ['id', 'username', 'email', 'first_name', 'last_name']
        read_only_fields = ['id']

    def validate_email(self, value):
        request = self.context.get('request')
        if User.objects.filter(email=value).exclude(pk=request.user.pk).exists():
            raise serializers.ValidationError('This email is already in use.')
        return value


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

## 2.4 `serializers/__init__.py`

```python
# store/serializers/__init__.py
from .auth import CustomTokenSerializer, CustomTokenView
from .user import (
    RegisterSerializer,
    UserSerializer,
    UserProfileSerializer,
    ChangePasswordSerializer,
)
```

---

## 2.5 Views de Auth

```python
# store/views/auth.py
from rest_framework import status
from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.exceptions import TokenError

from store.serializers.user import RegisterSerializer


class RegisterView(APIView):
    permission_classes = [AllowAny]

    def post(self, request):
        serializer = RegisterSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user    = serializer.save()
        refresh = RefreshToken.for_user(user)
        return Response({
            'access':   str(refresh.access_token),
            'refresh':  str(refresh),
            'user_id':  user.id,
            'username': user.username,
            'email':    user.email,
            'is_staff': user.is_staff,
        }, status=status.HTTP_201_CREATED)


class LogoutView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        refresh_token = request.data.get('refresh')
        if not refresh_token:
            return Response(
                {'error': 'Refresh token is required.'},
                status=status.HTTP_400_BAD_REQUEST,
            )
        try:
            RefreshToken(refresh_token).blacklist()
        except TokenError:
            return Response(
                {'error': 'Token is invalid or expired.'},
                status=status.HTTP_400_BAD_REQUEST,
            )
        return Response({'message': 'Session closed successfully.'})
```

---

## 2.6 Views de Usuarios

```python
# store/views/user.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.permissions import IsAuthenticated, IsAdminUser
from rest_framework.response import Response
from rest_framework.filters import SearchFilter, OrderingFilter
from django_filters.rest_framework import DjangoFilterBackend
from django.contrib.auth.models import User

from store.serializers.user import (
    UserSerializer,
    UserProfileSerializer,
    ChangePasswordSerializer,
)
from store.pagination import StandardPagination


class UserViewSet(viewsets.ModelViewSet):
    queryset           = User.objects.all().order_by('id')
    serializer_class   = UserSerializer
    permission_classes = [IsAdminUser]
    pagination_class   = StandardPagination
    filter_backends    = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields   = ['is_staff', 'is_active']
    search_fields      = ['username', 'email', 'first_name', 'last_name']
    ordering_fields    = ['id', 'username', 'date_joined']
    ordering           = ['id']

    @action(
        detail=False,
        methods=['get', 'patch'],
        permission_classes=[IsAuthenticated],
        url_path='profile',
    )
    def profile(self, request):
        if request.method == 'GET':
            return Response(
                UserProfileSerializer(request.user, context={'request': request}).data
            )
        serializer = UserProfileSerializer(
            request.user,
            data=request.data,
            partial=True,
            context={'request': request},
        )
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    @action(
        detail=False,
        methods=['post'],
        permission_classes=[IsAuthenticated],
        url_path='change-password',
    )
    def change_password(self, request):
        serializer = ChangePasswordSerializer(
            data=request.data,
            context={'request': request},
        )
        serializer.is_valid(raise_exception=True)
        request.user.set_password(serializer.validated_data['new_password'])
        request.user.save()
        return Response({'message': 'Password updated. Please log in again.'})

    @action(
        detail=True,
        methods=['post'],
        permission_classes=[IsAdminUser],
        url_path='toggle-active',
    )
    def toggle_active(self, request, pk=None):
        user = self.get_object()
        user.is_active = not user.is_active
        user.save(update_fields=['is_active'])
        state = 'activated' if user.is_active else 'deactivated'
        return Response({'message': f'User {state}.', 'is_active': user.is_active})

    @action(
        detail=False,
        methods=['get'],
        permission_classes=[IsAdminUser],
        url_path='stats',
    )
    def stats(self, request):
        qs = User.objects.all()
        return Response({
            'total':    qs.count(),
            'active':   qs.filter(is_active=True).count(),
            'inactive': qs.filter(is_active=False).count(),
            'staff':    qs.filter(is_staff=True).count(),
        })
```

---

## 2.7 `views/__init__.py`

```python
# store/views/__init__.py
```

---

## 2.8 URLs

```python
# store/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from rest_framework_simplejwt.views import TokenRefreshView, TokenVerifyView

from store.views.health import health_check
from store.views.auth   import RegisterView, LogoutView
from store.views.user   import UserViewSet
from store.serializers.auth import CustomTokenView

router = DefaultRouter()
router.register('users', UserViewSet, basename='user')

urlpatterns = [
    path('health/',             health_check),
    path('auth/register/',      RegisterView.as_view()),
    path('auth/login/',         CustomTokenView.as_view()),
    path('auth/token/refresh/', TokenRefreshView.as_view()),
    path('auth/token/verify/',  TokenVerifyView.as_view()),
    path('auth/logout/',        LogoutView.as_view()),
    path('', include(router.urls)),
]
```

---

## 2.9 Superusuario y arranque

```bash
uv run python manage.py createsuperuser
# username: admin | email: admin@shopapi.com | password: Admin1234!

uv run python manage.py runserver
```

---

## ✅ Checkpoint Etapa 2

| # | Endpoint | Método | Auth | Esperado |
|---|----------|--------|------|----------|
| 1 | `/api/auth/register/` | POST | No | 201 + tokens |
| 2 | `/api/auth/login/` | POST | No | 200 + tokens |
| 3 | `/api/auth/token/refresh/` | POST | No | 200 + nuevo access |
| 4 | `/api/users/profile/` | GET | Bearer | 200 perfil propio |
| 5 | `/api/users/profile/` | PATCH | Bearer | 200 perfil actualizado |
| 6 | `/api/users/change-password/` | POST | Bearer | 200 |
| 7 | `/api/users/` | GET | Staff | 200 lista paginada |
| 8 | `/api/users/?search=john&is_active=true` | GET | Staff | 200 filtrado |
| 9 | `/api/users/{id}/toggle-active/` | POST | Staff | 200 |
| 10 | `/api/users/stats/` | GET | Staff | 200 |
| 11 | `/api/auth/logout/` | POST | Bearer | 200 |

---

## 📦 Colección Postman — Etapa 2

```json
{
  "info": {
    "name": "ShopAPI — Stage 2: Auth + Users",
    "_postman_id": "shopapi-stage-02",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Auth",
      "item": [
        {
          "name": "Register",
          "event": [{ "listen": "test", "script": { "exec": [
            "pm.test('Status 201', () => pm.response.to.have.status(201));",
            "const j = pm.response.json();",
            "pm.collectionVariables.set('access',  j.access);",
            "pm.collectionVariables.set('refresh', j.refresh);"
          ]}}],
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": { "mode": "raw", "raw": "{\n  \"username\": \"john\",\n  \"email\": \"john@test.com\",\n  \"password\": \"Pass1234!\",\n  \"password2\": \"Pass1234!\"\n}" },
            "url": "{{base_url}}/auth/register/"
          }
        },
        {
          "name": "Login",
          "event": [{ "listen": "test", "script": { "exec": [
            "pm.test('Status 200', () => pm.response.to.have.status(200));",
            "const j = pm.response.json();",
            "pm.collectionVariables.set('access',  j.access);",
            "pm.collectionVariables.set('refresh', j.refresh);"
          ]}}],
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": { "mode": "raw", "raw": "{\n  \"username\": \"admin\",\n  \"password\": \"Admin1234!\"\n}" },
            "url": "{{base_url}}/auth/login/"
          }
        },
        {
          "name": "Refresh Token",
          "event": [{ "listen": "test", "script": { "exec": [
            "pm.test('Status 200', () => pm.response.to.have.status(200));",
            "const j = pm.response.json();",
            "pm.collectionVariables.set('access',  j.access);",
            "pm.collectionVariables.set('refresh', j.refresh);"
          ]}}],
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": { "mode": "raw", "raw": "{\n  \"refresh\": \"{{refresh}}\"\n}" },
            "url": "{{base_url}}/auth/token/refresh/"
          }
        },
        {
          "name": "Logout",
          "event": [{ "listen": "test", "script": { "exec": [
            "pm.test('Status 200', () => pm.response.to.have.status(200));"
          ]}}],
          "request": {
            "method": "POST",
            "header": [
              { "key": "Content-Type",  "value": "application/json" },
              { "key": "Authorization", "value": "Bearer {{access}}" }
            ],
            "body": { "mode": "raw", "raw": "{\n  \"refresh\": \"{{refresh}}\"\n}" },
            "url": "{{base_url}}/auth/logout/"
          }
        }
      ]
    },
    {
      "name": "Users",
      "item": [
        {
          "name": "Get my profile",
          "request": {
            "method": "GET",
            "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
            "url": "{{base_url}}/users/profile/"
          }
        },
        {
          "name": "Update my profile",
          "request": {
            "method": "PATCH",
            "header": [
              { "key": "Content-Type",  "value": "application/json" },
              { "key": "Authorization", "value": "Bearer {{access}}" }
            ],
            "body": { "mode": "raw", "raw": "{\n  \"first_name\": \"John\",\n  \"last_name\": \"Doe\"\n}" },
            "url": "{{base_url}}/users/profile/"
          }
        },
        {
          "name": "Change password",
          "request": {
            "method": "POST",
            "header": [
              { "key": "Content-Type",  "value": "application/json" },
              { "key": "Authorization", "value": "Bearer {{access}}" }
            ],
            "body": { "mode": "raw", "raw": "{\n  \"current_password\": \"Pass1234!\",\n  \"new_password\": \"New5678!\",\n  \"new_password2\": \"New5678!\"\n}" },
            "url": "{{base_url}}/users/change-password/"
          }
        },
        {
          "name": "[Staff] List users",
          "request": {
            "method": "GET",
            "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
            "url": {
              "raw": "{{base_url}}/users/?search=john&is_active=true&ordering=username",
              "query": [
                { "key": "search",    "value": "john" },
                { "key": "is_active", "value": "true" },
                { "key": "ordering",  "value": "username" }
              ]
            }
          }
        },
        {
          "name": "[Staff] Create user",
          "request": {
            "method": "POST",
            "header": [
              { "key": "Content-Type",  "value": "application/json" },
              { "key": "Authorization", "value": "Bearer {{access}}" }
            ],
            "body": { "mode": "raw", "raw": "{\n  \"username\": \"mary\",\n  \"email\": \"mary@test.com\",\n  \"password\": \"Pass1234!\",\n  \"first_name\": \"Mary\",\n  \"last_name\": \"Smith\"\n}" },
            "url": "{{base_url}}/users/"
          }
        },
        {
          "name": "[Staff] Toggle active",
          "request": {
            "method": "POST",
            "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
            "url": "{{base_url}}/users/2/toggle-active/"
          }
        },
        {
          "name": "[Staff] Stats",
          "request": {
            "method": "GET",
            "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
            "url": "{{base_url}}/users/stats/"
          }
        }
      ]
    }
  ],
  "variable": [
    { "key": "base_url", "value": "http://localhost:8000/api" },
    { "key": "access",   "value": "" },
    { "key": "refresh",  "value": "" }
  ]
}
```

---

## ⚠️ Actualización en Etapa 5

Cuando el modelo `Order` esté disponible, actualizar `UserSerializer` en `store/serializers/user.py`
agregando el campo `num_orders`:

```python
class UserSerializer(serializers.ModelSerializer):
    num_orders = serializers.SerializerMethodField()

    class Meta:
        model  = User
        fields = [
            'id', 'username', 'email', 'first_name', 'last_name',
            'is_staff', 'is_active', 'date_joined', 'num_orders',
        ]
        read_only_fields = ['id', 'date_joined']

    def get_num_orders(self, obj):
        return obj.orders.count()
```

---

## Resumen

| Elemento | Estado |
|---|---|
| `CustomTokenSerializer` con claims extra | ✅ |
| Registro con validaciones de unicidad | ✅ |
| Login / Refresh / Logout con blacklist | ✅ |
| CRUD de usuarios (staff) con paginación y filtros | ✅ |
| Acciones: `profile`, `change-password`, `toggle-active`, `stats` | ✅ |
| `num_orders` diferido a Etapa 5 — sin `AttributeError` | ✅ |
| Colección Postman con test scripts | ✅ |

**Siguiente etapa →** CRUD de Categorías