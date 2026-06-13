# Tutorial Flutter — Página 16
## Módulo 5 · Conectividad
### Notificaciones locales y push con FCM — construcción incremental

---

## Crear el proyecto

```bash
flutter create notificaciones_app
cd notificaciones_app
```

### `pubspec.yaml`

```yaml
name: notificaciones_app
description: "Demo de notificaciones locales y push FCM"
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

---

## Configuración nativa

### Android — `android/app/src/main/AndroidManifest.xml`

Añadir dentro de `<manifest>`, antes de `<application>`:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
<uses-permission android:name="android.permission.VIBRATE"/>
```

Añadir dentro de `<application>`:

```xml
<!-- Receptor para acciones de notificación -->
<receiver
    android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver"
    android:exported="false"/>
<receiver
    android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
    </intent-filter>
</receiver>
```

### iOS — `ios/Runner/Info.plist`

Añadir dentro de `<dict>`:

```xml
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
    <string>remote-notification</string>
</array>
```

### iOS — `ios/Runner/AppDelegate.swift`

```swift
import UIKit
import Flutter
import flutter_local_notifications   // añadir este import

@main
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    // Necesario para notificaciones locales en iOS
    FlutterLocalNotificationsPlugin.setPluginRegistrantCallback { registry in
      GeneratedPluginRegistrant.register(with: registry)
    }
    if #available(iOS 10.0, *) {
      UNUserNotificationCenter.current().delegate = self as UNUserNotificationCenterDelegate
    }
    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

### Firebase — configuración (para la parte FCM)

```
1. Ir a https://console.firebase.google.com
2. Crear proyecto → Agregar app Android con tu package name
3. Descargar google-services.json → copiar a android/app/
4. Crear app iOS con tu bundle id
5. Descargar GoogleService-Info.plist → copiar a ios/Runner/
```

**`android/build.gradle`** — añadir en `plugins`:
```groovy
id 'com.google.gms.google-services' version '4.4.2' apply false
```

**`android/app/build.gradle`** — añadir en `plugins`:
```groovy
id 'com.google.gms.google-services'
```

---

## Estructura final del proyecto

```
notificaciones_app/
├── android/
│   └── app/
│       ├── build.gradle               ← google-services plugin
│       ├── google-services.json       ← descargado de Firebase
│       └── src/main/AndroidManifest.xml
├── ios/
│   └── Runner/
│       ├── AppDelegate.swift
│       ├── Info.plist
│       └── GoogleService-Info.plist   ← descargado de Firebase
├── lib/
│   ├── main.dart
│   ├── core/
│   │   ├── notificaciones_locales.dart   ← servicio de notificaciones locales
│   │   └── fcm_service.dart              ← servicio FCM
│   └── features/
│       ├── pantalla_locales.dart         ← UI notificaciones locales
│       └── pantalla_push.dart            ← UI notificaciones push
└── pubspec.yaml
```

---

## Paso 1 — App mínima funcionando

**`lib/main.dart`** — reemplaza todo:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const ProviderScope(child: NotificacionesApp()));
}

class NotificacionesApp extends StatelessWidget {
  const NotificacionesApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Notificaciones App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const PantallaRaiz(),
    );
  }
}

class PantallaRaiz extends StatelessWidget {
  const PantallaRaiz({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Notificaciones App')),
      body: const Center(child: Text('Paso 1 — App funcionando ✅')),
    );
  }
}
```

```bash
flutter run
# Debe mostrar "Paso 1 — App funcionando ✅"
```

---

## Paso 2 — Notificaciones locales: inicialización

Las notificaciones locales se generan **desde la propia app**,
sin internet ni servidor. Primero inicializa el servicio y verifica
que el permiso se solicita correctamente:

**Crea `lib/core/notificaciones_locales.dart`:**

```dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

/// Servicio singleton de notificaciones locales.
/// Debe inicializarse en main() antes de runApp().
class NotificacionesLocales {
  NotificacionesLocales._();
  static final instance = NotificacionesLocales._();

  final _plugin = FlutterLocalNotificationsPlugin();

  // IDs de canales — cada canal tiene su propio sonido/vibración/importancia
  static const _canalGeneral     = 'canal_general';
  static const _canalAlertas     = 'canal_alertas';
  static const _canalDescargas   = 'canal_descargas';

  /// Inicializar el plugin y crear los canales (Android 8+).
  /// Llamar en main() antes de runApp().
  Future<void> inicializar() async {
    // Configuración por plataforma
    const android = AndroidInitializationSettings('@mipmap/ic_launcher');
    const ios     = DarwinInitializationSettings(
      requestAlertPermission:  true,
      requestBadgePermission:  true,
      requestSoundPermission:  true,
    );

    await _plugin.initialize(
      const InitializationSettings(android: android, iOS: ios),
      // Callback cuando el usuario toca la notificación
      onDidReceiveNotificationResponse: _onTap,
      // Callback cuando toca la notificación con la app cerrada (background)
      onDidReceiveBackgroundNotificationResponse: _onTapBackground,
    );

    // Crear canales de Android 8+
    await _crearCanales();
  }

  Future<void> _crearCanales() async {
    final androidPlugin = _plugin
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>();

    await androidPlugin?.createNotificationChannel(
      const AndroidNotificationChannel(
        _canalGeneral,
        'General',
        description:  'Notificaciones generales de la app',
        importance:   Importance.defaultImportance,
      ),
    );

    await androidPlugin?.createNotificationChannel(
      const AndroidNotificationChannel(
        _canalAlertas,
        'Alertas',
        description: 'Alertas urgentes del sistema',
        importance:  Importance.high,
        enableVibration: true,
      ),
    );

    await androidPlugin?.createNotificationChannel(
      const AndroidNotificationChannel(
        _canalDescargas,
        'Descargas',
        description: 'Progreso de descargas',
        importance:  Importance.low,
        playSound:   false,
      ),
    );
  }

  /// Solicitar permiso explícito en Android 13+ e iOS
  Future<bool> solicitarPermiso() async {
    final androidPlugin = _plugin
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>();
    final iosPlugin = _plugin
        .resolvePlatformSpecificImplementation<
            IOSFlutterLocalNotificationsPlugin>();

    final androidOk = await androidPlugin?.requestNotificationsPermission();
    final iosOk     = await iosPlugin?.requestPermissions(
      alert: true, badge: true, sound: true,
    );

    return (androidOk ?? true) || (iosOk ?? true);
  }

  void _onTap(NotificationResponse response) {
    print('Notificación tocada: ${response.payload}');
    // Aquí navegarías a la pantalla correspondiente
  }
}

// Función top-level — requerida para el callback en background
@pragma('vm:entry-point')
void _onTapBackground(NotificationResponse response) {
  print('Notificación tocada en background: ${response.payload}');
}
```

Actualiza `main.dart` para inicializar el servicio:

```dart
// lib/main.dart — actualizar main()

import 'core/notificaciones_locales.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Inicializar notificaciones locales antes de runApp
  await NotificacionesLocales.instance.inicializar();

  runApp(const ProviderScope(child: NotificacionesApp()));
}
```

Agrega un botón para verificar el permiso en `PantallaRaiz`:

```dart
// lib/main.dart — reemplaza PantallaRaiz

class PantallaRaiz extends StatefulWidget {
  const PantallaRaiz({super.key});

  @override
  State<PantallaRaiz> createState() => _PantallaRaizState();
}

class _PantallaRaizState extends State<PantallaRaiz> {
  String _estado = 'Sin verificar';

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 2 — Inicialización')),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text(_estado, style: const TextStyle(fontSize: 18)),
            const SizedBox(height: 24),
            FilledButton.icon(
              icon:  const Icon(Icons.notifications),
              label: const Text('Solicitar permiso'),
              onPressed: () async {
                final ok = await NotificacionesLocales.instance
                    .solicitarPermiso();
                setState(() => _estado = ok
                    ? '✅ Permiso concedido'
                    : '❌ Permiso denegado');
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

```bash
flutter run
# Presiona "Solicitar permiso"
# En Android 13+ aparece el diálogo del sistema
# En iOS aparece el diálogo de notificaciones
# El texto cambia según la respuesta
```

---

## Paso 3 — Enviar notificaciones simples

Ahora añade los métodos de envío al servicio y pruébalos:

**Actualiza `lib/core/notificaciones_locales.dart`** —
añade estos métodos a la clase `NotificacionesLocales`:

```dart
  // ── Notificación simple ───────────────────────────────────────

  Future<void> simple({
    required int    id,
    required String titulo,
    required String cuerpo,
    String?         payload,     // datos que recibes al tocar
    String          canal = _canalGeneral,
  }) async {
    await _plugin.show(
      id,
      titulo,
      cuerpo,
      NotificationDetails(
        android: AndroidNotificationDetails(
          canal,
          canal,
          importance:  Importance.defaultImportance,
          priority:    Priority.defaultPriority,
          styleInformation: const BigTextStyleInformation(''),
        ),
        iOS: const DarwinNotificationDetails(
          presentAlert: true,
          presentSound: true,
          presentBadge: true,
        ),
      ),
      payload: payload,
    );
  }

  // ── Notificación con barra de progreso ────────────────────────

  Future<void> conProgreso({
    required int    id,
    required String titulo,
    required int    progreso,    // 0..100
    bool            completa = false,
  }) async {
    await _plugin.show(
      id,
      titulo,
      completa ? 'Completado' : '$progreso%',
      NotificationDetails(
        android: AndroidNotificationDetails(
          _canalDescargas,
          'Descargas',
          importance:       Importance.low,
          priority:         Priority.low,
          showProgress:     true,
          maxProgress:      100,
          progress:         progreso,
          indeterminate:    false,
          onlyAlertOnce:    true,   // no suena en cada update
          ongoing:          !completa,
          autoCancel:       completa,
        ),
      ),
    );
  }

  // ── Cancelar notificaciones ───────────────────────────────────

  Future<void> cancelar(int id)   => _plugin.cancel(id);
  Future<void> cancelarTodas()    => _plugin.cancelAll();
