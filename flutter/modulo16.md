# Tutorial Flutter — Página 16
## Módulo 5 · Conectividad
### Notificaciones locales y push con FCM

---

Las notificaciones son el canal principal para reactivar usuarios: desde recordatorios programados sin internet hasta mensajes push enviados desde un servidor.
En este módulo combinas `flutter_local_notifications`, Firebase Cloud Messaging y Riverpod para construir un sistema completo y robusto.

## ¿Qué aprenderás?

```
Característica          | Notificaciones Locales        | Push FCM
------------------------|-------------------------------|-------------------------------
¿Quién las envía?       | La propia app en el dispositivo | Servidor externo via Firebase
¿Cuándo?                | Ahora o en un tiempo programado | Cuando el servidor lo decide
¿Necesita internet?     | No                            | Sí (FCM requiere conexión)
Complejidad de setup    | Media (permisos + plugin)     | Alta (Firebase + google-services)
Casos de uso típicos    | Alarmas, recordatorios, timers | Promociones, alertas, mensajes
Funciona app cerrada    | Con zonedSchedule + boot recv | Sí (background handler)
Personalización visual  | Total (canal, sonido, icono)  | Limitada (payload del servidor)
```

---

## Patrones clave

### Patrón 1: Inicialización de `FlutterLocalNotificationsPlugin` y permisos

```dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

final FlutterLocalNotificationsPlugin flutterLocalNotificationsPlugin =
    FlutterLocalNotificationsPlugin();

// Canal de alta importancia (Android 8+)
const AndroidNotificationChannel channel = AndroidNotificationChannel(
  'canal_principal',   // id — debe coincidir en cada show()
  'Notificaciones',    // nombre visible en ajustes del sistema
  importance: Importance.high,
);

Future<void> inicializarNotificaciones() async {
  // 1. Crear el canal en Android
  await flutterLocalNotificationsPlugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.createNotificationChannel(channel);

  // 2. Configurar init settings por plataforma
  const initSettings = InitializationSettings(
    android: AndroidInitializationSettings('@mipmap/ic_launcher'),
    iOS: DarwinInitializationSettings(
      requestAlertPermission: true,
      requestBadgePermission: true,
      requestSoundPermission: true,
    ),
  );
  await flutterLocalNotificationsPlugin.initialize(initSettings);

  // 3. Solicitar permiso POST_NOTIFICATIONS (Android 13+)
  await flutterLocalNotificationsPlugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.requestNotificationsPermission();
}
```

> **Mini-ejercicio ⏱ 5 min:** Llama a `inicializarNotificaciones()` desde `main()` antes de `runApp()`. Luego cambia el nombre del canal a `'Alertas de prueba'` y verifica en Ajustes del emulador (Configuracion → Aplicaciones → tu app → Notificaciones) que el nuevo nombre aparece.
Añade `playSound: true` en los `DarwinInitializationSettings` para iOS.

---

### Patrón 2: `show()` con `AndroidNotificationDetails` y `DarwinNotificationDetails`

```dart
Future<void> mostrarNotificacion({
  required int id,
  required String titulo,
  required String cuerpo,
}) async {
  const androidDetails = AndroidNotificationDetails(
    'canal_principal',  // mismo id del canal creado en init
    'Notificaciones',
    channelDescription: 'Canal principal de la app',
    importance: Importance.high,
    priority: Priority.high,
    playSound: true,
    icon: '@mipmap/ic_launcher',
  );

  const iosDetails = DarwinNotificationDetails(
    presentAlert: true,
    presentBadge: true,
    presentSound: true,
  );

  const detalles = NotificationDetails(
    android: androidDetails,
    iOS: iosDetails,
  );

  await flutterLocalNotificationsPlugin.show(
    id,       // id único — usar el mismo id reemplaza la notificación anterior
    titulo,
    cuerpo,
    detalles,
  );
}
```

> **Mini-ejercicio ⏱ 5 min:** Llama a `mostrarNotificacion(id: 1, titulo: 'Hola', cuerpo: 'Primera notificación')`. Luego llama de nuevo con el mismo `id: 1` pero diferente cuerpo. Observa que la notificación anterior se reemplaza, no se acumula. Después prueba con `id: 2` para que aparezcan dos notificaciones simultáneas.

---

### Patrón 3: Notificación programada con `zonedSchedule()` y `TZDateTime`

```dart
import 'package:timezone/timezone.dart' as tz;
import 'package:timezone/data/latest_all.dart' as tz;

// Llamar una sola vez al arrancar la app:
// tz.initializeTimeZones();

Future<void> programarNotificacion({
  required int id,
  required String titulo,
  required String cuerpo,
  required Duration demoraDesdeAhora,
}) async {
  final fechaHora = tz.TZDateTime.now(tz.local).add(demoraDesdeAhora);

  await flutterLocalNotificationsPlugin.zonedSchedule(
    id,
    titulo,
    cuerpo,
    fechaHora,
    const NotificationDetails(
      android: AndroidNotificationDetails(
        'canal_principal', 'Notificaciones',
        importance: Importance.high,
        priority: Priority.high,
      ),
      iOS: DarwinNotificationDetails(),
    ),
    androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
    uiLocalNotificationDateInterpretation:
        UILocalNotificationDateInterpretation.absoluteTime,
  );
}

// Cancelar una notificación programada:
// await flutterLocalNotificationsPlugin.cancel(id);
// Cancelar todas:
// await flutterLocalNotificationsPlugin.cancelAll();
```

> **Mini-ejercicio ⏱ 6 min:** Programa 3 notificaciones con `Duration(seconds: 10)`, `Duration(seconds: 20)` y `Duration(seconds: 30)`. Asigna ids 10, 20 y 30. Espera 12 segundos — solo debe llegar la primera. Luego cancela la de id 20 con `cancel(20)` antes de que llegue y verifica que la de 30 sí aparece.

---

### Patrón 4: FCM foreground con `FirebaseMessaging.onMessage.listen()` → notificación local

```dart
import 'package:firebase_messaging/firebase_messaging.dart';

void configurarFCMForeground(
    FlutterLocalNotificationsPlugin plugin) {
  // onMessage solo se dispara cuando la app está en PRIMER PLANO
  // FCM no muestra notificación visual en foreground por defecto — la mostramos nosotros
  FirebaseMessaging.onMessage.listen((RemoteMessage message) {
    final notif = message.notification;
    if (notif != null) {
      plugin.show(
        message.hashCode,
        notif.title ?? 'Nuevo mensaje',
        notif.body ?? '',
        const NotificationDetails(
          android: AndroidNotificationDetails(
            'canal_principal', 'Notificaciones',
            importance: Importance.high,
            priority: Priority.high,
          ),
          iOS: DarwinNotificationDetails(),
        ),
      );
    }
  });

  // Cuando el usuario toca la notificación y la app estaba en background:
  FirebaseMessaging.onMessageOpenedApp.listen((RemoteMessage message) {
    // Aquí puedes navegar a una pantalla específica según message.data
    print('Notificación tocada desde background: ${message.messageId}');
  });
}
```

> **Mini-ejercicio ⏱ 5 min:** Imprime `message.data` dentro del listener de `onMessage`. Envía un mensaje de prueba desde la consola de Firebase con un campo extra `{"pantalla": "detalle"}`. Observa en la consola de Flutter cómo llega ese mapa. Luego añade un `if (message.data['pantalla'] == 'detalle')` que imprima `'Navegar a detalle'`.

