# Curso Kotlin — Página 20
## Módulo 7 · Hardware y Sistema
### Notificaciones locales y push con Firebase Cloud Messaging

---

## 🚀 TechDash — Módulo de Notificaciones

> **Mismo proyecto TechDash** — continúa en `com.ute.techdash`.
> Esta página añade el sistema de notificaciones: **notificaciones locales**
> (canales, acciones, barra de progreso) y **push con Firebase Cloud Messaging**
> (token, topics, deep links). Al terminar, el dashboard podrá enviar alertas
> al usuario aunque la app esté en segundo plano.

### Árbol de archivos de esta página

```
app/src/main/java/com/example/techdash/
├── notificaciones/
│   ├── CanalNotificacion.kt                  ← 📄 NUEVO (constantes de canales)
│   ├── NotificacionesHelper.kt               ← 📄 NUEVO (helper para enviar notifs)
│   ├── AccionesNotificacionReceiver.kt        ← 📄 NUEVO (BroadcastReceiver)
│   ├── MiFirebaseMessagingService.kt          ← 📄 NUEVO (FCM)
│   └── TokenFcmRepository.kt                 ← 📄 NUEVO (token + topics)
└── ui/
    └── notificaciones/
        ├── PantallaNotificacionesLocales.kt   ← 📄 NUEVO
        └── PantallaConfigFCM.kt               ← 📄 NUEVO
```

### Archivos que crearás/modificarás en esta página

| Archivo | Acción | Qué hace |
|---------|--------|----------|
| `notificaciones/CanalNotificacion.kt` | 📄 Nuevo | Constantes de canales + función de creación |
| `notificaciones/NotificacionesHelper.kt` | 📄 Nuevo | Helper con 5 tipos de notificación |
| `notificaciones/AccionesNotificacionReceiver.kt` | 📄 Nuevo | Responde a botones sin abrir la app |
| `ui/notificaciones/PantallaNotificacionesLocales.kt` | 📄 Nuevo | Demo de los 4 tipos de notificación |
| `notificaciones/MiFirebaseMessagingService.kt` | 📄 Nuevo | Recibe mensajes FCM |
| `notificaciones/TokenFcmRepository.kt` | 📄 Nuevo | Obtiene token y gestiona topics |
| `ui/notificaciones/PantallaConfigFCM.kt` | 📄 Nuevo | Muestra token y permite suscribirse |
| `AndroidManifest.xml` | ✏️ Modifica | Permiso `POST_NOTIFICATIONS` + `<service>` FCM |
| `navigation/NavGraph.kt` | ✏️ Modifica | Añade rutas `notificaciones` y `config_fcm` |

---

## Dos tipos de notificaciones

```
Notificaciones LOCALES                  Notificaciones PUSH (FCM)
──────────────────────────────          ─────────────────────────────────────
Generadas por la propia app             Enviadas desde un servidor remoto
No necesitan internet                   Requieren internet + Firebase
Ejemplos: alarma, descarga,             Ejemplos: mensaje nuevo, oferta,
  recordatorio de tarea                   alerta del sistema
NotificationManager                     FirebaseMessagingService
Android 8+ requiere canales             Requiere google-services.json
```

---

## Parte 1 — Notificaciones locales

### Dependencias y permisos

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
}
```

```xml
<!-- AndroidManifest.xml -->
<!-- Android 13+ requiere permiso explícito para notificaciones -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<application ...>
    <!-- Receptor para acciones de notificación -->
    <receiver
        android:name=".notificaciones.AccionesNotificacionReceiver"
        android:exported="false" />
</application>
```

### Solicitar permiso de notificaciones (Android 13+)

```kotlin
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import android.os.Build

@Composable
fun SolicitarPermisoNotificaciones(onResultado: (Boolean) -> Unit) {
    val context = LocalContext.current

    // En Android < 13 el permiso no existe — se concede automáticamente
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) {
        LaunchedEffect(Unit) { onResultado(true) }
        return
    }

    val tienePermiso = remember {
        ContextCompat.checkSelfPermission(
            context,
            android.Manifest.permission.POST_NOTIFICATIONS
        ) == PackageManager.PERMISSION_GRANTED
    }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { onResultado(it) }

    LaunchedEffect(Unit) {
        if (!tienePermiso) {
            launcher.launch(android.Manifest.permission.POST_NOTIFICATIONS)
        } else {
            onResultado(true)
        }
    }
}
```

---

### Canales de notificación (obligatorio Android 8+)

```kotlin
// 📁 Nuevo archivo → notificaciones/CanalNotificacion.kt
import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import android.os.Build

