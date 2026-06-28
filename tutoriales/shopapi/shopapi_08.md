# ShopAPI — Etapa 8
## Servicio de correo electrónico

> **Objetivo:** Implementar un servicio de email reutilizable en Django para enviar correos en tres eventos clave. El código se construye en tres rodajas verticales: se escribe lo mínimo para cada evento y se prueba antes de continuar.

| Parada | Qué se prueba |
|--------|--------------|
| 🚦 1 | Registrar usuario → llega correo de bienvenida a Gmail |
| 🚦 2 | Solicitar reset → llega correo → confirmar token → login con nueva contraseña |
| 🚦 3 | Crear orden → llega correo de confirmación a Gmail |

---

## 8.1 Configuración de email — actualizar `config/settings.py`

Django soporta múltiples backends de correo. En desarrollo se usa el backend de consola (imprime en terminal); en producción se configura SMTP real.

```python
# config/settings.py  (agregar al final)
# --- Email -----------------------------------------------------------
EMAIL_BACKEND       = env('EMAIL_BACKEND', default='django.core.mail.backends.console.EmailBackend')
EMAIL_HOST          = env('EMAIL_HOST',    default='smtp.gmail.com')
EMAIL_PORT          = env.int('EMAIL_PORT', default=587)
EMAIL_USE_TLS       = env.bool('EMAIL_USE_TLS', default=True)
EMAIL_HOST_USER     = env('EMAIL_HOST_USER',     default='')
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD', default='')
DEFAULT_FROM_EMAIL  = env('DEFAULT_FROM_EMAIL',  default='ShopAPI <noreply@shopapi.local>')

# Timeout de conexión SMTP en segundos.
# CRÍTICO: sin este valor Django espera indefinidamente. Si el servidor SMTP
# no responde, gunicorn agota su propio timeout y mata el worker con SystemExit,
# que NO puede capturarse con `except Exception` y provoca un 500.
# Con EMAIL_TIMEOUT=5 Django lanza SMTPConnectError (subclase de Exception)
# antes de que gunicorn intervenga, y el error se puede manejar correctamente.
EMAIL_TIMEOUT = env.int('EMAIL_TIMEOUT', default=5)

# URL del frontend para armar enlaces en correos (recuperación de contraseña)
FRONTEND_URL = env('FRONTEND_URL', default='http://localhost:3000')

# Tiempo de validez del token de reset (en segundos). Por defecto Django usa 3 días.
PASSWORD_RESET_TIMEOUT = 86400  # 24 horas
```

> **Backends disponibles:**
> | Backend | Uso |
> |---------|-----|
> | `console.EmailBackend` | Desarrollo: imprime el correo completo en la terminal |
> | `filebased.EmailBackend` | Desarrollo: guarda cada correo como archivo en disco |
> | `smtp.EmailBackend` | Producción: envía correos reales vía SMTP |
>
> Cambiar entre backends solo requiere actualizar la variable de entorno, sin tocar código.

---

## 8.2 Variables de entorno — actualizar `.env`

```ini
# .env  (agregar)
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=tu_correo@gmail.com
EMAIL_HOST_PASSWORD=tu_app_password
DEFAULT_FROM_EMAIL=ShopAPI <noreply@shopapi.local>
FRONTEND_URL=http://localhost:3000
```

---

## 8.2.1 Configurar una cuenta Google personal para envío real

Gmail bloquea el acceso con la contraseña principal desde aplicaciones externas.
La solución oficial de Google es crear una **contraseña de aplicación** (App Password),
una clave de 16 caracteres generada exclusivamente para esta app.

### Paso 1 — Activar la verificación en dos pasos

