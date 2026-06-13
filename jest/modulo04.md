# Tutorial Jest — Página 4
## Módulo 4 · Casos borde y `test.each`
### Cómo identificar todos los escenarios y evitar repetición con tablas de datos

---

## ¿Qué es un caso borde?

Un **caso borde** (edge case) es una entrada que está en los límites
o casos extremos del comportamiento esperado: el mínimo posible,
el máximo posible, un valor nulo, un array vacío, un número negativo.

La mayoría de los bugs no ocurren con datos normales — ocurren
con los casos que el programador no contempló al escribir el código.

```
Función: calcularDescuento(precio, porcentaje)

Casos normales (los que todos prueban):
  calcularDescuento(100, 20)   →  80     ← precio con 20% de descuento

Casos borde (los que se olvidan):
  calcularDescuento(0, 20)     →  0      ← precio cero
  calcularDescuento(100, 0)    →  100    ← descuento cero
  calcularDescuento(100, 100)  →  0      ← descuento total
  calcularDescuento(-10, 20)   →  error  ← precio negativo
  calcularDescuento(100, -5)   →  error  ← descuento negativo
  calcularDescuento(100, 101)  →  error  ← descuento imposible
  calcularDescuento('abc', 20) →  error  ← tipo incorrecto
```

---

## Parte 1 — Identificar casos borde sistemáticamente

Para cualquier función, pregúntate:

```
1. ¿Qué pasa con el mínimo posible?         (0, '', [], null)
2. ¿Qué pasa con el máximo posible?         (Number.MAX_SAFE_INTEGER, array enorme)
3. ¿Qué pasa con valores negativos?         (-1, -Infinity)
4. ¿Qué pasa si el tipo es incorrecto?      (string en lugar de número)
5. ¿Qué pasa con valores especiales?        (NaN, Infinity, undefined)
6. ¿Qué pasa en los límites exactos?        (el valor justo en el borde)
7. ¿Qué pasa si la lista está vacía?        ([] en lugar de [1,2,3])
8. ¿Qué pasa con un solo elemento?          ([42] en lugar de [1,2,3])
```

### Ejemplo — función de nota

```javascript
// src/notas.js

/**
 * Convierte una nota numérica (0-10) a su calificación textual.
 */
function calificar(nota) {
  if (typeof nota !== 'number' || isNaN(nota)) {
    throw new TypeError('La nota debe ser un número');
  }
  if (nota < 0 || nota > 10) {
    throw new RangeError('La nota debe estar entre 0 y 10');
  }

  if (nota >= 9)  return 'Sobresaliente';
  if (nota >= 7)  return 'Notable';
  if (nota >= 5)  return 'Aprobado';
  return 'Suspenso';
}

module.exports = { calificar };
```

