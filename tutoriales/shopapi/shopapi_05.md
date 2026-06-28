# ShopAPI — Etapa 5
## CRUD de Pedidos

> **Objetivo:** Modelos `Order` + `OrderItem` con lógica de negocio, serializers anidados, paginación, filtros por estado/fecha y acciones para gestionar el ciclo de vida del pedido.
> **Checkpoint final:** crear pedido, agregar ítems, confirmar y actualizar estado funcionando con permisos por propietario.

---

## 5.1 Modelos

```python
# store/models/order.py
from django.db import models
from django.contrib.auth.models import User
from .product import Product


class Order(models.Model):
    STATUS_CHOICES = [
        ('pending',   'Pending'),
        ('confirmed', 'Confirmed'),
        ('shipped',   'Shipped'),
        ('delivered', 'Delivered'),
        ('cancelled', 'Cancelled'),
    ]

    user       = models.ForeignKey(User, on_delete=models.CASCADE, related_name='orders')
    status     = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    total      = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return f'Order #{self.id} — {self.user.username} ({self.status})'

    def calculate_total(self):
        self.total = sum(
            item.unit_price * item.quantity
            for item in self.items.all()
        )
        self.save(update_fields=['total'])


class OrderItem(models.Model):
    order      = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    product    = models.ForeignKey(Product, on_delete=models.PROTECT)
    quantity   = models.PositiveIntegerField(default=1)
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)

    @property
    def subtotal(self):
        return float(self.unit_price) * self.quantity

    def __str__(self):
        return f'{self.quantity}x {self.product.name}'
```

### `store/models/__init__.py` — actualizar

```python
# store/models/__init__.py
from .category import Category
from .product  import Product
from .order    import Order, OrderItem

__all__ = ['Category', 'Product', 'Order', 'OrderItem']
```

### Migración

```bash
uv run python manage.py makemigrations store
uv run python manage.py migrate
```

---

## 5.2 Permiso de propietario — actualizar `store/permissions.py`

```python
# store/permissions.py
from rest_framework.permissions import BasePermission, SAFE_METHODS


class IsStaffOrReadOnly(BasePermission):
    def has_permission(self, request, view):
        if request.method in SAFE_METHODS:
            return bool(request.user and request.user.is_authenticated)
        return bool(request.user and request.user.is_staff)


class IsOwnerOrStaff(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.user == request.user or request.user.is_staff
```

---

## 5.2.1 Actualizar `UserSerializer` — agregar `num_orders`

> **Este es el momento.** El modelo `Order` ya existe con `related_name='orders'`
> en el FK a `User`. Ahora `obj.orders.count()` funciona correctamente.
> Esto fue diferido desde la Etapa 2 para evitar el `AttributeError`.

```python
# store/serializers/user.py — actualizar UserSerializer
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

    class Meta:
        model  = User
        fields = [
            'id', 'username', 'email', 'first_name', 'last_name',
            'is_staff', 'is_active', 'date_joined', 'num_orders',
        ]
        read_only_fields = ['id', 'date_joined']

    def get_num_orders(self, obj):
        return obj.orders.count()


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

## 5.3 Filtro — actualizar `store/filters.py`

```python
# store/filters.py
import django_filters
from store.models import Category, Product, Order


class CategoryFilter(django_filters.FilterSet):
    name = django_filters.CharFilter(lookup_expr='icontains')

    class Meta:
        model  = Category
        fields = ['is_active']


class ProductFilter(django_filters.FilterSet):
    name          = django_filters.CharFilter(lookup_expr='icontains')
    price_min     = django_filters.NumberFilter(field_name='price', lookup_expr='gte')
    price_max     = django_filters.NumberFilter(field_name='price', lookup_expr='lte')
    stock_min     = django_filters.NumberFilter(field_name='stock', lookup_expr='gte')
    stock_max     = django_filters.NumberFilter(field_name='stock', lookup_expr='lte')
    category_name = django_filters.CharFilter(
        field_name='category__name', lookup_expr='icontains'
    )

    class Meta:
        model  = Product
        fields = ['is_active', 'category']