```

Prueba los métodos desde `PantallaRaiz`:

```dart
// lib/main.dart — reemplaza PantallaRaiz

import 'dart:async';
import 'core/notificaciones_locales.dart';

class PantallaRaiz extends StatefulWidget {
  const PantallaRaiz({super.key});

  @override
  State<PantallaRaiz> createState() => _PantallaRaizState();
}

class _PantallaRaizState extends State<PantallaRaiz> {
  int _progreso = 0;
  Timer? _timer;

  @override
  void dispose() {
    _timer?.cancel();
    super.dispose();
  }

  void _simularDescarga() {
    _progreso = 0;
    _timer?.cancel();
    _timer = Timer.periodic(const Duration(milliseconds: 400), (t) async {
      _progreso += 10;
      if (_progreso >= 100) {
        t.cancel();
        await NotificacionesLocales.instance.conProgreso(
          id: 99, titulo: 'Descarga completa', progreso: 100,
          completa: true,
        );
      } else {
        await NotificacionesLocales.instance.conProgreso(
          id: 99, titulo: 'Descargando archivo...', progreso: _progreso,
        );
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final svc = NotificacionesLocales.instance;
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 3 — Notificaciones')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            FilledButton.icon(
              icon:  const Icon(Icons.notifications),
              label: const Text('Notificación simple'),
              onPressed: () => svc.simple(
                id:     1,
                titulo: 'Hola desde Flutter',
                cuerpo: 'Esta es una notificación local simple',
              ),
            ),
            const SizedBox(height: 8),
            FilledButton.icon(
              icon:  const Icon(Icons.warning_amber),
              label: const Text('Notificación de alerta'),
              onPressed: () => svc.simple(
                id:     2,
                titulo: '⚠️ Alerta del sistema',
                cuerpo: 'CPU al 98% — revisa el servidor prod-01',
                canal:  'canal_alertas',
              ),
            ),
            const SizedBox(height: 8),
            FilledButton.tonal(
              onPressed: _simularDescarga,
              child: const Text('Simular descarga (con progreso)'),
            ),
            const SizedBox(height: 8),
            OutlinedButton.icon(
              icon:  const Icon(Icons.delete_outline),
              label: const Text('Cancelar todas'),
              onPressed: () => svc.cancelarTodas(),
            ),
          ],
        ),
      ),
    );
  }
}
```

```bash
flutter run
# Presiona "Notificación simple" → aparece en la barra del sistema
# Presiona "Alerta" → aparece con más prominencia (canal_alertas)
# Presiona "Simular descarga" → barra de progreso que avanza en tiempo real
# Presiona "Cancelar todas" → desaparecen todas las notificaciones
```

---

## Paso 4 — Notificaciones con acciones

Las acciones permiten al usuario responder desde la notificación
sin abrir la app:

**Actualiza `lib/core/notificaciones_locales.dart`** —
añade el método `conAcciones`:

```dart
  // ── Notificación con botones de acción ────────────────────────

  Future<void> conAcciones({
    required int    id,
    required String titulo,
    required String cuerpo,
    required List<({String id, String label, IconData? icono})> acciones,
    String?         payload,
  }) async {
    final androidAcciones = acciones.map((a) =>
      AndroidNotificationAction(
        a.id,
        a.label,
        showsUserInterface: false,   // true = abre la app al presionar
        cancelNotification: true,
      ),
    ).toList();

    await _plugin.show(
      id,
      titulo,
      cuerpo,
      NotificationDetails(
        android: AndroidNotificationDetails(
          _canalAlertas,
          'Alertas',
          importance:  Importance.high,
          priority:    Priority.high,
          actions:     androidAcciones,
        ),
        iOS: DarwinNotificationDetails(
          categoryIdentifier: 'acciones_$id',
        ),
      ),
      payload: payload,
    );
  }
