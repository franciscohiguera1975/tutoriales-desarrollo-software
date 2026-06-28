# Tutorial Flutter — Página 9
## Módulo 2 · Flutter
### Formularios y listas: `TextField`, `Form`, `ListView` y `GridView`

---

## Formularios y listas en Flutter

Flutter gestiona la entrada de texto con **`TextField`** (libre, sin validación) y **`Form` + `TextFormField`** (con validación centralizada). Para mostrar colecciones largas usa **`ListView.builder`** (virtualizado) o **`GridView.builder`** (cuadrícula). La `SearchBar` de M3 integra filtrado en tiempo real.

```
TextField (libre)          Form + TextFormField
────────────────           ────────────────────────────────
onChanged / onSubmitted    validator por campo
leer: _ctrl.text           validar todos: _key.validate()
sin estado de error        muestra error automáticamente
```

---

## Patrones clave — de un vistazo

Referencia rápida de los patrones del módulo. Léelos ahora y vuelve
a consultarlos conforme avanzas en cada paso.

---

### `TextField` y `TextEditingController`

```dart
// TextField básico — sin controller
TextField(
  decoration: const InputDecoration(
    labelText:  'Hostname',
    hintText:   'prod-web-01',
    prefixIcon: Icon(Icons.dns),
    border:     OutlineInputBorder(),
  ),
  onChanged:   (v) => print('Escribiendo: $v'),
  onSubmitted: (v) => print('Enviado: $v'),
)

// Con controller — leer, escribir y limpiar programáticamente
final _ctrl = TextEditingController(text: '22'); // valor inicial

// Leer el valor:   _ctrl.text
// Cambiar valor:   _ctrl.text = '2222'
// Limpiar campo:   _ctrl.clear()

// SIEMPRE liberar en dispose():
@override
void dispose() {
  _ctrl.dispose();
  super.dispose();
}
```

> **Mini-ejercicio ⏱ 5 min** — Agrega un cuarto `TextField` con `labelText: 'Usuario SSH'`, valor inicial `'root'` y `prefixIcon: Icons.person`. Muestra `usuario@ip:puerto` en el SnackBar del botón Conectar.

---

### `FocusNode` y flujo entre campos

```dart
final _focusIp     = FocusNode();
final _focusPuerto = FocusNode();

// En dispose():
_focusIp.dispose();
_focusPuerto.dispose();

// TextField con foco controlado:
TextField(
  textInputAction: TextInputAction.next,           // tecla "Siguiente"
  onSubmitted:     (_) => _focusIp.requestFocus(), // salta al campo IP
),
TextField(
  focusNode:       _focusIp,
  textInputAction: TextInputAction.next,
  onSubmitted:     (_) => _focusPuerto.requestFocus(),
),
TextField(
  focusNode:       _focusPuerto,
  textInputAction: TextInputAction.done,
  onSubmitted:     (_) => FocusScope.of(context).unfocus(), // cierra teclado
),
```

> **Mini-ejercicio ⏱ 5 min** — Agrega al flujo un `FocusNode` para el campo "Usuario SSH" entre Puerto y el cierre de teclado: Hostname → IP → Puerto → Usuario → cierra teclado.

---

### `Form` y `TextFormField`

```dart
// Form agrupa campos y valida todos con una sola llamada
final _formKey = GlobalKey<FormState>();

Form(
  key: _formKey,
  child: Column(children: [

    TextFormField(
      decoration: const InputDecoration(
        labelText:  'Nombre',
        prefixIcon: Icon(Icons.dns),
        border:     OutlineInputBorder(),
      ),
      validator: (v) {
        if (v == null || v.trim().isEmpty) return 'Campo requerido';
        if (v.length < 3)                  return 'Mínimo 3 caracteres';
        return null;   // null = válido
      },
    ),

    DropdownButtonFormField<String>(
      value: 'Ubuntu 24.04',
      decoration: const InputDecoration(
        labelText:  'Sistema Operativo',
        prefixIcon: Icon(Icons.computer),
        border:     OutlineInputBorder(),
      ),
      items: ['Ubuntu 24.04', 'Debian 12', 'Alpine Linux']
          .map((so) => DropdownMenuItem(value: so, child: Text(so)))
          .toList(),
      onChanged: (v) {},
    ),

    SwitchListTile(
      title:     const Text('Conexión SSL'),
      value:     true,
      onChanged: (v) {},
      secondary: const Icon(Icons.security),
    ),
  ]),
)

// Validar todos los campos de una vez:
if (_formKey.currentState!.validate()) { /* todos válidos */ }

// Limpiar todos los campos:
_formKey.currentState?.reset();
```

> **Mini-ejercicio ⏱ 5 min** — Agrega un `validator` al campo IP que rechace valores sin al menos un punto (`!v.contains('.')`). Mensaje de error: `'Formato inválido (ej: 192.168.1.1)'`.

---

### `ListView` — variantes

```dart
// ListView estático — pocas filas conocidas en tiempo de compilación
ListView(
  children: [
    ListTile(title: Text('Servidor 1')),
    const Divider(),
    ListTile(title: Text('Servidor 2')),
  ],
)

// ListView.builder — virtualizado, eficiente para muchos elementos
// Solo construye los widgets actualmente visibles en pantalla
ListView.builder(
  itemCount:   items.length,
  itemBuilder: (context, i) => TarjetaItem(item: items[i]),
)

// ListView.separated — separador automático entre elementos
ListView.separated(
  itemCount:        items.length,
  separatorBuilder: (_, __) => const Divider(height: 1, indent: 72),
  itemBuilder:      (context, i) => ListTile(title: Text(items[i])),
)
```

> **Mini-ejercicio ⏱ 5 min** — Convierte el `ListView.builder` del ejemplo en `ListView.separated` con `Divider(height: 1, indent: 72)`. ¿Qué cambia visualmente?

---

### `ListTile` — fila estándar

```dart
ListTile(
  leading: CircleAvatar(child: Icon(Icons.dns)),     // widget izquierda
  title:    const Text('prod-web-01'),
  subtitle: const Text('10.0.2.10 · Ubuntu 24.04'),
  trailing: Row(                                     // widget derecha
    mainAxisSize: MainAxisSize.min,
    children: [
      IconButton(icon: Icon(Icons.star_border), onPressed: () {}),
      IconButton(icon: Icon(Icons.delete_outline), onPressed: () {}),
    ],
  ),
  onTap: () {},
  contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
)
```

> **Mini-ejercicio ⏱ 5 min** — Modifica el `leading` para que muestre un ícono diferente según el SO: `Icons.terminal` para Linux y `Icons.window` para otros. Pista: usa un ternario en `CircleAvatar`.

---

### `GridView` — cuadrícula

```dart
// Columnas fijas
GridView.builder(
  padding: const EdgeInsets.all(12),
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount:   2,    // 2 columnas
    childAspectRatio: 1.1,  // ancho / alto de cada celda
    crossAxisSpacing: 8,
    mainAxisSpacing:  8,
  ),
  itemCount:   items.length,
  itemBuilder: (context, i) => TarjetaItem(item: items[i]),
)

// Columnas adaptativas — se ajustan al ancho de pantalla
GridView.builder(
  gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
    maxCrossAxisExtent: 200, // cada columna máximo 200px de ancho
    childAspectRatio:   1.2,
    crossAxisSpacing:   8,
    mainAxisSpacing:    8,
  ),
  itemCount:   items.length,
  itemBuilder: (_, i) => TarjetaItem(item: items[i]),
)
```

> **Mini-ejercicio ⏱ 5 min** — Cambia `crossAxisCount` de `2` a `3`. ¿Qué le pasa a las tarjetas? Ajusta `childAspectRatio` hasta que se vean bien (prueba valores entre `0.8` y `1.4`).

---

### `SearchBar` — barra de búsqueda M3