1. Ir a [myaccount.google.com](https://myaccount.google.com)
2. En el menú lateral elegir **Seguridad**
3. Buscar la sección **"Cómo inicias sesión en Google"**
4. Hacer clic en **Verificación en 2 pasos**
5. Seguir los pasos del asistente (número de teléfono o app autenticadora)
6. Al terminar, el estado debe mostrar **Activada**

> Si ya tienes la verificación en dos pasos activa, pasar directamente al Paso 2.

---

### Paso 2 — Crear una contraseña de aplicación

1. Ir a [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
   *(también: Seguridad → Verificación en 2 pasos → bajar al final → Contraseñas de aplicaciones)*
2. En el campo **"Nombre de la app"** escribir: `ShopAPI Django`
3. Hacer clic en **Crear**
4. Google muestra una contraseña de **16 caracteres** en 4 grupos (ej.: `abcd efgh ijkl mnop`)
5. **Copiarla ahora** — Google no la vuelve a mostrar

> La contraseña tiene espacios solo por legibilidad. Al pegarla en `.env` se escribe **sin espacios**: `abcdefghijklmnop`

---

### Paso 3 — Actualizar `.env` con las credenciales reales

```ini
# .env  (reemplazar los valores de email)
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=tu_correo@gmail.com
EMAIL_HOST_PASSWORD=abcdefghijklmnop
DEFAULT_FROM_EMAIL=ShopAPI <tu_correo@gmail.com>
FRONTEND_URL=http://localhost:3000
```

> **Cambios respecto al valor de desarrollo:**
> - `EMAIL_BACKEND` cambia de `console` a `smtp`
> - `EMAIL_HOST_USER` es tu dirección Gmail completa
> - `EMAIL_HOST_PASSWORD` es la App Password de 16 caracteres (sin espacios)
> - `DEFAULT_FROM_EMAIL` debe coincidir con `EMAIL_HOST_USER` para que Gmail acepte el envío

---

> **Errores comunes de conexión SMTP:**
>
> | Error | Causa | Solución |
> |-------|-------|----------|
> | `SMTPAuthenticationError (535)` | Contraseña incorrecta o sin App Password | Usar la App Password, no la contraseña principal |
> | `SMTPAuthenticationError (534)` | Verificación en dos pasos no activada | Completar el Paso 1 |
> | `Connection refused` | Puerto bloqueado por la red | Probar `EMAIL_PORT=465`, `EMAIL_USE_TLS=False`, `EMAIL_USE_SSL=True` |
> | Correo en **Spam** | `DEFAULT_FROM_EMAIL` no coincide con `EMAIL_HOST_USER` | Asegurarse de que coincidan exactamente |

---

## ── Parte 1 · Correo de bienvenida ──────────────────────────────────────────

## 8.3 Templates de bienvenida

Crear la carpeta y los dos archivos:

```
store/
  templates/
    emails/
      welcome.html
      welcome.txt
```

### `store/templates/emails/welcome.html`

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: Arial, sans-serif; background: #f4f4f4; margin: 0; padding: 20px; }
    .container { background: #fff; max-width: 600px; margin: 0 auto; padding: 30px; border-radius: 8px; }
    h1 { color: #333; }
    .footer { margin-top: 30px; font-size: 12px; color: #999; }
  </style>
</head>
<body>
  <div class="container">
    <h1>¡Bienvenido a ShopAPI, {{ username }}!</h1>
    <p>Tu cuenta ha sido creada exitosamente con el correo <strong>{{ email }}</strong>.</p>
    <p>Ya puedes explorar nuestros productos y realizar pedidos.</p>
    <div class="footer">
      <p>Este es un correo automático, por favor no respondas.</p>
    </div>
  </div>
</body>
</html>
```

### `store/templates/emails/welcome.txt`

```
¡Bienvenido a ShopAPI, {{ username }}!

Tu cuenta ha sido creada exitosamente con el correo {{ email }}.

Ya puedes explorar nuestros productos y realizar pedidos.

---
Este es un correo automático, por favor no respondas.
```

---

## 8.4 Servicio de email — nuevo archivo `store/services/email.py`

Crear la carpeta y el módulo:

```bash
mkdir store/services
touch store/services/__init__.py
```

```python
# store/services/email.py
from django.core.mail import EmailMultiAlternatives
from django.template.loader import render_to_string
from django.conf import settings


def _send(subject: str, to: str, txt_template: str, html_template: str, context: dict) -> None:
    """
    Helper privado: renderiza ambas versiones del correo (texto plano + HTML)
    y lo envía con EmailMultiAlternatives.

    EmailMultiAlternatives envía texto plano como cuerpo principal y adjunta
    la versión HTML como alternativa. Los clientes que soportan HTML muestran
    la versión HTML; el resto usa texto plano.
    """
    text_body = render_to_string(txt_template, context)
    html_body = render_to_string(html_template, context)

    msg = EmailMultiAlternatives(
        subject=subject,
        body=text_body,
        from_email=settings.DEFAULT_FROM_EMAIL,
        to=[to],
    )
    msg.attach_alternative(html_body, 'text/html')
    msg.send(fail_silently=False)


def send_welcome_email(user) -> None:
    """Envía correo de bienvenida cuando se registra un nuevo usuario."""
    _send(
        subject='¡Bienvenido a ShopAPI!',
        to=user.email,
        txt_template='emails/welcome.txt',
        html_template='emails/welcome.html',
        context={
            'username': user.username,
            'email':    user.email,
        },
    )
```

> **Diseño:** `_send()` es el único lugar donde se usa `EmailMultiAlternatives`. Si mañana se cambia al SDK de SendGrid o Mailgun, solo se modifica esta función, sin tocar las funciones de negocio.

---

## 8.5 Señal de bienvenida — actualizar `store/signals.py`

El import del servicio se hace **dentro de la función** para evitar importaciones circulares: `signals.py` se carga desde `apps.py` muy temprano, antes de que todos los modelos estén listos.

```python
# store/signals.py  (reemplazar el contenido completo)
import logging
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User
from store.models.profile import UserProfile

logger = logging.getLogger(__name__)


@receiver(post_save, sender=User)
def user_post_save(sender, instance, created, **kwargs):
    if created:
        UserProfile.objects.create(user=instance)
        _send_welcome(instance)


def _send_welcome(user):
    """Envía correo de bienvenida. Falla en silencio para no bloquear el registro."""
    if not user.email:
        return
    try:
        from store.services.email import send_welcome_email
        send_welcome_email(user)
    except Exception:
        logger.exception('Error enviando correo de bienvenida a %s', user.email)
```

> **¿Por qué no propagar el error?** La señal se ejecuta dentro de la transacción de la vista. Si el SMTP está caído, no queremos revertir el registro del usuario. El error queda en el log del servidor.

---

## 🚦 PARADA 1 — Correo de bienvenida

> **Qué está listo:** configuración SMTP/Gmail, template `welcome.html`/`welcome.txt`, servicio `send_welcome_email`, señal que se dispara al crear un usuario.
> **Qué vas a probar:** que al registrar un usuario nuevo llega el correo de bienvenida.

**Registrar un usuario nuevo**

```
POST /api/auth/register/
Content-Type: application/json

{
  "username": "carlos",
  "email": "tu_correo@gmail.com",
  "password": "test1234!!",
  "password2": "test1234!!"
}
```

Respuesta esperada: `201 Created`.

Abrir Gmail → debe aparecer un correo con asunto **¡Bienvenido a ShopAPI!** con el nombre de usuario en el cuerpo.

> **Si llega el `201` pero no el correo:** revisar la terminal del servidor. Si hay un error en el traceback, el correo falló silenciosamente. Los errores más comunes:
> - `TemplateDoesNotExist: emails/welcome.html` → la carpeta debe ser `store/templates/emails/` y `APP_DIRS: True` en settings
> - `SMTPAuthenticationError` → revisar el App Password en `.env`
>
> **Para probar sin gastar cuota de Gmail**, cambiar en `.env`:
> ```ini
> EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
> ```
> El correo aparecerá impreso en la terminal del servidor.

---

## ── Parte 2 · Recuperación de contraseña ────────────────────────────────────

## 8.6 Templates de reset

Agregar dos archivos nuevos a `store/templates/emails/`:

### `store/templates/emails/password_reset.html`

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: Arial, sans-serif; background: #f4f4f4; margin: 0; padding: 20px; }
    .container { background: #fff; max-width: 600px; margin: 0 auto; padding: 30px; border-radius: 8px; }
    h1 { color: #333; }
    .btn { display: inline-block; padding: 12px 24px; background: #007bff; color: #fff;
           text-decoration: none; border-radius: 4px; margin: 20px 0; }
    .warning { color: #666; font-size: 13px; }
    .footer { margin-top: 30px; font-size: 12px; color: #999; }
  </style>
</head>
<body>
  <div class="container">
    <h1>Recuperación de contraseña</h1>
    <p>Hola <strong>{{ username }}</strong>, recibimos una solicitud para restablecer tu contraseña.</p>
    <p>Haz clic en el siguiente botón para continuar:</p>
    <a class="btn" href="{{ reset_url }}">Restablecer contraseña</a>
    <p class="warning">
      Este enlace es válido por <strong>24 horas</strong>.
      Si no solicitaste este cambio, ignora este correo.
    </p>
    <div class="footer">
      <p>Este es un correo automático, por favor no respondas.</p>
    </div>
  </div>
</body>
</html>
```

### `store/templates/emails/password_reset.txt`

```
Recuperación de contraseña

Hola {{ username }}, recibimos una solicitud para restablecer tu contraseña.

Visita el siguiente enlace para continuar:
{{ reset_url }}

Este enlace es válido por 24 horas.
Si no solicitaste este cambio, ignora este correo.

---
Este es un correo automático.
```

---

## 8.7 Agregar `send_password_reset_email` al servicio

Agregar al final de `store/services/email.py`:

```python
# store/services/email.py  (agregar al final)

def send_password_reset_email(user, uid: str, token: str) -> None:
    """
    Envía correo con enlace de recuperación de contraseña.
    El enlace apunta al frontend, que luego llamará al endpoint de confirmación de la API.
    """
    reset_url = f"{settings.FRONTEND_URL}/password-reset/confirm/?uid={uid}&token={token}"

    _send(
        subject='Recuperación de contraseña — ShopAPI',
        to=user.email,
        txt_template='emails/password_reset.txt',
        html_template='emails/password_reset.html',
        context={
            'username':  user.username,
            'reset_url': reset_url,
        },
    )
```

---

## 8.8 Serializers para recuperación de contraseña — actualizar `store/serializers/user.py`

Agregar al final del archivo:

```python
# store/serializers/user.py  (agregar al final)
from django.utils.encoding import force_bytes, force_str
from django.utils.http import urlsafe_base64_encode, urlsafe_base64_decode
from django.contrib.auth.tokens import default_token_generator


class PasswordResetRequestSerializer(serializers.Serializer):
    """Paso 1: el usuario envía su email para solicitar el reset."""
    email = serializers.EmailField()

    # No validamos si el email existe en esta capa:
    # la vista lo maneja en silencio para evitar enumeración de usuarios.


class PasswordResetConfirmSerializer(serializers.Serializer):
    """Paso 2: el usuario envía uid + token + nueva contraseña."""
    uid           = serializers.CharField()
    token         = serializers.CharField()
    new_password  = serializers.CharField(min_length=8, write_only=True)
    new_password2 = serializers.CharField(write_only=True)

    def validate(self, data):
        # Decodificar el uid para obtener el usuario
        try:
            pk   = force_str(urlsafe_base64_decode(data['uid']))
            user = User.objects.get(pk=pk)
        except (TypeError, ValueError, OverflowError, User.DoesNotExist):
            raise serializers.ValidationError({'uid': 'Enlace inválido o expirado.'})

        # Verificar el token con el generador nativo de Django
        if not default_token_generator.check_token(user, data['token']):
            raise serializers.ValidationError({'token': 'Token inválido o expirado.'})

        if data['new_password'] != data['new_password2']:
            raise serializers.ValidationError({'new_password2': 'Las contraseñas no coinciden.'})

        data['user'] = user
        return data

    def save(self):
        user = self.validated_data['user']
        user.set_password(self.validated_data['new_password'])
        user.save()
        return user
```

> **¿Cómo funciona `default_token_generator`?**
> Django genera el token combinando el `pk` del usuario, su `last_login` y el hash de su contraseña actual. Esto garantiza que:
> 1. El token **expira automáticamente** al cambiar la contraseña (el hash cambia).
> 2. El token **expira por tiempo** según `PASSWORD_RESET_TIMEOUT` (24 h, configurado en 8.1).
> 3. No se necesita tabla adicional en la base de datos.

---

## 8.9 Vistas para recuperación de contraseña — actualizar `store/views/auth.py`

Agregar al final del archivo:

```python
# store/views/auth.py  (agregar al final)
import logging
from django.contrib.auth.models import User
from django.utils.encoding import force_bytes
from django.utils.http import urlsafe_base64_encode
from django.contrib.auth.tokens import default_token_generator
from store.serializers.user import PasswordResetRequestSerializer, PasswordResetConfirmSerializer
from store.services.email import send_password_reset_email

logger = logging.getLogger(__name__)


class PasswordResetRequestView(APIView):
    """
    POST /api/auth/password-reset/
    Body: { "email": "user@example.com" }

    Genera un token y envía el correo. Siempre responde 200
    independientemente de si el email existe (anti-enumeración de usuarios).
    """
    permission_classes = []

    def post(self, request):
        serializer = PasswordResetRequestSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        email = serializer.validated_data['email']
        try:
            user  = User.objects.get(email=email, is_active=True)
            uid   = urlsafe_base64_encode(force_bytes(user.pk))
            token = default_token_generator.make_token(user)
            try:
                send_password_reset_email(user, uid, token)
            except Exception:
                # El SMTP puede fallar (credenciales, red, timeout).
                # Registramos el error pero NO propagamos la excepción:
                # el endpoint siempre responde 200 para no revelar si el
                # email está registrado (anti-enumeración).
                logger.exception('Error enviando email de reset a %s', email)
        except User.DoesNotExist:
            pass  # No revelar si el email está registrado

        return Response(
            {'detail': 'Si el correo está registrado, recibirás un enlace de recuperación.'},
            status=status.HTTP_200_OK,
        )


class PasswordResetConfirmView(APIView):
    """
    POST /api/auth/password-reset/confirm/
    Body: { "uid": "...", "token": "...", "new_password": "...", "new_password2": "..." }

    Valida el token y actualiza la contraseña. Una vez usado, el token queda inválido.
    """
    permission_classes = []

    def post(self, request):
        serializer = PasswordResetConfirmSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(
            {'detail': 'Contraseña actualizada correctamente.'},
            status=status.HTTP_200_OK,
        )
```

> **¿Por qué dos try/except separados?**
> El `except User.DoesNotExist` protege contra enumeración de usuarios. El `except Exception` interno protege contra fallos de SMTP. Si se unificaran, un error SMTP también caería en el `pass`, silenciando el problema sin loguearlo.
>
> **¿Por qué `EMAIL_TIMEOUT` es imprescindible?**
> Sin él, si el servidor SMTP no responde, Django bloquea al worker de gunicorn hasta que gunicorn agota su propio timeout y lanza `SystemExit`. `SystemExit` hereda de `BaseException`, **no de `Exception`**, por lo que `except Exception` **no lo captura** y el request termina en 500. Con `EMAIL_TIMEOUT=5`, Django lanza `SMTPConnectError` (subclase de `Exception`) antes de que gunicorn intervenga, y el error queda correctamente manejado.

---

## 8.10 Actualizar `store/urls.py`

```python
# store/urls.py  (agregar las nuevas rutas)
from store.views.auth import (
    RegisterView, LoginView, LogoutView, RefreshTokenView,
    PasswordResetRequestView, PasswordResetConfirmView,
)

urlpatterns = [
    path('auth/register/',               RegisterView.as_view(),             name='auth-register'),
    path('auth/login/',                  LoginView.as_view(),                name='auth-login'),
    path('auth/logout/',                 LogoutView.as_view(),               name='auth-logout'),
    path('auth/token/refresh/',          RefreshTokenView.as_view(),         name='auth-token-refresh'),
    path('auth/password-reset/',         PasswordResetRequestView.as_view(), name='auth-password-reset'),       # ← nuevo
    path('auth/password-reset/confirm/', PasswordResetConfirmView.as_view(), name='auth-password-reset-confirm'), # ← nuevo
    # ... resto de rutas sin cambios ...
]
```

---

## 🚦 PARADA 2 — Recuperación de contraseña

> **Qué está listo:** templates de reset, función `send_password_reset_email`, serializers, vistas y URLs.
> **Qué vas a probar:** el flujo completo de 4 pasos.

**Paso A — Solicitar el reset**

```
POST /api/auth/password-reset/
Content-Type: application/json

{ "email": "tu_correo@gmail.com" }
```

Respuesta: `200 OK` con `"detail": "Si el correo está registrado..."`.
Abrir Gmail → debe llegar el correo con un botón **Restablecer contraseña**.

---

**Paso B — Copiar uid y token del enlace**

El enlace del botón tiene esta forma:
```
http://localhost:3000/password-reset/confirm/?uid=MQ&token=abc-defg-hij
```

Copiar los valores de `uid` y `token`.

> Con `EMAIL_BACKEND=console.EmailBackend` el enlace aparece en la terminal del servidor. Buscar la línea que contiene `password-reset/confirm/?uid=`.

---

**Paso C — Confirmar el reset**

```
POST /api/auth/password-reset/confirm/
Content-Type: application/json

{
  "uid": "MQ",
  "token": "abc-defg-hij",
  "new_password": "nuevapass123",
  "new_password2": "nuevapass123"
}
```

Respuesta esperada: `200 OK` con `"detail": "Contraseña actualizada correctamente."`.

---

**Paso D — Login con la nueva contraseña**

```
POST /api/auth/login/
Content-Type: application/json

{
  "username": "carlos",
  "password": "nuevapass123"
}
```

Respuesta esperada: `200 OK` con nuevos tokens JWT.

---

**Casos de borde:**

| Acción | Esperado |
|--------|----------|
| Usar el mismo token dos veces | `400` — `token: inválido o expirado` |
| Enviar `uid` inventado | `400` — `uid: Enlace inválido o expirado` |
| Solicitar reset para email inexistente | `200` — misma respuesta (anti-enumeración) |

---

## ── Parte 3 · Confirmación de orden ─────────────────────────────────────────

## 8.11 Templates de confirmación de orden

Agregar dos archivos nuevos a `store/templates/emails/`:

### `store/templates/emails/order_confirmation.html`

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: Arial, sans-serif; background: #f4f4f4; margin: 0; padding: 20px; }
    .container { background: #fff; max-width: 600px; margin: 0 auto; padding: 30px; border-radius: 8px; }
    h1 { color: #333; }
    table { width: 100%; border-collapse: collapse; margin: 20px 0; }
    th, td { text-align: left; padding: 8px 12px; border-bottom: 1px solid #eee; }
    th { background: #f8f8f8; }
    .total-row td { font-weight: bold; font-size: 16px; }
    .footer { margin-top: 30px; font-size: 12px; color: #999; }
  </style>
</head>
<body>
  <div class="container">
    <h1>Confirmación de pedido #{{ order_id }}</h1>
    <p>Hola <strong>{{ username }}</strong>, hemos recibido tu pedido correctamente.</p>
    <table>
      <thead>
        <tr>
          <th>Producto</th><th>Cantidad</th><th>Precio unit.</th><th>Subtotal</th>
        </tr>
      </thead>
      <tbody>
        {% for item in items %}
        <tr>
          <td>{{ item.product_name }}</td>
          <td>{{ item.quantity }}</td>
          <td>${{ item.unit_price }}</td>
          <td>${{ item.subtotal }}</td>
        </tr>
        {% endfor %}
      </tbody>
      <tfoot>
        <tr class="total-row">
          <td colspan="3">Total</td>
          <td>${{ total }}</td>
        </tr>
      </tfoot>
    </table>
    <p>Estado: <strong>{{ status }}</strong></p>
    <div class="footer">
      <p>Pedido creado el {{ created_at }}. Este es un correo automático.</p>
    </div>
  </div>
</body>
</html>
```

### `store/templates/emails/order_confirmation.txt`

```
Confirmación de pedido #{{ order_id }}

Hola {{ username }}, hemos recibido tu pedido correctamente.

Productos:
{% for item in items %}- {{ item.product_name }}: {{ item.quantity }} x ${{ item.unit_price }} = ${{ item.subtotal }}
{% endfor %}
Total: ${{ total }}
Estado: {{ status }}

Pedido creado el {{ created_at }}.
---
Este es un correo automático.
```

---

## 8.12 Agregar `send_order_confirmation_email` al servicio

Agregar al final de `store/services/email.py`:

```python
# store/services/email.py  (agregar al final)

def send_order_confirmation_email(order) -> None:
    """
    Envía correo de confirmación cuando se crea una nueva orden.
    Incluye el detalle de ítems y el total.
    """
    items = [
        {
            'product_name': item.product.name,
            'quantity':     item.quantity,
            'unit_price':   item.unit_price,
            'subtotal':     round(item.quantity * float(item.unit_price), 2),
        }
        for item in order.items.select_related('product').all()
    ]

    _send(
        subject=f'Confirmación de pedido #{order.id} — ShopAPI',
        to=order.user.email,
        txt_template='emails/order_confirmation.txt',
        html_template='emails/order_confirmation.html',
        context={
            'username':   order.user.username,
            'order_id':   order.id,
            'items':      items,
            'total':      order.total,
            'status':     order.status,
            'created_at': order.created_at.strftime('%d/%m/%Y %H:%M'),
        },
    )
```

---

## 8.13 Señal de confirmación de orden — completar `store/signals.py`

Agregar al final del archivo:

```python
# store/signals.py  (agregar al final)
from store.models.order import Order


@receiver(post_save, sender=Order)
def order_post_save(sender, instance, created, **kwargs):
    if created:
        _send_order_confirmation(instance)


def _send_order_confirmation(order):
    """
    Envía correo de confirmación. Falla en silencio para no bloquear la transacción.

    Nota: la señal se dispara cuando la orden se crea. Si los ítems se agregan
    en una segunda request (add-item), el correo saldrá con la lista vacía.
    Para correos con ítems completos, considera disparar el envío desde la
    vista de 'confirmar pedido' en una etapa futura.
    """
    if not order.user.email:
        return
    try:
        from store.services.email import send_order_confirmation_email
        send_order_confirmation_email(order)
    except Exception:
        logger.exception('Error enviando confirmación de orden #%s', order.id)
```

---

## 🚦 PARADA 3 — Confirmación de orden

> **Qué está listo:** templates de orden, función `send_order_confirmation_email`, señal que se dispara al crear una orden.
> **Qué vas a probar:** que al crear una orden llega el correo de confirmación.

**Crear una orden**

```
POST /api/orders/
Authorization: Bearer <access_token>
Content-Type: application/json

{}
```

Respuesta esperada: `201 Created` con el objeto de la orden.

Abrir Gmail → debe llegar el correo con asunto **Confirmación de pedido #N — ShopAPI**.

> **Nota sobre los ítems:** si los productos se agregan en un paso posterior (`add-item`), el correo llegará con la tabla de productos vacía. Esto es esperado en esta etapa y se abordará en una etapa futura.
>
> **Para probar con ítems en el correo**, agregar los ítems dentro de la misma request de creación (si tu implementación lo soporta), o disparar el correo manualmente desde la vista de confirmación.

---

## ── Parte 4 · Notificaciones manuales ───────────────────────────────────────

Un endpoint exclusivo para staff que permite enviar un correo personalizado a un usuario específico o, si no se indica ninguno, enviarlo masivamente a todos los usuarios activos que no son staff.

Casos de uso: anunciar promociones, avisos de mantenimiento, newsletters, descuentos por temporada.

---

## 8.14 Template de notificación

Crear en `store/templates/emails/`:

### `store/templates/emails/notification.html`

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: Arial, sans-serif; background: #f4f4f4; margin: 0; padding: 20px; }
    .container { background: #fff; max-width: 600px; margin: 0 auto; padding: 30px; border-radius: 8px; }
    h1 { color: #333; font-size: 22px; }
    .message { font-size: 15px; line-height: 1.6; color: #444; white-space: pre-line; }
    .footer { margin-top: 30px; font-size: 12px; color: #999; border-top: 1px solid #eee; padding-top: 15px; }
  </style>
</head>
<body>
  <div class="container">
    <p>Hola <strong>{{ username }}</strong>,</p>
    <h1>{{ subject }}</h1>
    <div class="message">{{ message }}</div>
    <div class="footer">
      <p>Este mensaje fue enviado por el equipo de ShopAPI. No respondas a este correo.</p>
    </div>
  </div>
</body>
</html>
```

### `store/templates/emails/notification.txt`

```
Hola {{ username }},

{{ subject }}

{{ message }}

---
Este mensaje fue enviado por el equipo de ShopAPI.
```

---

## 8.15 Agregar `send_notification_email` al servicio

Agregar al final de `store/services/email.py`:

```python
# store/services/email.py  (agregar al final)

def send_notification_email(user, subject: str, message: str) -> None:
    """
    Envía un correo de notificación personalizada a un usuario.
    Usada tanto para envíos individuales como masivos.
    """
    _send(
        subject=subject,
        to=user.email,
        txt_template='emails/notification.txt',
        html_template='emails/notification.html',
        context={
            'username': user.username,
            'subject':  subject,
            'message':  message,
        },
    )
```

---

## 8.16 Serializer de notificación — actualizar `store/serializers/user.py`

Agregar al final del archivo:

```python
# store/serializers/user.py  (agregar al final)

class SendNotificationSerializer(serializers.Serializer):
    """
    Cuerpo del request para enviar una notificación manual.

    - Si se provee `user_id`, el correo va solo a ese usuario.
    - Si `user_id` es null u omitido, el correo se envía a todos los
      usuarios activos que no son staff (envío masivo).
    """
    subject = serializers.CharField(max_length=200)
    message = serializers.CharField()
    user_id = serializers.IntegerField(required=False, allow_null=True)

    def validate_user_id(self, value):
        if value is not None:
            if not User.objects.filter(pk=value, is_active=True, is_staff=False).exists():
                raise serializers.ValidationError(
                    'Usuario no encontrado, inactivo o es staff.'
                )
        return value
```

---

## 8.17 Vista de notificación — nuevo archivo o actualizar `store/views/email.py`

Crear el archivo (o añadir a las vistas existentes):

```python
# store/views/email.py
import logging
from django.contrib.auth.models import User
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAdminUser
from store.serializers.user import SendNotificationSerializer
from store.services.email import send_notification_email

logger = logging.getLogger(__name__)


class SendNotificationView(APIView):
    """
    POST /api/emails/send/
    Solo accesible para usuarios staff (is_staff=True).

    Envío individual:
        { "subject": "...", "message": "...", "user_id": 5 }

    Envío masivo (sin user_id o user_id: null):
        { "subject": "...", "message": "..." }
        → Envía a todos los usuarios activos que no son staff y tienen email.

    Responde con el número de correos enviados y fallidos.
    """
    permission_classes = [IsAdminUser]

    def post(self, request):
        serializer = SendNotificationSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        subject = serializer.validated_data['subject']
        message = serializer.validated_data['message']
        user_id = serializer.validated_data.get('user_id')

        if user_id:
            recipients = User.objects.filter(pk=user_id)
        else:
            # Masivo: todos los usuarios activos no-staff que tienen email
            recipients = User.objects.filter(
                is_staff=False,
                is_active=True,
            ).exclude(email='')

        sent   = 0
        failed = 0

        for user in recipients:
            try:
                send_notification_email(user, subject, message)
                sent += 1
            except Exception:
                failed += 1
                logger.exception('Error enviando notificación a %s', user.email)

        return Response(
            {
                'detail': f'Correo enviado a {sent} usuario(s).',
                'sent':   sent,
                'failed': failed,
            },
            status=status.HTTP_200_OK,
        )
```

> **`IsAdminUser`** en DRF verifica que `request.user.is_staff == True`. No requiere `is_superuser`. Cualquier usuario marcado como staff en el admin puede usar este endpoint.

> **Envío masivo y rendimiento:** el bucle `for user in recipients` llama al servidor SMTP una vez por usuario. Con decenas de usuarios no hay problema. Para cientos o miles, lo correcto es delegar el envío a una cola de tareas asíncrona (Celery + Redis), que se abordará en una etapa futura.

---

## 8.18 Actualizar `store/urls.py`

```python
# store/urls.py  (agregar)
from store.views.email import SendNotificationView

urlpatterns = [
    # ... rutas existentes ...
    path('emails/send/', SendNotificationView.as_view(), name='email-send-notification'),
]
```

---

## 🚦 PARADA 4 — Notificaciones manuales

> **Qué está listo:** template de notificación, función `send_notification_email`, serializer, vista protegida con `IsAdminUser`, ruta registrada.
> **Qué vas a probar:** envío individual a un usuario, envío masivo, y que usuarios sin staff no pueden acceder.

**1 — Envío individual (solo un usuario)**

```
POST /api/emails/send/
Authorization: Bearer <access_token_staff>
Content-Type: application/json

{
  "subject": "¡Promoción exclusiva para ti!",
  "message": "Hola,\n\nEste mes tienes un 20% de descuento en todos los productos.\n\nUsa el código PROMO20 al hacer tu pedido.\n\n¡Aprovéchalo antes del 31!",
  "user_id": 2
}
```

Respuesta esperada:
```json
{ "detail": "Correo enviado a 1 usuario(s).", "sent": 1, "failed": 0 }
```

Abrir Gmail → llega el correo con el saludo personalizado (`Hola <username>`) y el mensaje formateado.

---

**2 — Envío masivo (todos los usuarios no-staff)**

```
POST /api/emails/send/
Authorization: Bearer <access_token_staff>
Content-Type: application/json

{
  "subject": "Novedades de ShopAPI — Junio 2025",
  "message": "Estimado cliente,\n\nTenemos nuevos productos disponibles en nuestra tienda.\n\nVisita nuestro catálogo y descubre las últimas incorporaciones.\n\n¡Gracias por tu preferencia!"
}
```

Respuesta esperada:
```json
{ "detail": "Correo enviado a 3 usuario(s).", "sent": 3, "failed": 0 }
```

Cada usuario registrado recibe el correo con su propio nombre en el saludo.

---

**3 — Intentar sin token staff (esperar 403)**

```
POST /api/emails/send/
Authorization: Bearer <access_token_usuario_normal>
Content-Type: application/json

{ "subject": "test", "message": "test" }
```

Respuesta esperada: `403 Forbidden`.

---

**Casos de borde:**

| Caso | Esperado |
|------|----------|
| `user_id` de un usuario staff | `400` — "es staff" no es destinatario válido |
| `user_id` de usuario inactivo | `400` — "inactivo" no es destinatario válido |
| Envío masivo con 0 usuarios no-staff activos | `200` — `{"sent": 0, "failed": 0}` |
| Un usuario sin email en envío masivo | Se omite automáticamente (filtro `exclude(email='')`) |

---

## ✅ Checkpoint Etapa 8

| Parada | Qué se validó |
|--------|--------------|
| 🚦 1 | Registrar usuario → correo de bienvenida en Gmail con nombre y email |
| 🚦 2A | Solicitar reset → correo con enlace en Gmail |
| 🚦 2B | Confirmar reset con token → `200 OK` |
| 🚦 2C | Login con nueva contraseña → nuevos tokens JWT |
| 🚦 2D | Token ya usado → `400` inválido |
| 🚦 3 | Crear orden → correo de confirmación en Gmail |
| 🚦 4A | Envío individual a usuario → `sent: 1`, correo personalizado llega |
| 🚦 4B | Envío masivo sin `user_id` → `sent: N`, cada usuario recibe su correo |
| 🚦 4C | Sin token staff → `403 Forbidden` |

**Casos de borde adicionales a verificar:**

| Caso | Esperado |
|------|----------|
| Registrar usuario sin email | `201` — correo no se envía, señal no falla |
| SMTP caído durante registro | `201` — error en log del servidor, usuario creado igual |
| Reset para email inexistente | `200` (anti-enumeración) |
| Un fallo de SMTP en envío masivo | El resto continúa — la respuesta incluye `"failed": N` |

---

## 📦 Colección Postman — Etapa 8

```json
{
  "info": {
    "name": "ShopAPI — Stage 8: Email Service",
    "_postman_id": "shopapi-stage-08",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "── PARADA 1: Bienvenida ──",
      "item": [
        {
          "name": "Register (triggers welcome email)",
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"username\": \"carlos\",\n  \"email\": \"tu_correo@gmail.com\",\n  \"password\": \"test1234!!\",\n  \"password2\": \"test1234!!\"\n}"
            },
            "url": "{{base_url}}/auth/register/"
          }
        }
      ]
    },
    {
      "name": "── PARADA 2: Reset de contraseña ──",
      "item": [
        {
          "name": "Request password reset",
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"email\": \"tu_correo@gmail.com\"\n}"
            },
            "url": "{{base_url}}/auth/password-reset/"
          }
        },
        {
          "name": "Confirm password reset",
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"uid\": \"{{reset_uid}}\",\n  \"token\": \"{{reset_token}}\",\n  \"new_password\": \"nuevapass123\",\n  \"new_password2\": \"nuevapass123\"\n}"
            },
            "url": "{{base_url}}/auth/password-reset/confirm/"
          }
        },
        {
          "name": "Confirm reset with invalid token (expect 400)",
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"uid\": \"{{reset_uid}}\",\n  \"token\": \"invalid-token-000\",\n  \"new_password\": \"nuevapass123\",\n  \"new_password2\": \"nuevapass123\"\n}"
            },
            "url": "{{base_url}}/auth/password-reset/confirm/"
          }
        },
        {
          "name": "Login with new password",
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"username\": \"carlos\",\n  \"password\": \"nuevapass123\"\n}"
            },
            "url": "{{base_url}}/auth/login/"
          }
        },
        {
          "name": "Request reset for non-existent email (expect 200)",
          "request": {
            "method": "POST",
            "header": [{ "key": "Content-Type", "value": "application/json" }],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"email\": \"noexiste@ejemplo.com\"\n}"
            },
            "url": "{{base_url}}/auth/password-reset/"
          }
        }
      ]
    },
    {
      "name": "── PARADA 3: Confirmación de orden ──",
      "item": [
        {
          "name": "Create order (triggers confirmation email)",
          "request": {
            "method": "POST",
            "header": [
              { "key": "Authorization", "value": "Bearer {{access}}" },
              { "key": "Content-Type",  "value": "application/json" }
            ],
            "body": { "mode": "raw", "raw": "{}" },
            "url": "{{base_url}}/orders/"
          }
        }
      ]
    },
    {
      "name": "── PARADA 4: Notificaciones manuales ──",
      "item": [
        {
          "name": "Send to single user",
          "request": {
            "method": "POST",
            "header": [
              { "key": "Authorization", "value": "Bearer {{access_staff}}" },
              { "key": "Content-Type",  "value": "application/json" }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"subject\": \"¡Promoción exclusiva para ti!\",\n  \"message\": \"Este mes tienes un 20% de descuento.\\nUsa el código PROMO20 al hacer tu pedido.\",\n  \"user_id\": {{target_user_id}}\n}"
            },
            "url": "{{base_url}}/emails/send/"
          }
        },
        {
          "name": "Broadcast to all non-staff users",
          "request": {
            "method": "POST",
            "header": [
              { "key": "Authorization", "value": "Bearer {{access_staff}}" },
              { "key": "Content-Type",  "value": "application/json" }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"subject\": \"Novedades de ShopAPI\",\n  \"message\": \"Tenemos nuevos productos disponibles.\\n\\nVisita nuestro catálogo y descubre las últimas incorporaciones.\"\n}"
            },
            "url": "{{base_url}}/emails/send/"
          }
        },
        {
          "name": "Send without staff token (expect 403)",
          "request": {
            "method": "POST",
            "header": [
              { "key": "Authorization", "value": "Bearer {{access}}" },
              { "key": "Content-Type",  "value": "application/json" }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"subject\": \"test\",\n  \"message\": \"test\"\n}"
            },
            "url": "{{base_url}}/emails/send/"
          }
        },
        {
          "name": "Send to staff user_id (expect 400)",
          "request": {
            "method": "POST",
            "header": [
              { "key": "Authorization", "value": "Bearer {{access_staff}}" },
              { "key": "Content-Type",  "value": "application/json" }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"subject\": \"test\",\n  \"message\": \"test\",\n  \"user_id\": 1\n}"
            },
            "url": "{{base_url}}/emails/send/"
          }
        }
      ]
    }
  ],
  "variable": [
    { "key": "base_url",       "value": "http://localhost:8000/api" },
    { "key": "access",         "value": "" },
    { "key": "access_staff",   "value": "" },
    { "key": "reset_uid",      "value": "" },
    { "key": "reset_token",    "value": "" },
    { "key": "target_user_id", "value": "2" }
  ]
}
```

---

## Resumen

| Elemento | Parada |
|----------|--------|
| `EMAIL_BACKEND` configurable por entorno | base |
| `DEFAULT_FROM_EMAIL`, `FRONTEND_URL`, `PASSWORD_RESET_TIMEOUT` en settings | base |
| Cuenta Google con App Password configurada | base |
| Template `welcome.html` / `welcome.txt` | 🚦 1 |
| `send_welcome_email()` en el servicio | 🚦 1 |
| Señal `post_save` en `User` → correo de bienvenida | 🚦 1 |
| Template `password_reset.html` / `password_reset.txt` | 🚦 2 |
| `send_password_reset_email()` en el servicio | 🚦 2 |
| `PasswordResetRequestView` — anti-enumeración (siempre `200`) | 🚦 2 |
| Token `default_token_generator` — expira con cambio de contraseña | 🚦 2 |
| `PasswordResetConfirmView` — token queda inválido tras el primer uso | 🚦 2 |
| Template `order_confirmation.html` / `order_confirmation.txt` | 🚦 3 |
| `send_order_confirmation_email()` en el servicio | 🚦 3 |
| Señal `post_save` en `Order` → correo de confirmación | 🚦 3 |
| Template `notification.html` / `notification.txt` | 🚦 4 |
| `send_notification_email()` en el servicio | 🚦 4 |
| `SendNotificationView` — protegida con `IsAdminUser` | 🚦 4 |
| Envío individual (`user_id`) y masivo (sin `user_id`) | 🚦 4 |
| Fallos de SMTP aislados por usuario en envío masivo | 🚦 4 |
| Errores de envío logueados sin bloquear la transacción | todas |

**Siguiente etapa →** Tests del servicio de email con `mail.outbox` + mocking de señales
