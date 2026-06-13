# Tutorial Flutter — Página 9
## Módulo 2 · Flutter
### Formularios y listas: `TextField`, `Form`, `ListView` y `GridView`

---

## `TextField` — entrada de texto

```dart
import 'package:flutter/material.dart';

// TextField básico
TextField(
  decoration: const InputDecoration(
    labelText:   'Hostname',
    hintText:    'ej. prod-web-01',
    prefixIcon:  Icon(Icons.dns),
    border:      OutlineInputBorder(),
  ),
  onChanged: (valor) => print('Escribiendo: $valor'),
  onSubmitted: (valor) => print('Enviado: $valor'),
)
```

### `TextEditingController` — controlar el valor

```dart
class _FormularioConController extends StatefulWidget {
  const _FormularioConController();

  @override
  State<_FormularioConController> createState() =>
      _FormularioConControllerState();
}

class _FormularioConControllerState
    extends State<_FormularioConController> {

  // El controller da acceso al texto, posición y selección
  final _ctrlHostname  = TextEditingController();
  final _ctrlIp        = TextEditingController();
  final _ctrlPuerto    = TextEditingController(text: '22');  // valor inicial
  final _foco          = FocusNode();

  @override
  void dispose() {
    // SIEMPRE limpiar controllers y FocusNodes
    _ctrlHostname.dispose();
    _ctrlIp.dispose();
    _ctrlPuerto.dispose();
    _foco.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        TextField(
          controller:     _ctrlHostname,
          focusNode:      _foco,
          decoration:     const InputDecoration(
            labelText:  'Hostname',
            prefixIcon: Icon(Icons.dns),
            border:     OutlineInputBorder(),
          ),
          textInputAction: TextInputAction.next,   // "Siguiente" en el teclado
          onSubmitted: (_) => FocusScope.of(context).nextFocus(),
        ),
        const SizedBox(height: 12),
        TextField(
          controller:      _ctrlIp,
          decoration:      const InputDecoration(
            labelText:   'Dirección IP',
            prefixIcon:  Icon(Icons.router),
            hintText:    '192.168.1.100',
            border:      OutlineInputBorder(),
          ),
          keyboardType:    TextInputType.number,
          textInputAction: TextInputAction.next,
          onSubmitted:     (_) => FocusScope.of(context).nextFocus(),
        ),
        const SizedBox(height: 12),
        TextField(
          controller:      _ctrlPuerto,
          decoration:      const InputDecoration(
            labelText:   'Puerto SSH',
            prefixIcon:  Icon(Icons.lock_outline),
            border:      OutlineInputBorder(),
          ),
          keyboardType:    TextInputType.number,
          textInputAction: TextInputAction.done,
        ),
        const SizedBox(height: 16),
        FilledButton(
          onPressed: () {
            print('${_ctrlHostname.text} — '
                  '${_ctrlIp.text}:${_ctrlPuerto.text}');
            // Limpiar campos
            _ctrlHostname.clear();
            _ctrlIp.clear();
            _ctrlPuerto.text = '22';
          },
          child: const Text('Conectar'),
        ),
      ],
    );
  }
}
```

---

## `Form` y `TextFormField` — validación integrada

`Form` agrupa campos y permite validarlos todos a la vez:

```dart
class FormularioServidor extends StatefulWidget {
  final void Function(Map<String, String> datos) onGuardar;

  const FormularioServidor({super.key, required this.onGuardar});

  @override
  State<FormularioServidor> createState() => _FormularioServidorState();
}

class _FormularioServidorState extends State<FormularioServidor> {
  // GlobalKey para acceder al estado del Form
  final _formKey = GlobalKey<FormState>();

  final _ctrlNombre   = TextEditingController();
  final _ctrlIp       = TextEditingController();
  final _ctrlPuerto   = TextEditingController(text: '22');
  final _ctrlUsuario  = TextEditingController(text: 'root');

  String _so          = 'Ubuntu 24.04';
  bool   _ssl         = true;

  // Expresión regular para validar IPv4
  static final _regexIp = RegExp(
    r'^(\d{1,3}\.){3}\d{1,3}$',
  );

  @override
  void dispose() {
    _ctrlNombre.dispose();
    _ctrlIp.dispose();
    _ctrlPuerto.dispose();
    _ctrlUsuario.dispose();
    super.dispose();
  }

  void _guardar() {
    // validate() llama al validator de TODOS los TextFormField
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

          // Nombre del servidor
          TextFormField(
            controller:  _ctrlNombre,
            decoration:  const InputDecoration(
              labelText:  'Nombre del servidor',
              hintText:   'prod-web-01',
              prefixIcon: Icon(Icons.dns),
              border:     OutlineInputBorder(),
            ),
            textInputAction: TextInputAction.next,
            validator: (v) {
              if (v == null || v.trim().isEmpty) {
                return 'El nombre es obligatorio';
              }
              if (v.length < 3) return 'Mínimo 3 caracteres';
              if (!RegExp(r'^[a-zA-Z0-9\-\_]+$').hasMatch(v)) {
                return 'Solo letras, números, guiones y guiones bajos';
              }
              return null;  // null = válido
            },
          ),
          const SizedBox(height: 12),

          // Dirección IP
          TextFormField(
            controller:      _ctrlIp,
            decoration:      const InputDecoration(
              labelText:   'Dirección IP',
              hintText:    '192.168.1.100',
              prefixIcon:  Icon(Icons.router),
              border:      OutlineInputBorder(),
            ),
            keyboardType:    TextInputType.number,
            textInputAction: TextInputAction.next,
            validator: (v) {
              if (v == null || v.isEmpty) return 'La IP es obligatoria';
              if (!_regexIp.hasMatch(v)) return 'Formato IPv4 inválido';
              // Validar cada octeto
              final octetos = v.split('.').map(int.parse).toList();
              if (octetos.any((o) => o > 255)) {
                return 'Octeto fuera de rango (0-255)';
              }
              return null;
            },
          ),
          const SizedBox(height: 12),

          // Puerto SSH
          TextFormField(
            controller:      _ctrlPuerto,
            decoration:      const InputDecoration(
              labelText:   'Puerto',
              prefixIcon:  Icon(Icons.lock_outline),
              border:      OutlineInputBorder(),
            ),
            keyboardType:    TextInputType.number,
            textInputAction: TextInputAction.next,
            validator: (v) {
              final puerto = int.tryParse(v ?? '');
              if (puerto == null) return 'Puerto debe ser un número';
              if (puerto < 1 || puerto > 65535) {
                return 'Puerto debe estar entre 1 y 65535';
              }
              return null;
            },
          ),
          const SizedBox(height: 12),

          // Sistema Operativo — DropdownButtonFormField
          DropdownButtonFormField<String>(
            value:       _so,
            decoration:  const InputDecoration(
              labelText:  'Sistema Operativo',
              prefixIcon: Icon(Icons.computer),
              border:     OutlineInputBorder(),
            ),
            items: [
              'Ubuntu 24.04', 'Debian 12', 'CentOS Stream 9',
              'Rocky Linux 9', 'Alpine Linux',
            ].map((so) => DropdownMenuItem(value: so, child: Text(so)))
             .toList(),
            onChanged: (v) => setState(() => _so = v!),
          ),
          const SizedBox(height: 8),

          // SSL — SwitchListTile
          SwitchListTile(
            title:    const Text('Conexión SSL/TLS'),
            subtitle: const Text('Cifrar la comunicación'),
            value:    _ssl,
            onChanged: (v) => setState(() => _ssl = v),
            secondary: const Icon(Icons.security),
          ),

          const SizedBox(height: 16),

          // Botones
          Row(
            children: [
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
            ],
          ),
        ],
      ),
    );
  }
}
```

---

## `ListView` — listas eficientes

```dart
// ListView básico — para pocas filas estáticas
ListView(
  children: const [
    ListTile(title: Text('Elemento 1')),
    ListTile(title: Text('Elemento 2')),
    Divider(),
    ListTile(title: Text('Elemento 3')),
  ],
)

// ListView.builder — virtualizado, eficiente para muchos elementos
// Solo construye los elementos visibles en pantalla
ListView.builder(
  itemCount:   servidores.length,
  itemBuilder: (context, index) {
    final s = servidores[index];
    return TarjetaServidor(servidor: s);
  },
)

// ListView.separated — con separadores entre elementos
ListView.separated(
  itemCount:        servidores.length,
  separatorBuilder: (_, __) => const Divider(height: 1),
  itemBuilder:      (context, index) =>
      ListTile(title: Text(servidores[index])),
)
```

### `ListTile` — fila estándar de lista