```dart
SearchBar(
  hintText: 'Buscar por nombre o IP...',
  leading:  const Icon(Icons.search),
  trailing: _busqueda.isNotEmpty
      ? [IconButton(icon: const Icon(Icons.clear), onPressed: () => setState(() => _busqueda = ''))]
      : null,
  onChanged: (v) => setState(() => _busqueda = v),
  padding: const WidgetStatePropertyAll(
    EdgeInsets.symmetric(horizontal: 16),
  ),
)

// Filtrar lista con getter
List<Servidor> get _filtrados => _servidores
    .where((s) =>
        s.nombre.toLowerCase().contains(_busqueda.toLowerCase()) ||
        s.ip.contains(_busqueda))
    .toList();
```

> **Mini-ejercicio ⏱ 5 min** — Amplía el filtro para que si el texto ingresado es numérico, busque también por puerto: añade `|| s.puerto.toString().contains(_busqueda)` al `where`.

---

## Crea el proyecto

```bash
flutter create modulo09_formularios
cd modulo09_formularios
```

Borra todo el contenido de `lib/main.dart` y déjalo vacío.
El Paso 1 parte desde cero en ese archivo.

```
modulo09_formularios/
├── lib/
│   ├── main.dart              ← aquí trabajamos
│   ├── models/                ← se crea al llegar al Paso 3
│   ├── widgets/               ← se crea al llegar al Paso 2
│   └── screens/               ← se crea al llegar al Paso 4
├── pubspec.yaml
└── ...
```

Para correr la app:
```bash
flutter run
```

---

## Paso 1 — `TextField` básico + `TextEditingController`

Escribe esto en `lib/main.dart`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';

void main() => runApp(const AppGestorSSH());

class AppGestorSSH extends StatelessWidget {
  const AppGestorSSH({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF1B5E20), // verde oscuro
        ),
        useMaterial3: true,
      ),
      home: const _ConexionSSH(),
    );
  }
}

class _ConexionSSH extends StatefulWidget {
  const _ConexionSSH();
  @override
  State<_ConexionSSH> createState() => _ConexionSSHState();
}

class _ConexionSSHState extends State<_ConexionSSH> {
  // Controller da acceso al texto, posición y selección del campo
  final _ctrlHostname = TextEditingController();
  final _ctrlIp       = TextEditingController();
  final _ctrlPuerto   = TextEditingController(text: '22'); // valor inicial

  final _focusIp     = FocusNode();
  final _focusPuerto = FocusNode();

  @override
  void dispose() {
    // SIEMPRE liberar controllers y FocusNodes en dispose
    _ctrlHostname.dispose();
    _ctrlIp.dispose();
    _ctrlPuerto.dispose();
    _focusIp.dispose();
    _focusPuerto.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Conexión SSH'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            TextField(
              controller:      _ctrlHostname,
              decoration:      const InputDecoration(
                labelText:  'Hostname',
                hintText:   'prod-web-01',
                prefixIcon: Icon(Icons.dns),
                border:     OutlineInputBorder(),
              ),
              textInputAction: TextInputAction.next,              // tecla "Siguiente"
              onSubmitted:     (_) => _focusIp.requestFocus(),   // salta a IP
            ),
            const SizedBox(height: 12),
            TextField(
              controller:      _ctrlIp,
              focusNode:       _focusIp,
              decoration:      const InputDecoration(
                labelText:  'Dirección IP',
                hintText:   '192.168.1.100',
                prefixIcon: Icon(Icons.router),
                border:     OutlineInputBorder(),
              ),
              keyboardType:    TextInputType.number,
              textInputAction: TextInputAction.next,
              onSubmitted:     (_) => _focusPuerto.requestFocus(),
            ),
            const SizedBox(height: 12),
            TextField(
              controller:      _ctrlPuerto,
              focusNode:       _focusPuerto,
              decoration:      const InputDecoration(
                labelText:  'Puerto SSH',
                prefixIcon: Icon(Icons.lock_outline),
                border:     OutlineInputBorder(),
              ),
              keyboardType:    TextInputType.number,
              textInputAction: TextInputAction.done,
              onSubmitted:     (_) => FocusScope.of(context).unfocus(),
            ),
            const SizedBox(height: 20),
            FilledButton.icon(
              onPressed: () {
                FocusScope.of(context).unfocus();
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(
                    content: Text(
                      'Conectando a ${_ctrlHostname.text} '
                      '(${_ctrlIp.text}:${_ctrlPuerto.text})',
                    ),
                    behavior: SnackBarBehavior.floating,
                  ),
                );
              },
              icon:  const Icon(Icons.terminal),
              label: const Text('Conectar'),
            ),
            const SizedBox(height: 8),
            OutlinedButton(
              onPressed: () {
                _ctrlHostname.clear();
                _ctrlIp.clear();
                _ctrlPuerto.text = '22'; // restablecer valor por defecto
              },
              child: const Text('Limpiar campos'),
            ),
          ],
        ),
      ),
    );
  }
}
```

`TextInputAction.next` muestra la tecla "Siguiente" en el teclado virtual y
`onSubmitted` salta al siguiente campo. `dispose()` es obligatorio — los
controllers y FocusNodes consumen memoria hasta que los liberas.

### Prueba esto

- Escribe en Hostname y presiona "Siguiente" en el teclado — el foco salta a IP
- Presiona "Siguiente" desde IP — salta a Puerto; desde Puerto — cierra el teclado
- Presiona **Conectar** — aparece SnackBar con los valores de cada campo
- Presiona **Limpiar campos** — Hostname e IP quedan vacíos, Puerto vuelve a `22`
- Cambia `border: OutlineInputBorder()` a `UnderlineInputBorder()` — el estilo del campo cambia

---

## Convierte `main.dart` en selector de pasos

Antes de avanzar al Paso 2, actualiza `main.dart` **una sola vez**.
A partir de aquí solo cambias el número de `paso` para navegar entre ejemplos
— el Paso 1 sigue funcionando igual con `paso = 1`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  TextField + TextEditingController + FocusNode       │
// │  2  Paso 2  Form + TextFormField + validación                   │
// │  3  Paso 3  Modelo + ListView.builder + ListTile acciones       │
// │  4  Paso 4  GridView.builder + toggle lista/grid                │
// │  5  Paso 5  SearchBar + filtrado en tiempo real                 │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() => runApp(MaterialApp(
  debugShowCheckedModeBanner: false,
  theme: ThemeData(
    colorScheme: ColorScheme.fromSeed(
      seedColor: const Color(0xFF1B5E20),
    ),
    useMaterial3: true,
  ),
  home: switch (paso) {
    1 => const _Paso1(),
    _ => Scaffold(
        body: Center(child: Text('Paso $paso: crea el widget primero'))),
  },
));

// ─── Paso 1 — vive en main.dart ────────────────────────────────────────
class _Paso1 extends StatefulWidget {
  const _Paso1();
  @override
  State<_Paso1> createState() => _Paso1State();
}

class _Paso1State extends State<_Paso1> {
  final _ctrlHostname = TextEditingController();
  final _ctrlIp       = TextEditingController();
  final _ctrlPuerto   = TextEditingController(text: '22');
  final _focusIp      = FocusNode();
  final _focusPuerto  = FocusNode();

  @override
  void dispose() {
    _ctrlHostname.dispose();
    _ctrlIp.dispose();
    _ctrlPuerto.dispose();
    _focusIp.dispose();
    _focusPuerto.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Conexión SSH'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            TextField(
              controller:      _ctrlHostname,
              decoration:      const InputDecoration(
                labelText:  'Hostname',
                hintText:   'prod-web-01',
                prefixIcon: Icon(Icons.dns),
                border:     OutlineInputBorder(),
              ),
              textInputAction: TextInputAction.next,
              onSubmitted:     (_) => _focusIp.requestFocus(),
            ),
            const SizedBox(height: 12),
            TextField(
              controller:      _ctrlIp,
              focusNode:       _focusIp,
              decoration:      const InputDecoration(
                labelText:  'Dirección IP',
                hintText:   '192.168.1.100',
                prefixIcon: Icon(Icons.router),
                border:     OutlineInputBorder(),
              ),
              keyboardType:    TextInputType.number,
              textInputAction: TextInputAction.next,
              onSubmitted:     (_) => _focusPuerto.requestFocus(),
            ),
            const SizedBox(height: 12),
            TextField(
              controller:  _ctrlPuerto,
              focusNode:   _focusPuerto,
              decoration:  const InputDecoration(
                labelText:  'Puerto SSH',
                prefixIcon: Icon(Icons.lock_outline),
                border:     OutlineInputBorder(),
              ),
              keyboardType:    TextInputType.number,
              textInputAction: TextInputAction.done,
              onSubmitted:     (_) => FocusScope.of(context).unfocus(),
            ),
            const SizedBox(height: 20),
            FilledButton.icon(
              onPressed: () {
                FocusScope.of(context).unfocus();
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(
                    content: Text(
                      'Conectando a ${_ctrlHostname.text} '
                      '(${_ctrlIp.text}:${_ctrlPuerto.text})',
                    ),
                    behavior: SnackBarBehavior.floating,
                  ),
                );
              },
              icon:  const Icon(Icons.terminal),
              label: const Text('Conectar'),
            ),
            const SizedBox(height: 8),
            OutlinedButton(
              onPressed: () {
                _ctrlHostname.clear();
                _ctrlIp.clear();
                _ctrlPuerto.text = '22';
              },
              child: const Text('Limpiar campos'),
            ),
          ],
        ),
      ),
    );
  }
}
```

