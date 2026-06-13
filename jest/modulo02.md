# Tutorial Jest — Página 2
## Módulo 2 · Matchers
### Los comparadores de Jest para verificar cualquier tipo de dato

---

## ¿Qué es un matcher?

Un **matcher** es el método que va después de `expect()` y define
cómo se compara el valor real con el valor esperado.

```javascript
expect(valorReal).matcher(valorEsperado);
//                ↑ aquí va el matcher
```

Cada sección de esta página tiene dos archivos:
un archivo `src/` con las funciones a probar y un archivo
`tests/` con los tests que usan el matcher explicado.

---

## Matchers de igualdad: `toBe` y `toEqual`

### `toBe` — igualdad estricta (===)

Usa `toBe` para valores primitivos: números, strings, booleanos.
Funciona igual que el operador `===` de JavaScript.

```javascript
// src/calculadora.js

function sumar(a, b) {
  return a + b;
}

function restar(a, b) {
  return a - b;
}

function multiplicar(a, b) {
  return a * b;
}

// Devuelve true si el número es par, false si es impar
function esPar(numero) {
  return numero % 2 === 0;
}

// Devuelve el saludo según la hora del día (0-23)
function obtenerSaludo(hora) {
  if (hora < 12) return 'Buenos días';
  if (hora < 20) return 'Buenas tardes';
  return 'Buenas noches';
}

module.exports = { sumar, restar, multiplicar, esPar, obtenerSaludo };
```

```javascript
// tests/calculadora.test.js
const { sumar, restar, multiplicar, esPar, obtenerSaludo } = require('../src/calculadora');

// toBe compara con === — ideal para números, strings y booleanos

test('sumar 3 + 4 devuelve 7', () => {
  expect(sumar(3, 4)).toBe(7);
});

test('restar 10 - 6 devuelve 4', () => {
  expect(restar(10, 6)).toBe(4);
});

test('multiplicar 5 × 3 devuelve 15', () => {
  expect(multiplicar(5, 3)).toBe(15);
});

test('esPar devuelve true para números pares', () => {
  expect(esPar(8)).toBe(true);
  expect(esPar(0)).toBe(true);
});

test('esPar devuelve false para números impares', () => {
  expect(esPar(7)).toBe(false);
});

test('obtenerSaludo a las 9h devuelve Buenos días', () => {
  expect(obtenerSaludo(9)).toBe('Buenos días');
});

test('obtenerSaludo a las 15h devuelve Buenas tardes', () => {
  expect(obtenerSaludo(15)).toBe('Buenas tardes');
});

test('obtenerSaludo a las 22h devuelve Buenas noches', () => {
  expect(obtenerSaludo(22)).toBe('Buenas noches');
});
```

> **Importante:** `toBe` NO funciona con objetos ni arrays porque
> compara la referencia en memoria, no el contenido.
> Para objetos y arrays usa siempre `toEqual`.

---

### `toEqual` — igualdad profunda de contenido

Compara el **contenido completo** de objetos y arrays,
sin importar si son referencias distintas en memoria.

```javascript
// src/carrito.js

// Crea un objeto producto con su total ya calculado
function crearProducto(nombre, precio, cantidad) {
  return {
    nombre,
    precio,
    cantidad,
    total: precio * cantidad,
  };
}

// Recibe un array de productos y devuelve los que cuestan menos que el límite
function filtrarBaratos(productos, limite) {
  return productos.filter(p => p.precio < limite);
}

// Devuelve el nombre de los productos en un array de strings
function obtenerNombres(productos) {
  return productos.map(p => p.nombre);
}

module.exports = { crearProducto, filtrarBaratos, obtenerNombres };
```