```

Prueba en `PantallaRaiz` — añade este botón al `Column` del paso anterior:

```dart
// Añadir al Column de PantallaRaiz en main.dart

const SizedBox(height: 8),
FilledButton.icon(
  icon:  const Icon(Icons.task_alt),
  label: const Text('Notificación con acciones'),
  onPressed: () => svc.conAcciones(
    id:      3,
    titulo:  'Nuevo ticket de soporte',
    cuerpo:  'Servidor prod-01 reporta latencia > 500ms',
    payload: 'ticket:1234',
    acciones: [
      (id: 'RESOLVER',  label: 'Resolver', icono: null),
      (id: 'IGNORAR',   label: 'Ignorar',  icono: null),
    ],
  ),
),
```

```bash
flutter run
# Presiona "Notificación con acciones"
# En la barra de notificaciones expande la notificación
# Verás los botones "Resolver" e "Ignorar"
# Al presionar, la notificación se descarta (cancelNotification: true)
```

---

## Paso 5 — Notificaciones programadas

```dart
// Añadir a lib/core/notificaciones_locales.dart
// Agregar import al inicio del archivo:
// import 'package:timezone/timezone.dart' as tz;
// import 'package:timezone/data/latest.dart' as tz_data;

  // ── Notificación programada ───────────────────────────────────
  // Requiere el paquete timezone — añadir a pubspec.yaml:
  // timezone: ^0.9.4

  Future<void> programada({
    required int      id,
    required String   titulo,
    required String   cuerpo,
    required Duration en,         // cuánto tiempo desde ahora
    String?           payload,
  }) async {
    final ahora    = tz.TZDateTime.now(tz.local);
    final momento  = ahora.add(en);

    await _plugin.zonedSchedule(
      id,
      titulo,
      cuerpo,
      momento,
      NotificationDetails(
        android: AndroidNotificationDetails(
          _canalGeneral, 'General',
          importance: Importance.defaultImportance,
        ),
        iOS: const DarwinNotificationDetails(),
      ),
      androidScheduleMode:
          AndroidScheduleMode.exactAllowWhileIdle,
      payload: payload,
    );
  }

  Future<void> cancelarProgramada(int id) => _plugin.cancel(id);
```

Para usar `timezone`, añade en `pubspec.yaml`:
```yaml
  timezone: ^0.9.4
```

Y en `main()` inicializa los datos de zona horaria:
```dart
// lib/main.dart — actualizar main()
import 'package:timezone/data/latest.dart' as tz_data;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  tz_data.initializeTimeZones();   // necesario para zonedSchedule
  await NotificacionesLocales.instance.inicializar();
  runApp(const ProviderScope(child: NotificacionesApp()));
}
```

Prueba añadiendo este botón:

```dart
// En el Column de PantallaRaiz

const SizedBox(height: 8),
FilledButton.tonal(
  onPressed: () {
    svc.programada(
      id:     10,
      titulo: '⏰ Recordatorio',
      cuerpo: 'Han pasado 5 segundos',
      en:     const Duration(seconds: 5),
    );
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(
        content: Text('Notificación programada en 5 segundos'),
        behavior: SnackBarBehavior.floating,
      ),
    );
  },
  child: const Text('Programar en 5 segundos'),
),
```

```bash
flutter run
# Presiona "Programar en 5 segundos"
# El Snackbar confirma que quedó programada
# Minimiza la app o bloquea la pantalla
# En 5 segundos aparece la notificación
```

---

## Paso 6 — Push con Firebase Cloud Messaging (FCM)

Las notificaciones push llegan **desde un servidor remoto** a través
de Firebase. Requieren la configuración de Firebase del inicio de la página.

**Crea `lib/core/fcm_service.dart`:**

```dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'notificaciones_locales.dart';

/// Servicio FCM — recibe mensajes push y los muestra como notificaciones locales.
/// Debe inicializarse en main() después de Firebase.initializeApp().
class FcmService {
  FcmService._();
  static final instance = FcmService._();

  final _messaging = FirebaseMessaging.instance;