Guarda y verifica que la app sigue mostrando el formulario SSH. Listo — de ahora
en adelante cada paso nuevo solo requiere agregar un `import` y un `case` al switch.

---

## Paso 2 — `Form` + `TextFormField` + validación

`Form` agrupa campos y los valida todos a la vez con `_formKey.currentState!.validate()`.
El `validator` devuelve `null` si el campo es válido o un `String` con el mensaje de error.

### Archivo: `lib/widgets/formulario_servidor.dart`

Crea la carpeta `lib/widgets/` y dentro el archivo:

```dart
// lib/widgets/formulario_servidor.dart
import 'package:flutter/material.dart';

class FormularioServidor extends StatefulWidget {
  final void Function(Map<String, String> datos) onGuardar;
  const FormularioServidor({super.key, required this.onGuardar});

  @override
  State<FormularioServidor> createState() => _FormularioServidorState();
}

class _FormularioServidorState extends State<FormularioServidor> {
  final _formKey  = GlobalKey<FormState>();

  final _ctrlNombre  = TextEditingController();
  final _ctrlIp      = TextEditingController();
  final _ctrlPuerto  = TextEditingController(text: '22');
  final _ctrlUsuario = TextEditingController(text: 'root');

  final _focusIp      = FocusNode();
  final _focusPuerto  = FocusNode();
  final _focusUsuario = FocusNode();

  String _so  = 'Ubuntu 24.04';
  bool   _ssl = true;

  // Expresión regular para validar IPv4
  static final _regexIp = RegExp(r'^(\d{1,3}\.){3}\d{1,3}$');

  @override
  void dispose() {
    _ctrlNombre.dispose();
    _ctrlIp.dispose();
    _ctrlPuerto.dispose();
    _ctrlUsuario.dispose();
    _focusIp.dispose();
    _focusPuerto.dispose();
    _focusUsuario.dispose();
    super.dispose();
  }

  void _guardar() {
    // validate() llama al validator de TODOS los TextFormField del Form
    if (!_formKey.currentState!.validate()) return;

    widget.onGuardar({
      'nombre':  _ctrlNombre.text,
      'ip':      _ctrlIp.text,
      'puerto':  _ctrlPuerto.text,
      'usuario': _ctrlUsuario.text,
      'so':      _so,
      'ssl':     _ssl.toString(),
    });
  }

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.stretch,
        children: [

          // ── Nombre del servidor ───────────────────────────────────
          TextFormField(
            controller:      _ctrlNombre,
            decoration:      const InputDecoration(
              labelText:  'Nombre del servidor',
              hintText:   'prod-web-01',
              prefixIcon: Icon(Icons.dns),
              border:     OutlineInputBorder(),
            ),
            textInputAction: TextInputAction.next,
            onFieldSubmitted: (_) => _focusIp.requestFocus(),
            validator: (v) {
              if (v == null || v.trim().isEmpty) return 'El nombre es obligatorio';
              if (v.length < 3)                  return 'Mínimo 3 caracteres';
              if (!RegExp(r'^[a-zA-Z0-9\-\_]+$').hasMatch(v))
                return 'Solo letras, números, guiones y guiones bajos';
              return null;
            },
          ),
          const SizedBox(height: 12),

          // ── Dirección IP ──────────────────────────────────────────
          TextFormField(
            controller:      _ctrlIp,
            focusNode:       _focusIp,
            decoration:      const InputDecoration(
              labelText:  'Dirección IP',
              hintText:   '192.168.1.100',
              prefixIcon: Icon(Icons.router),
              border:     OutlineInputBorder(),
            ),
            keyboardType:    TextInputType.number,
            textInputAction: TextInputAction.next,
            onFieldSubmitted: (_) => _focusPuerto.requestFocus(),
            validator: (v) {
              if (v == null || v.isEmpty) return 'La IP es obligatoria';
              if (!_regexIp.hasMatch(v))  return 'Formato IPv4 inválido (ej. 192.168.1.10)';
              final octetos = v.split('.').map(int.parse).toList();
              if (octetos.any((o) => o > 255)) return 'Octeto fuera de rango (0–255)';
              return null;
            },
          ),
          const SizedBox(height: 12),

          // ── Puerto SSH ────────────────────────────────────────────
          TextFormField(
            controller:      _ctrlPuerto,
            focusNode:       _focusPuerto,
            decoration:      const InputDecoration(
              labelText:  'Puerto',
              prefixIcon: Icon(Icons.lock_outline),
              border:     OutlineInputBorder(),
            ),
            keyboardType:    TextInputType.number,
            textInputAction: TextInputAction.next,
            onFieldSubmitted: (_) => _focusUsuario.requestFocus(),
            validator: (v) {
              final puerto = int.tryParse(v ?? '');
              if (puerto == null)              return 'Puerto debe ser un número';
              if (puerto < 1 || puerto > 65535) return 'Puerto entre 1 y 65535';
              return null;
            },
          ),
          const SizedBox(height: 12),

          // ── Usuario ───────────────────────────────────────────────
          TextFormField(
            controller:      _ctrlUsuario,
            focusNode:       _focusUsuario,
            decoration:      const InputDecoration(
              labelText:  'Usuario',
              prefixIcon: Icon(Icons.person_outline),
              border:     OutlineInputBorder(),
            ),
            textInputAction: TextInputAction.next,
            validator: (v) =>
                v == null || v.trim().isEmpty ? 'El usuario es obligatorio' : null,
          ),
          const SizedBox(height: 12),

          // ── Sistema Operativo — DropdownButtonFormField ────────────
          DropdownButtonFormField<String>(
            value:      _so,
            decoration: const InputDecoration(
              labelText:  'Sistema Operativo',
              prefixIcon: Icon(Icons.computer),
              border:     OutlineInputBorder(),
            ),
            items: [
              'Ubuntu 24.04', 'Debian 12', 'CentOS Stream 9',
              'Rocky Linux 9', 'Alpine Linux',
            ].map((s) => DropdownMenuItem(value: s, child: Text(s))).toList(),
            onChanged: (v) => setState(() => _so = v!),
          ),
          const SizedBox(height: 8),

          // ── SSL — SwitchListTile ──────────────────────────────────
          SwitchListTile(
            title:     const Text('Conexión SSL/TLS'),
            subtitle:  const Text('Cifrar la comunicación'),
            value:     _ssl,
            onChanged: (v) => setState(() => _ssl = v),
            secondary: const Icon(Icons.security),
          ),
          const SizedBox(height: 16),

          // ── Botones ───────────────────────────────────────────────
          Row(children: [
            Expanded(
              child: OutlinedButton(
                onPressed: () => _formKey.currentState?.reset(),
                child: const Text('Limpiar'),
              ),
            ),
            const SizedBox(width: 12),
            Expanded(
              flex: 2,
              child: FilledButton.icon(
                onPressed: _guardar,
                icon:  const Icon(Icons.save),
                label: const Text('Guardar servidor'),
              ),
            ),
          ]),
        ],
      ),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'widgets/formulario_servidor.dart';
```