// Identificadores de canales — constantes en un objeto
object CanalNotificacion {
    const val GENERAL       = "canal_general"
    const val RECORDATORIOS = "canal_recordatorios"
    const val DESCARGAS     = "canal_descargas"
    const val ALERTAS       = "canal_alertas"
}

fun crearCanalesNotificacion(context: Context) {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) return

    val manager = context.getSystemService(
        Context.NOTIFICATION_SERVICE
    ) as NotificationManager

    val canales = listOf(
        NotificationChannel(
            CanalNotificacion.GENERAL,
            "General",
            NotificationManager.IMPORTANCE_DEFAULT
        ).apply {
            description = "Notificaciones generales de la app"
        },

        NotificationChannel(
            CanalNotificacion.RECORDATORIOS,
            "Recordatorios",
            NotificationManager.IMPORTANCE_HIGH   // ← cabecera + sonido
        ).apply {
            description    = "Recordatorios de tareas pendientes"
            enableVibration(true)
        },

        NotificationChannel(
            CanalNotificacion.DESCARGAS,
            "Descargas",
            NotificationManager.IMPORTANCE_LOW    // ← sin sonido
        ).apply {
            description = "Progreso de descargas"
        },

        NotificationChannel(
            CanalNotificacion.ALERTAS,
            "Alertas urgentes",
            NotificationManager.IMPORTANCE_HIGH
        ).apply {
            description    = "Alertas críticas del sistema"
            enableLights(true)
            lightColor     = android.graphics.Color.RED
            enableVibration(true)
        }
    )

    canales.forEach { manager.createNotificationChannel(it) }
}

// Llamar en Application.onCreate() o en MainActivity.onCreate()
class MiApp : android.app.Application() {
    override fun onCreate() {
        super.onCreate()
        crearCanalesNotificacion(this)
    }
}
```

---

### `NotificacionesHelper` — enviar notificaciones

```kotlin
// 📁 Nuevo archivo → notificaciones/NotificacionesHelper.kt
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat

object NotificacionesHelper {

    // Notificación simple
    fun simple(
        context:  Context,
        id:       Int,
        titulo:   String,
        mensaje:  String,
        canal:    String = CanalNotificacion.GENERAL
    ) {
        val notif = NotificationCompat.Builder(context, canal)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(titulo)
            .setContentText(mensaje)
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .setAutoCancel(true)   // desaparece al tocarla
            .build()

        NotificationManagerCompat.from(context).notify(id, notif)
    }

    // Notificación con acción y deep link a una pantalla
    fun conAccion(
        context:       Context,
        id:            Int,
        titulo:        String,
        mensaje:       String,
        textoAccion:   String,
        intentAccion:  Intent
    ) {
        val pendingIntent = PendingIntent.getActivity(
            context, id, intentAccion,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        val notif = NotificationCompat.Builder(context, CanalNotificacion.GENERAL)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(titulo)
            .setContentText(mensaje)
            .addAction(R.drawable.ic_check, textoAccion, pendingIntent)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(context).notify(id, notif)
    }

    // Notificación expandible con texto largo
    fun expandible(
        context:   Context,
        id:        Int,
        titulo:    String,
        resumen:   String,
        textoLargo: String
    ) {
        val notif = NotificationCompat.Builder(context, CanalNotificacion.GENERAL)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(titulo)
            .setContentText(resumen)
            .setStyle(
                NotificationCompat.BigTextStyle()
                    .bigText(textoLargo)
                    .setBigContentTitle(titulo)
                    .setSummaryText(resumen)
            )
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(context).notify(id, notif)
    }

    // Notificación con barra de progreso
    fun conProgreso(
        context:    Context,
        id:         Int,
        titulo:     String,
        progreso:   Int,    // 0..100
        completa:   Boolean = false
    ) {
        val notif = NotificationCompat.Builder(context, CanalNotificacion.DESCARGAS)
            .setSmallIcon(R.drawable.ic_download)
            .setContentTitle(titulo)
            .setContentText(if (completa) "Completado" else "$progreso%")
            .setProgress(100, progreso, false)
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .setOngoing(!completa)        // no deslizable hasta completar
            .setAutoCancel(completa)
            .build()

        NotificationManagerCompat.from(context).notify(id, notif)
    }

    // Notificación con botones de acción (sin abrir la app)
    fun conBotones(
        context:  Context,
        id:       Int,
        titulo:   String,
        mensaje:  String,
        tareaId:  Int
    ) {
        val intentCompletar = Intent(context, AccionesNotificacionReceiver::class.java).apply {
            action = "COMPLETAR_TAREA"
            putExtra("tarea_id", tareaId)
            putExtra("notif_id", id)
        }
        val intentEliminar  = Intent(context, AccionesNotificacionReceiver::class.java).apply {
            action = "ELIMINAR_TAREA"
            putExtra("tarea_id", tareaId)
            putExtra("notif_id", id)
        }

        fun pendingBroadcast(intent: Intent, reqCode: Int) =
            PendingIntent.getBroadcast(
                context, reqCode, intent,
                PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
            )

        val notif = NotificationCompat.Builder(context, CanalNotificacion.RECORDATORIOS)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(titulo)
            .setContentText(mensaje)
            .addAction(R.drawable.ic_check,  "Completar",
                pendingBroadcast(intentCompletar, tareaId * 10))
            .addAction(R.drawable.ic_delete, "Eliminar",
                pendingBroadcast(intentEliminar, tareaId * 10 + 1))
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(context).notify(id, notif)
    }

    fun cancelar(context: Context, id: Int) {
        NotificationManagerCompat.from(context).cancel(id)
    }

    fun cancelarTodas(context: Context) {
        NotificationManagerCompat.from(context).cancelAll()
    }
}
```

### `BroadcastReceiver` para acciones

```kotlin
// 📁 Nuevo archivo → notificaciones/AccionesNotificacionReceiver.kt
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent

// Recibe las acciones de los botones de notificación
class AccionesNotificacionReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val tareaId = intent.getIntExtra("tarea_id", -1)
        val notifId = intent.getIntExtra("notif_id", -1)