class OrderFilter(django_filters.FilterSet):
    from_date = django_filters.DateFilter(field_name='created_at', lookup_expr='date__gte')
    to_date   = django_filters.DateFilter(field_name='created_at', lookup_expr='date__lte')

    class Meta:
        model  = Order
        fields = ['status']
```

---

## 5.4 Serializers

```python
# store/serializers/order.py
from rest_framework import serializers
from store.models import Order, OrderItem, Product
from store.serializers.product import ProductSummarySerializer


class OrderItemSerializer(serializers.ModelSerializer):
    product = ProductSummarySerializer(read_only=True)
    subtotal = serializers.SerializerMethodField()

    class Meta:
        model  = OrderItem
        fields = ['id', 'product', 'quantity', 'unit_price', 'subtotal']
        read_only_fields = ['id', 'unit_price']

    def get_subtotal(self, obj):
        return obj.subtotal


class OrderSerializer(serializers.ModelSerializer):
    items         = OrderItemSerializer(many=True, read_only=True)
    username      = serializers.CharField(source='user.username', read_only=True)
    num_items     = serializers.SerializerMethodField()

    class Meta:
        model  = Order
        fields = [
            'id', 'username', 'status', 'total',
            'num_items', 'items', 'created_at', 'updated_at',
        ]
        read_only_fields = ['id', 'total', 'created_at', 'updated_at']

    def get_num_items(self, obj):
        return obj.items.count()


class AddItemSerializer(serializers.Serializer):
    product_id = serializers.IntegerField()
    quantity   = serializers.IntegerField(min_value=1)

    def validate_product_id(self, value):
        try:
            Product.objects.get(pk=value, is_active=True)
        except Product.DoesNotExist:
            raise serializers.ValidationError(
                f'Product {value} not found or inactive.'
            )
        return value

    def validate(self, data):
        product = Product.objects.get(pk=data['product_id'])
        if product.stock < data['quantity']:
            raise serializers.ValidationError(
                f'Insufficient stock: only {product.stock} units available.'
            )
        return data
```

### `store/serializers/__init__.py` — actualizar

```python
# store/serializers/__init__.py
from .auth    import CustomTokenSerializer, CustomTokenView
from .user    import (
    RegisterSerializer,
    UserSerializer,
    UserProfileSerializer,
    ChangePasswordSerializer,
)
from .category import CategorySerializer
from .product  import ProductSerializer, ProductSummarySerializer
from .order    import OrderSerializer, OrderItemSerializer, AddItemSerializer
```

---

## 5.5 View

```python
# store/views/order.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.permissions import IsAuthenticated, IsAdminUser
from rest_framework.response import Response
from rest_framework.filters import OrderingFilter
from django_filters.rest_framework import DjangoFilterBackend

from store.models import Order, OrderItem, Product
from store.serializers.order import OrderSerializer, AddItemSerializer
from store.permissions import IsOwnerOrStaff
from store.filters    import OrderFilter
from store.pagination import StandardPagination