```dart
2 => Scaffold(
       appBar: AppBar(title: const Text('Nuevo servidor')),
       body: SingleChildScrollView(
         padding: const EdgeInsets.all(16),
         child: FormularioServidor(
           onGuardar: (datos) {
             ScaffoldMessenger.of(
               Navigator.of(context).context,
             );
           },
         ),
       ),
     ),
```

Espera — dentro del `switch` del `MaterialApp` no hay `BuildContext` disponible.
La solución es envolver en un `StatelessWidget`:

```dart
2 => const _Paso2(),
```

Y al final de `main.dart`, fuera del `main()`:

```dart
class _Paso2 extends StatelessWidget {
  const _Paso2();

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Nuevo servidor'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: FormularioServidor(
          onGuardar: (datos) {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(
                content: Text(
                    'Guardado: ${datos['nombre']} — ${datos['ip']}:${datos['puerto']}'),
                behavior: SnackBarBehavior.floating,
              ),
            );
          },
        ),
      ),
    );
  }
}
```

Cambia `paso = 2` y guarda.

### Prueba esto

- Presiona **Guardar servidor** con todos los campos vacíos — cada campo muestra su error en rojo
- Escribe `ab` en Nombre — aparece "Mínimo 3 caracteres"
- Escribe `999.0.0.1` en IP — aparece el error de octeto fuera de rango
- Escribe `99999` en Puerto — aparece el error de rango
- Completa todos los campos correctamente y presiona Guardar — aparece el SnackBar
- Presiona **Limpiar** — todos los campos vuelven a sus valores iniciales
- Desmarca SSL y cambia el SO con el dropdown — los cambios persisten al cambiar de campo

---

## Paso 3 — Modelo + `ListView.builder` + acciones

Un `ListView.separated` con `ListTile` que muestra datos de un modelo.
Las acciones (favorito / eliminar) van en el `trailing` del `ListTile`.

### Archivo: `lib/models/servidor_ssh.dart`

Crea la carpeta `lib/models/` y dentro el archivo:

```dart
// lib/models/servidor_ssh.dart
class ServidorSSH {
  final String id;
  final String nombre;
  final String ip;
  final int    puerto;
  final String usuario;
  final String so;
  final bool   ssl;
  bool         favorito;    // mutable — puede cambiar sin recrear el objeto

  ServidorSSH({
    required this.id,
    required this.nombre,
    required this.ip,
    required this.puerto,
    required this.usuario,
    required this.so,
    required this.ssl,
    this.favorito = false,
  });
}
```

### Archivo: `lib/widgets/fila_servidor.dart`

```dart
// lib/widgets/fila_servidor.dart
import 'package:flutter/material.dart';
import '../models/servidor_ssh.dart';

class FilaServidor extends StatelessWidget {
  final ServidorSSH  servidor;
  final VoidCallback onFavorito;
  final VoidCallback onEliminar;

  const FilaServidor({
    super.key,
    required this.servidor,
    required this.onFavorito,
    required this.onEliminar,
  });

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return ListTile(
      // leading — icono con color según SSL
      leading: CircleAvatar(
        backgroundColor: servidor.ssl
            ? cs.primaryContainer
            : cs.surfaceContainerHighest,
        child: Icon(
          Icons.dns,
          color: servidor.ssl ? cs.onPrimaryContainer : cs.onSurfaceVariant,
        ),
      ),
      title: Text(
        servidor.nombre,
        style: const TextStyle(fontWeight: FontWeight.w600),
      ),
      subtitle: Text(
        '${servidor.usuario}@${servidor.ip}:${servidor.puerto}',
        style: TextStyle(fontSize: 12, color: cs.onSurfaceVariant),
      ),
      // trailing — dos acciones compactas
      trailing: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          IconButton(
            icon: Icon(
              servidor.favorito ? Icons.star : Icons.star_border,
              color: servidor.favorito ? Colors.amber : cs.outline,
            ),
            onPressed:     onFavorito,
            visualDensity: VisualDensity.compact,
            tooltip:       servidor.favorito ? 'Quitar favorito' : 'Agregar a favoritos',
          ),
          IconButton(
            icon:          Icon(Icons.delete_outline, color: cs.error),
            onPressed:     onEliminar,
            visualDensity: VisualDensity.compact,
            tooltip:       'Eliminar',
          ),
        ],
      ),
      contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'models/servidor_ssh.dart';
import 'widgets/fila_servidor.dart';
```

```dart
3 => const _Paso3(),
```

Al final de `main.dart`:

```dart
class _Paso3 extends StatefulWidget {
  const _Paso3();
  @override
  State<_Paso3> createState() => _Paso3State();
}

class _Paso3State extends State<_Paso3> {
  final _servidores = [
    ServidorSSH(id:'1', nombre:'prod-web-01',  ip:'10.0.2.10',   puerto:22,   usuario:'deploy',   so:'Ubuntu 24.04', ssl:true,  favorito:true),
    ServidorSSH(id:'2', nombre:'prod-db-01',   ip:'10.0.2.20',   puerto:22,   usuario:'postgres', so:'Debian 12',    ssl:true),
    ServidorSSH(id:'3', nombre:'staging-api',  ip:'10.0.3.10',   puerto:2222, usuario:'ubuntu',   so:'Ubuntu 24.04', ssl:false),
    ServidorSSH(id:'4', nombre:'dev-sandbox',  ip:'192.168.1.5', puerto:22,   usuario:'vagrant',  so:'Alpine Linux', ssl:false),
  ];

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           Text('Servidores (${_servidores.length})'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: _servidores.isEmpty
          ? Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Icon(Icons.dns_outlined, size: 56, color: cs.onSurfaceVariant),
                  const SizedBox(height: 12),
                  Text('Sin servidores',
                      style: TextStyle(color: cs.onSurfaceVariant)),
                ],
              ),
            )
          : ListView.separated(
              itemCount:        _servidores.length,
              separatorBuilder: (_, __) =>
                  const Divider(height: 1, indent: 72),
              itemBuilder: (ctx, i) => FilaServidor(
                servidor:   _servidores[i],
                onFavorito: () => setState(() =>
                    _servidores[i].favorito = !_servidores[i].favorito),
                onEliminar: () => setState(() => _servidores.removeAt(i)),
              ),
            ),
    );
  }
}
```

Cambia `paso = 3` y guarda.

### Prueba esto

- Presiona la estrella de `prod-web-01` — se vuelve amarilla
- Presiona el icono de eliminar de cualquier servidor — desaparece de la lista
- Elimina todos los servidores — aparece el estado vacío con el icono
- Observa el `Divider(indent: 72)` — el separador no llega al borde izquierdo,
  dejando espacio para el avatar circular

---

## Paso 4 — `GridView.builder` + toggle lista/grid

`GridView.builder` virtualiza la cuadrícula igual que `ListView.builder`.
Un `bool _modoGrid` en el estado controla qué vista se muestra.

### Archivo: `lib/widgets/tarjeta_servidor_grid.dart`