```javascript
// tests/notas.test.js
const { calificar } = require('../src/notas');

describe('calificar', () => {
  // Casos normales
  describe('casos normales', () => {
    test('nota 10 → Sobresaliente', () => expect(calificar(10)).toBe('Sobresaliente'));
    test('nota 8  → Notable',       () => expect(calificar(8)).toBe('Notable'));
    test('nota 6  → Aprobado',      () => expect(calificar(6)).toBe('Aprobado'));
    test('nota 3  → Suspenso',      () => expect(calificar(3)).toBe('Suspenso'));
  });

  // Casos borde — los límites exactos de cada categoría
  describe('límites entre categorías', () => {
    test('nota 9  → Sobresaliente (límite inferior)', () => {
      expect(calificar(9)).toBe('Sobresaliente');
    });

    test('nota 8.9 → Notable (justo bajo Sobresaliente)', () => {
      expect(calificar(8.9)).toBe('Notable');
    });

    test('nota 7  → Notable (límite inferior)', () => {
      expect(calificar(7)).toBe('Notable');
    });

    test('nota 6.9 → Aprobado (justo bajo Notable)', () => {
      expect(calificar(6.9)).toBe('Aprobado');
    });

    test('nota 5  → Aprobado (límite inferior, el mínimo para pasar)', () => {
      expect(calificar(5)).toBe('Aprobado');
    });

    test('nota 4.9 → Suspenso (justo bajo el aprobado)', () => {
      expect(calificar(4.9)).toBe('Suspenso');
    });

    test('nota 0  → Suspenso (el mínimo absoluto)', () => {
      expect(calificar(0)).toBe('Suspenso');
    });
  });

  // Casos de error
  describe('entradas inválidas', () => {
    test('nota -1 lanza RangeError', () => {
      expect(() => calificar(-1)).toThrow(RangeError);
    });

    test('nota 10.1 lanza RangeError', () => {
      expect(() => calificar(10.1)).toThrow(RangeError);
    });

    test('NaN lanza TypeError', () => {
      expect(() => calificar(NaN)).toThrow(TypeError);
    });

    test('string lanza TypeError', () => {
      expect(() => calificar('8')).toThrow(TypeError);
    });

    test('undefined lanza TypeError', () => {
      expect(() => calificar(undefined)).toThrow(TypeError);
    });

    test('null lanza TypeError', () => {
      expect(() => calificar(null)).toThrow(TypeError);
    });
  });
});
```

---

## Parte 2 — `test.each` — tablas de datos

Cuando tienes muchos tests con la misma estructura pero distintos datos,
`test.each` elimina la repetición y hace el código mucho más conciso.

### Sin `test.each` — repetitivo

```javascript
// Esto funciona, pero es mucho código repetido
test('calificar(10) = Sobresaliente', () => expect(calificar(10)).toBe('Sobresaliente'));
test('calificar(9)  = Sobresaliente', () => expect(calificar(9)).toBe('Sobresaliente'));
test('calificar(8)  = Notable',       () => expect(calificar(8)).toBe('Notable'));
test('calificar(7)  = Notable',       () => expect(calificar(7)).toBe('Notable'));
test('calificar(6)  = Aprobado',      () => expect(calificar(6)).toBe('Aprobado'));
test('calificar(5)  = Aprobado',      () => expect(calificar(5)).toBe('Aprobado'));
test('calificar(4)  = Suspenso',      () => expect(calificar(4)).toBe('Suspenso'));
test('calificar(0)  = Suspenso',      () => expect(calificar(0)).toBe('Suspenso'));
```

### Con `test.each` — compacto y claro

```javascript
// test.each recibe una tabla de datos (array de arrays)
// y genera un test por cada fila
test.each([
  // [nota, calificación esperada]
  [10,  'Sobresaliente'],
  [9,   'Sobresaliente'],
  [8.9, 'Notable'],
  [7,   'Notable'],
  [6.9, 'Aprobado'],
  [5,   'Aprobado'],
  [4.9, 'Suspenso'],
  [0,   'Suspenso'],
])('calificar(%s) devuelve "%s"', (nota, esperado) => {
  //            ↑ %s se sustituye por cada valor de la fila
  expect(calificar(nota)).toBe(esperado);
});
```

Salida:

```
  ✓ calificar(10) devuelve "Sobresaliente"
  ✓ calificar(9) devuelve "Sobresaliente"
  ✓ calificar(8.9) devuelve "Notable"
  ✓ calificar(7) devuelve "Notable"
  ✓ calificar(6.9) devuelve "Aprobado"
  ✓ calificar(5) devuelve "Aprobado"
  ✓ calificar(4.9) devuelve "Suspenso"
  ✓ calificar(0) devuelve "Suspenso"
```

### `test.each` con objetos nombrados

Para tablas con muchas columnas, los objetos son más legibles que los arrays:

```javascript
// src/envio.js
function calcularCostoEnvio(pesoKg, distanciaKm, express) {
  if (pesoKg <= 0)      throw new Error('El peso debe ser positivo');
  if (distanciaKm <= 0) throw new Error('La distancia debe ser positiva');

  const base = pesoKg * 0.5 + distanciaKm * 0.1;
  return express ? base * 1.5 : base;
}
module.exports = { calcularCostoEnvio };
```

```javascript
const { calcularCostoEnvio } = require('../src/envio');

test.each([
  // Cada fila es un objeto con propiedades nombradas
  { peso: 1,  distancia: 10,  express: false, esperado: 1.5  },
  { peso: 2,  distancia: 20,  express: false, esperado: 3.0  },
  { peso: 1,  distancia: 10,  express: true,  esperado: 2.25 },
  { peso: 5,  distancia: 100, express: false, esperado: 12.5 },
  { peso: 5,  distancia: 100, express: true,  esperado: 18.75},
])(
  'peso=$peso kg, dist=$distancia km, express=$express → $esperado',
  ({ peso, distancia, express, esperado }) => {
    expect(calcularCostoEnvio(peso, distancia, express)).toBeCloseTo(esperado, 2);
  }
);
```

Salida:

```
  ✓ peso=1 kg, dist=10 km, express=false → 1.5
  ✓ peso=2 kg, dist=20 km, express=false → 3
  ✓ peso=1 kg, dist=10 km, express=true → 2.25
  ✓ peso=5 kg, dist=100 km, express=false → 12.5
  ✓ peso=5 kg, dist=100 km, express=true → 18.75
```

### `test.each` con template literals

Otra sintaxis más visual usando template literals:

```javascript
// src/conversor.js
function celsiusAFahrenheit(c) {
  return (c * 9 / 5) + 32;
}
module.exports = { celsiusAFahrenheit };
```

```javascript
const { celsiusAFahrenheit } = require('../src/conversor');

// Sintaxis con template literal — la primera fila son los nombres de columna
test.each`
  celsius  | fahrenheit
  ${0}     | ${32}
  ${100}   | ${212}
  ${-40}   | ${-40}
  ${37}    | ${98.6}
  ${20}    | ${68}
`('$celsius°C equivale a $fahrenheit°F', ({ celsius, fahrenheit }) => {
  expect(celsiusAFahrenheit(celsius)).toBeCloseTo(fahrenheit, 1);
});
```

### `describe.each` — agrupar varios tests por cada conjunto de datos

```javascript
// Cuando para cada conjunto de datos necesitas varios tests
describe.each([
  { nombre: 'Ana',  email: 'ana@ejemplo.com',  edad: 25 },
  { nombre: 'Luis', email: 'luis@ejemplo.com', edad: 30 },
  { nombre: 'María',email: 'maria@ejemplo.com',edad: 22 },
])('usuario $nombre', ({ nombre, email, edad }) => {
  let usuario;

  beforeEach(() => {
    usuario = crearUsuario(nombre, email, edad);
  });

  test('tiene el nombre correcto', () => {
    expect(usuario.nombre).toBe(nombre);
  });

  test('tiene el email en minúsculas', () => {
    expect(usuario.email).toBe(email.toLowerCase());
  });

  test('está activo al crearse', () => {
    expect(usuario.activo).toBe(true);
  });
});
```

---

## Parte 3 — Ejemplo completo: función de búsqueda con casos borde

```javascript
// src/busqueda.js

/**
 * Busca productos por nombre (búsqueda parcial, sin distinción de mayúsculas).
 * Devuelve los resultados ordenados por precio ascendente.
 */
function buscarProductos(catalogo, termino) {
  if (!Array.isArray(catalogo)) {
    throw new TypeError('El catálogo debe ser un array');
  }
  if (typeof termino !== 'string') {
    throw new TypeError('El término de búsqueda debe ser un string');
  }

  const terminoLimpio = termino.trim().toLowerCase();

  // Término vacío devuelve todo el catálogo ordenado
  if (terminoLimpio === '') {
    return [...catalogo].sort((a, b) => a.precio - b.precio);
  }

  return catalogo
    .filter(p => p.nombre.toLowerCase().includes(terminoLimpio))
    .sort((a, b) => a.precio - b.precio);
}

module.exports = { buscarProductos };
```