```javascript
// tests/carrito.test.js
const { crearProducto, filtrarBaratos, obtenerNombres } = require('../src/carrito');

// toEqual compara el contenido — perfecto para objetos y arrays

test('crearProducto devuelve el objeto correcto con su total', () => {
  const producto = crearProducto('Teclado', 50, 2);

  expect(producto).toEqual({
    nombre:   'Teclado',
    precio:   50,
    cantidad: 2,
    total:    100,
  });
});

test('filtrarBaratos devuelve solo los productos bajo el límite', () => {
  const productos = [
    { nombre: 'Ratón',   precio: 20 },
    { nombre: 'Teclado', precio: 50 },
    { nombre: 'Cable',   precio: 8  },
  ];

  const baratos = filtrarBaratos(productos, 30);

  expect(baratos).toEqual([
    { nombre: 'Ratón', precio: 20 },
    { nombre: 'Cable', precio: 8  },
  ]);
});

test('obtenerNombres devuelve un array de strings con los nombres', () => {
  const productos = [
    { nombre: 'Ratón',   precio: 20 },
    { nombre: 'Teclado', precio: 50 },
  ];

  expect(obtenerNombres(productos)).toEqual(['Ratón', 'Teclado']);
});

test('filtrarBaratos con límite muy bajo devuelve array vacío', () => {
  const productos = [{ nombre: 'Monitor', precio: 300 }];
  expect(filtrarBaratos(productos, 10)).toEqual([]);
});
```

---

## Matchers de verdad y falsedad

### `toBeTruthy`, `toBeFalsy`, `toBeNull`, `toBeDefined`

```javascript
// src/sesion.js

// Devuelve el usuario si el token es válido, o null si no lo es
function obtenerUsuarioPorToken(token, usuarios) {
  if (!token) return null;
  return usuarios.find(u => u.token === token) ?? null;
}

// Devuelve el mensaje de bienvenida si el usuario existe, o undefined si no
function mensajeBienvenida(usuario) {
  if (!usuario) return undefined;
  return `Bienvenido, ${usuario.nombre}`;
}

// Devuelve true si el usuario tiene sesión activa, false si no
function tieneSesionActiva(usuario) {
  return Boolean(usuario && usuario.activo);
}

module.exports = { obtenerUsuarioPorToken, mensajeBienvenida, tieneSesionActiva };
```

```javascript
// tests/sesion.test.js
const { obtenerUsuarioPorToken, mensajeBienvenida, tieneSesionActiva } = require('../src/sesion');

const usuarios = [
  { nombre: 'Ana',  token: 'abc123', activo: true  },
  { nombre: 'Luis', token: 'xyz789', activo: false },
];

// toBeDefined — el valor NO es undefined
test('token válido devuelve un usuario (no undefined)', () => {
  const usuario = obtenerUsuarioPorToken('abc123', usuarios);
  expect(usuario).toBeDefined();
});

// toBeNull — el valor ES exactamente null
test('token inválido devuelve null', () => {
  const usuario = obtenerUsuarioPorToken('tokenFalso', usuarios);
  expect(usuario).toBeNull();
});

// toBeNull — token vacío también devuelve null
test('token vacío devuelve null', () => {
  expect(obtenerUsuarioPorToken('', usuarios)).toBeNull();
});

// not.toBeNull — el valor NO es null
test('token correcto devuelve un usuario que no es null', () => {
  expect(obtenerUsuarioPorToken('abc123', usuarios)).not.toBeNull();
});

// toBeUndefined — el valor ES exactamente undefined
test('mensajeBienvenida sin usuario devuelve undefined', () => {
  expect(mensajeBienvenida(null)).toBeUndefined();
});

// toBeDefined — el valor NO es undefined
test('mensajeBienvenida con usuario devuelve un string definido', () => {
  const usuario = { nombre: 'Ana' };
  expect(mensajeBienvenida(usuario)).toBeDefined();
});

// toBeTruthy — cualquier valor "verdadero" (no vacío, no cero, no null...)
test('usuario con sesión activa → truthy', () => {
  expect(tieneSesionActiva(usuarios[0])).toBeTruthy();
});

// toBeFalsy — false, 0, null, undefined, '' o NaN
test('usuario con sesión inactiva → falsy', () => {
  expect(tieneSesionActiva(usuarios[1])).toBeFalsy();
});

test('sin usuario → falsy', () => {
  expect(tieneSesionActiva(null)).toBeFalsy();
});
```

### El operador `.not`

`.not` invierte cualquier matcher. Funciona con todos ellos:

```javascript
// El mismo archivo tests/sesion.test.js puede incluir estos tests

test('not.toBe — el saludo de la mañana no es el de la noche', () => {
  const { obtenerSaludo } = require('../src/calculadora');
  expect(obtenerSaludo(9)).not.toBe('Buenas noches');
});

test('not.toBeNull — usuario encontrado no es null', () => {
  const usuario = obtenerUsuarioPorToken('abc123', usuarios);
  expect(usuario).not.toBeNull();
});

test('not.toBeTruthy — sesión inactiva no es truthy', () => {
  expect(tieneSesionActiva(usuarios[1])).not.toBeTruthy();
});
```

---

## Matchers numéricos

### `toBeGreaterThan`, `toBeLessThan`, `toBeCloseTo`

```javascript
// src/notas.js

// Calcula el promedio de un array de números
function calcularPromedio(numeros) {
  if (numeros.length === 0) return 0;
  const suma = numeros.reduce((acc, n) => acc + n, 0);
  return suma / numeros.length;
}

// Calcula el precio final aplicando un porcentaje de descuento
function aplicarDescuento(precio, porcentaje) {
  return precio * (1 - porcentaje / 100);
}

// Devuelve la nota más alta del array
function notaMaxima(notas) {
  return Math.max(...notas);
}

// Devuelve la nota más baja del array
function notaMinima(notas) {
  return Math.min(...notas);
}

module.exports = { calcularPromedio, aplicarDescuento, notaMaxima, notaMinima };
```

```javascript
// tests/notas.test.js
const { calcularPromedio, aplicarDescuento, notaMaxima, notaMinima } = require('../src/notas');

// toBeGreaterThan — el valor es mayor que el esperado
test('el promedio de [6, 8, 10] es mayor que 7', () => {
  expect(calcularPromedio([6, 8, 10])).toBeGreaterThan(7);
});

// toBeGreaterThanOrEqual — mayor o igual
test('la nota máxima de [5, 7, 9] es mayor o igual que 9', () => {
  expect(notaMaxima([5, 7, 9])).toBeGreaterThanOrEqual(9);
});

// toBeLessThan — el valor es menor que el esperado
test('la nota mínima de [5, 7, 9] es menor que 6', () => {
  expect(notaMinima([5, 7, 9])).toBeLessThan(6);
});

// toBeLessThanOrEqual — menor o igual
test('el descuento nunca eleva el precio original', () => {
  const precioFinal = aplicarDescuento(100, 10);
  expect(precioFinal).toBeLessThanOrEqual(100);
});

// toBeCloseTo — comparar decimales (0.1 + 0.2 ≠ 0.3 exacto en JS)
test('promedio de [1, 2] es cercano a 1.5', () => {
  // Sin toBeCloseTo algunos decimales fallan por precisión de punto flotante
  expect(calcularPromedio([1, 2])).toBeCloseTo(1.5, 1);
});

test('descuento del 33% sobre 100 es cercano a 67', () => {
  // 100 * (1 - 33/100) = 67.00000... — seguro con toBeCloseTo
  expect(aplicarDescuento(100, 33)).toBeCloseTo(67, 2);
});

test('promedio con decimales — toBeCloseTo evita errores de precisión', () => {
  // 1/3 + 2/3 + ... pueden acumular error de punto flotante
  expect(calcularPromedio([0.1, 0.2, 0.3])).toBeCloseTo(0.2, 5);
});
```

---

## Matchers de strings: `toMatch`

```javascript
// src/mensajes.js

// Devuelve un mensaje de error formateado
function formatearError(codigo, detalle) {
  return `[ERROR-${codigo}] ${detalle}`;
}

// Devuelve true si el email tiene formato válido (básico)
function esEmailValido(email) {
  return email.includes('@') && email.includes('.');
}

// Genera un código de pedido con fecha y número
function generarCodigoPedido(numero) {
  const hoy = new Date().toISOString().slice(0, 10);   // "2024-03-15"
  return `PED-${hoy}-${String(numero).padStart(4, '0')}`;
}

module.exports = { formatearError, esEmailValido, generarCodigoPedido };
```