---

### Patrón 5: Handler de background/terminated con `@pragma('vm:entry-point')`

```dart
// IMPORTANTE: debe ser función de NIVEL SUPERIOR, no un método de clase
// Si está dentro de una clase, FCM no puede invocarla cuando la app está cerrada
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  // Reinicializar Firebase porque la app puede no estar iniciada
  await Firebase.initializeApp();
  print('Mensaje en background/terminated: ${message.messageId}');
  // En background, FCM muestra la notificación del sistema automáticamente
  // si el mensaje incluye el campo "notification" (no solo "data")
}

// Registrar ANTES de runApp(), en main():
// FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);
```

> **Mini-ejercicio ⏱ 5 min:** Registra el handler en `main()` antes de `runApp()`. Cierra completamente la app (swipe en el task manager del emulador). Envía un push desde Firebase Console con un campo `notification` (titulo + body). Verifica que aparece la notificación del sistema. Luego envía un mensaje solo con `data` (sin `notification`) — observa que FCM no muestra nada visual automáticamente en ese caso.

---

### Patrón 6: `NotifierProvider` para historial de notificaciones

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Modelo simple de notificación recibida
class NotificacionLocal {
  final int id;
  final String titulo;
  final String cuerpo;
  final DateTime fechaHora;
  final TipoNotificacion tipo;
  final bool leida;

  const NotificacionLocal({
    required this.id,
    required this.titulo,
    required this.cuerpo,
    required this.fechaHora,
    required this.tipo,
    this.leida = false,
  });

  NotificacionLocal copyWith({bool? leida}) =>
      NotificacionLocal(
        id: id, titulo: titulo, cuerpo: cuerpo,
        fechaHora: fechaHora, tipo: tipo,
        leida: leida ?? this.leida,
      );
}

enum TipoNotificacion { local, push }

// Notifier: gestiona la lista inmutable
class NotificacionesNotifier extends Notifier<List<NotificacionLocal>> {
  @override
  List<NotificacionLocal> build() => [];

  void agregar(NotificacionLocal n) => state = [n, ...state];

  void marcarLeida(int id) {
    state = [
      for (final n in state)
        if (n.id == id) n.copyWith(leida: true) else n,
    ];
  }

  void limpiar() => state = [];
}

final notificacionesProvider =
    NotifierProvider<NotificacionesNotifier, List<NotificacionLocal>>(
  NotificacionesNotifier.new,
);
```

> **Mini-ejercicio ⏱ 6 min:** Añade un método `eliminar(int id)` al notifier que elimine una notificación por id. En el widget que muestra la lista, añade un `IconButton(icon: Icon(Icons.delete))` en cada `ListTile` que llame a `ref.read(notificacionesProvider.notifier).eliminar(n.id)`. Verifica que el item desaparece de la lista sin recargar la pantalla.

---

## Crea el proyecto

```bash
flutter create modulo16_notificaciones
cd modulo16_notificaciones
```

### `pubspec.yaml`

```yaml
name: modulo16_notificaciones
description: "Notificaciones locales y push FCM con Riverpod"
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.7.0

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod:              ^3.3.0
  flutter_local_notifications:   ^17.2.4
  firebase_core:                 ^3.9.0
  firebase_messaging:            ^15.2.4

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0

flutter:
  uses-material-design: true
```

```bash
flutter pub get
```

> La dependencia `timezone` es transitiva de `flutter_local_notifications` — no necesitas declararla explícitamente.

### Configuración nativa

#### Android — `android/app/src/main/AndroidManifest.xml`

Añadir dentro de `<manifest>`, antes de `<application>`:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
<uses-permission android:name="android.permission.VIBRATE"/>
```

Añadir dentro de `<application>`:

```xml
<!-- Receptor para notificaciones programadas -->
<receiver
    android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver"
    android:exported="false"/>
<receiver
    android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver"
    android:exported="false">
  <intent-filter>
    <action android:name="android.intent.action.BOOT_COMPLETED"/>
    <action android:name="android.intent.action.MY_PACKAGE_REPLACED"/>
    <action android:name="android.intent.action.QUICKBOOT_POWERON"/>
  </intent-filter>
</receiver>

<!-- FCM: servicio de mensajería en background -->
<service
    android:name="com.google.firebase.messaging.FirebaseMessagingService"
    android:exported="false">
  <intent-filter>
    <action android:name="com.google.firebase.MESSAGING_EVENT"/>
  </intent-filter>
</service>
```

#### iOS — `ios/Runner/Info.plist`

```xml
<key>UIBackgroundModes</key>
<array>
  <string>fetch</string>
  <string>remote-notification</string>
</array>
```

Verificar que `ios/Runner/AppDelegate.swift` no sobreescriba el handler de FCM.
En proyectos nuevos con Flutter 3.x el `AppDelegate` generado es compatible sin cambios.

#### Firebase — `google-services.json`

Coloca el archivo descargado desde Firebase Console en `android/app/google-services.json`.
En el `android/build.gradle` (nivel proyecto) asegúrate de tener:

```groovy
// android/build.gradle
dependencies {
    classpath 'com.google.gms:google-services:4.4.2'
}
```

Y en `android/app/build.gradle` (nivel app) al final:

```groovy
apply plugin: 'com.google.gms.google-services'
```

> **Nota de clase:** Si no tienes un proyecto Firebase configurado, los pasos 1 y 2 funcionan sin Firebase. Los pasos 3-5 requieren `google-services.json` real. Puedes dejar un archivo vacío de placeholder y el paso 3 simplemente fallará con un error claro.

---

## Selector de pasos

Cada paso es un archivo `lib/main.dart` independiente y ejecutable.
Cambia este número para saltar entre pasos:

```dart
// lib/main.dart  ← cambia este número para cada paso
const int paso = 1;
```

---

## Paso 1 — Init + permisos + primera notificación

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/data/latest_all.dart' as tz;

final FlutterLocalNotificationsPlugin _plugin =
    FlutterLocalNotificationsPlugin();

const AndroidNotificationChannel _canal = AndroidNotificationChannel(
  'canal_principal',
  'Notificaciones',
  importance: Importance.high,
);

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Inicializar zona horaria (requerido por zonedSchedule en pasos siguientes)
  tz.initializeTimeZones();

  // Crear canal en Android
  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.createNotificationChannel(_canal);

  // Configurar init settings
  await _plugin.initialize(
    const InitializationSettings(
      android: AndroidInitializationSettings('@mipmap/ic_launcher'),
      iOS: DarwinInitializationSettings(),
    ),
  );

  // Solicitar permisos Android 13+
  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.requestNotificationsPermission();

  runApp(const AppPaso1());
}

class AppPaso1 extends StatelessWidget {
  const AppPaso1({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Paso 1 — Notificación básica',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const PantallaPaso1(),
    );
  }
}

class PantallaPaso1 extends StatelessWidget {
  const PantallaPaso1({super.key});