        when (intent.action) {
            "COMPLETAR_TAREA" -> {
                // Aquí actualizarías la BD con Room
                println("Tarea $tareaId marcada como completada desde notificación")
                NotificacionesHelper.cancelar(context, notifId)
            }
            "ELIMINAR_TAREA"  -> {
                println("Tarea $tareaId eliminada desde notificación")
                NotificacionesHelper.cancelar(context, notifId)
            }
        }
    }
}
```

### Pantalla de demostración de notificaciones

```kotlin
// 📁 Nuevo archivo → ui/notificaciones/PantallaNotificacionesLocales.kt
@Composable
fun PantallaNotificacionesLocales() {
    val context = LocalContext.current
    var permisoNotif by remember { mutableStateOf(
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU)
            ContextCompat.checkSelfPermission(
                context, android.Manifest.permission.POST_NOTIFICATIONS
            ) == PackageManager.PERMISSION_GRANTED
        else true
    )}
    var progreso by remember { mutableStateOf(0) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { permisoNotif = it }

    Scaffold(
        topBar = { TopAppBar(title = { Text("Notificaciones locales",
            fontWeight = FontWeight.Bold) }) }
    ) { padding ->
        LazyColumn(
            modifier        = Modifier.padding(padding).fillMaxSize(),
            contentPadding  = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            // Permiso
            if (!permisoNotif) {
                item {
                    ElevatedCard(
                        colors = CardDefaults.elevatedCardColors(
                            containerColor = MaterialTheme.colorScheme.errorContainer)
                    ) {
                        Row(
                            Modifier.padding(16.dp).fillMaxWidth(),
                            horizontalArrangement = Arrangement.SpaceBetween,
                            verticalAlignment     = Alignment.CenterVertically
                        ) {
                            Text("Permiso de notificaciones requerido",
                                color = MaterialTheme.colorScheme.onErrorContainer)
                            Button(onClick = {
                                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU)
                                    launcher.launch(android.Manifest.permission.POST_NOTIFICATIONS)
                            }) { Text("Conceder") }
                        }
                    }
                }
            }

            item {
                Button(
                    onClick  = {
                        NotificacionesHelper.simple(
                            context, 1,
                            "¡Hola desde Kotlin!",
                            "Esta es una notificación simple."
                        )
                    },
                    enabled  = permisoNotif,
                    modifier = Modifier.fillMaxWidth()
                ) { Text("Notificación simple") }
            }

            item {
                Button(
                    onClick  = {
                        NotificacionesHelper.expandible(
                            context, 2,
                            "Resumen largo",
                            "Toca para expandir...",
                            "Este es el texto completo de la notificación. " +
                            "Puede contener varios párrafos y mucha información detallada " +
                            "que no cabe en una sola línea de la barra de notificaciones."
                        )
                    },
                    enabled  = permisoNotif,
                    modifier = Modifier.fillMaxWidth()
                ) { Text("Notificación expandible") }
            }

            item {
                Button(
                    onClick  = {
                        NotificacionesHelper.conBotones(
                            context, 3,
                            "Recordatorio de tarea",
                            "¿Completaste 'Revisar el código'?",
                            tareaId = 42
                        )
                    },
                    enabled  = permisoNotif,
                    modifier = Modifier.fillMaxWidth()
                ) { Text("Notificación con botones") }
            }

            item {
                Column {
                    Text("Progreso: $progreso%",
                        style = MaterialTheme.typography.labelMedium)
                    Spacer(Modifier.height(8.dp))
                    Slider(
                        value         = progreso.toFloat(),
                        onValueChange = {
                            progreso = it.toInt()
                            NotificacionesHelper.conProgreso(
                                context, 4,
                                "Descargando archivo...",
                                progreso,
                                completa = progreso >= 100
                            )
                        },
                        valueRange = 0f..100f,
                        modifier   = Modifier.fillMaxWidth()
                    )
                }
            }
        }
    }
}
```

---

### ▶ Checkpoint 1 — Notificaciones locales en acción

**Conecta los archivos al proyecto:**

1. Añade `<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />` en `AndroidManifest.xml`.
2. Declara `AccionesNotificacionReceiver` en el manifest (ver bloque XML de la sección anterior).
3. Crea el paquete `notificaciones/` y copia `CanalNotificacion.kt`, `NotificacionesHelper.kt` y `AccionesNotificacionReceiver.kt`.
4. Crea el paquete `ui/notificaciones/` y copia `PantallaNotificacionesLocales.kt`.
5. En la clase `Application` (o en `MainActivity.onCreate()`), llama a `crearCanalesNotificacion(this)`.
6. En `NavGraph.kt`, añade:
   ```kotlin
   composable("notificaciones") { PantallaNotificacionesLocales() }
   ```
7. En `PantallaDashboard`, añade un botón 🔔 que navegue a `"notificaciones"`.
8. Ejecuta en el emulador (Android 13+: acepta el permiso cuando aparezca el diálogo).

📁 **Reemplaza TODO el contenido de `ui/NavGraph.kt`** con este código:

```kotlin
package com.ute.techdash.ui