```dart
// lib/widgets/tarjeta_servidor_grid.dart
import 'package:flutter/material.dart';
import '../models/servidor_ssh.dart';

class TarjetaServidorGrid extends StatelessWidget {
  final ServidorSSH  servidor;
  final VoidCallback onFavorito;
  final VoidCallback onEliminar;

  const TarjetaServidorGrid({
    super.key,
    required this.servidor,
    required this.onFavorito,
    required this.onEliminar,
  });

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(10),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Fila superior: icono + favorito
            Row(children: [
              Icon(
                Icons.dns,
                color: servidor.ssl ? cs.primary : cs.outline,
                size: 18,
              ),
              const Spacer(),
              GestureDetector(
                onTap: onFavorito,
                child: Icon(
                  servidor.favorito ? Icons.star : Icons.star_border,
                  color: servidor.favorito ? Colors.amber : cs.outline,
                  size: 18,
                ),
              ),
            ]),
            const SizedBox(height: 6),

            // Nombre e IP
            Text(
              servidor.nombre,
              style: text.titleSmall?.copyWith(fontWeight: FontWeight.bold),
              maxLines: 1,
              overflow: TextOverflow.ellipsis,
            ),
            Text(
              servidor.ip,
              style: text.bodySmall?.copyWith(color: cs.onSurfaceVariant),
            ),

            const Spacer(),

            // Fila inferior: SSL + SO + eliminar
            Row(children: [
              if (servidor.ssl)
                Padding(
                  padding: const EdgeInsets.only(right: 4),
                  child: Icon(Icons.lock, size: 12, color: cs.primary),
                ),
              Expanded(
                child: Text(
                  servidor.so,
                  style: text.labelSmall?.copyWith(color: cs.onSurfaceVariant),
                  overflow: TextOverflow.ellipsis,
                ),
              ),
              GestureDetector(
                onTap: onEliminar,
                child: Icon(Icons.delete_outline, size: 16, color: cs.error),
              ),
            ]),
          ],
        ),
      ),
    );
  }
}
```

### Archivo: `lib/screens/pantalla_servidores.dart`

Crea la carpeta `lib/screens/` y dentro el archivo:

```dart
// lib/screens/pantalla_servidores.dart
import 'package:flutter/material.dart';
import '../models/servidor_ssh.dart';
import '../widgets/fila_servidor.dart';
import '../widgets/tarjeta_servidor_grid.dart';

class PantallaServidores extends StatefulWidget {
  const PantallaServidores({super.key});
  @override
  State<PantallaServidores> createState() => _PantallaServidoresState();
}

class _PantallaServidoresState extends State<PantallaServidores> {
  final _servidores = [
    ServidorSSH(id:'1', nombre:'prod-web-01',  ip:'10.0.2.10',   puerto:22,   usuario:'deploy',   so:'Ubuntu 24.04', ssl:true,  favorito:true),
    ServidorSSH(id:'2', nombre:'prod-db-01',   ip:'10.0.2.20',   puerto:22,   usuario:'postgres', so:'Debian 12',    ssl:true),
    ServidorSSH(id:'3', nombre:'staging-api',  ip:'10.0.3.10',   puerto:2222, usuario:'ubuntu',   so:'Ubuntu 24.04', ssl:false),
    ServidorSSH(id:'4', nombre:'dev-sandbox',  ip:'192.168.1.5', puerto:22,   usuario:'vagrant',  so:'Alpine Linux', ssl:false),
  ];

  bool _modoGrid = false;   // false = lista, true = cuadrícula

  void _toggleFavorito(int i) =>
      setState(() => _servidores[i].favorito = !_servidores[i].favorito);

  void _eliminar(int i) => setState(() => _servidores.removeAt(i));

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           Text('Servidores (${_servidores.length})'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          // Toggle lista / cuadrícula
          IconButton(
            icon:    Icon(_modoGrid ? Icons.list : Icons.grid_view),
            onPressed: () => setState(() => _modoGrid = !_modoGrid),
            tooltip: _modoGrid ? 'Vista lista' : 'Vista cuadrícula',
          ),
        ],
      ),
      body: _modoGrid
          ? GridView.builder(
              padding: const EdgeInsets.all(12),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount:   2,
                childAspectRatio: 1.1,
                crossAxisSpacing: 8,
                mainAxisSpacing:  8,
              ),
              itemCount:   _servidores.length,
              itemBuilder: (ctx, i) => TarjetaServidorGrid(
                servidor:   _servidores[i],
                onFavorito: () => _toggleFavorito(i),
                onEliminar: () => _eliminar(i),
              ),
            )
          : ListView.separated(
              itemCount:        _servidores.length,
              separatorBuilder: (_, __) =>
                  const Divider(height: 1, indent: 72),
              itemBuilder: (ctx, i) => FilaServidor(
                servidor:   _servidores[i],
                onFavorito: () => _toggleFavorito(i),
                onEliminar: () => _eliminar(i),
              ),
            ),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_servidores.dart';
import 'widgets/tarjeta_servidor_grid.dart';
```

```dart
4 => const PantallaServidores(),
```

Cambia `paso = 4` y guarda.

### Prueba esto

- Presiona el ícono de la AppBar — alterna entre lista y cuadrícula con animación
- En modo cuadrícula, cambia `crossAxisCount: 2` a `3` — aparecen 3 columnas
- Cambia `childAspectRatio: 1.1` a `1.5` — las tarjetas se vuelven más anchas que altas
- Prueba `SliverGridDelegateWithMaxCrossAxisExtent(maxCrossAxisExtent: 180)` en lugar de
  `WithFixedCrossAxisCount` — las columnas se adaptan automáticamente al ancho de pantalla
- Marca favoritos en la lista, cambia a cuadrícula — los favoritos se conservan

---

## Paso 5 — `SearchBar` y filtrado en tiempo real

`SearchBar` es el componente M3 para búsqueda. El filtrado usa un getter
calculado sobre la lista original para no perder elementos al borrar el texto.

### Archivo: `lib/screens/pantalla_busqueda.dart`