  Future<void> _mostrar() async {
    await _plugin.show(
      1,
      'Hola mundo',
      'Primera notificación local funcionando',
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'canal_principal', 'Notificaciones',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 1 — Notificación básica')),
      body: Center(
        child: ElevatedButton.icon(
          onPressed: _mostrar,
          icon: const Icon(Icons.notifications),
          label: const Text('Enviar notificación'),
        ),
      ),
    );
  }
}
```

> **Prueba esto:** Pulsa el botón. En Android verás la notificación en la barra de estado; en iOS en el banner superior.
> Si no aparece nada, revisa que el emulador tiene activadas las notificaciones para la app (Ajustes → Aplicaciones → modulo16_notificaciones → Notificaciones → Activado).

Salida esperada en consola: ninguna (la notificación es silenciosa en consola). En el dispositivo: banner de notificación con título "Hola mundo".

---

## Paso 2 — Notificaciones programadas con `zonedSchedule`

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/timezone.dart' as tz;
import 'package:timezone/data/latest_all.dart' as tz_data;

final FlutterLocalNotificationsPlugin _plugin =
    FlutterLocalNotificationsPlugin();

const AndroidNotificationChannel _canal = AndroidNotificationChannel(
  'canal_principal', 'Notificaciones',
  importance: Importance.high,
);

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  tz_data.initializeTimeZones();  // OBLIGATORIO antes de zonedSchedule

  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.createNotificationChannel(_canal);

  await _plugin.initialize(
    const InitializationSettings(
      android: AndroidInitializationSettings('@mipmap/ic_launcher'),
      iOS: DarwinInitializationSettings(),
    ),
  );

  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.requestNotificationsPermission();

  runApp(const AppPaso2());
}

class AppPaso2 extends StatelessWidget {
  const AppPaso2({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Paso 2 — Programada',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaPaso2(),
    );
  }
}

class PantallaPaso2 extends StatelessWidget {
  const PantallaPaso2({super.key});

  Future<void> _programar(int segundos, int id) async {
    final cuando = tz.TZDateTime.now(tz.local).add(Duration(seconds: segundos));
    await _plugin.zonedSchedule(
      id,
      'Recordatorio programado',
      'Han pasado $segundos segundos desde que pulsaste el botón',
      cuando,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'canal_principal', 'Notificaciones',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
    );
  }

  Future<void> _cancelar(int id) async {
    await _plugin.cancel(id);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 2 — Programada')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => _programar(5, 5),
              child: const Text('Notificación en 5 segundos (id=5)'),
            ),
            const SizedBox(height: 12),
            ElevatedButton(
              onPressed: () => _programar(10, 10),
              child: const Text('Notificación en 10 segundos (id=10)'),
            ),
            const SizedBox(height: 12),
            OutlinedButton.icon(
              onPressed: () => _cancelar(10),
              icon: const Icon(Icons.cancel),
              label: const Text('Cancelar la de 10 s (id=10)'),
            ),
            const SizedBox(height: 12),
            OutlinedButton.icon(
              onPressed: () => _plugin.cancelAll(),
              icon: const Icon(Icons.delete_sweep),
              label: const Text('Cancelar todas'),
            ),
          ],
        ),
      ),
    );
  }
}
```

> **Prueba esto:** Pulsa "Notificación en 5 segundos" y pon la app en segundo plano. A los 5 s debe aparecer el banner. Luego pulsa la de 10 s, espera 3 s y cancela — no debe llegar ningún banner.

Salida esperada: banner "Recordatorio programado — Han pasado 5 segundos desde que pulsaste el botón" exactamente a los 5 segundos.

---

## Paso 3 — FCM: token + escucha de mensajes en foreground

```dart
// lib/main.dart
// REQUISITO: google-services.json en android/app/ y Firebase configurado
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/data/latest_all.dart' as tz;

final FlutterLocalNotificationsPlugin _plugin =
    FlutterLocalNotificationsPlugin();

const AndroidNotificationChannel _canal = AndroidNotificationChannel(
  'canal_principal', 'Notificaciones',
  importance: Importance.high,
);

// Handler de background — función de nivel superior OBLIGATORIO
@pragma('vm:entry-point')
Future<void> _handlerBackground(RemoteMessage message) async {
  await Firebase.initializeApp();
  print('[BG] Mensaje recibido: ${message.messageId}');
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  tz.initializeTimeZones();
  await Firebase.initializeApp();

  // Registrar handler de background ANTES de runApp
  FirebaseMessaging.onBackgroundMessage(_handlerBackground);

  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.createNotificationChannel(_canal);

  await _plugin.initialize(
    const InitializationSettings(
      android: AndroidInitializationSettings('@mipmap/ic_launcher'),
      iOS: DarwinInitializationSettings(),
    ),
  );

  // Permiso iOS para FCM
  await FirebaseMessaging.instance.requestPermission(
    alert: true, badge: true, sound: true,
  );

  // Permiso Android 13+
  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.requestNotificationsPermission();

  runApp(const AppPaso3());
}

class AppPaso3 extends StatelessWidget {
  const AppPaso3({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Paso 3 — FCM',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const PantallaPaso3(),
    );
  }
}

class PantallaPaso3 extends StatefulWidget {
  const PantallaPaso3({super.key});

  @override
  State<PantallaPaso3> createState() => _PantallaPaso3State();
}

class _PantallaPaso3State extends State<PantallaPaso3> {
  String _token = 'Obteniendo token...';
  String _ultimoMensaje = 'Ninguno';

  @override
  void initState() {
    super.initState();
    _obtenerToken();
    _escucharForeground();
  }

  Future<void> _obtenerToken() async {
    final token = await FirebaseMessaging.instance.getToken();
    setState(() => _token = token ?? 'No disponible');
    print('[FCM] Token: $token');
  }

  void _escucharForeground() {
    FirebaseMessaging.onMessage.listen((RemoteMessage message) {
      final notif = message.notification;
      if (notif != null) {
        setState(() => _ultimoMensaje = '${notif.title}: ${notif.body}');
        _plugin.show(
          message.hashCode,
          notif.title,
          notif.body,
          const NotificationDetails(
            android: AndroidNotificationDetails(
              'canal_principal', 'Notificaciones',
              importance: Importance.high, priority: Priority.high,
            ),
            iOS: DarwinNotificationDetails(),
          ),
        );
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 3 — FCM Token')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text('Token FCM del dispositivo:',
                style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            SelectableText(_token,
                style: const TextStyle(fontSize: 11, fontFamily: 'monospace')),
            const SizedBox(height: 24),
            const Text('Último mensaje FCM en foreground:',
                style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            Text(_ultimoMensaje, style: const TextStyle(fontSize: 14)),
          ],
        ),
      ),
    );
  }
}
```

> **Prueba esto:** Ejecuta la app. Copia el token que aparece en pantalla (o en la consola de Flutter). Ve a Firebase Console → Cloud Messaging → Enviar mensaje de prueba → pega el token como "dispositivo de destino". La app debe mostrar el banner Y actualizar el texto "Último mensaje FCM en foreground".

Salida esperada en consola: `[FCM] Token: f3Kz9...` (cadena larga única por dispositivo/instalación).

---

## Paso 4 — Riverpod: historial de notificaciones FCM

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/data/latest_all.dart' as tz;

// ---------- Modelo ----------
enum TipoNotificacion { local, push }

class NotificacionLocal {
  final int id;
  final String titulo;
  final String cuerpo;
  final DateTime fechaHora;
  final TipoNotificacion tipo;
  final bool leida;

  const NotificacionLocal({
    required this.id,
    required this.titulo,
    required this.cuerpo,
    required this.fechaHora,
    required this.tipo,
    this.leida = false,
  });