import androidx.compose.runtime.Composable
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.ute.techdash.ui.ajustes.PantallaAjustes
import com.ute.techdash.ui.hardware.gps.PantallaGPS
import com.ute.techdash.ui.hardware.sensores.PantallaSensores
import com.ute.techdash.ui.notificaciones.PantallaNotificacionesLocales
import com.ute.techdash.ui.tareas.PantallaTareas

@Composable
fun TechDashNavGraph() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = "dashboard") {
        composable("dashboard")      { PantallaDashboard(navController) }
        composable("gps")            { PantallaGPS() }
        composable("sensores")       { PantallaSensores() }
        composable("ajustes")        { PantallaAjustes() }
        composable("tareas")         { PantallaTareas() }
        composable("notificaciones") { PantallaNotificacionesLocales() }  // ← nuevo (p20)
    }
}
```

> `MainActivity.kt` no cambia — sigue llamando a `TechDashNavGraph()`.

**¿Qué deberías ver?**
- 🔔 "Notificación simple" → aparece en la barra de estado.
- 📄 "Notificación expandible" → se despliega al deslizar hacia abajo.
- ⚡ "Notificación con botones" → muestra dos acciones: **Completar** y **Eliminar**.
- 📊 El slider de progreso actualiza la notificación de descarga en tiempo real.

> 💡 Si la notificación no aparece en Android 13+: revisa que el permiso está en el manifest **y** que el usuario lo concedió en el diálogo del sistema.
> Si los botones de acción no hacen nada: verifica que `AccionesNotificacionReceiver` está declarado con `android:exported="false"` en el manifest.

### 🧪 Prueba esto

> **Reto 1 — Canal de urgencia:** Crea un nuevo canal `ALERTAS_CRITICAS` con `IMPORTANCE_HIGH`, vibración y luz roja. Añade un botón "⚠️ Alerta crítica" en `PantallaNotificacionesLocales` que lo use. Compara el comportamiento visual con el canal `GENERAL`: ¿cuál muestra cabecera emergente (heads-up)?
>
> **Reto 2 — Conectar con Room:** Modifica `AccionesNotificacionReceiver` para que, al pulsar "Completar", marque la tarea como completada en Room (usa `AppDatabase.getInstance(context)`). Pruébalo enviando la notificación desde `PantallaTareas` al añadir una tarea nueva.
>
> **Reflexión:** Las notificaciones locales con `setOngoing(true)` no se pueden deslizar para descartar. ¿Cuándo es apropiado usar esto y cuándo sería molesto para el usuario? ¿Qué relación tiene con el canal de descarga (`IMPORTANCE_LOW`)?

---

## Parte 2 — Push con Firebase Cloud Messaging (FCM)

### Configuración del proyecto Firebase

```
1. Ir a https://console.firebase.google.com
2. Crear proyecto → Agregar app Android
3. Registrar package name de la app (ej. com.ejemplo.productsapp)
4. Descargar google-services.json → copiar a app/
5. Seguir instrucciones de los plugins de Gradle
```

### `build.gradle.kts` — plugins y dependencias

```kotlin
// build.gradle.kts (proyecto raíz)
plugins {
    id("com.google.gms.google-services") version "4.4.2" apply false
}