class OrderViewSet(viewsets.ModelViewSet):
    serializer_class   = OrderSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrStaff]
    pagination_class   = StandardPagination
    filter_backends    = [DjangoFilterBackend, OrderingFilter]
    filterset_class    = OrderFilter
    ordering_fields    = ['created_at', 'total']
    ordering           = ['-created_at']
    http_method_names  = ['get', 'post', 'patch', 'delete', 'head', 'options']

    def get_queryset(self):
        if self.request.user.is_staff:
            return (
                Order.objects
                .select_related('user')
                .prefetch_related('items__product')
                .all()
            )
        return (
            Order.objects
            .filter(user=self.request.user)
            .prefetch_related('items__product')
        )

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)

    @action(detail=True, methods=['post'], url_path='add-item')
    def add_item(self, request, pk=None):
        order = self.get_object()
        if order.status != 'pending':
            return Response(
                {'error': f'Cannot modify an order with status "{order.status}".'},
                status=status.HTTP_400_BAD_REQUEST,
            )
        serializer = AddItemSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        product  = Product.objects.get(pk=serializer.validated_data['product_id'])
        quantity = serializer.validated_data['quantity']

        item, created = OrderItem.objects.get_or_create(
            order=order,
            product=product,
            defaults={'unit_price': product.price, 'quantity': quantity},
        )
        if not created:
            item.quantity += quantity
            item.save(update_fields=['quantity'])

        product.stock -= quantity
        product.save(update_fields=['stock'])
        order.calculate_total()

        return Response(OrderSerializer(order).data)

    @action(detail=True, methods=['post'], url_path='confirm')
    def confirm(self, request, pk=None):
        order = self.get_object()
        if order.status != 'pending':
            return Response(
                {'error': 'Only pending orders can be confirmed.'},
                status=status.HTTP_400_BAD_REQUEST,
            )
        if not order.items.exists():
            return Response(
                {'error': 'Cannot confirm an order with no items.'},
                status=status.HTTP_400_BAD_REQUEST,
            )
        order.status = 'confirmed'
        order.save(update_fields=['status'])
        return Response(OrderSerializer(order).data)

    @action(
        detail=True,
        methods=['post'],
        permission_classes=[IsAdminUser],
        url_path='update-status',
    )
    def update_status(self, request, pk=None):
        order         = self.get_object()
        new_status    = request.data.get('status')
        valid_statuses = [s[0] for s in Order.STATUS_CHOICES]

        if new_status not in valid_statuses:
            return Response(
                {'error': f'Invalid status. Valid options: {valid_statuses}'},
                status=status.HTTP_400_BAD_REQUEST,
            )
        order.status = new_status
        order.save(update_fields=['status'])
        return Response(OrderSerializer(order).data)

    @action(
        detail=False,
        methods=['get'],
        permission_classes=[IsAdminUser],
        url_path='stats',
    )
    def stats(self, request):
        from django.db.models import Count, Sum
        qs     = Order.objects.all()
        totals = qs.aggregate(
            total_orders = Count('id'),
            total_revenue = Sum('total'),
        )
        by_status = {
            s: qs.filter(status=s).count()
            for s, _ in Order.STATUS_CHOICES
        }
        return Response({
            'total_orders':   totals['total_orders'],
            'total_revenue':  float(totals['total_revenue'] or 0),
            'by_status':      by_status,
        })
```

---

## 5.6 URLs — actualizar `store/urls.py`

```python
# store/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from rest_framework_simplejwt.views import TokenRefreshView, TokenVerifyView

from store.views.health    import health_check
from store.views.auth      import RegisterView, LogoutView
from store.views.user      import UserViewSet
from store.views.category  import CategoryViewSet
from store.views.product   import ProductViewSet
from store.views.order     import OrderViewSet
from store.serializers.auth import CustomTokenView

router = DefaultRouter()
router.register('users',      UserViewSet,     basename='user')
router.register('categories', CategoryViewSet, basename='category')
router.register('products',   ProductViewSet,  basename='product')
router.register('orders',     OrderViewSet,    basename='order')

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

## 5.7 Admin — actualizar `store/admin.py`

```python
# store/admin.py
from django.contrib import admin
from store.models import Category, Product, Order, OrderItem


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
```

---

## ✅ Checkpoint Etapa 5

Flujo completo de un pedido a probar en orden:

| # | Endpoint | Método | Auth | Esperado |
|---|----------|--------|------|----------|
| 1 | `/api/orders/` | POST | Bearer | 201 pedido vacío |
| 2 | `/api/orders/{id}/add-item/` | POST | Bearer | 200 ítem agregado |
| 3 | `/api/orders/{id}/add-item/` | POST | Bearer | 200 cantidad incrementada |
| 4 | `/api/orders/{id}/` | GET | Bearer | 200 detalle con ítems |
| 5 | `/api/orders/{id}/confirm/` | POST | Bearer | 200 status=confirmed |
| 6 | `/api/orders/?status=confirmed` | GET | Bearer | 200 filtrado |
| 7 | `/api/orders/?from_date=2025-01-01` | GET | Bearer | 200 filtrado por fecha |
| 8 | `/api/orders/{id}/update-status/` | POST | Staff | 200 status=shipped |
| 9 | `/api/orders/stats/` | GET | Staff | 200 resumen |

---

## 📦 Colección Postman — Etapa 5

```json
{
  "info": {
    "name": "ShopAPI — Stage 5: Orders",
    "_postman_id": "shopapi-stage-05",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Create order",
      "event": [{ "listen": "test", "script": { "exec": [
        "pm.test('Status 201', () => pm.response.to.have.status(201));",
        "pm.collectionVariables.set('order_id', pm.response.json().id);"
      ]}}],
      "request": {
        "method": "POST",
        "header": [
          { "key": "Content-Type",  "value": "application/json" },
          { "key": "Authorization", "value": "Bearer {{access}}" }
        ],
        "body": { "mode": "raw", "raw": "{}" },
        "url": "{{base_url}}/orders/"
      }
    },
    {
      "name": "Add item to order",
      "request": {
        "method": "POST",
        "header": [
          { "key": "Content-Type",  "value": "application/json" },
          { "key": "Authorization", "value": "Bearer {{access}}" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"product_id\": {{product_id}},\n  \"quantity\": 2\n}"
        },
        "url": "{{base_url}}/orders/{{order_id}}/add-item/"
      }
    },
    {
      "name": "Get order detail",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": "{{base_url}}/orders/{{order_id}}/"
      }
    },
    {
      "name": "Confirm order",
      "request": {
        "method": "POST",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": "{{base_url}}/orders/{{order_id}}/confirm/"
      }
    },
    {
      "name": "List my orders",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": "{{base_url}}/orders/"
      }
    },
    {
      "name": "Filter by status",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": {
          "raw": "{{base_url}}/orders/?status=confirmed",
          "query": [{ "key": "status", "value": "confirmed" }]
        }
      }
    },
    {
      "name": "Filter by date range",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": {
          "raw": "{{base_url}}/orders/?from_date=2025-01-01&to_date=2025-12-31",
          "query": [
            { "key": "from_date", "value": "2025-01-01" },
            { "key": "to_date",   "value": "2025-12-31" }
          ]
        }
      }
    },
    {
      "name": "[Staff] Update status",
      "request": {
        "method": "POST",
        "header": [
          { "key": "Content-Type",  "value": "application/json" },
          { "key": "Authorization", "value": "Bearer {{access}}" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"status\": \"shipped\"\n}"
        },
        "url": "{{base_url}}/orders/{{order_id}}/update-status/"
      }
    },
    {
      "name": "[Staff] Stats",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": "{{base_url}}/orders/stats/"
      }
    }
  ],
  "variable": [
    { "key": "base_url",    "value": "http://localhost:8000/api" },
    { "key": "access",      "value": "" },
    { "key": "refresh",     "value": "" },
    { "key": "product_id",  "value": "" },
    { "key": "order_id",    "value": "" }
  ]
}
```

---

## Resumen

| Elemento | Estado |
|---|---|
| Modelos `Order` + `OrderItem` + migración | ✅ |
| Permiso `IsOwnerOrStaff` | ✅ |
| `OrderFilter` por status y rango de fechas | ✅ |
| Serializers anidados con `subtotal` calculado | ✅ |
| `get_queryset` diferenciado usuario/staff | ✅ |
| `select_related` + `prefetch_related` para evitar N+1 | ✅ |
| Acciones: `add_item`, `confirm`, `update_status`, `stats` | ✅ |
| Admin con inline de ítems | ✅ |
| Colección Postman con flujo completo | ✅ |

**Siguiente etapa →** Testing completo