  NotificacionLocal copyWith({bool? leida}) => NotificacionLocal(
        id: id, titulo: titulo, cuerpo: cuerpo,
        fechaHora: fechaHora, tipo: tipo,
        leida: leida ?? this.leida,
      );
}

// ---------- Notifier ----------
class NotificacionesNotifier extends Notifier<List<NotificacionLocal>> {
  @override
  List<NotificacionLocal> build() => [];

  void agregar(NotificacionLocal n) => state = [n, ...state];
  void marcarLeida(int id) {
    state = [
      for (final n in state)
        if (n.id == id) n.copyWith(leida: true) else n,
    ];
  }
  void limpiar() => state = [];
}

final notificacionesProvider =
    NotifierProvider<NotificacionesNotifier, List<NotificacionLocal>>(
  NotificacionesNotifier.new,
);

// ---------- Infraestructura global ----------
final FlutterLocalNotificationsPlugin _plugin =
    FlutterLocalNotificationsPlugin();

@pragma('vm:entry-point')
Future<void> _handlerBackground(RemoteMessage message) async {
  await Firebase.initializeApp();
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  tz.initializeTimeZones();
  await Firebase.initializeApp();
  FirebaseMessaging.onBackgroundMessage(_handlerBackground);

  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.createNotificationChannel(const AndroidNotificationChannel(
          'canal_principal', 'Notificaciones',
          importance: Importance.high));

  await _plugin.initialize(const InitializationSettings(
    android: AndroidInitializationSettings('@mipmap/ic_launcher'),
    iOS: DarwinInitializationSettings(),
  ));

  await FirebaseMessaging.instance.requestPermission();
  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.requestNotificationsPermission();

  runApp(const ProviderScope(child: AppPaso4()));
}

// ---------- App ----------
class AppPaso4 extends StatelessWidget {
  const AppPaso4({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Paso 4 — Riverpod + FCM',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.orange),
        useMaterial3: true,
      ),
      home: const PantallaPaso4(),
    );
  }
}

class PantallaPaso4 extends ConsumerStatefulWidget {
  const PantallaPaso4({super.key});

  @override
  ConsumerState<PantallaPaso4> createState() => _PantallaPaso4State();
}