```dart
// ListTile es el componente estándar para filas de lista en M3
ListTile(
  leading:    const CircleAvatar(child: Icon(Icons.dns)),
  title:       Text('prod-web-01'),
  subtitle:    Text('10.0.2.10 · Ubuntu 24.04'),
  trailing:    Row(
    mainAxisSize: MainAxisSize.min,
    children: [
      const Icon(Icons.circle, color: Colors.green, size: 10),
      const SizedBox(width: 4),
      Text('Activo', style: TextStyle(color: Colors.green.shade700)),
    ],
  ),
  onTap: () {},
  contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
)
```

---

## `GridView` — cuadrícula

```dart
// GridView.builder — virtualizado para muchos elementos
GridView.builder(
  padding:     const EdgeInsets.all(12),
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount:   2,     // columnas
    childAspectRatio: 1.4,   // ancho / alto de cada celda
    crossAxisSpacing: 8,
    mainAxisSpacing:  8,
  ),
  itemCount:   servidores.length,
  itemBuilder: (context, i) => TarjetaServidor(servidor: servidores[i]),
)

// Con columnas adaptativas — se ajusta al ancho de pantalla
GridView.builder(
  gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
    maxCrossAxisExtent: 200,  // cada columna hasta 200px de ancho
    childAspectRatio:   1.2,
    crossAxisSpacing:   8,
    mainAxisSpacing:    8,
  ),
  itemCount:   items.length,
  itemBuilder: (_, i) => ItemWidget(item: items[i]),
)
```

---

## Programa completo — gestor de servidores SSH