```dart
// lib/screens/pantalla_busqueda.dart
import 'package:flutter/material.dart';
import '../models/servidor_ssh.dart';
import '../widgets/fila_servidor.dart';
import '../widgets/tarjeta_servidor_grid.dart';

class PantallaBusqueda extends StatefulWidget {
  const PantallaBusqueda({super.key});
  @override
  State<PantallaBusqueda> createState() => _PantallaBusquedaState();
}

class _PantallaBusquedaState extends State<PantallaBusqueda> {
  final _servidores = [
    ServidorSSH(id:'1', nombre:'prod-web-01',  ip:'10.0.2.10',   puerto:22,   usuario:'deploy',   so:'Ubuntu 24.04', ssl:true,  favorito:true),
    ServidorSSH(id:'2', nombre:'prod-db-01',   ip:'10.0.2.20',   puerto:22,   usuario:'postgres', so:'Debian 12',    ssl:true),
    ServidorSSH(id:'3', nombre:'staging-api',  ip:'10.0.3.10',   puerto:2222, usuario:'ubuntu',   so:'Ubuntu 24.04', ssl:false),
    ServidorSSH(id:'4', nombre:'dev-sandbox',  ip:'192.168.1.5', puerto:22,   usuario:'vagrant',  so:'Alpine Linux', ssl:false),
  ];

  String _busqueda = '';     // texto actual de la búsqueda
  bool   _modoGrid = false;

  // Getter calculado — filtra sin modificar _servidores
  List<ServidorSSH> get _filtrados => _servidores
      .where((s) =>
          s.nombre.toLowerCase().contains(_busqueda.toLowerCase()) ||
          s.ip.contains(_busqueda) ||
          s.usuario.toLowerCase().contains(_busqueda.toLowerCase()))
      .toList();

  void _toggleFavorito(ServidorSSH s) =>
      setState(() => s.favorito = !s.favorito);

  void _eliminar(ServidorSSH s) =>
      setState(() => _servidores.removeWhere((x) => x.id == s.id));

  @override
  Widget build(BuildContext context) {
    final cs       = Theme.of(context).colorScheme;
    final filtrados = _filtrados;   // evalúa el getter una sola vez

    return Scaffold(
      appBar: AppBar(
        title:           Text('Servidores (${_servidores.length})'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          IconButton(
            icon:      Icon(_modoGrid ? Icons.list : Icons.grid_view),
            onPressed: () => setState(() => _modoGrid = !_modoGrid),
            tooltip:   _modoGrid ? 'Vista lista' : 'Vista cuadrícula',
          ),
        ],
      ),
      body: Column(
        children: [
          // ── SearchBar ─────────────────────────────────────────────
          Padding(
            padding: const EdgeInsets.fromLTRB(16, 12, 16, 8),
            child: SearchBar(
              hintText: 'Buscar por nombre, IP o usuario...',
              leading:  const Icon(Icons.search),
              trailing: _busqueda.isNotEmpty
                  ? [
                      IconButton(
                        icon:      const Icon(Icons.clear),
                        onPressed: () => setState(() => _busqueda = ''),
                      ),
                    ]
                  : null,
              onChanged: (v) => setState(() => _busqueda = v),
              padding: const WidgetStatePropertyAll(
                EdgeInsets.symmetric(horizontal: 16),
              ),
            ),
          ),

          // ── Contador de resultados ────────────────────────────────
          if (_busqueda.isNotEmpty)
            Padding(
              padding: const EdgeInsets.only(left: 16, bottom: 4),
              child: Align(
                alignment: Alignment.centerLeft,
                child: Text(
                  '${filtrados.length} resultado${filtrados.length == 1 ? '' : 's'}',
                  style: Theme.of(context).textTheme.labelMedium?.copyWith(
                    color: cs.onSurfaceVariant,
                  ),
                ),
              ),
            ),

          // ── Lista o Grid ──────────────────────────────────────────
          Expanded(
            child: filtrados.isEmpty
                ? Center(
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        Icon(Icons.search_off,
                            size: 56, color: cs.onSurfaceVariant),
                        const SizedBox(height: 12),
                        Text(
                          'Sin resultados para "$_busqueda"',
                          style: TextStyle(color: cs.onSurfaceVariant),
                        ),
                        const SizedBox(height: 8),
                        TextButton(
                          onPressed: () => setState(() => _busqueda = ''),
                          child: const Text('Limpiar búsqueda'),
                        ),
                      ],
                    ),
                  )
                : _modoGrid
                    ? GridView.builder(
                        padding: const EdgeInsets.all(12),
                        gridDelegate:
                            const SliverGridDelegateWithFixedCrossAxisCount(
                          crossAxisCount:   2,
                          childAspectRatio: 1.1,
                          crossAxisSpacing: 8,
                          mainAxisSpacing:  8,
                        ),
                        itemCount:   filtrados.length,
                        itemBuilder: (ctx, i) => TarjetaServidorGrid(
                          servidor:   filtrados[i],
                          onFavorito: () => _toggleFavorito(filtrados[i]),
                          onEliminar: () => _eliminar(filtrados[i]),
                        ),
                      )
                    : ListView.separated(
                        itemCount:        filtrados.length,
                        separatorBuilder: (_, __) =>
                            const Divider(height: 1, indent: 72),
                        itemBuilder: (ctx, i) => FilaServidor(
                          servidor:   filtrados[i],
                          onFavorito: () => _toggleFavorito(filtrados[i]),
                          onEliminar: () => _eliminar(filtrados[i]),
                        ),
                      ),
          ),
        ],
      ),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_busqueda.dart';
```

```dart
5 => const PantallaBusqueda(),
```

Cambia `paso = 5` y guarda.

### Prueba esto

- Escribe `prod` en la búsqueda — filtra solo los servidores que contienen "prod"
- Escribe `10.0.3` — filtra por IP
- Escribe `ubuntu` — filtra por usuario
- Presiona la `X` del SearchBar — regresa a la lista completa
- Escribe algo que no exista (ej. `xyz`) — aparece el estado vacío con botón "Limpiar búsqueda"
- Marca un favorito, escribe una búsqueda, borra el texto — el favorito se mantiene
  (el getter filtra `_servidores` sin modificarlo)

---

## `main.dart` completo — referencia

Al terminar todos los pasos, el archivo queda así:

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'widgets/formulario_servidor.dart';
import 'models/servidor_ssh.dart';
import 'widgets/fila_servidor.dart';
import 'screens/pantalla_servidores.dart';
import 'screens/pantalla_busqueda.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  TextField + TextEditingController + FocusNode       │
// │  2  Paso 2  Form + TextFormField + validación                   │
// │  3  Paso 3  Modelo + ListView.builder + ListTile acciones       │
// │  4  Paso 4  GridView.builder + toggle lista/grid                │
// │  5  Paso 5  SearchBar + filtrado en tiempo real                 │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() => runApp(MaterialApp(
  debugShowCheckedModeBanner: false,
  theme: ThemeData(
    colorScheme: ColorScheme.fromSeed(
      seedColor: const Color(0xFF1B5E20),
    ),
    useMaterial3: true,
  ),
  home: switch (paso) {
    1 => const _Paso1(),
    2 => const _Paso2(),
    3 => const _Paso3(),
    4 => const PantallaServidores(),
    5 => const PantallaBusqueda(),
    _ => Scaffold(body: Center(child: Text('Paso $paso no definido'))),
  },
));

// ─── Paso 1 ────────────────────────────────────────────────────────────
class _Paso1 extends StatefulWidget {
  const _Paso1();
  @override
  State<_Paso1> createState() => _Paso1State();
}

class _Paso1State extends State<_Paso1> {
  final _ctrlHostname = TextEditingController();
  final _ctrlIp       = TextEditingController();
  final _ctrlPuerto   = TextEditingController(text: '22');
  final _focusIp      = FocusNode();
  final _focusPuerto  = FocusNode();

  @override
  void dispose() {
    _ctrlHostname.dispose();
    _ctrlIp.dispose();
    _ctrlPuerto.dispose();
    _focusIp.dispose();
    _focusPuerto.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Conexión SSH'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            TextField(
              controller:      _ctrlHostname,
              decoration:      const InputDecoration(
                labelText:  'Hostname',
                hintText:   'prod-web-01',
                prefixIcon: Icon(Icons.dns),
                border:     OutlineInputBorder(),
              ),
              textInputAction: TextInputAction.next,
              onSubmitted:     (_) => _focusIp.requestFocus(),
            ),
            const SizedBox(height: 12),
            TextField(
              controller:      _ctrlIp,
              focusNode:       _focusIp,
              decoration:      const InputDecoration(
                labelText:  'Dirección IP',
                hintText:   '192.168.1.100',
                prefixIcon: Icon(Icons.router),
                border:     OutlineInputBorder(),
              ),
              keyboardType:    TextInputType.number,
              textInputAction: TextInputAction.next,
              onSubmitted:     (_) => _focusPuerto.requestFocus(),
            ),
            const SizedBox(height: 12),
            TextField(
              controller:      _ctrlPuerto,
              focusNode:       _focusPuerto,
              decoration:      const InputDecoration(
                labelText:  'Puerto SSH',
                prefixIcon: Icon(Icons.lock_outline),
                border:     OutlineInputBorder(),
              ),
              keyboardType:    TextInputType.number,
              textInputAction: TextInputAction.done,
              onSubmitted:     (_) => FocusScope.of(context).unfocus(),
            ),
            const SizedBox(height: 20),
            FilledButton.icon(
              onPressed: () {
                FocusScope.of(context).unfocus();
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(
                    content: Text(
                      'Conectando a ${_ctrlHostname.text} '
                      '(${_ctrlIp.text}:${_ctrlPuerto.text})',
                    ),
                    behavior: SnackBarBehavior.floating,
                  ),
                );
              },
              icon:  const Icon(Icons.terminal),
              label: const Text('Conectar'),
            ),
            const SizedBox(height: 8),
            OutlinedButton(
              onPressed: () {
                _ctrlHostname.clear();
                _ctrlIp.clear();
                _ctrlPuerto.text = '22';
              },
              child: const Text('Limpiar campos'),
            ),
          ],
        ),
      ),
    );
  }
}