  /// Inicializar FCM y configurar los handlers de mensajes.
  Future<void> inicializar() async {
    // 1. Solicitar permiso de notificaciones (iOS — en Android ya se hizo)
    await _messaging.requestPermission(
      alert:      true,
      announcement: false,
      badge:      true,
      carPlay:    false,
      criticalAlert: false,
      provisional: false,
      sound:      true,
    );

    // 2. Configurar presentación en primer plano (iOS)
    await FirebaseMessaging.instance
        .setForegroundNotificationPresentationOptions(
      alert: true,
      badge: true,
      sound: true,
    );

    // 3. Handler para mensajes en PRIMER PLANO
    // En Android FCM NO muestra la notificación automáticamente en primer plano
    // — hay que mostrarla manualmente con flutter_local_notifications
    FirebaseMessaging.onMessage.listen(_onMensajePrimerPlano);

    // 4. Handler cuando el usuario toca la notificación con la app en FONDO
    FirebaseMessaging.onMessageOpenedApp.listen(_onAbrirDesdeBackground);

    // 5. Verificar si la app se abrió desde una notificación (app cerrada)
    final mensajeInicial = await _messaging.getInitialMessage();
    if (mensajeInicial != null) {
      _procesarMensaje(mensajeInicial, origen: 'inicial');
    }

    // 6. Obtener y registrar el token del dispositivo
    final token = await obtenerToken();
    print('🔑 Token FCM: $token');
    // En producción: enviar este token a tu backend
  }

  // Mensaje recibido con la app en PRIMER PLANO
  Future<void> _onMensajePrimerPlano(RemoteMessage mensaje) async {
    print('📨 FCM primer plano: ${mensaje.messageId}');

    final notif = mensaje.notification;
    if (notif != null) {
      // Mostrar como notificación local porque FCM no la muestra solo en primer plano
      await NotificacionesLocales.instance.simple(
        id:     mensaje.hashCode,
        titulo: notif.title ?? 'Nuevo mensaje',
        cuerpo: notif.body  ?? '',
        // Pasar los datos para el handler de tap
        payload: mensaje.data['pantalla'],
      );
    }
    _procesarMensaje(mensaje, origen: 'primer_plano');
  }

  // App en fondo — usuario toca la notificación
  void _onAbrirDesdeBackground(RemoteMessage mensaje) {
    print('📨 FCM abierto desde fondo: ${mensaje.messageId}');
    _procesarMensaje(mensaje, origen: 'background');
  }

  // Lógica compartida para procesar cualquier mensaje
  void _procesarMensaje(RemoteMessage mensaje, {required String origen}) {
    final data     = mensaje.data;
    final pantalla = data['pantalla'] as String?;
    final payload  = data['payload']  as String?;

    print('Origen: $origen | Pantalla: $pantalla | Payload: $payload');

    // Aquí navegarías según la pantalla usando GoRouter o Navigator:
    // if (pantalla == 'alertas') context.go('/alertas');
  }

  /// Obtener el token FCM del dispositivo.
  Future<String?> obtenerToken() => _messaging.getToken();

  /// Suscribirse a un topic — todos los dispositivos suscritos
  /// reciben los mensajes enviados a ese topic
  Future<void> suscribirA(String topic) async {
    await _messaging.subscribeToTopic(topic);
    print('✅ Suscrito a topic: $topic');
  }

  Future<void> desuscribirDe(String topic) async {
    await _messaging.unsubscribeFromTopic(topic);
    print('🚫 Desuscrito de topic: $topic');
  }
}

/// Handler de mensajes en SEGUNDO PLANO — función top-level obligatoria.
/// Se ejecuta en un isolate separado — no puede acceder a estado de la app.
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage mensaje) async {
  // No necesitas mostrar nada aquí — FCM lo hace automáticamente en segundo plano
  print('📨 FCM background handler: ${mensaje.messageId}');
}
```

Actualiza `main.dart` para inicializar Firebase y FCM:

```dart
// lib/main.dart — versión completa con Firebase

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:timezone/data/latest.dart' as tz_data;
import 'core/notificaciones_locales.dart';
import 'core/fcm_service.dart';
import 'features/pantalla_locales.dart';
import 'features/pantalla_push.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Timezone para notificaciones programadas
  tz_data.initializeTimeZones();

  // Firebase debe inicializarse antes de FCM
  await Firebase.initializeApp();

  // Registrar handler de background ANTES de runApp
  FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);

  // Notificaciones locales
  await NotificacionesLocales.instance.inicializar();

  // FCM
  await FcmService.instance.inicializar();

  runApp(const ProviderScope(child: NotificacionesApp()));
}

// La función de background debe ser top-level (fuera de cualquier clase)
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage mensaje) async {
  await Firebase.initializeApp();
  print('Background FCM: ${mensaje.messageId}');
}

class NotificacionesApp extends StatelessWidget {
  const NotificacionesApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Notificaciones',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const PantallaRaiz(),
    );
  }
}