// build.gradle.kts (módulo app)
plugins {
    id("com.google.gms.google-services")   // ← aplica el plugin aquí
}

dependencies {
    // Firebase BOM — versiones compatibles entre sí
    implementation(platform("com.google.firebase:firebase-bom:33.7.0"))
    implementation("com.google.firebase:firebase-messaging-ktx")
    implementation("com.google.firebase:firebase-analytics-ktx")  // opcional
}
```

### `FirebaseMessagingService` — recibir mensajes

```kotlin
// 📁 Nuevo archivo → notificaciones/MiFirebaseMessagingService.kt
import com.google.firebase.messaging.FirebaseMessagingService
import com.google.firebase.messaging.RemoteMessage

class MiFirebaseMessagingService : FirebaseMessagingService() {

    // Se llama cuando llega un mensaje PUSH
    // Puede llegar en primer plano o en fondo
    override fun onMessageReceived(message: RemoteMessage) {
        super.onMessageReceived(message)

        // Datos del mensaje — dos tipos de payload
        val titulo   = message.notification?.title ?: message.data["titulo"] ?: "Notificación"
        val cuerpo   = message.notification?.body  ?: message.data["cuerpo"] ?: ""
        val pantalla = message.data["pantalla"]     // para deep link

        println("FCM recibido — Tipo: ${if (message.notification != null) "notification" else "data"}")
        println("Título: $titulo | Cuerpo: $cuerpo | Pantalla: $pantalla")

        // Mostrar notificación local cuando la app está en PRIMER PLANO
        // En fondo, Firebase la muestra automáticamente (solo si tiene notification payload)
        if (isAppEnPrimerPlano()) {
            mostrarNotificacionLocal(titulo, cuerpo, pantalla)
        }
    }

    // Se llama cuando el token FCM cambia (primera instalación o regeneración)
    override fun onNewToken(token: String) {
        super.onNewToken(token)
        println("Nuevo token FCM: $token")
        // Enviar el token al backend para futuras notificaciones push
        enviarTokenAlBackend(token)
    }

    private fun mostrarNotificacionLocal(
        titulo:    String,
        cuerpo:    String,
        pantalla:  String?
    ) {
        // Construir intent de destino para deep link
        val intent = Intent(this, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
            pantalla?.let { putExtra("destino", it) }
        }
        NotificacionesHelper.conAccion(
            context      = this,
            id           = System.currentTimeMillis().toInt(),
            titulo       = titulo,
            mensaje      = cuerpo,
            textoAccion  = "Abrir",
            intentAccion = intent
        )
    }

    private fun enviarTokenAlBackend(token: String) {
        // Aquí harías una llamada a tu API para registrar el token
        // ej: api.registrarToken(TokenRequest(token))
    }

    private fun isAppEnPrimerPlano(): Boolean {
        val activityManager = getSystemService(ACTIVITY_SERVICE)
            as android.app.ActivityManager
        val appProcesses    = activityManager.runningAppProcesses ?: return false
        return appProcesses.any {
            it.importance == android.app.ActivityManager.RunningAppProcessInfo.IMPORTANCE_FOREGROUND
                    && it.processName == packageName
        }
    }
}
```

### Registrar el servicio en `AndroidManifest.xml`

```xml
<application ...>

    <service
        android:name=".notificaciones.MiFirebaseMessagingService"
        android:exported="false">
        <intent-filter>
            <action android:name="com.google.firebase.MESSAGING_EVENT" />
        </intent-filter>
    </service>

    <!-- Icono y color por defecto para notificaciones FCM -->
    <meta-data
        android:name="com.google.firebase.messaging.default_notification_icon"
        android:resource="@drawable/ic_notification" />
    <meta-data
        android:name="com.google.firebase.messaging.default_notification_color"
        android:resource="@color/colorPrimary" />
    <meta-data
        android:name="com.google.firebase.messaging.default_notification_channel_id"
        android:value="canal_general" />