```javascript
// tests/mensajes.test.js
const { formatearError, esEmailValido, generarCodigoPedido } = require('../src/mensajes');

// toMatch con substring — verifica que el string contiene ese texto
test('formatearError incluye el código en el mensaje', () => {
  const mensaje = formatearError(404, 'Página no encontrada');
  expect(mensaje).toMatch('404');
});

test('formatearError incluye el detalle en el mensaje', () => {
  const mensaje = formatearError(500, 'Error interno del servidor');
  expect(mensaje).toMatch('Error interno del servidor');
});

test('formatearError empieza con [ERROR-', () => {
  const mensaje = formatearError(403, 'Sin permisos');
  // toMatch con expresión regular
  expect(mensaje).toMatch(/^\[ERROR-/);
});

// toMatch con regex — patrón más flexible
test('el código de pedido tiene el formato correcto', () => {
  const codigo = generarCodigoPedido(1);
  // Formato esperado: PED-YYYY-MM-DD-0001
  expect(codigo).toMatch(/^PED-\d{4}-\d{2}-\d{2}-\d{4}$/);
});

test('el código de pedido contiene el número con ceros a la izquierda', () => {
  expect(generarCodigoPedido(7)).toMatch('0007');
  expect(generarCodigoPedido(42)).toMatch('0042');
});

// not.toMatch — el string NO contiene ese texto
test('email inválido sin @ — no contiene símbolo de arroba', () => {
  // esEmailValido devuelve boolean, pero aquí probamos el resultado con toMatch
  // en un string generado
  const mensaje = formatearError(422, 'Email sin arroba');
  expect(mensaje).not.toMatch('200');   // no es un código de éxito
});
```

---

## Matchers de arrays

### `toContain` y `toHaveLength`

```javascript
// src/permisos.js

// Devuelve la lista de acciones permitidas según el rol
function obtenerPermisos(rol) {
  const tabla = {
    admin:     ['leer', 'escribir', 'eliminar', 'configurar'],
    editor:    ['leer', 'escribir'],
    visitante: ['leer'],
  };
  return tabla[rol] ?? [];
}

// Devuelve true si el rol puede realizar la acción
function puedeRealizarAccion(rol, accion) {
  return obtenerPermisos(rol).includes(accion);
}

// Devuelve los roles que tienen al menos un permiso
function rolesConAcceso() {
  return ['admin', 'editor', 'visitante'];
}

module.exports = { obtenerPermisos, puedeRealizarAccion, rolesConAcceso };
```

```javascript
// tests/permisos.test.js
const { obtenerPermisos, puedeRealizarAccion, rolesConAcceso } = require('../src/permisos');

// toContain — el array contiene ese elemento primitivo (string, número...)
test('el admin tiene el permiso "eliminar"', () => {
  expect(obtenerPermisos('admin')).toContain('eliminar');
});

test('el editor tiene el permiso "leer"', () => {
  expect(obtenerPermisos('editor')).toContain('leer');
});

// not.toContain — el array NO contiene ese elemento
test('el editor NO tiene el permiso "eliminar"', () => {
  expect(obtenerPermisos('editor')).not.toContain('eliminar');
});

test('el visitante NO tiene el permiso "escribir"', () => {
  expect(obtenerPermisos('visitante')).not.toContain('escribir');
});

// toHaveLength — el array o string tiene esa longitud
test('el admin tiene exactamente 4 permisos', () => {
  expect(obtenerPermisos('admin')).toHaveLength(4);
});

test('el visitante tiene exactamente 1 permiso', () => {
  expect(obtenerPermisos('visitante')).toHaveLength(1);
});

test('rol desconocido devuelve array vacío', () => {
  expect(obtenerPermisos('desconocido')).toHaveLength(0);
});

// toContain en el array de roles
test('la lista de roles contiene "editor"', () => {
  expect(rolesConAcceso()).toContain('editor');
});

test('la lista de roles tiene 3 elementos', () => {
  expect(rolesConAcceso()).toHaveLength(3);
});
```

### `toContainEqual` — objetos dentro de arrays

```javascript
// src/tienda.js

// Devuelve el catálogo completo de productos
function obtenerCatalogo() {
  return [
    { id: 1, nombre: 'Ratón',    precio: 25, categoria: 'periféricos' },
    { id: 2, nombre: 'Teclado',  precio: 45, categoria: 'periféricos' },
    { id: 3, nombre: 'Monitor',  precio: 280, categoria: 'pantallas'  },
    { id: 4, nombre: 'Cable USB', precio: 8,  categoria: 'accesorios' },
  ];
}

// Devuelve los productos de una categoría
function filtrarPorCategoria(productos, categoria) {
  return productos.filter(p => p.categoria === categoria);
}

module.exports = { obtenerCatalogo, filtrarPorCategoria };
```