```javascript
// tests/busqueda.test.js
const { buscarProductos } = require('../src/busqueda');

const CATALOGO = [
  { id: 1, nombre: 'Teclado mecánico',   precio: 120 },
  { id: 2, nombre: 'Teclado de membrana',precio: 35  },
  { id: 3, nombre: 'Ratón inalámbrico',  precio: 45  },
  { id: 4, nombre: 'Ratón con cable',    precio: 20  },
  { id: 5, nombre: 'Monitor 24"',        precio: 280 },
  { id: 6, nombre: 'Monitor 27" curvo',  precio: 450 },
];

describe('buscarProductos', () => {
  describe('búsqueda normal', () => {
    test('encuentra productos que contienen el término', () => {
      const resultados = buscarProductos(CATALOGO, 'teclado');
      expect(resultados).toHaveLength(2);
    });

    test('la búsqueda es insensible a mayúsculas', () => {
      expect(buscarProductos(CATALOGO, 'RATÓN')).toHaveLength(2);
      expect(buscarProductos(CATALOGO, 'Ratón')).toHaveLength(2);
      expect(buscarProductos(CATALOGO, 'ratón')).toHaveLength(2);
    });

    test('los resultados están ordenados por precio ascendente', () => {
      const resultados = buscarProductos(CATALOGO, 'teclado');
      expect(resultados[0].precio).toBeLessThanOrEqual(resultados[1].precio);
    });

    test('devuelve el producto con los datos correctos', () => {
      const resultados = buscarProductos(CATALOGO, 'monitor 24');
      expect(resultados).toHaveLength(1);
      expect(resultados[0]).toEqual(
        expect.objectContaining({ nombre: 'Monitor 24"', precio: 280 })
      );
    });
  });

  describe('casos borde', () => {
    test('término vacío devuelve todo el catálogo ordenado', () => {
      const resultados = buscarProductos(CATALOGO, '');
      expect(resultados).toHaveLength(CATALOGO.length);
      // Verifica orden ascendente de precios
      for (let i = 0; i < resultados.length - 1; i++) {
        expect(resultados[i].precio).toBeLessThanOrEqual(resultados[i + 1].precio);
      }
    });

    test('término con espacios se limpia antes de buscar', () => {
      const resultados = buscarProductos(CATALOGO, '  ratón  ');
      expect(resultados).toHaveLength(2);
    });

    test('término sin resultados devuelve array vacío', () => {
      const resultados = buscarProductos(CATALOGO, 'auriculares');
      expect(resultados).toEqual([]);
      expect(resultados).toHaveLength(0);
    });

    test('catálogo vacío devuelve array vacío', () => {
      const resultados = buscarProductos([], 'teclado');
      expect(resultados).toEqual([]);
    });

    test('catálogo con un solo producto y coincidencia', () => {
      const catalogo = [{ id: 1, nombre: 'Teclado', precio: 100 }];
      expect(buscarProductos(catalogo, 'teclado')).toHaveLength(1);
    });

    test('catálogo con un solo producto sin coincidencia', () => {
      const catalogo = [{ id: 1, nombre: 'Teclado', precio: 100 }];
      expect(buscarProductos(catalogo, 'ratón')).toHaveLength(0);
    });
  });

  describe('entradas inválidas', () => {
    test.each([
      [null,      'teclado'],
      [undefined, 'teclado'],
      ['cadena',  'teclado'],
      [42,        'teclado'],
    ])('catálogo=%p lanza TypeError', (catálogoInvalido, termino) => {
      expect(() => buscarProductos(catálogoInvalido, termino)).toThrow(TypeError);
    });

    test.each([
      [CATALOGO, null],
      [CATALOGO, undefined],
      [CATALOGO, 42],
      [CATALOGO, true],
    ])('término=%p lanza TypeError', (catalogo, terminoInvalido) => {
      expect(() => buscarProductos(catalogo, terminoInvalido)).toThrow(TypeError);
    });
  });
});
```