class PantallaRaiz extends ConsumerWidget {
  const PantallaRaiz({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final indice = ref.watch(_tabProvider);
    return Scaffold(
      body: switch (indice) {
        0 => const PantallaLocales(),
        1 => const PantallaPush(),
        _ => const PantallaLocales(),
      },
      bottomNavigationBar: NavigationBar(
        selectedIndex:         indice,
        onDestinationSelected: (i) =>
            ref.read(_tabProvider.notifier).state = i,
        destinations: const [
          NavigationDestination(
            icon:         Icon(Icons.notifications_outlined),
            selectedIcon: Icon(Icons.notifications),
            label:        'Locales',
          ),
          NavigationDestination(
            icon:         Icon(Icons.cloud_outlined),
            selectedIcon: Icon(Icons.cloud),
            label:        'Push',
          ),
        ],
      ),
    );
  }
}

final _tabProvider = StateProvider<int>((ref) => 0);
```

---

## Paso 7 — Pantallas finales

**Crea `lib/features/pantalla_locales.dart`:**

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import '../core/notificaciones_locales.dart';

class PantallaLocales extends StatefulWidget {
  const PantallaLocales({super.key});

  @override
  State<PantallaLocales> createState() => _PantallaLocalesState();
}

class _PantallaLocalesState extends State<PantallaLocales> {
  final _svc      = NotificacionesLocales.instance;
  int   _progreso = 0;
  Timer? _timer;

  @override
  void dispose() {
    _timer?.cancel();
    super.dispose();
  }

  void _simularDescarga() {
    _progreso = 0;
    _timer?.cancel();
    _timer = Timer.periodic(const Duration(milliseconds: 300), (t) async {
      _progreso += 10;
      if (_progreso >= 100) {
        t.cancel();
        await _svc.conProgreso(
          id: 99, titulo: '✅ Descarga completa',
          progreso: 100, completa: true,
        );
      } else {
        await _svc.conProgreso(
          id: 99, titulo: 'Descargando...', progreso: _progreso,
        );
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Notificaciones locales'),
        backgroundColor: cs.primaryContainer,
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [

          _Seccion('Básicas'),
          _Boton(
            label:   'Notificación simple',
            icono:   Icons.notifications,
            color:   cs.primary,
            onPress: () => _svc.simple(
              id:     1,
              titulo: 'Hola desde Flutter 👋',
              cuerpo: 'Esta es una notificación local simple',
            ),
          ),
          const SizedBox(height: 8),
          _Boton(
            label:   'Notificación de alerta',
            icono:   Icons.warning_amber,
            color:   Colors.orange,
            onPress: () => _svc.simple(
              id:     2,
              titulo: '⚠️ Alerta del sistema',
              cuerpo: 'CPU al 98% en prod-01',
              canal:  'canal_alertas',
            ),
          ),

          const SizedBox(height: 16),
          _Seccion('Con interacción'),
          _Boton(
            label:   'Con botones de acción',
            icono:   Icons.task_alt,
            color:   Colors.teal,
            onPress: () => _svc.conAcciones(
              id:      3,
              titulo:  'Nuevo ticket de soporte',
              cuerpo:  'prod-01 reporta latencia > 500ms',
              payload: 'ticket:1234',
              acciones: [
                (id: 'RESOLVER', label: 'Resolver', icono: null),
                (id: 'IGNORAR',  label: 'Ignorar',  icono: null),
              ],
            ),
          ),
          const SizedBox(height: 8),
          _Boton(
            label:   'Simular descarga (progreso)',
            icono:   Icons.download,
            color:   Colors.blue,
            onPress: _simularDescarga,
          ),

          const SizedBox(height: 16),
          _Seccion('Programadas'),
          _Boton(
            label:   'En 5 segundos',
            icono:   Icons.schedule,
            color:   Colors.purple,
            onPress: () {
              _svc.programada(
                id:     10,
                titulo: '⏰ Recordatorio',
                cuerpo: 'Han pasado 5 segundos',
                en:     const Duration(seconds: 5),
              );
              ScaffoldMessenger.of(context).showSnackBar(
                const SnackBar(
                  content: Text('Programada en 5s — minimiza la app'),
                  behavior: SnackBarBehavior.floating,
                ),
              );
            },
          ),

          const SizedBox(height: 16),
          _Seccion('Gestión'),
          OutlinedButton.icon(
            onPressed: () => _svc.cancelarTodas(),
            icon:  const Icon(Icons.delete_sweep_outlined),
            label: const Text('Cancelar todas'),
          ),
        ],
      ),
    );
  }
}

class _Seccion extends StatelessWidget {
  final String texto;
  const _Seccion(this.texto);

  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.only(bottom: 8),
    child: Text(texto,
        style: Theme.of(context).textTheme.labelLarge
            ?.copyWith(color: Theme.of(context).colorScheme.primary)),
  );
}

class _Boton extends StatelessWidget {
  final String       label;
  final IconData     icono;
  final Color        color;
  final VoidCallback onPress;
  const _Boton({
    required this.label,
    required this.icono,
    required this.color,
    required this.onPress,
  });

  @override
  Widget build(BuildContext context) => FilledButton.icon(
    style:    FilledButton.styleFrom(backgroundColor: color),
    onPressed: onPress,
    icon:      Icon(icono),
    label:     Text(label),
  );
}
```

**Crea `lib/features/pantalla_push.dart`:**

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../core/fcm_service.dart';

// Provider del token FCM
final tokenProvider = FutureProvider<String?>((ref) async {
  return FcmService.instance.obtenerToken();
});

// Provider de topics suscritos
final topicsProvider = StateProvider<Set<String>>((ref) => {});

class PantallaPush extends ConsumerWidget {
  const PantallaPush({super.key});