class _PantallaPaso4State extends ConsumerState<PantallaPaso4> {
  @override
  void initState() {
    super.initState();
    FirebaseMessaging.onMessage.listen((RemoteMessage message) {
      final notif = message.notification;
      if (notif != null) {
        final n = NotificacionLocal(
          id: message.hashCode,
          titulo: notif.title ?? 'Sin título',
          cuerpo: notif.body ?? '',
          fechaHora: DateTime.now(),
          tipo: TipoNotificacion.push,
        );
        ref.read(notificacionesProvider.notifier).agregar(n);
        _plugin.show(
          n.id, n.titulo, n.cuerpo,
          const NotificationDetails(
            android: AndroidNotificationDetails(
                'canal_principal', 'Notificaciones',
                importance: Importance.high, priority: Priority.high),
            iOS: DarwinNotificationDetails(),
          ),
        );
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final notificaciones = ref.watch(notificacionesProvider);
    return Scaffold(
      appBar: AppBar(
        title: const Text('Paso 4 — Historial FCM'),
        actions: [
          IconButton(
            icon: const Icon(Icons.delete_sweep),
            onPressed: () =>
                ref.read(notificacionesProvider.notifier).limpiar(),
          ),
        ],
      ),
      body: notificaciones.isEmpty
          ? const Center(
              child: Text('Envía un push desde Firebase Console',
                  style: TextStyle(color: Colors.grey)))
          : ListView.builder(
              itemCount: notificaciones.length,
              itemBuilder: (context, i) {
                final n = notificaciones[i];
                return ListTile(
                  leading: Icon(
                    n.leida ? Icons.mark_email_read : Icons.notifications,
                    color: n.leida ? Colors.grey : Colors.orange,
                  ),
                  title: Text(n.titulo,
                      style: TextStyle(
                          fontWeight: n.leida
                              ? FontWeight.normal
                              : FontWeight.bold)),
                  subtitle: Text(n.cuerpo),
                  trailing: Text(
                    '${n.fechaHora.hour}:${n.fechaHora.minute.toString().padLeft(2, '0')}',
                    style: const TextStyle(fontSize: 12),
                  ),
                  onTap: () =>
                      ref.read(notificacionesProvider.notifier).marcarLeida(n.id),
                );
              },
            ),
    );
  }
}
```

> **Prueba esto:** Envía 3 mensajes push desde Firebase Console. Deben aparecer en la lista con fondo en negrita. Toca uno — el ícono cambia y el texto vuelve a peso normal (marcado como leído). Pulsa el ícono de papelera en el `AppBar` — la lista se vacía.

Salida esperada: lista de tarjetas con la hora de recepción, marcado visual de leído/no leído.

---

## Paso 5 — App completa con `NavigationBar`: locales + push

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/timezone.dart' as tz;
import 'package:timezone/data/latest_all.dart' as tz_data;

// ---------- Modelo ----------
enum TipoNotificacion { local, push }

class NotificacionLocal {
  final int id;
  final String titulo;
  final String cuerpo;
  final DateTime fechaHora;
  final TipoNotificacion tipo;
  final bool leida;

  const NotificacionLocal({
    required this.id,
    required this.titulo,
    required this.cuerpo,
    required this.fechaHora,
    required this.tipo,
    this.leida = false,
  });

  NotificacionLocal copyWith({bool? leida}) => NotificacionLocal(
        id: id, titulo: titulo, cuerpo: cuerpo,
        fechaHora: fechaHora, tipo: tipo,
        leida: leida ?? this.leida,
      );
}

// ---------- Provider ----------
class NotificacionesNotifier extends Notifier<List<NotificacionLocal>> {
  @override
  List<NotificacionLocal> build() => [];
  void agregar(NotificacionLocal n) => state = [n, ...state];
  void marcarLeida(int id) {
    state = [
      for (final n in state)
        if (n.id == id) n.copyWith(leida: true) else n,
    ];
  }
  void limpiar() => state = [];
}

final notificacionesProvider =
    NotifierProvider<NotificacionesNotifier, List<NotificacionLocal>>(
  NotificacionesNotifier.new,
);

final tabProvider = StateProvider<int>((ref) => 0);

// ---------- Infraestructura ----------
final FlutterLocalNotificationsPlugin _plugin =
    FlutterLocalNotificationsPlugin();

const AndroidNotificationChannel _canal = AndroidNotificationChannel(
  'canal_principal', 'Notificaciones',
  importance: Importance.high,
);

@pragma('vm:entry-point')
Future<void> _handlerBackground(RemoteMessage message) async {
  await Firebase.initializeApp();
}

Future<void> _mostrar(int id, String titulo, String cuerpo) async {
  await _plugin.show(
    id, titulo, cuerpo,
    const NotificationDetails(
      android: AndroidNotificationDetails(
          'canal_principal', 'Notificaciones',
          importance: Importance.high, priority: Priority.high),
      iOS: DarwinNotificationDetails(),
    ),
  );
}

Future<void> _programar(int id, String titulo, String cuerpo,
    Duration demora) async {
  await _plugin.zonedSchedule(
    id, titulo, cuerpo,
    tz.TZDateTime.now(tz.local).add(demora),
    const NotificationDetails(
      android: AndroidNotificationDetails(
          'canal_principal', 'Notificaciones',
          importance: Importance.high, priority: Priority.high),
      iOS: DarwinNotificationDetails(),
    ),
    androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
    uiLocalNotificationDateInterpretation:
        UILocalNotificationDateInterpretation.absoluteTime,
  );
}

// ---------- main ----------
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  tz_data.initializeTimeZones();
  await Firebase.initializeApp();
  FirebaseMessaging.onBackgroundMessage(_handlerBackground);

  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.createNotificationChannel(_canal);

  await _plugin.initialize(const InitializationSettings(
    android: AndroidInitializationSettings('@mipmap/ic_launcher'),
    iOS: DarwinInitializationSettings(),
  ));

  await FirebaseMessaging.instance.requestPermission(
      alert: true, badge: true, sound: true);
  await _plugin
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.requestNotificationsPermission();

  runApp(const ProviderScope(child: NotificacionesApp()));
}

// ---------- App raíz ----------
class NotificacionesApp extends StatelessWidget {
  const NotificacionesApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Notificaciones — Módulo 16',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const PantallaRaiz(),
    );
  }
}

class PantallaRaiz extends ConsumerStatefulWidget {
  const PantallaRaiz({super.key});

  @override
  ConsumerState<PantallaRaiz> createState() => _PantallaRaizState();
}

class _PantallaRaizState extends ConsumerState<PantallaRaiz> {
  @override
  void initState() {
    super.initState();
    // Escuchar mensajes FCM en foreground
    FirebaseMessaging.onMessage.listen((RemoteMessage message) {
      final notif = message.notification;
      if (notif != null) {
        final n = NotificacionLocal(
          id: message.hashCode,
          titulo: notif.title ?? 'Push recibido',
          cuerpo: notif.body ?? '',
          fechaHora: DateTime.now(),
          tipo: TipoNotificacion.push,
        );
        ref.read(notificacionesProvider.notifier).agregar(n);
        _mostrar(n.id, n.titulo, n.cuerpo);
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final tabActual = ref.watch(tabProvider);
    const tabs = [
      PantallaLocales(),
      PantallaPush(),
    ];
    return Scaffold(
      body: tabs[tabActual],
      bottomNavigationBar: NavigationBar(
        selectedIndex: tabActual,
        onDestinationSelected: (i) =>
            ref.read(tabProvider.notifier).state = i,
        destinations: const [
          NavigationDestination(
              icon: Icon(Icons.notifications_outlined),
              selectedIcon: Icon(Icons.notifications),
              label: 'Locales'),
          NavigationDestination(
              icon: Icon(Icons.cloud_outlined),
              selectedIcon: Icon(Icons.cloud),
              label: 'Push FCM'),
        ],
      ),
    );
  }
}

// ---------- Tab 0: Locales ----------
class PantallaLocales extends ConsumerStatefulWidget {
  const PantallaLocales({super.key});

  @override
  ConsumerState<PantallaLocales> createState() => _PantallaLocalesState();
}

class _PantallaLocalesState extends ConsumerState<PantallaLocales> {
  int _contadorId = 100;

  Future<void> _enviarAhora() async {
    final id = _contadorId++;
    final n = NotificacionLocal(
      id: id,
      titulo: 'Notificación local #$id',
      cuerpo: 'Enviada a las ${TimeOfDay.now().format(context)}',
      fechaHora: DateTime.now(),
      tipo: TipoNotificacion.local,
    );
    ref.read(notificacionesProvider.notifier).agregar(n);
    await _mostrar(n.id, n.titulo, n.cuerpo);
  }

  Future<void> _programar5s() async {
    final id = _contadorId++;
    final n = NotificacionLocal(
      id: id,
      titulo: 'Recordatorio #$id',
      cuerpo: 'Programado 5 segundos desde ${TimeOfDay.now().format(context)}',
      fechaHora: DateTime.now().add(const Duration(seconds: 5)),
      tipo: TipoNotificacion.local,
    );
    ref.read(notificacionesProvider.notifier).agregar(n);
    await _programar(n.id, n.titulo, n.cuerpo,
        const Duration(seconds: 5));
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Llega en 5 segundos...')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    final notifs = ref.watch(notificacionesProvider)
        .where((n) => n.tipo == TipoNotificacion.local)
        .toList();
    return Scaffold(
      appBar: AppBar(
        title: const Text('Notificaciones Locales'),
        actions: [
          IconButton(
            icon: const Icon(Icons.delete_sweep),
            onPressed: () =>
                ref.read(notificacionesProvider.notifier).limpiar(),
          ),
        ],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              children: [
                Expanded(
                  child: ElevatedButton.icon(
                    onPressed: _enviarAhora,
                    icon: const Icon(Icons.send),
                    label: const Text('Ahora'),
                  ),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: ElevatedButton.icon(
                    onPressed: _programar5s,
                    icon: const Icon(Icons.timer),
                    label: const Text('En 5 s'),
                  ),
                ),
              ],
            ),
          ),
          Expanded(
            child: notifs.isEmpty
                ? const Center(
                    child: Text('Sin notificaciones locales aún',
                        style: TextStyle(color: Colors.grey)))
                : ListView.builder(
                    itemCount: notifs.length,
                    itemBuilder: (context, i) {
                      final n = notifs[i];
                      return ListTile(
                        leading: const Icon(Icons.notifications,
                            color: Colors.indigo),
                        title: Text(n.titulo),
                        subtitle: Text(n.cuerpo),
                        trailing: Text(
                          '${n.fechaHora.hour}:${n.fechaHora.minute.toString().padLeft(2, '0')}',
                          style: const TextStyle(fontSize: 11),
                        ),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}

// ---------- Tab 1: Push FCM ----------
class PantallaPush extends ConsumerStatefulWidget {
  const PantallaPush({super.key});

  @override
  ConsumerState<PantallaPush> createState() => _PantallaPushState();
}

class _PantallaPushState extends ConsumerState<PantallaPush> {
  String _token = 'Obteniendo...';

  @override
  void initState() {
    super.initState();
    _obtenerToken();
  }

  Future<void> _obtenerToken() async {
    final t = await FirebaseMessaging.instance.getToken();
    if (mounted) setState(() => _token = t ?? 'No disponible');
  }

  @override
  Widget build(BuildContext context) {
    final pushNotifs = ref.watch(notificacionesProvider)
        .where((n) => n.tipo == TipoNotificacion.push)
        .toList();
    return Scaffold(
      appBar: AppBar(
        title: const Text('Push FCM'),
        actions: [
          IconButton(
            icon: const Icon(Icons.delete_sweep),
            onPressed: () =>
                ref.read(notificacionesProvider.notifier).limpiar(),
          ),
        ],
      ),
      body: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                const Text('Token FCM:',
                    style: TextStyle(fontWeight: FontWeight.bold)),
                const SizedBox(height: 6),
                SelectableText(
                  _token,
                  style: const TextStyle(
                      fontSize: 10, fontFamily: 'monospace'),
                ),
              ],
            ),
          ),
          const Divider(),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Text(
              'Mensajes recibidos (${pushNotifs.length})',
              style: const TextStyle(fontWeight: FontWeight.bold),
            ),
          ),
          Expanded(
            child: pushNotifs.isEmpty
                ? const Center(
                    child: Text('Envía un push desde Firebase Console',
                        style: TextStyle(color: Colors.grey)))
                : ListView.builder(
                    itemCount: pushNotifs.length,
                    itemBuilder: (context, i) {
                      final n = pushNotifs[i];
                      return ListTile(
                        leading: Icon(
                          n.leida
                              ? Icons.mark_email_read
                              : Icons.mark_email_unread,
                          color: n.leida ? Colors.grey : Colors.deepPurple,
                        ),
                        title: Text(n.titulo,
                            style: TextStyle(
                                fontWeight: n.leida
                                    ? FontWeight.normal
                                    : FontWeight.bold)),
                        subtitle: Text(n.cuerpo),
                        trailing: Text(
                          '${n.fechaHora.hour}:${n.fechaHora.minute.toString().padLeft(2, '0')}',
                          style: const TextStyle(fontSize: 11),
                        ),
                        onTap: () => ref
                            .read(notificacionesProvider.notifier)
                            .marcarLeida(n.id),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}
```

> **Prueba esto:** Tab "Locales" → pulsa "Ahora" 3 veces y luego "En 5 s". A los 5 s llega la notificación programada. Ve al tab "Push FCM" → copia el token → envía un push desde Firebase Console → el mensaje aparece en la lista. Toca un mensaje para marcarlo como leído.

Salida esperada: dos tabs funcionales, historial separado por tipo, marcado de leído sin perder los otros items.

---

## `main.dart` completo — referencia

El archivo del Paso 5 es el `main.dart` completo de referencia. Incluye todos los patrones del módulo integrados en una sola app funcional con `NavigationBar`.

Para el proyecto final estructurado en múltiples archivos, ver la sección "Proyecto final" a continuación.

---

## Proyecto final

### Estructura

```
modulo16_notificaciones/
├── android/
│   └── app/
│       ├── src/main/AndroidManifest.xml   (permisos + receivers FCM)
│       └── google-services.json           (desde Firebase Console)
├── ios/
│   └── Runner/
│       └── Info.plist                     (UIBackgroundModes)
├── lib/
│   ├── main.dart
│   ├── models/
│   │   └── notificacion_local.dart
│   ├── services/
│   │   └── servicio_notificaciones.dart
│   ├── providers/
│   │   └── notificaciones_provider.dart
│   └── screens/
│       ├── pantalla_locales.dart
│       └── pantalla_push.dart
└── pubspec.yaml
```

### Archivos clave

#### `lib/models/notificacion_local.dart`

```dart
enum TipoNotificacion { local, push }

class NotificacionLocal {
  final int id;
  final String titulo;
  final String cuerpo;
  final DateTime fechaHora;
  final TipoNotificacion tipo;
  final bool leida;

  const NotificacionLocal({
    required this.id,
    required this.titulo,
    required this.cuerpo,
    required this.fechaHora,
    required this.tipo,
    this.leida = false,
  });

  NotificacionLocal copyWith({bool? leida}) => NotificacionLocal(
        id: id,
        titulo: titulo,
        cuerpo: cuerpo,
        fechaHora: fechaHora,
        tipo: tipo,
        leida: leida ?? this.leida,
      );

  @override
  String toString() =>
      'NotificacionLocal(id: $id, titulo: $titulo, tipo: $tipo, leida: $leida)';
}
```

#### `lib/services/servicio_notificaciones.dart`

```dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/timezone.dart' as tz;

class ServicioNotificaciones {
  // Singleton
  ServicioNotificaciones._();
  static final ServicioNotificaciones instance = ServicioNotificaciones._();

  final FlutterLocalNotificationsPlugin _plugin =
      FlutterLocalNotificationsPlugin();

  static const AndroidNotificationChannel canal = AndroidNotificationChannel(
    'canal_principal',
    'Notificaciones',
    importance: Importance.high,
  );

  Future<void> init() async {
    // Crear canal en Android
    await _plugin
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(canal);

    // Inicializar plugin
    await _plugin.initialize(
      const InitializationSettings(
        android: AndroidInitializationSettings('@mipmap/ic_launcher'),
        iOS: DarwinInitializationSettings(
          requestAlertPermission: true,
          requestBadgePermission: true,
          requestSoundPermission: true,
        ),
      ),
    );

    // Solicitar permiso Android 13+
    await _plugin
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.requestNotificationsPermission();
  }

  Future<void> mostrar({
    required int id,
    required String titulo,
    required String cuerpo,
  }) async {
    await _plugin.show(
      id,
      titulo,
      cuerpo,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'canal_principal',
          'Notificaciones',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
    );
  }

  Future<void> programar({
    required int id,
    required String titulo,
    required String cuerpo,
    required Duration demora,
  }) async {
    final cuando = tz.TZDateTime.now(tz.local).add(demora);
    await _plugin.zonedSchedule(
      id,
      titulo,
      cuerpo,
      cuando,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'canal_principal',
          'Notificaciones',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
    );
  }

  Future<void> cancelar(int id) async => _plugin.cancel(id);

  Future<void> cancelarTodas() async => _plugin.cancelAll();
}
```

#### `lib/providers/notificaciones_provider.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/notificacion_local.dart';

class NotificacionesNotifier extends Notifier<List<NotificacionLocal>> {
  @override
  List<NotificacionLocal> build() => [];

  void agregar(NotificacionLocal n) => state = [n, ...state];

  void marcarLeida(int id) {
    state = [
      for (final n in state)
        if (n.id == id) n.copyWith(leida: true) else n,
    ];
  }

  void eliminar(int id) {
    state = state.where((n) => n.id != id).toList();
  }

  void limpiar() => state = [];
}

final notificacionesProvider =
    NotifierProvider<NotificacionesNotifier, List<NotificacionLocal>>(
  NotificacionesNotifier.new,
);

// Provider derivado para contar no leídas
final noLeidasProvider = Provider<int>((ref) {
  return ref.watch(notificacionesProvider).where((n) => !n.leida).length;
});
```

#### `lib/screens/pantalla_locales.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/notificacion_local.dart';
import '../providers/notificaciones_provider.dart';
import '../services/servicio_notificaciones.dart';

class PantallaLocales extends ConsumerStatefulWidget {
  const PantallaLocales({super.key});

  @override
  ConsumerState<PantallaLocales> createState() => _PantallaLocalesState();
}

class _PantallaLocalesState extends ConsumerState<PantallaLocales> {
  int _contadorId = 100;
  final ServicioNotificaciones _servicio = ServicioNotificaciones.instance;

  Future<void> _enviarAhora() async {
    final id = _contadorId++;
    final n = NotificacionLocal(
      id: id,
      titulo: 'Notificación local #$id',
      cuerpo: 'Enviada a las ${TimeOfDay.now().format(context)}',
      fechaHora: DateTime.now(),
      tipo: TipoNotificacion.local,
    );
    ref.read(notificacionesProvider.notifier).agregar(n);
    await _servicio.mostrar(id: n.id, titulo: n.titulo, cuerpo: n.cuerpo);
  }

  Future<void> _programar(Duration demora) async {
    final id = _contadorId++;
    final segundos = demora.inSeconds;
    final n = NotificacionLocal(
      id: id,
      titulo: 'Recordatorio #$id',
      cuerpo: 'Programado para $segundos s desde ahora',
      fechaHora: DateTime.now().add(demora),
      tipo: TipoNotificacion.local,
    );
    ref.read(notificacionesProvider.notifier).agregar(n);
    await _servicio.programar(
        id: n.id, titulo: n.titulo, cuerpo: n.cuerpo, demora: demora);
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Llega en $segundos segundos (id: $id)')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    final locales = ref.watch(notificacionesProvider)
        .where((n) => n.tipo == TipoNotificacion.local)
        .toList();

    return Scaffold(
      appBar: AppBar(
        title: const Text('Notificaciones Locales'),
        actions: [
          IconButton(
            icon: const Icon(Icons.delete_sweep),
            tooltip: 'Limpiar historial',
            onPressed: () =>
                ref.read(notificacionesProvider.notifier).limpiar(),
          ),
        ],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Wrap(
              spacing: 8,
              runSpacing: 8,
              children: [
                ElevatedButton.icon(
                  onPressed: _enviarAhora,
                  icon: const Icon(Icons.send),
                  label: const Text('Ahora'),
                ),
                ElevatedButton.icon(
                  onPressed: () => _programar(const Duration(seconds: 5)),
                  icon: const Icon(Icons.timer),
                  label: const Text('En 5 s'),
                ),
                ElevatedButton.icon(
                  onPressed: () => _programar(const Duration(seconds: 30)),
                  icon: const Icon(Icons.timer_outlined),
                  label: const Text('En 30 s'),
                ),
                OutlinedButton.icon(
                  onPressed: () => _servicio.cancelarTodas(),
                  icon: const Icon(Icons.cancel_outlined),
                  label: const Text('Cancelar todas'),
                ),
              ],
            ),
          ),
          const Divider(),
          Expanded(
            child: locales.isEmpty
                ? const Center(
                    child: Text('Sin notificaciones locales',
                        style: TextStyle(color: Colors.grey)))
                : ListView.builder(
                    itemCount: locales.length,
                    itemBuilder: (context, i) {
                      final n = locales[i];
                      return Dismissible(
                        key: Key('local_${n.id}'),
                        direction: DismissDirection.endToStart,
                        background: Container(
                          alignment: Alignment.centerRight,
                          color: Colors.red,
                          padding: const EdgeInsets.only(right: 16),
                          child: const Icon(Icons.delete,
                              color: Colors.white),
                        ),
                        onDismissed: (_) => ref
                            .read(notificacionesProvider.notifier)
                            .eliminar(n.id),
                        child: ListTile(
                          leading: const Icon(Icons.notifications,
                              color: Colors.indigo),
                          title: Text(n.titulo),
                          subtitle: Text(n.cuerpo),
                          trailing: Text(
                            '${n.fechaHora.hour}:${n.fechaHora.minute.toString().padLeft(2, '0')}',
                            style: const TextStyle(fontSize: 11),
                          ),
                        ),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}
```

#### `lib/screens/pantalla_push.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import '../models/notificacion_local.dart';
import '../providers/notificaciones_provider.dart';

class PantallaPush extends ConsumerStatefulWidget {
  const PantallaPush({super.key});

  @override
  ConsumerState<PantallaPush> createState() => _PantallaPushState();
}

class _PantallaPushState extends ConsumerState<PantallaPush> {
  String _token = 'Obteniendo token...';

  @override
  void initState() {
    super.initState();
    _obtenerToken();
  }

  Future<void> _obtenerToken() async {
    try {
      final token = await FirebaseMessaging.instance.getToken();
      if (mounted) setState(() => _token = token ?? 'No disponible');
    } catch (e) {
      if (mounted) setState(() => _token = 'Error: $e');
    }
  }

  @override
  Widget build(BuildContext context) {
    final pushNotifs = ref.watch(notificacionesProvider)
        .where((n) => n.tipo == TipoNotificacion.push)
        .toList();
    final noLeidas =
        pushNotifs.where((n) => !n.leida).length;

    return Scaffold(
      appBar: AppBar(
        title: Text('Push FCM'
            '${noLeidas > 0 ? ' ($noLeidas sin leer)' : ''}'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: _obtenerToken,
            tooltip: 'Refrescar token',
          ),
          IconButton(
            icon: const Icon(Icons.delete_sweep),
            onPressed: () =>
                ref.read(notificacionesProvider.notifier).limpiar(),
          ),
        ],
      ),
      body: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Token FCM
          Container(
            width: double.infinity,
            margin: const EdgeInsets.all(16),
            padding: const EdgeInsets.all(12),
            decoration: BoxDecoration(
              color: Theme.of(context).colorScheme.surfaceContainerHighest,
              borderRadius: BorderRadius.circular(8),
            ),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  children: [
                    const Icon(Icons.key, size: 16),
                    const SizedBox(width: 6),
                    const Text('Token FCM (selecciona para copiar):',
                        style: TextStyle(fontWeight: FontWeight.bold,
                            fontSize: 12)),
                  ],
                ),
                const SizedBox(height: 8),
                SelectableText(
                  _token,
                  style: const TextStyle(
                      fontSize: 10, fontFamily: 'monospace'),
                ),
              ],
            ),
          ),
          const Divider(),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
            child: Text(
              'Mensajes recibidos (${pushNotifs.length})',
              style: const TextStyle(
                  fontWeight: FontWeight.bold, fontSize: 13),
            ),
          ),
          Expanded(
            child: pushNotifs.isEmpty
                ? Center(
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      children: const [
                        Icon(Icons.cloud_off, size: 48, color: Colors.grey),
                        SizedBox(height: 8),
                        Text(
                          'Envía un push desde Firebase Console\ncopiando el token de arriba',
                          textAlign: TextAlign.center,
                          style: TextStyle(color: Colors.grey),
                        ),
                      ],
                    ),
                  )
                : ListView.builder(
                    itemCount: pushNotifs.length,
                    itemBuilder: (context, i) {
                      final n = pushNotifs[i];
                      return ListTile(
                        leading: Icon(
                          n.leida
                              ? Icons.mark_email_read
                              : Icons.mark_email_unread,
                          color:
                              n.leida ? Colors.grey : Colors.deepPurple,
                        ),
                        title: Text(n.titulo,
                            style: TextStyle(
                                fontWeight: n.leida
                                    ? FontWeight.normal
                                    : FontWeight.bold)),
                        subtitle: Text(n.cuerpo),
                        trailing: Text(
                          '${n.fechaHora.hour}:${n.fechaHora.minute.toString().padLeft(2, '0')}',
                          style: const TextStyle(fontSize: 11),
                        ),
                        onTap: () => ref
                            .read(notificacionesProvider.notifier)
                            .marcarLeida(n.id),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}
```

#### `lib/main.dart` (proyecto final)

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:timezone/data/latest_all.dart' as tz;
import 'models/notificacion_local.dart';
import 'providers/notificaciones_provider.dart';
import 'services/servicio_notificaciones.dart';
import 'screens/pantalla_locales.dart';
import 'screens/pantalla_push.dart';

// Handler de background — función de nivel superior (obligatorio)
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  // En background/terminated FCM muestra la notificación automáticamente
  // si el mensaje incluye el campo "notification"
}

Future<void> main() async {
  // 1. Inicializar bindings de Flutter
  WidgetsFlutterBinding.ensureInitialized();

  // 2. Inicializar zonas horarias (requerido por zonedSchedule)
  tz.initializeTimeZones();

  // 3. Inicializar Firebase
  await Firebase.initializeApp();

  // 4. Registrar handler de background ANTES de runApp
  FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);

  // 5. Inicializar servicio de notificaciones locales
  await ServicioNotificaciones.instance.init();

  // 6. Solicitar permiso FCM en iOS
  await FirebaseMessaging.instance.requestPermission(
    alert: true,
    badge: true,
    sound: true,
  );

  // 7. Lanzar app con ProviderScope
  runApp(const ProviderScope(child: NotificacionesApp()));
}

final _tabProvider = StateProvider<int>((ref) => 0);

class NotificacionesApp extends StatelessWidget {
  const NotificacionesApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Notificaciones — Módulo 16',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const PantallaRaiz(),
    );
  }
}

class PantallaRaiz extends ConsumerStatefulWidget {
  const PantallaRaiz({super.key});

  @override
  ConsumerState<PantallaRaiz> createState() => _PantallaRaizState();
}

class _PantallaRaizState extends ConsumerState<PantallaRaiz> {
  final ServicioNotificaciones _servicio = ServicioNotificaciones.instance;

  @override
  void initState() {
    super.initState();
    _configurarFCMForeground();
    _configurarAperturaDesdeNotificacion();
  }

  void _configurarFCMForeground() {
    // Mensaje cuando app está en PRIMER PLANO
    FirebaseMessaging.onMessage.listen((RemoteMessage message) {
      final notif = message.notification;
      if (notif != null) {
        final n = NotificacionLocal(
          id: message.hashCode,
          titulo: notif.title ?? 'Nuevo mensaje',
          cuerpo: notif.body ?? '',
          fechaHora: DateTime.now(),
          tipo: TipoNotificacion.push,
        );
        ref.read(notificacionesProvider.notifier).agregar(n);
        _servicio.mostrar(id: n.id, titulo: n.titulo, cuerpo: n.cuerpo);
      }
    });
  }

  void _configurarAperturaDesdeNotificacion() {
    // Usuario toca la notificación cuando app estaba en background
    FirebaseMessaging.onMessageOpenedApp.listen((RemoteMessage message) {
      // Navegar al tab de Push cuando se abre desde una notificación
      ref.read(_tabProvider.notifier).state = 1;
    });
  }

  @override
  Widget build(BuildContext context) {
    final tabActual = ref.watch(_tabProvider);
    final noLeidas = ref.watch(noLeidasProvider);
    const tabs = [
      PantallaLocales(),
      PantallaPush(),
    ];
    return Scaffold(
      body: tabs[tabActual],
      bottomNavigationBar: NavigationBar(
        selectedIndex: tabActual,
        onDestinationSelected: (i) =>
            ref.read(_tabProvider.notifier).state = i,
        destinations: [
          const NavigationDestination(
            icon: Icon(Icons.notifications_outlined),
            selectedIcon: Icon(Icons.notifications),
            label: 'Locales',
          ),
          NavigationDestination(
            icon: Badge(
              isLabelVisible: noLeidas > 0,
              label: Text('$noLeidas'),
              child: const Icon(Icons.cloud_outlined),
            ),
            selectedIcon: Badge(
              isLabelVisible: noLeidas > 0,
              label: Text('$noLeidas'),
              child: const Icon(Icons.cloud),
            ),
            label: 'Push FCM',
          ),
        ],
      ),
    );
  }
}
```

---

## Guía rápida de imports

```dart
// Plugin de notificaciones locales
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

// Zonas horarias (transitive dep — no declarar en pubspec)
import 'package:timezone/timezone.dart' as tz;
import 'package:timezone/data/latest_all.dart' as tz;  // alias tz_data en algunos pasos

// Firebase
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';

// Riverpod
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Flutter
import 'package:flutter/material.dart';
```

---

## Cuándo usar qué

```
Situacion                                     | Solucion
----------------------------------------------|--------------------------------------------------
Sin internet, aviso inmediato                 | show()
Sin internet, aviso en N segundos/minutos     | zonedSchedule() con TZDateTime.now().add(duration)
Cancelar notificacion pendiente por id        | cancel(id)
Cancelar todas las notificaciones pendientes  | cancelAll()
Mensaje desde servidor a usuario con app open | onMessage.listen() → show()
Mensaje cuando app cerrada o background       | FCM background handler + campo "notification"
App abierta tocando notificacion push         | onMessageOpenedApp.listen()
Mostrar badge en tab con mensajes sin leer    | Badge widget + provider derivado noLeidasProvider
Canal con sonido personalizado Android        | AndroidNotificationChannel(sound: RawResourceAndroidNotificationSound('nombre'))
Grupo de notificaciones Android               | AndroidNotificationDetails(groupKey, setAsGroupSummary: true)
Mantener historial de notificaciones          | NotificacionesNotifier extends Notifier<List<NotificacionLocal>>
Marcar mensajes como leidos                   | copyWith(leida: true) + state actualizado
```

---

## Ejercicios propuestos

1. **Sonido personalizado:** Crea un canal `'canal_alarmas'` con `importance: Importance.max` y añade un botón "Alarma urgente" que use ese canal. Verifica que en Android el sonido es diferente al canal normal.

2. **Notificaciones recurrentes:** Usa `periodicallyShow()` de `flutter_local_notifications` para mostrar un recordatorio de hidratacion cada hora. Añade un switch en la UI para activar/desactivar el recordatorio.

3. **Deep link desde push:** En `onMessageOpenedApp`, lee `message.data['pantalla']`. Si el valor es `'push'`, navega al tab 1. Si es `'locales'`, navega al tab 0. Envía dos mensajes desde Firebase Console con distintos valores y verifica la navegación.

4. **Persistencia del historial:** Usa `shared_preferences` para serializar y deserializar la lista de `NotificacionLocal`. Al relanzar la app, el historial debe seguir visible. Añade un campo `toJson()` / `fromJson()` al modelo.

---

## Resumen

- `FlutterLocalNotificationsPlugin` requiere un `AndroidNotificationChannel` creado antes de mostrar notificaciones; en iOS los permisos se solicitan en `DarwinInitializationSettings`.
- `tz.initializeTimeZones()` debe llamarse antes de cualquier `zonedSchedule()`; `timezone` es una dependencia transitiva de `flutter_local_notifications`.
- Para Android 13+ el permiso `POST_NOTIFICATIONS` se solicita en runtime con `requestNotificationsPermission()`.
- El handler de background de FCM (`onBackgroundMessage`) debe ser una función de nivel superior anotada con `@pragma('vm:entry-point')`, no un método de clase.
- El orden de inicializacion en `main()` es fijo: `ensureInitialized` → `initializeTimeZones` → `Firebase.initializeApp()` → `onBackgroundMessage` → `ServicioNotificaciones.init()` → `runApp`.
- `onMessage` maneja mensajes FCM cuando la app está en primer plano; `onMessageOpenedApp` detecta la apertura desde una notificacion en background; el handler de background cubre el estado terminado.
- `NotifierProvider` con `copyWith` garantiza inmutabilidad del estado: nunca modificar la lista directamente, siempre producir una nueva.
- El widget `Badge` de Material 3 permite mostrar contadores de no leídos directamente en las `NavigationDestination` sin librerías adicionales.

> **Siguiente página →** Página 17: Bluetooth y comunicación en tiempo real con WebSockets.