```javascript
// tests/tienda.test.js
const { obtenerCatalogo, filtrarPorCategoria } = require('../src/tienda');

// toContainEqual — el array contiene un objeto con ese contenido
test('el catálogo contiene el producto Teclado', () => {
  const catalogo = obtenerCatalogo();

  expect(catalogo).toContainEqual({
    id: 2, nombre: 'Teclado', precio: 45, categoria: 'periféricos',
  });
});

test('el catálogo NO contiene un producto con id=99', () => {
  const catalogo = obtenerCatalogo();

  expect(catalogo).not.toContainEqual(
    expect.objectContaining({ id: 99 })
  );
});

test('filtrar periféricos devuelve ratón y teclado', () => {
  const catalogo = obtenerCatalogo();
  const perifericos = filtrarPorCategoria(catalogo, 'periféricos');

  expect(perifericos).toHaveLength(2);
  expect(perifericos).toContainEqual(
    expect.objectContaining({ nombre: 'Ratón' })
  );
  expect(perifericos).toContainEqual(
    expect.objectContaining({ nombre: 'Teclado' })
  );
});
```

---

## Matchers de objetos

### `toHaveProperty` y `expect.objectContaining`

```javascript
// src/usuarios.js

// Crea un objeto usuario con todos sus campos
function crearUsuario(nombre, email) {
  return {
    id:        Math.floor(Math.random() * 1000),  // número dinámico
    nombre,
    email:     email.toLowerCase(),
    activo:    true,
    creadoEn:  new Date().toISOString(),          // fecha dinámica
    direccion: {
      ciudad: 'Sin asignar',
      cp:     '00000',
    },
  };
}

// Actualiza la ciudad del usuario y devuelve el objeto modificado
function actualizarCiudad(usuario, ciudad) {
  return { ...usuario, direccion: { ...usuario.direccion, ciudad } };
}

module.exports = { crearUsuario, actualizarCiudad };
```

```javascript
// tests/usuarios.test.js
const { crearUsuario, actualizarCiudad } = require('../src/usuarios');

// toHaveProperty — el objeto tiene esa propiedad (y opcionalmente ese valor)
test('el usuario creado tiene la propiedad nombre', () => {
  const usuario = crearUsuario('Ana', 'ana@ejemplo.com');
  expect(usuario).toHaveProperty('nombre');
});

test('el email se guarda en minúsculas', () => {
  const usuario = crearUsuario('Ana', 'ANA@EJEMPLO.COM');
  expect(usuario).toHaveProperty('email', 'ana@ejemplo.com');
});

test('el usuario empieza activo', () => {
  const usuario = crearUsuario('Luis', 'luis@ejemplo.com');
  expect(usuario).toHaveProperty('activo', true);
});

// Propiedad anidada con punto
test('el usuario tiene dirección con ciudad', () => {
  const usuario = crearUsuario('Ana', 'ana@ejemplo.com');
  expect(usuario).toHaveProperty('direccion.ciudad', 'Sin asignar');
});

test('actualizarCiudad cambia la ciudad correctamente', () => {
  const usuario = crearUsuario('Ana', 'ana@ejemplo.com');
  const actualizado = actualizarCiudad(usuario, 'Madrid');
  expect(actualizado).toHaveProperty('direccion.ciudad', 'Madrid');
});

// expect.objectContaining — verifica solo las propiedades que nos importan
// Útil cuando el objeto tiene campos dinámicos (id, fechas)
test('el usuario tiene el nombre y email correctos (sin importar el id)', () => {
  const usuario = crearUsuario('Ana', 'ana@ejemplo.com');

  // id y creadoEn son dinámicos — no los verificamos
  expect(usuario).toEqual(
    expect.objectContaining({
      nombre: 'Ana',
      email:  'ana@ejemplo.com',
      activo: true,
    })
  );
});

test('el array de usuarios contiene a Ana (búsqueda parcial)', () => {
  const lista = [
    crearUsuario('Ana',  'ana@ejemplo.com'),
    crearUsuario('Luis', 'luis@ejemplo.com'),
  ];

  expect(lista).toContainEqual(
    expect.objectContaining({ nombre: 'Ana' })
  );
});
```