  static const _topicsDisponibles = [
    (id: 'noticias',       nombre: 'Noticias generales'),
    (id: 'alertas_prod',   nombre: 'Alertas de producción'),
    (id: 'actualizaciones', nombre: 'Actualizaciones de la app'),
  ];

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final tokenAsync = ref.watch(tokenProvider);
    final topics     = ref.watch(topicsProvider);
    final cs         = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Notificaciones push'),
        backgroundColor: cs.primaryContainer,
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [

          // ── Token FCM ─────────────────────────────────────────
          Text('Token del dispositivo',
              style: Theme.of(context).textTheme.labelLarge
                  ?.copyWith(color: cs.primary)),
          const SizedBox(height: 8),
          Card(
            child: Padding(
              padding: const EdgeInsets.all(12),
              child: tokenAsync.when(
                loading: () => const Center(
                    child: CircularProgressIndicator()),
                error: (e, _) => Text('Error: $e',
                    style: TextStyle(color: cs.error)),
                data: (token) => Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      token ?? 'Token no disponible',
                      style: const TextStyle(
                          fontSize: 11, fontFamily: 'monospace'),
                      maxLines: 3,
                      overflow: TextOverflow.ellipsis,
                    ),
                    const SizedBox(height: 8),
                    if (token != null)
                      Align(
                        alignment: Alignment.centerRight,
                        child: OutlinedButton.icon(
                          onPressed: () {
                            Clipboard.setData(
                                ClipboardData(text: token));
                            ScaffoldMessenger.of(context)
                                .showSnackBar(
                              const SnackBar(
                                content: Text('Token copiado'),
                                behavior: SnackBarBehavior.floating,
                              ),
                            );
                          },
                          icon:  const Icon(Icons.copy, size: 16),
                          label: const Text('Copiar token'),
                        ),
                      ),
                  ],
                ),
              ),
            ),
          ),

          const SizedBox(height: 24),

          // ── Topics ────────────────────────────────────────────
          Text('Topics',
              style: Theme.of(context).textTheme.labelLarge
                  ?.copyWith(color: cs.primary)),
          const SizedBox(height: 4),
          Text(
            'Suscríbete a un topic para recibir mensajes '
            'enviados a ese grupo de dispositivos',
            style: TextStyle(color: cs.onSurfaceVariant, fontSize: 12),
          ),
          const SizedBox(height: 8),
          ..._topicsDisponibles.map((t) {
            final suscrito = topics.contains(t.id);
            return Card(
              margin: const EdgeInsets.only(bottom: 8),
              child: SwitchListTile(
                title:    Text(t.nombre),
                subtitle: Text(t.id,
                    style: TextStyle(
                        fontSize: 11, color: cs.onSurfaceVariant)),
                value:    suscrito,
                onChanged: (v) async {
                  if (v) {
                    await FcmService.instance.suscribirA(t.id);
                    ref.read(topicsProvider.notifier).update(
                        (s) => {...s, t.id});
                  } else {
                    await FcmService.instance.desuscribirDe(t.id);
                    ref.read(topicsProvider.notifier).update(
                        (s) => s.where((x) => x != t.id).toSet());
                  }
                },
              ),
            );
          }),

          const SizedBox(height: 16),

          // ── Instrucciones de prueba ───────────────────────────
          Card(
            color: cs.surfaceContainerHighest,
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Row(children: [
                    Icon(Icons.info_outline,
                        size: 16, color: cs.onSurfaceVariant),
                    const SizedBox(width: 6),
                    Text('Cómo probar el push',
                        style: TextStyle(
                            fontWeight: FontWeight.bold,
                            color:      cs.onSurfaceVariant)),
                  ]),
                  const SizedBox(height: 8),
                  Text(
                    '1. Copia el token de arriba\n'
                    '2. Ve a Firebase Console → Messaging\n'
                    '3. Crea una nueva campaña → Notification\n'
                    '4. Pega el token en "Send test message"\n'
                    '5. Envía — la notificación llegará al dispositivo',
                    style: TextStyle(
                        fontSize: 12, color: cs.onSurfaceVariant),
                  ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

```bash
flutter run
# Tab "Locales" — prueba todos los tipos de notificación
# Tab "Push" — copia el token y prueba desde Firebase Console
```

---

## Flujo completo de un mensaje push

```
                    ┌──────────────┐
                    │   Backend    │  POST a API de FCM con token o topic
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Firebase    │  Enruta al dispositivo correcto
                    └──────┬───────┘
                           │
              ┌────────────┼────────────────┐
              │            │                │
    ┌─────────▼──┐  ┌──────▼──────┐  ┌─────▼───────────┐
    │ App en     │  │ App en      │  │ App              │
    │ PRIMER     │  │ FONDO       │  │ CERRADA          │
    │ PLANO      │  │             │  │                  │
    └─────────┬──┘  └──────┬──────┘  └─────┬───────────┘
              │            │               │
    onMessage  │   FCM muestra  │   FCM muestra │
    → mostrar  │   automática   │   automática  │
    con        │   mente ✅     │   mente ✅    │
    local_notif│               │               │
              │   onMessageOpened  getInitialMessage
              │   App cuando el    App cuando el
              │   usuario toca     usuario toca
```

---

## Ejercicios propuestos

1. **Deep link desde notificación** — En `_onTap` de `NotificacionesLocales`
   y en `_procesarMensaje` de `FcmService`, lee el `payload` recibido
   (ej: `'pantalla:alertas'`). Usando un `GlobalKey<NavigatorState>` pasado
   a `MaterialApp`, navega programáticamente a la pantalla correcta
   cuando el usuario toca la notificación.

2. **Notificación de recordatorio diario** — Usa `zonedSchedule` con
   `matchDateTimeComponents: DateTimeComponents.time` para programar
   una notificación que se repita todos los días a la misma hora.
   Añade un `TimePicker` en `PantallaLocales` para que el usuario elija la hora.

3. **Badge de la app** — Instala `flutter_app_badger`. En `FcmService`,
   incrementa el badge en `_onMensajePrimerPlano`. Cuando el usuario
   abre la tab de Push, llama `FlutterAppBadger.removeBadge()`.
   Guarda el contador en `StateProvider<int>` de Riverpod.

4. **Payload estructurado** — En lugar de `String payload`, usa JSON en el
   campo `data` del mensaje FCM: `{"pantalla": "alertas", "id": "42"}`.
   Parsea el JSON en `_procesarMensaje` con `jsonDecode` y crea un
   `sealed class NavDestino` con los casos posibles.

---

## Resumen de la página 16

- Las notificaciones **locales** se generan desde la app — no necesitan internet. Las **push** llegan desde un servidor remoto vía Firebase.
- `FlutterLocalNotificationsPlugin` debe inicializarse en `main()` antes de `runApp()`. Los canales de Android se crean en la inicialización.
- Android 13+ requiere el permiso `POST_NOTIFICATIONS` en runtime. iOS siempre requiere el diálogo de permiso del sistema.
- `onlyAlertOnce: true` en la notificación de progreso evita que suene en cada actualización — solo suena la primera vez.
- `FirebaseMessaging.onBackgroundMessage` recibe un handler **top-level** con `@pragma('vm:entry-point')` — no puede ser un método de clase.
- En **primer plano**, FCM no muestra la notificación automáticamente en Android — hay que mostrarla manualmente con `flutter_local_notifications`.
- En **fondo y app cerrada**, FCM la muestra automáticamente si el mensaje tiene `notification` payload.
- Los **topics** permiten enviar a grupos de usuarios sin conocer cada token. `subscribeToTopic` y `unsubscribeFromTopic` son operaciones async.
- `getInitialMessage()` detecta si la app se abrió tocando una notificación push cuando estaba completamente cerrada.

---

> **Siguiente página →** Página 17: Audio y video streaming con `audioplayers`,
> `video_player` y controles de reproducción en segundo plano.