// ─── Paso 2 ────────────────────────────────────────────────────────────
class _Paso2 extends StatelessWidget {
  const _Paso2();

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Nuevo servidor'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: FormularioServidor(
          onGuardar: (datos) {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(
                content: Text(
                    'Guardado: ${datos['nombre']} — ${datos['ip']}:${datos['puerto']}'),
                behavior: SnackBarBehavior.floating,
              ),
            );
          },
        ),
      ),
    );
  }
}

// ─── Paso 3 ────────────────────────────────────────────────────────────
class _Paso3 extends StatefulWidget {
  const _Paso3();
  @override
  State<_Paso3> createState() => _Paso3State();
}

class _Paso3State extends State<_Paso3> {
  final _servidores = [
    ServidorSSH(id:'1', nombre:'prod-web-01',  ip:'10.0.2.10',   puerto:22,   usuario:'deploy',   so:'Ubuntu 24.04', ssl:true,  favorito:true),
    ServidorSSH(id:'2', nombre:'prod-db-01',   ip:'10.0.2.20',   puerto:22,   usuario:'postgres', so:'Debian 12',    ssl:true),
    ServidorSSH(id:'3', nombre:'staging-api',  ip:'10.0.3.10',   puerto:2222, usuario:'ubuntu',   so:'Ubuntu 24.04', ssl:false),
    ServidorSSH(id:'4', nombre:'dev-sandbox',  ip:'192.168.1.5', puerto:22,   usuario:'vagrant',  so:'Alpine Linux', ssl:false),
  ];

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           Text('Servidores (${_servidores.length})'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: _servidores.isEmpty
          ? Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Icon(Icons.dns_outlined, size: 56, color: cs.onSurfaceVariant),
                  const SizedBox(height: 12),
                  Text('Sin servidores',
                      style: TextStyle(color: cs.onSurfaceVariant)),
                ],
              ),
            )
          : ListView.separated(
              itemCount:        _servidores.length,
              separatorBuilder: (_, __) =>
                  const Divider(height: 1, indent: 72),
              itemBuilder: (ctx, i) => FilaServidor(
                servidor:   _servidores[i],
                onFavorito: () => setState(() =>
                    _servidores[i].favorito = !_servidores[i].favorito),
                onEliminar: () =>
                    setState(() => _servidores.removeAt(i)),
              ),
            ),
    );
  }
}
```

---

## Proyecto — Gestor de Servidores SSH

Proyecto separado que integra formulario + búsqueda + lista/grid + confirmación de eliminación.
La pantalla principal muestra la lista con el `SearchBar`; el FAB muestra el formulario.

```
modulo09_formularios/
├── lib/
│   ├── main.dart
│   ├── models/
│   │   └── servidor_ssh.dart      ← del Paso 3
│   ├── widgets/
│   │   ├── formulario_servidor.dart   ← del Paso 2
│   │   ├── fila_servidor.dart         ← del Paso 3
│   │   └── tarjeta_servidor_grid.dart ← del Paso 4
│   └── screens/
│       └── pantalla_gestor.dart   ← nuevo
```

### `lib/screens/pantalla_gestor.dart`

```dart
// lib/screens/pantalla_gestor.dart
import 'package:flutter/material.dart';
import '../models/servidor_ssh.dart';
import '../widgets/formulario_servidor.dart';
import '../widgets/fila_servidor.dart';
import '../widgets/tarjeta_servidor_grid.dart';

class PantallaGestor extends StatefulWidget {
  const PantallaGestor({super.key});
  @override
  State<PantallaGestor> createState() => _PantallaGestorState();
}

class _PantallaGestorState extends State<PantallaGestor> {
  final _servidores = [
    ServidorSSH(id:'1', nombre:'prod-web-01',  ip:'10.0.2.10',   puerto:22,   usuario:'deploy',   so:'Ubuntu 24.04', ssl:true,  favorito:true),
    ServidorSSH(id:'2', nombre:'prod-db-01',   ip:'10.0.2.20',   puerto:22,   usuario:'postgres', so:'Debian 12',    ssl:true),
    ServidorSSH(id:'3', nombre:'staging-api',  ip:'10.0.3.10',   puerto:2222, usuario:'ubuntu',   so:'Ubuntu 24.04', ssl:false),
    ServidorSSH(id:'4', nombre:'dev-sandbox',  ip:'192.168.1.5', puerto:22,   usuario:'vagrant',  so:'Alpine Linux', ssl:false),
  ];

  String _busqueda    = '';
  bool   _mostrarForm = false;
  bool   _modoGrid    = false;

  List<ServidorSSH> get _filtrados => _servidores
      .where((s) =>
          s.nombre.toLowerCase().contains(_busqueda.toLowerCase()) ||
          s.ip.contains(_busqueda) ||
          s.usuario.toLowerCase().contains(_busqueda.toLowerCase()))
      .toList();