</application>
```

### Obtener y gestionar el token FCM

```kotlin
// 📁 Nuevo archivo → notificaciones/TokenFcmRepository.kt
import com.google.firebase.messaging.FirebaseMessaging
import kotlinx.coroutines.tasks.await

class TokenFcmRepository(
    private val preferencias: PreferenciasRepository  // DataStore de página 19
) {
    private val tokenKey = stringPreferencesKey("fcm_token")

    // Obtener token actual
    suspend fun obtenerToken(): String? {
        return try {
            FirebaseMessaging.getInstance().token.await()
        } catch (e: Exception) {
            null
        }
    }

    // Suscribirse a un topic (broadcast a grupos de usuarios)
    suspend fun suscribirA(topic: String) {
        try {
            FirebaseMessaging.getInstance().subscribeToTopic(topic).await()
            println("Suscrito al topic: $topic")
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }

    suspend fun desuscribirDe(topic: String) {
        try {
            FirebaseMessaging.getInstance().unsubscribeFromTopic(topic).await()
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }
}
```

### Tipos de mensajes FCM

```kotlin
// Los mensajes FCM tienen DOS formatos de payload:

// 1. "notification" payload — Firebase lo muestra automáticamente en fondo
//    Solo llega a onMessageReceived() cuando la app está en PRIMER PLANO
val payloadNotification = """
{
  "to": "TOKEN_DEL_DISPOSITIVO",
  "notification": {
    "title": "Nuevo mensaje",
    "body": "Tienes un mensaje de Ana García",
    "click_action": "ABRIR_CHAT"
  }
}
"""

// 2. "data" payload — SIEMPRE llega a onMessageReceived() (primer y segundo plano)
//    Tú controlas cómo se muestra
val payloadData = """
{
  "to": "TOKEN_DEL_DISPOSITIVO",
  "data": {
    "titulo": "Nuevo pedido",
    "cuerpo": "El pedido #1234 fue confirmado",
    "pantalla": "pedidos",
    "pedido_id": "1234"
  }
}
"""

// 3. Combinado — data en fondo, notification en primer plano
val payloadCombinado = """
{
  "to": "TOKEN_DEL_DISPOSITIVO",
  "notification": { "title": "...", "body": "..." },
  "data": { "pantalla": "pedidos", "id": "1234" }
}
"""
```

### Pantalla de configuración de notificaciones push

```kotlin
// 📁 Nuevos archivos → ui/notificaciones/PantallaConfigFCM.kt  +  NotificacionesViewModel.kt
@Composable
fun PantallaConfigFCM(
    vm: NotificacionesViewModel = viewModel()
) {
    val token      by vm.token.collectAsStateWithLifecycle("")
    val suscripciones by vm.suscripciones.collectAsStateWithLifecycle(emptySet())

    val topics = listOf(
        "noticias"        to "Noticias generales",
        "ofertas"         to "Ofertas y descuentos",
        "actualizaciones" to "Actualizaciones de la app"
    )

    Scaffold(
        topBar = { TopAppBar(title = { Text("Push Notifications",
            fontWeight = FontWeight.Bold) }) }
    ) { padding ->
        LazyColumn(
            modifier        = Modifier.padding(padding).fillMaxSize(),
            contentPadding  = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            // Token FCM
            item {
                ElevatedCard(Modifier.fillMaxWidth()) {
                    Column(Modifier.padding(16.dp)) {
                        Text("Token del dispositivo",
                            fontWeight = FontWeight.SemiBold)
                        Spacer(Modifier.height(8.dp))
                        Text(
                            token.ifEmpty { "Obteniendo token..." },
                            style    = MaterialTheme.typography.bodySmall,
                            color    = MaterialTheme.colorScheme.onSurfaceVariant,
                            maxLines = 3,
                            overflow = TextOverflow.Ellipsis
                        )
                        if (token.isNotEmpty()) {
                            Spacer(Modifier.height(8.dp))
                            OutlinedButton(
                                onClick  = { vm.copiarToken() },
                                modifier = Modifier.align(Alignment.End)
                            ) {
                                Icon(Icons.Default.ContentCopy, null,
                                    modifier = Modifier.size(16.dp))
                                Spacer(Modifier.width(4.dp))
                                Text("Copiar token")
                            }
                        }
                    }
                }
            }

            // Suscripciones a topics
            item {
                Text("Topics",
                    style      = MaterialTheme.typography.labelLarge,
                    color      = MaterialTheme.colorScheme.primary,
                    fontWeight = FontWeight.Bold)
            }

            items(topics) { (topicId, nombre) ->
                val estasSuscrito = topicId in suscripciones
                ElevatedCard(Modifier.fillMaxWidth()) {
                    Row(
                        Modifier.padding(horizontal = 16.dp, vertical = 12.dp),
                        horizontalArrangement = Arrangement.SpaceBetween,
                        verticalAlignment     = Alignment.CenterVertically
                    ) {
                        Column(Modifier.weight(1f)) {
                            Text(nombre, fontWeight = FontWeight.Medium)
                            Text(
                                if (estasSuscrito) "Suscrito" else "No suscrito",
                                style = MaterialTheme.typography.bodySmall,
                                color = if (estasSuscrito) MaterialTheme.colorScheme.primary
                                        else MaterialTheme.colorScheme.onSurfaceVariant
                            )
                        }
                        Switch(
                            checked         = estasSuscrito,
                            onCheckedChange = { vm.toggleTopic(topicId, it) }
                        )
                    }
                }
            }
        }
    }
}

class NotificacionesViewModel(
    private val tokenRepo: TokenFcmRepository
) : ViewModel() {

    private val _token = MutableStateFlow("")
    val token: StateFlow<String> = _token.asStateFlow()

    private val _suscripciones = MutableStateFlow(setOf<String>())
    val suscripciones: StateFlow<Set<String>> = _suscripciones.asStateFlow()

    init {
        viewModelScope.launch {
            _token.value = tokenRepo.obtenerToken() ?: ""
        }
    }

    fun toggleTopic(topic: String, suscribir: Boolean) {
        viewModelScope.launch {
            if (suscribir) {
                tokenRepo.suscribirA(topic)
                _suscripciones.update { it + topic }
            } else {
                tokenRepo.desuscribirDe(topic)
                _suscripciones.update { it - topic }
            }
        }
    }

    fun copiarToken() {
        // Aquí usarías ClipboardManager para copiar al portapapeles
        println("Token copiado: ${_token.value}")
    }
}
```

---

### ▶ Checkpoint 2 — Push con Firebase Cloud Messaging

**Configuración de Firebase (hazlo antes de compilar):**

1. Ve a [console.firebase.google.com](https://console.firebase.google.com) y crea un proyecto.
2. Añade una app Android con el paquete `com.ute.techdash`.
3. Descarga `google-services.json` y cópialo en la carpeta `app/` (no en la raíz).
4. Aplica los plugins de Gradle según el wizard de Firebase.
5. Crea el paquete `notificaciones/` y copia `MiFirebaseMessagingService.kt` y `TokenFcmRepository.kt`.
6. Copia `PantallaConfigFCM.kt` en `ui/notificaciones/`.
7. Registra el servicio en `AndroidManifest.xml` (usa el bloque XML de la sección anterior).
8. En `NavGraph.kt`, añade:
   ```kotlin
   composable("config_fcm") { PantallaConfigFCM() }
   ```
9. En `PantallaDashboard`, añade un botón 📨 que navegue a `"config_fcm"`.
10. Ejecuta en el emulador.

📁 **Reemplaza TODO el contenido de `ui/NavGraph.kt`** con este código:

```kotlin
package com.ute.techdash.ui

import androidx.compose.runtime.Composable
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.ute.techdash.ui.ajustes.PantallaAjustes
import com.ute.techdash.ui.hardware.gps.PantallaGPS
import com.ute.techdash.ui.hardware.sensores.PantallaSensores
import com.ute.techdash.ui.notificaciones.PantallaConfigFCM
import com.ute.techdash.ui.notificaciones.PantallaNotificacionesLocales
import com.ute.techdash.ui.tareas.PantallaTareas

@Composable
fun TechDashNavGraph() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = "dashboard") {
        composable("dashboard")      { PantallaDashboard(navController) }
        composable("gps")            { PantallaGPS() }
        composable("sensores")       { PantallaSensores() }
        composable("ajustes")        { PantallaAjustes() }
        composable("tareas")         { PantallaTareas() }
        composable("notificaciones") { PantallaNotificacionesLocales() }
        composable("config_fcm")     { PantallaConfigFCM() }              // ← nuevo (p20)
    }
}
```

> `MainActivity.kt` no cambia — sigue llamando a `TechDashNavGraph()`.

**¿Qué deberías ver?**
- 🔑 La pantalla muestra el token FCM del dispositivo (puede tardar unos segundos en aparecer).
- 📋 El botón "Copiar token" lo pone en el portapapeles — úsalo en la consola de Firebase.
- 🔔 Desde **Firebase Console → Cloud Messaging → Enviar mensaje de prueba**, pega el token y envía — la notificación debe aparecer en el dispositivo.
- 🔁 Los switches de topics llaman a `subscribeToTopic` / `unsubscribeFromTopic` (el estado se resetea al reiniciar; el Reto 1 te pide persistirlo).

> 💡 Si el token no aparece: verifica que `google-services.json` está en `app/` y que el plugin `com.google.gms.google-services` está aplicado en el módulo `app`, no en la raíz.
> Si FCM no entrega el mensaje al emulador: asegúrate de usar un emulador con Google Play (Play Store icon en AVD Manager).

### 🧪 Prueba esto

> **Reto 1 — Persistir suscripciones:** Los topics seleccionados en `PantallaConfigFCM` se pierden al reiniciar la app. Usa `PreferenciasRepository` (DataStore de la página 19) para guardar un `Set<String>` de topics suscritos. Carga el estado guardado en `NotificacionesViewModel.init {}` y muéstralo correctamente al abrir la pantalla.
>
> **Reto 2 — Deep link desde notificación:** Modifica `MiFirebaseMessagingService` para que cuando el payload `data` contenga `"pantalla": "tareas"`, al tocar la notificación navegue directamente a `PantallaTareas`. Usa `Intent.putExtra("destino", pantalla)` y maneja el extra en `MainActivity.onCreate()` con `navController.navigate(destino)`.
>
> **Reflexión:** FCM tiene dos tipos de payload: `notification` (Firebase lo muestra automáticamente en segundo plano) y `data` (siempre llega a `onMessageReceived`). ¿Por qué Android hace esta distinción? Si envías un payload combinado (`notification` + `data`), ¿cuándo llegaría a `onMessageReceived` y cuándo lo mostraría Firebase directamente?

---

## Ejercicios propuestos

1. **Notificación programada con WorkManager** — Usa `WorkManager` para
   enviar una notificación local 10 segundos después de que el usuario
   añade una tarea. Crea un `NotificationWorker` que extienda `CoroutineWorker`
   y llame a `NotificacionesHelper.conBotones()` con los datos de la tarea.

2. **Deep link desde notificación** — Cuando el usuario toca una notificación
   push con `pantalla = "detalle"` e `id = "5"`, la app debe navegar
   directamente a `DetalleScreen(5)`. Maneja el `Intent` en `MainActivity`
   con `intent?.getStringExtra("destino")` y navega con `navController`.

3. **Conteo en el icono de la app** — Usa `ShortcutBadger` o la API nativa
   `NotificationBadge` para mostrar el número de notificaciones no leídas en
   el icono de la app. Actualiza el contador cada vez que llega una
   notificación push y bájalo a 0 al abrir la app.

4. **Payload data vs notification** — Implementa un servidor local simple con
   Ktor (1 endpoint `POST /enviar-push`) que use la API de FCM para enviar
   un mensaje. Prueba los tres tipos de payload (notification, data, combinado)
   y observa el comportamiento en primer y segundo plano.

---

## Resumen de la página 20

- Las notificaciones **locales** se generan desde la app con `NotificationManager`. Las **push** llegan desde un servidor vía Firebase Cloud Messaging.
- Android 8+ requiere **canales de notificación** — sin canal la notificación no se muestra. Android 13+ requiere además el permiso `POST_NOTIFICATIONS`.
- `NotificationCompat.Builder` crea notificaciones compatibles con todas las versiones. `setAutoCancel(true)` la descarta al tocarla. `setOngoing(true)` la fija en la barra.
- Los botones de acción en notificaciones usan `PendingIntent.getBroadcast` + `BroadcastReceiver` — la app no necesita estar en primer plano.
- FCM divide los mensajes en dos payloads: `notification` (Firebase los muestra automáticamente en fondo) y `data` (siempre llegan a `onMessageReceived`).
- `onNewToken` se llama en la primera instalación y cuando el token se regenera — siempre enviar el nuevo token al backend.
- Los **topics** permiten enviar a grupos de usuarios sin conocer cada token. Un usuario se suscribe con `subscribeToTopic("nombre")`.
- `FirebaseMessaging.getInstance().token.await()` obtiene el token actual de forma asíncrona con coroutines.

---

> **Siguiente página →** Página 21: Audio y video streaming con
> Media3/ExoPlayer — reproducción desde URL, playlist, `MediaSession`
> y notificación de medios en segundo plano.