---

## Matchers de errores: `toThrow`

```javascript
// src/validador.js

// Valida que la edad sea un número entre 0 y 120
function validarEdad(edad) {
  if (typeof edad !== 'number') {
    throw new TypeError('La edad debe ser un número');
  }
  if (edad < 0 || edad > 120) {
    throw new RangeError('La edad debe estar entre 0 y 120');
  }
  return true;
}

// Valida que el nombre no esté vacío y tenga al menos 2 caracteres
function validarNombre(nombre) {
  if (!nombre || nombre.trim() === '') {
    throw new Error('El nombre no puede estar vacío');
  }
  if (nombre.trim().length < 2) {
    throw new Error('El nombre debe tener al menos 2 caracteres');
  }
  return true;
}

// Divide dos números — lanza error si el divisor es cero
function dividir(a, b) {
  if (b === 0) {
    throw new Error('No se puede dividir entre cero');
  }
  return a / b;
}

module.exports = { validarEdad, validarNombre, dividir };
```

```javascript
// tests/validador.test.js
const { validarEdad, validarNombre, dividir } = require('../src/validador');

// IMPORTANTE: siempre envuelve la llamada en una función flecha → () =>
// Sin la función flecha Jest no puede capturar el error

// toThrow() — lanza ALGÚN error (sin importar cuál)
test('edad negativa lanza algún error', () => {
  expect(() => validarEdad(-1)).toThrow();
});

// toThrow('mensaje') — lanza un error con ese mensaje exacto
test('edad mayor a 120 lanza RangeError con el mensaje correcto', () => {
  expect(() => validarEdad(200)).toThrow('La edad debe estar entre 0 y 120');
});

// toThrow(TypeError) — verifica el tipo de error
test('edad como string lanza TypeError', () => {
  expect(() => validarEdad('veinte')).toThrow(TypeError);
});

// toThrow(RangeError)
test('edad fuera de rango lanza RangeError', () => {
  expect(() => validarEdad(-5)).toThrow(RangeError);
  expect(() => validarEdad(150)).toThrow(RangeError);
});

// toThrow(/regex/) — el mensaje contiene ese patrón
test('nombre vacío lanza error que menciona "vacío"', () => {
  expect(() => validarNombre('')).toThrow(/vacío/);
});

test('nombre de 1 carácter lanza error de longitud', () => {
  expect(() => validarNombre('A')).toThrow('al menos 2 caracteres');
});

// not.toThrow() — la función NO lanza error (funciona correctamente)
test('edad válida no lanza error', () => {
  expect(() => validarEdad(25)).not.toThrow();
  expect(() => validarEdad(0)).not.toThrow();
  expect(() => validarEdad(120)).not.toThrow();
});

test('nombre válido no lanza error', () => {
  expect(() => validarNombre('Ana')).not.toThrow();
});

test('dividir entre cero lanza error', () => {
  expect(() => dividir(10, 0)).toThrow('No se puede dividir entre cero');
});

test('dividir números válidos no lanza error', () => {
  expect(() => dividir(10, 2)).not.toThrow();
  expect(dividir(10, 2)).toBe(5);
});
```

---

## Tabla resumen de matchers

| Matcher | Para qué sirve |
|---|---|
| `toBe(valor)` | Igualdad estricta (`===`) — primitivos |
| `toEqual(valor)` | Igualdad de contenido — objetos y arrays |
| `toBeTruthy()` | Cualquier valor truthy |
| `toBeFalsy()` | Cualquier valor falsy |
| `toBeNull()` | Exactamente `null` |
| `toBeUndefined()` | Exactamente `undefined` |
| `toBeDefined()` | Cualquier valor que no sea `undefined` |
| `toBeGreaterThan(n)` | Mayor que n |
| `toBeLessThan(n)` | Menor que n |
| `toBeCloseTo(n, dec)` | Aproximación decimal |
| `toMatch(str \| regex)` | Substring o expresión regular |
| `toContain(valor)` | Array contiene el valor (primitivo) |
| `toContainEqual(obj)` | Array contiene el objeto |
| `toHaveLength(n)` | Array o string de longitud n |
| `toHaveProperty(key, val)` | Objeto tiene la propiedad |
| `toThrow(msg \| tipo)` | Función lanza un error |
| `.not.matcher()` | Invierte cualquier matcher |