  void _agregarServidor(Map<String, String> datos) {
    setState(() {
      _servidores.add(ServidorSSH(
        id:      DateTime.now().millisecondsSinceEpoch.toString(),
        nombre:  datos['nombre']!,
        ip:      datos['ip']!,
        puerto:  int.parse(datos['puerto']!),
        usuario: datos['usuario']!,
        so:      datos['so']!,
        ssl:     datos['ssl'] == 'true',
      ));
      _mostrarForm = false;
    });

    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content:  Text('Servidor "${datos['nombre']}" agregado'),
        behavior: SnackBarBehavior.floating,
      ),
    );
  }

  Future<void> _confirmarEliminar(ServidorSSH s) async {
    final confirma = await showDialog<bool>(
      context: context,
      builder: (ctx) => AlertDialog(
        icon:    const Icon(Icons.warning_amber, color: Colors.orange),
        title:   const Text('Eliminar servidor'),
        content: Text('¿Eliminar "${s.nombre}" (${s.ip})?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx, false),
            child:     const Text('Cancelar'),
          ),
          FilledButton(
            style: FilledButton.styleFrom(
              backgroundColor: Theme.of(ctx).colorScheme.error,
            ),
            onPressed: () => Navigator.pop(ctx, true),
            child: const Text('Eliminar'),
          ),
        ],
      ),
    );

    if (confirma == true) {
      setState(() => _servidores.removeWhere((x) => x.id == s.id));
    }
  }

  @override
  Widget build(BuildContext context) {
    final cs      = Theme.of(context).colorScheme;
    final filtrados = _filtrados;

    return Scaffold(
      appBar: AppBar(
        title: _mostrarForm
            ? const Text('Nuevo servidor')
            : Text('Servidores SSH (${_servidores.length})'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        leading: _mostrarForm
            ? IconButton(
                icon:      const Icon(Icons.arrow_back),
                onPressed: () => setState(() => _mostrarForm = false),
              )
            : null,
        actions: _mostrarForm
            ? []
            : [
                IconButton(
                  icon:      Icon(_modoGrid ? Icons.list : Icons.grid_view),
                  onPressed: () =>
                      setState(() => _modoGrid = !_modoGrid),
                  tooltip:   _modoGrid ? 'Vista lista' : 'Vista cuadrícula',
                ),
              ],
      ),

      body: _mostrarForm
          // ── Formulario ────────────────────────────────────────────
          ? SingleChildScrollView(
              padding: const EdgeInsets.all(16),
              child: FormularioServidor(onGuardar: _agregarServidor),
            )
          // ── Lista / Grid con búsqueda ─────────────────────────────
          : Column(
              children: [
                Padding(
                  padding: const EdgeInsets.fromLTRB(16, 12, 16, 8),
                  child: SearchBar(
                    hintText: 'Buscar por nombre, IP o usuario...',
                    leading:  const Icon(Icons.search),
                    trailing: _busqueda.isNotEmpty
                        ? [
                            IconButton(
                              icon:      const Icon(Icons.clear),
                              onPressed: () =>
                                  setState(() => _busqueda = ''),
                            ),
                          ]
                        : null,
                    onChanged: (v) => setState(() => _busqueda = v),
                    padding: const WidgetStatePropertyAll(
                      EdgeInsets.symmetric(horizontal: 16),
                    ),
                  ),
                ),

                if (_busqueda.isNotEmpty)
                  Padding(
                    padding: const EdgeInsets.only(left: 16, bottom: 4),
                    child: Align(
                      alignment: Alignment.centerLeft,
                      child: Text(
                        '${filtrados.length} resultado${filtrados.length == 1 ? '' : 's'}',
                        style: Theme.of(context).textTheme.labelMedium
                            ?.copyWith(color: cs.onSurfaceVariant),
                      ),
                    ),
                  ),

                Expanded(
                  child: filtrados.isEmpty
                      ? Center(
                          child: Column(
                            mainAxisSize: MainAxisSize.min,
                            children: [
                              Icon(Icons.search_off,
                                  size: 56, color: cs.onSurfaceVariant),
                              const SizedBox(height: 12),
                              Text(
                                'Sin resultados para "$_busqueda"',
                                style:
                                    TextStyle(color: cs.onSurfaceVariant),
                              ),
                              const SizedBox(height: 8),
                              TextButton(
                                onPressed: () =>
                                    setState(() => _busqueda = ''),
                                child: const Text('Limpiar búsqueda'),
                              ),
                            ],
                          ),
                        )
                      : _modoGrid
                          ? GridView.builder(
                              padding: const EdgeInsets.all(12),
                              gridDelegate:
                                  const SliverGridDelegateWithFixedCrossAxisCount(
                                crossAxisCount:   2,
                                childAspectRatio: 1.1,
                                crossAxisSpacing: 8,
                                mainAxisSpacing:  8,
                              ),
                              itemCount:   filtrados.length,
                              itemBuilder: (ctx, i) =>
                                  TarjetaServidorGrid(
                                servidor:   filtrados[i],
                                onFavorito: () => setState(() =>
                                    filtrados[i].favorito =
                                        !filtrados[i].favorito),
                                onEliminar: () =>
                                    _confirmarEliminar(filtrados[i]),
                              ),
                            )
                          : ListView.separated(
                              itemCount:        filtrados.length,
                              separatorBuilder: (_, __) =>
                                  const Divider(height: 1, indent: 72),
                              itemBuilder: (ctx, i) => FilaServidor(
                                servidor:   filtrados[i],
                                onFavorito: () => setState(() =>
                                    filtrados[i].favorito =
                                        !filtrados[i].favorito),
                                onEliminar: () =>
                                    _confirmarEliminar(filtrados[i]),
                              ),
                            ),
                ),
              ],
            ),

      // FAB solo en vista de lista/grid
      floatingActionButton: _mostrarForm
          ? null
          : FloatingActionButton.extended(
              onPressed: () => setState(() => _mostrarForm = true),
              icon:  const Icon(Icons.add),
              label: const Text('Nuevo servidor'),
            ),
    );
  }
}
```

### `lib/main.dart` (proyecto)

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'screens/pantalla_gestor.dart';

void main() => runApp(const AppGestorSSH());

class AppGestorSSH extends StatelessWidget {
  const AppGestorSSH({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title:     'Gestor SSH',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF1B5E20), // verde oscuro
        ),
        useMaterial3: true,
      ),
      home: const PantallaGestor(),
    );
  }
}
```

---

## Guía rápida de imports

```dart
import 'package:flutter/material.dart';  // todo Material 3

// Acceso al tema:
final cs   = Theme.of(context).colorScheme;
final text = Theme.of(context).textTheme;

// Validar Form:
if (_formKey.currentState!.validate()) { /* todos válidos */ }

// Limpiar Form:
_formKey.currentState?.reset();

// FocusNode:
_focusNode.requestFocus();          // dar foco
FocusScope.of(context).unfocus();   // cerrar teclado
```

---

## Cuándo usar qué lista

```
Widget                 Cuándo usarlo
─────────────────────  ─────────────────────────────────────────────────────
ListView               Pocos elementos estáticos conocidos en compilación
ListView.builder       Lista grande o dinámica — virtualiza el scroll
ListView.separated     Como builder pero con separadores automáticos
GridView.builder       Cuadrícula grande o dinámica — virtualiza igual
SliverGridDelegateWithFixedCrossAxisCount    Columnas fijas (2, 3, 4...)
SliverGridDelegateWithMaxCrossAxisExtent     Columnas adaptativas al ancho
```

---

## Ejercicios propuestos

1. **Formulario de reglas de firewall** — Crea un `Form` con los campos:
   protocolo (`DropdownButtonFormField`: TCP/UDP/ICMP), IP origen, IP destino,
   puerto destino y acción (ALLOW/DENY con `SegmentedButton`). Valida que
   los puertos estén entre 1 y 65535 y que las IPs sean IPv4 válidas.

2. **Lista con swipe to dismiss** — Envuelve cada `ListTile` en `Dismissible`.
   Deslizar a la derecha marca el servidor como favorito (fondo verde),
   deslizar a la izquierda lo elimina (fondo rojo). Usa `confirmDismiss`
   para mostrar un `AlertDialog` antes de eliminar.

3. **Grid adaptativo** — Modifica `PantallaBusqueda` para que use
   `SliverGridDelegateWithMaxCrossAxisExtent(maxCrossAxisExtent: 180)`
   en lugar de columnas fijas. Verifica que en tablet aparecen más columnas
   automáticamente sin código adicional.

4. **Búsqueda con debounce** — La búsqueda actual filtra en cada tecla.
   Implementa un debounce de 300ms usando `Timer` para que el filtrado
   se ejecute 300ms después de que el usuario deje de escribir.
   Usa `Timer.cancel()` para cancelar el timer anterior en cada tecla.

---

## Resumen de la página 9

- `TextField` + `TextEditingController` permite leer, escribir y limpiar texto programáticamente. El controller debe liberarse en `dispose()`.
- `FocusNode` controla qué campo tiene el foco. `TextInputAction.next` muestra "Siguiente" en el teclado; `onSubmitted` salta al siguiente campo con `focusNode.requestFocus()`.
- `Form` + `GlobalKey<FormState>` + `TextFormField`: `_formKey.currentState!.validate()` valida todos los campos. El `validator` devuelve `null` si válido o `String` con el mensaje de error.
- `DropdownButtonFormField` integra un selector dentro de un `Form`. `SwitchListTile` integra un switch con título y subtítulo en una sola fila.
- `ListView.builder` virtualiza la lista — solo construye los widgets visibles. Para listas estáticas pequeñas, `ListView` directo es suficiente.
- `ListView.separated` añade separadores entre elementos. `Divider(indent: 72)` alinea el separador con el texto, dejando espacio para el `leading`.
- `GridView.builder` con `SliverGridDelegateWithFixedCrossAxisCount` usa columnas fijas. `WithMaxCrossAxisExtent` adapta las columnas al ancho de pantalla automáticamente.
- `SearchBar` es el componente M3 para búsqueda. El filtrado usa un getter calculado sobre la lista original para no perder elementos al borrar la búsqueda.

---

> **Siguiente página →** Página 10: Estado con Riverpod —
> `ProviderScope`, `ConsumerWidget`, `Provider`, `StateNotifierProvider`
> y `AsyncNotifierProvider`.