```dart
import 'package:flutter/material.dart';

void main() => runApp(const AppGestorSSH());

class AppGestorSSH extends StatelessWidget {
  const AppGestorSSH({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Gestor SSH',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF1B5E20)),
        useMaterial3: true,
      ),
      home: const PantallaGestorSSH(),
    );
  }
}

// Modelo
class ServidorSSH {
  final String id;
  final String nombre;
  final String ip;
  final int    puerto;
  final String usuario;
  final String so;
  final bool   ssl;
  bool         favorito;

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

class PantallaGestorSSH extends StatefulWidget {
  const PantallaGestorSSH({super.key});

  @override
  State<PantallaGestorSSH> createState() => _PantallaGestorSSHState();
}

class _PantallaGestorSSHState extends State<PantallaGestorSSH> {
  final List<ServidorSSH> _servidores = [
    ServidorSSH(id: '1', nombre: 'prod-web-01',  ip: '10.0.2.10',  puerto: 22,   usuario: 'deploy',  so: 'Ubuntu 24.04', ssl: true,  favorito: true),
    ServidorSSH(id: '2', nombre: 'prod-db-01',   ip: '10.0.2.20',  puerto: 22,   usuario: 'postgres', so: 'Debian 12',   ssl: true),
    ServidorSSH(id: '3', nombre: 'staging-api',  ip: '10.0.3.10',  puerto: 2222, usuario: 'ubuntu',   so: 'Ubuntu 24.04', ssl: false),
    ServidorSSH(id: '4', nombre: 'dev-sandbox',  ip: '192.168.1.5',puerto: 22,   usuario: 'vagrant',  so: 'Alpine Linux', ssl: false),
  ];

  String _busqueda    = '';
  bool   _mostrarForm = false;
  String _vistaActual = 'lista';  // 'lista' o 'grid'

  List<ServidorSSH> get _filtrados => _servidores
      .where((s) =>
          s.nombre.toLowerCase().contains(_busqueda.toLowerCase()) ||
          s.ip.contains(_busqueda))
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

    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text('Servidor "${datos['nombre']}" agregado'),
        behavior: SnackBarBehavior.floating,
      ),
    );
  }

  Future<void> _confirmarEliminar(ServidorSSH s) async {
    final confirma = await showDialog<bool>(
      context: context,
      builder: (ctx) => AlertDialog(
        icon:  const Icon(Icons.warning_amber, color: Colors.orange),
        title: const Text('Eliminar servidor'),
        content: Text('¿Eliminar "${s.nombre}" (${s.ip})?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx, false),
            child: const Text('Cancelar'),
          ),
          FilledButton(
            style: FilledButton.styleFrom(
              backgroundColor: Theme.of(context).colorScheme.error,
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
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: _mostrarForm
            ? const Text('Nuevo servidor')
            : Text('Servidores SSH (${_servidores.length})'),
        backgroundColor: cs.primaryContainer,
        leading: _mostrarForm
            ? IconButton(
                icon:      const Icon(Icons.arrow_back),
                onPressed: () => setState(() => _mostrarForm = false),
              )
            : null,
        actions: _mostrarForm
            ? []
            : [
                // Toggle lista / grid
                IconButton(
                  icon: Icon(_vistaActual == 'lista'
                      ? Icons.grid_view
                      : Icons.list),
                  onPressed: () => setState(() =>
                      _vistaActual = _vistaActual == 'lista' ? 'grid' : 'lista'),
                  tooltip: 'Cambiar vista',
                ),
              ],
      ),

      body: _mostrarForm
          // ── Vista formulario ──────────────────────────────────
          ? SingleChildScrollView(
              padding: const EdgeInsets.all(16),
              child: FormularioServidor(onGuardar: _agregarServidor),
            )
          // ── Vista lista / grid ───────────────────────────────
          : Column(
              children: [
                // Buscador
                Padding(
                  padding: const EdgeInsets.fromLTRB(16, 12, 16, 8),
                  child: SearchBar(
                    hintText: 'Buscar por nombre o IP...',
                    leading: const Icon(Icons.search),
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

                // Lista o Grid
                Expanded(
                  child: _filtrados.isEmpty
                      ? Center(
                          child: Column(
                            mainAxisSize: MainAxisSize.min,
                            children: [
                              Icon(Icons.search_off,
                                  size: 56,
                                  color: cs.onSurfaceVariant),
                              const SizedBox(height: 12),
                              Text(
                                'Sin resultados para "$_busqueda"',
                                style: TextStyle(color: cs.onSurfaceVariant),
                              ),
                            ],
                          ),
                        )
                      : _vistaActual == 'lista'
                          ? ListView.separated(
                              itemCount:        _filtrados.length,
                              separatorBuilder: (_, __) =>
                                  const Divider(height: 1, indent: 72),
                              itemBuilder: (context, i) =>
                                  _FilaServidor(
                                servidor:   _filtrados[i],
                                onFavorito: () => setState(() =>
                                    _filtrados[i].favorito =
                                        !_filtrados[i].favorito),
                                onEliminar: () =>
                                    _confirmarEliminar(_filtrados[i]),
                              ),
                            )
                          : GridView.builder(
                              padding: const EdgeInsets.all(12),
                              gridDelegate:
                                  const SliverGridDelegateWithFixedCrossAxisCount(
                                crossAxisCount:   2,
                                childAspectRatio: 1.1,
                                crossAxisSpacing: 8,
                                mainAxisSpacing:  8,
                              ),
                              itemCount:   _filtrados.length,
                              itemBuilder: (context, i) =>
                                  _TarjetaServidorGrid(
                                servidor:   _filtrados[i],
                                onFavorito: () => setState(() =>
                                    _filtrados[i].favorito =
                                        !_filtrados[i].favorito),
                                onEliminar: () =>
                                    _confirmarEliminar(_filtrados[i]),
                              ),
                            ),
                ),
              ],
            ),

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

// Fila para vista lista
class _FilaServidor extends StatelessWidget {
  final ServidorSSH  servidor;
  final VoidCallback onFavorito;
  final VoidCallback onEliminar;

  const _FilaServidor({
    required this.servidor,
    required this.onFavorito,
    required this.onEliminar,
  });

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: CircleAvatar(
        backgroundColor: servidor.ssl
            ? Colors.green.shade100
            : Colors.grey.shade100,
        child: Icon(
          Icons.dns,
          color: servidor.ssl ? Colors.green.shade700 : Colors.grey,
        ),
      ),
      title: Text(
        servidor.nombre,
        style: const TextStyle(fontWeight: FontWeight.w600),
      ),
      subtitle: Text(
        '${servidor.usuario}@${servidor.ip}:${servidor.puerto}',
        style: const TextStyle(fontSize: 12),
      ),
      trailing: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          IconButton(
            icon: Icon(
              servidor.favorito ? Icons.star : Icons.star_border,
              color: servidor.favorito ? Colors.amber : null,
            ),
            onPressed: onFavorito,
            visualDensity: VisualDensity.compact,
          ),
          IconButton(
            icon: const Icon(Icons.delete_outline, color: Colors.red),
            onPressed: onEliminar,
            visualDensity: VisualDensity.compact,
          ),
        ],
      ),
    );
  }
}

// Tarjeta para vista grid
class _TarjetaServidorGrid extends StatelessWidget {
  final ServidorSSH  servidor;
  final VoidCallback onFavorito;
  final VoidCallback onEliminar;

  const _TarjetaServidorGrid({
    required this.servidor,
    required this.onFavorito,
    required this.onEliminar,
  });

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(10),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Icon(Icons.dns,
                    color: servidor.ssl ? cs.primary : cs.outline,
                    size: 18),
                const Spacer(),
                GestureDetector(
                  onTap: onFavorito,
                  child: Icon(
                    servidor.favorito ? Icons.star : Icons.star_border,
                    color: servidor.favorito ? Colors.amber : cs.outline,
                    size: 18,
                  ),
                ),
              ],
            ),
            const SizedBox(height: 6),
            Text(
              servidor.nombre,
              style: const TextStyle(
                  fontWeight: FontWeight.bold, fontSize: 13),
              maxLines: 1,
              overflow: TextOverflow.ellipsis,
            ),
            Text(
              servidor.ip,
              style: TextStyle(fontSize: 11, color: cs.onSurfaceVariant),
            ),
            const Spacer(),
            Row(
              children: [
                if (servidor.ssl)
                  Icon(Icons.lock, size: 12, color: cs.primary),
                const SizedBox(width: 4),
                Expanded(
                  child: Text(
                    servidor.so,
                    style: TextStyle(
                        fontSize: 10, color: cs.onSurfaceVariant),
                    overflow: TextOverflow.ellipsis,
                  ),
                ),
                GestureDetector(
                  onTap: onEliminar,
                  child: Icon(Icons.delete_outline,
                      size: 16, color: cs.error),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## Ejercicios propuestos

1. **Formulario de reglas de firewall** — Crea un `Form` con los campos:
   protocolo (`DropdownButtonFormField`: TCP/UDP/ICMP), IP origen, IP destino,
   puerto destino y acción (ALLOW/DENY con `SegmentedButton`). Valida que
   los puertos estén entre 1 y 65535 y que las IPs sean IPv4 válidas.

2. **Lista con swipe to dismiss** — Usa `Dismissible` envolviendo cada
   `ListTile`. Deslizar a la derecha marca el servidor como favorito
   (fondo verde), deslizar a la izquierda lo elimina (fondo rojo).
   Usa `onDismissed` para actualizar el estado y `confirmDismiss` para
   mostrar un `AlertDialog` antes de eliminar.

3. **Grid adaptativo** — Modifica la vista grid del programa completo para
   que use `SliverGridDelegateWithMaxCrossAxisExtent(maxCrossAxisExtent: 180)`
   en lugar de columnas fijas. Verifica que en tablet aparecen más columnas
   automáticamente sin código adicional.

4. **Búsqueda con debounce** — La búsqueda actual filtra en cada tecla.
   Implementa un debounce de 300ms usando `Timer` para que la búsqueda
   solo se ejecute 300ms después de que el usuario deje de escribir.
   Usa `Timer.cancel()` para cancelar el timer anterior en cada keystroke.

---

## Resumen de la página 9

- `TextField` con `TextEditingController` permite leer, escribir y limpiar el texto programáticamente. El controller debe liberarse en `dispose()`.
- `FocusNode` controla el foco entre campos. `FocusScope.of(context).nextFocus()` mueve al siguiente campo como hace el botón "Siguiente" del teclado.
- `Form` + `GlobalKey<FormState>` + `TextFormField` permite validar todos los campos con un solo `_formKey.currentState!.validate()`. El `validator` devuelve `null` si es válido o el mensaje de error.
- `DropdownButtonFormField` es el selector dentro de un `Form`. `SwitchListTile` integra un switch con título y subtítulo en una fila.
- `ListView.builder` virtualiza la lista — solo construye los widgets visibles. Para pocos elementos estáticos, `ListView` directo es suficiente.
- `ListView.separated` añade separadores entre elementos sin código extra.
- `GridView.builder` con `SliverGridDelegateWithFixedCrossAxisCount` usa columnas fijas. `SliverGridDelegateWithMaxCrossAxisExtent` adapta las columnas automáticamente al ancho de la pantalla.
- `SearchBar` es el componente M3 recomendado para búsqueda — reemplaza el `TextField` en `AppBar`.

---

> **Siguiente página →** Página 10: Estado con Riverpod —
> `ProviderScope`, `ConsumerWidget`, `Provider`, `StateNotifierProvider`
> y `AsyncNotifierProvider`.