### Archivos del módulo

```
src/
├── calculadora.js   ← sumar, restar, multiplicar, esPar, obtenerSaludo
├── carrito.js       ← crearProducto, filtrarBaratos, obtenerNombres
├── sesion.js        ← obtenerUsuarioPorToken, mensajeBienvenida, tieneSesionActiva
├── notas.js         ← calcularPromedio, aplicarDescuento, notaMaxima, notaMinima
├── mensajes.js      ← formatearError, esEmailValido, generarCodigoPedido
├── permisos.js      ← obtenerPermisos, puedeRealizarAccion, rolesConAcceso
├── tienda.js        ← obtenerCatalogo, filtrarPorCategoria
├── usuarios.js      ← crearUsuario, actualizarCiudad
└── validador.js     ← validarEdad, validarNombre, dividir

tests/
├── calculadora.test.js
├── carrito.test.js
├── sesion.test.js
├── notas.test.js
├── mensajes.test.js
├── permisos.test.js
├── tienda.test.js
├── usuarios.test.js
└── validador.test.js
```

---

## Ejercicios propuestos

1. **Conversor de unidades** — Crea `src/conversor.js` con las funciones
   `kmAMillas(km)` (1 km = 0.621371 millas) y `celsiusAFahrenheit(c)`.
   En `tests/conversor.test.js` usa `toBeCloseTo` para verificar que
   `kmAMillas(1)` ≈ 0.62 y que `celsiusAFahrenheit(0)` = 32 exacto.

2. **Lista de tareas** — Crea `src/tareas.js` con `agregarTarea(lista, texto)`
   que devuelve una nueva lista con la tarea añadida, y `completarTarea(lista, id)`
   que devuelve la lista con esa tarea marcada como completada.
   Usa `toHaveLength`, `toContainEqual` y `toHaveProperty` en los tests.

3. **Validar contraseña** — Crea `src/contrasena.js` con `validarContrasena(pwd)`
   que lanza `Error('muy corta')` si tiene menos de 8 caracteres,
   `Error('sin números')` si no tiene ningún dígito, y devuelve `true` si es válida.
   Usa `toThrow`, `toThrow(/regex/)` y `not.toThrow` para cubrir los tres casos.

4. **Buscar en lista** — Crea `src/lista.js` con `buscarPorNombre(personas, nombre)`
   que devuelve la persona si existe o `null` si no. Usa `toBeNull`,
   `not.toBeNull`, `toBeDefined` y `toEqual` para verificar ambos casos.

---

## Resumen de la página 2

- `toBe` usa `===` — solo para primitivos. Para objetos y arrays siempre usa `toEqual`.
- `toEqual` compara el **contenido completo** sin importar la referencia en memoria — ideal para objetos y arrays.
- `toBeNull` / `toBeUndefined` / `toBeDefined` son más precisos que `toBeTruthy`/`toBeFalsy` — úsalos cuando el tipo exacto importa.
- `toBeCloseTo` es imprescindible para decimales — `0.1 + 0.2` no es exactamente `0.3` en JavaScript.
- `toContain` para elementos primitivos en arrays. `toContainEqual` para objetos dentro de arrays.
- `toHaveProperty('clave', valor)` verifica propiedades de objetos. Con punto accede a propiedades anidadas: `'direccion.ciudad'`.
- `expect.objectContaining({...})` verifica solo las propiedades que te importan — perfecto cuando el objeto tiene campos dinámicos como IDs o fechas.
- `toThrow` **siempre** necesita una función flecha: `expect(() => fn()).toThrow()`. Sin ella, el error no se captura.
- `.not` invierte cualquier matcher: `not.toBe`, `not.toContain`, `not.toThrow`, etc.

---

> **Siguiente página →** Página 3: Organización de tests con `describe`,
> `beforeEach`, `afterEach`, `beforeAll` y `afterAll` — cómo agrupar
> tests relacionados y compartir configuración entre ellos.