---

## Parte 4 — Funciones puras: la base del testing fácil

Una **función pura** siempre devuelve el mismo resultado para los mismos
argumentos y no tiene efectos secundarios. Son las más fáciles de probar.

```javascript
// PURA — misma entrada → mismo resultado siempre
function sumar(a, b) {
  return a + b;
}

// IMPURA — resultado depende de algo externo (fecha actual)
function obtenerSaludo() {
  const hora = new Date().getHours();
  return hora < 12 ? 'Buenos días' : 'Buenas tardes';
}

// IMPURA — tiene efecto secundario (modifica el array original)
function agregarAlFinal(lista, elemento) {
  lista.push(elemento);  // modifica el argumento
  return lista;
}

// PURA — versión corregida (no modifica el argumento)
function agregarAlFinalPura(lista, elemento) {
  return [...lista, elemento];  // devuelve una nueva lista
}
```

Cuando puedes elegir, escribe funciones puras. Tu código será más fácil
de probar, de leer y de mantener.

---

## Ejercicios propuestos

1. **test.each para conversores** — Tienes estas funciones:
   `kmAMillas(km)` y `kgALibras(kg)`. Usa `test.each` para verificar
   al menos 5 conversiones conocidas de cada una. Incluye cero, valores
   negativos y decimales.

2. **Casos borde de una función de pagos** — Crea `src/pagos.js` con
   `dividirCuentaEquitativamente(total, numPersonas)`. Identifica y escribe
   tests para: división exacta, división con decimales (¿cuánto da
   `10 / 3`?), cero personas (error), personas negativas (error),
   total negativo (error) y total igual a cero.

3. **Tabla de códigos de estado HTTP** — Crea `src/http.js` con
   `describirCodigo(codigo)` que devuelva la descripción del código
   HTTP. Usa `test.each` con al menos 8 filas (200, 201, 301, 400,
   401, 403, 404, 500) y un caso para código desconocido.

4. **Bug hunting** — Esta función tiene al menos 3 bugs relacionados con
   casos borde. Escribe tests que los descubran **antes** de leer el código:
   ```javascript
   function promediar(numeros) {
     let suma = 0;
     for (let i = 0; i <= numeros.length; i++) {  // bug 1
       suma += numeros[i];
     }
     return suma / numeros.length;  // bug 2 (hint: array vacío)
   }
   ```

---

## Resumen de la página 4

- Los bugs suelen ocurrir en los **casos borde** — mínimos, máximos, cero, vacío, tipos incorrectos — no en los casos normales.
- Antes de escribir tests, hazte estas preguntas: ¿qué pasa con el mínimo? ¿con el máximo? ¿con tipos incorrectos? ¿con colecciones vacías?
- `test.each(tabla)(descripción, fn)` genera un test por cada fila de la tabla — elimina repetición cuando la estructura del test es idéntica y solo cambian los datos.
- En la descripción de `test.each`, `%s` inserta el valor como string, `%i` como entero, `%f` como decimal, `%p` como pretty-print. Con objetos, `$propiedad` accede al campo.
- `describe.each` crea bloques de tests completos por cada conjunto de datos — útil cuando cada input necesita varios tests relacionados.
- Las **funciones puras** (misma entrada → mismo resultado, sin efectos secundarios) son las más sencillas de probar. Favorécelas en tu diseño.

---

> **Siguiente página →** Página 5: Testing asíncrono — cómo probar funciones
> que devuelven `Promise`, usan `async/await` o trabajan con callbacks.
