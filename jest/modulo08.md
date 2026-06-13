# Tutorial Jest — Página 8
## Módulo 8 · Cobertura de código y buenas prácticas
### Coverage reports, qué métricas importan y cómo escribir tests de valor real

---

## ¿Qué es la cobertura de código?

La **cobertura de código** (code coverage) mide qué porcentaje de tu código
fue ejecutado durante los tests. No mide si tus tests son buenos —
mide si hay partes del código que ningún test toca.

```
Código sin tests             Código con cobertura alta
─────────────────────────    ─────────────────────────────
function calcular(a, b) {    function calcular(a, b) {
  if (a > 0) {           ✗     if (a > 0) {           ✓
    return a + b;        ✗       return a + b;         ✓
  } else {               ✗     } else {                ✓
    return b - a;        ✗       return b - a;         ✓
  }                      ✗     }
}                        ✗   }
```

---

## Parte 1 — Generar el reporte de cobertura

```bash
# Ejecutar tests con reporte de cobertura
npm test -- --coverage

# O añadir el flag en package.json
```

```json
{
  "scripts": {
    "test":     "jest",
    "coverage": "jest --coverage"
  }
}
```

La salida en consola:

```
 PASS  tests/calculadora.test.js
 PASS  tests/validador.test.js

----------|---------|----------|---------|---------|-------------------
File      | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
----------|---------|----------|---------|---------|-------------------
All files |   87.50 |    75.00 |  100.00 |   87.50 |
 src/
  calcu.. |  100.00 |   100.00 |  100.00 |  100.00 |
  valid.. |   75.00 |    50.00 |  100.00 |   75.00 | 18,24
----------|---------|----------|---------|---------|-------------------
```

### Las cuatro métricas de cobertura

```
% Statements (Stmts)  → qué porcentaje de instrucciones se ejecutó
                         let x = 5; ← una instrucción
                         return x;  ← otra instrucción

% Branches            → qué porcentaje de ramas de decisión se tomó
                         if (x > 0) { ... } else { ... }
                         ↑ dos ramas: el "sí" y el "no"

% Functions (Funcs)   → qué porcentaje de funciones fue llamada

% Lines               → qué porcentaje de líneas fue ejecutada
```

También genera un reporte HTML en `coverage/lcov-report/index.html`
que puedes abrir en el navegador para ver línea por línea qué está cubierto.

---

## Parte 2 — Configurar umbrales mínimos de cobertura

Puedes hacer que Jest falle si la cobertura cae por debajo de ciertos
umbrales. Añade esto a `package.json` o a `jest.config.js`:

```json
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "statements": 80,
        "branches":   75,
        "functions":  90,
        "lines":      80
      }
    }
  }
}
```

Si la cobertura cae por debajo de estos valores, `npm test` falla:

```
Jest: "global" coverage threshold for branches (75%) not met: 60%
```

---

## Parte 3 — Excluir archivos del reporte

No todo el código necesita ser testeado (archivos de configuración,
puntos de entrada, código generado automáticamente):

```json
{
  "jest": {
    "collectCoverageFrom": [
      "src/**/*.js",           ← incluir todo src/
      "!src/index.js",         ← excluir el punto de entrada
      "!src/config/*.js",      ← excluir configuraciones
      "!src/**/*.mock.js"      ← excluir archivos de mock
    ]
  }
}
```

---

## Parte 4 — Ejemplo: descubrir ramas no cubiertas

```javascript
// src/descuentos.js
function calcularDescuento(usuario, total) {
  // Cliente premium: 20% de descuento
  if (usuario.tipo === 'premium') {
    if (total > 500) {
      return total * 0.20;   // ← ¿está probada esta rama?
    }
    return total * 0.10;
  }

  // Cliente normal: 5% si supera 200
  if (total > 200) {
    return total * 0.05;
  }

  return 0;  // ← ¿y esta?
}

module.exports = { calcularDescuento };
```

```javascript
// tests/descuentos.test.js — versión con cobertura incompleta
const { calcularDescuento } = require('../src/descuentos');

test('premium con total alto', () => {
  expect(calcularDescuento({ tipo: 'premium' }, 600)).toBe(120);
});

test('normal con total alto', () => {
  expect(calcularDescuento({ tipo: 'normal' }, 300)).toBe(15);
});

// Cobertura: 75% de branches — faltan dos ramas:
//   - premium con total <= 500 (línea del 10%)
//   - normal con total <= 200  (línea del return 0)
```

```bash
npm run coverage
```

```
File          | % Stmts | % Branch | % Funcs | % Lines | Uncovered
descuentos.js |   83.33 |    75.00 |  100.00 |   83.33 | 11,17
```

El reporte indica que las líneas 11 y 17 no están cubiertas. Las añadimos:

```javascript
// tests/descuentos.test.js — versión completa
const { calcularDescuento } = require('../src/descuentos');

describe('calcularDescuento', () => {
  describe('usuario premium', () => {
    test('total > 500 → 20%', () => {
      expect(calcularDescuento({ tipo: 'premium' }, 600)).toBe(120);
    });

    test('total <= 500 → 10%', () => {
      // Esta es la rama que faltaba
      expect(calcularDescuento({ tipo: 'premium' }, 400)).toBe(40);
    });
  });

  describe('usuario normal', () => {
    test('total > 200 → 5%', () => {
      expect(calcularDescuento({ tipo: 'normal' }, 300)).toBe(15);
    });

    test('total <= 200 → sin descuento', () => {
      // Esta también faltaba
      expect(calcularDescuento({ tipo: 'normal' }, 100)).toBe(0);
    });

    test('exactamente 200 → sin descuento (límite)', () => {
      expect(calcularDescuento({ tipo: 'normal' }, 200)).toBe(0);
    });
  });
});
```

```
File          | % Stmts | % Branch | % Funcs | % Lines | Uncovered
descuentos.js |  100.00 |   100.00 |  100.00 |  100.00 |
```

---

## Parte 5 — Buenas prácticas

### 1. El nombre del test describe el comportamiento, no la implementación

```javascript
// MAL — describe la implementación (el cómo)
test('el if de la condición premium ejecuta el bloque de 20%', () => { ... });

// BIEN — describe el comportamiento (el qué)
test('usuario premium con compra mayor a 500€ recibe 20% de descuento', () => { ... });
```

### 2. Un test verifica una sola cosa

```javascript
// MAL — un test que verifica demasiadas cosas
test('registro de usuario funciona', async () => {
  const usuario = await registrar({ nombre: 'Ana', email: 'ana@ej.com' });
  expect(usuario.id).toBeDefined();
  expect(usuario.nombre).toBe('Ana');
  expect(usuario.activo).toBe(true);
  expect(mockEmail).toHaveBeenCalled();
  expect(mockEmail).toHaveBeenCalledWith(expect.objectContaining({ para: 'ana@ej.com' }));
  expect(mockBD).toHaveBeenCalledTimes(1);
});

// BIEN — cada test verifica un aspecto
test('el usuario registrado tiene un id asignado', async () => {
  const usuario = await registrar({ nombre: 'Ana', email: 'ana@ej.com' });
  expect(usuario.id).toBeDefined();
});

test('el email de bienvenida se envía al correo del usuario', async () => {
  await registrar({ nombre: 'Ana', email: 'ana@ej.com' });
  expect(mockEmail).toHaveBeenCalledWith(
    expect.objectContaining({ para: 'ana@ej.com' })
  );
});
```

### 3. Los tests no deben depender de datos compartidos mutables

```javascript
// MAL — todos los tests modifican el mismo objeto
const USUARIOS = [{ id: 1, nombre: 'Ana', activo: true }];

test('desactivar usuario', () => {
  USUARIOS[0].activo = false;  // modifica el array compartido
  expect(USUARIOS[0].activo).toBe(false);
});

test('usuario empieza activo', () => {
  // FALLA si el test anterior se ejecutó primero
  expect(USUARIOS[0].activo).toBe(true);
});

// BIEN — cada test crea sus propios datos
describe('operaciones de usuario', () => {
  function crearUsuarioPrueba() {
    return { id: 1, nombre: 'Ana', activo: true };
  }

  test('desactivar usuario', () => {
    const usuario = crearUsuarioPrueba();
    usuario.activo = false;
    expect(usuario.activo).toBe(false);
  });

  test('usuario empieza activo', () => {
    const usuario = crearUsuarioPrueba();
    expect(usuario.activo).toBe(true);
  });
});
```

### 4. Describe el contexto con el patrón given/when/then

```javascript
describe('calcularDescuento', () => {
  // Given (dado que)
  describe('dado que el usuario es premium', () => {
    // When (cuando)
    describe('cuando el total supera 500€', () => {
      // Then (entonces)
      test('aplica el 20% de descuento', () => {
        expect(calcularDescuento({ tipo: 'premium' }, 600)).toBe(120);
      });
    });

    describe('cuando el total es menor o igual a 500€', () => {
      test('aplica el 10% de descuento', () => {
        expect(calcularDescuento({ tipo: 'premium' }, 400)).toBe(40);
      });
    });
  });
});
```

### 5. No testees los detalles de implementación

```javascript
// MAL — prueba cómo funciona internamente (implementación)
test('usa el método sort del array', () => {
  const spy = jest.spyOn(Array.prototype, 'sort');
  ordenarPorPrecio(productos);
  expect(spy).toHaveBeenCalled();  // ← ¿importa si usa sort o no?
});

// BIEN — prueba el comportamiento observable (resultado)
test('devuelve los productos ordenados de menor a mayor precio', () => {
  const productos = [
    { nombre: 'A', precio: 100 },
    { nombre: 'B', precio: 20  },
    { nombre: 'C', precio: 50  },
  ];
  const ordenados = ordenarPorPrecio(productos);
  expect(ordenados[0].precio).toBe(20);
  expect(ordenados[1].precio).toBe(50);
  expect(ordenados[2].precio).toBe(100);
});
```

### 6. Los tests deben ser rápidos

```
Test rápido    < 50ms    ← ideal
Test normal    < 500ms   ← aceptable
Test lento     > 1s      ← revisar, probablemente necesita mocks
Test muy lento > 5s      ← problema, Jest lo marcará como timeout
```

Si un test es lento, revisa si:
- Llama a servicios externos reales (usa mocks)
- Tiene un `setTimeout` o `setInterval` (usa `jest.useFakeTimers()`)
- Lee o escribe archivos del disco (mockea `fs`)

---

## Parte 6 — Timers falsos con `jest.useFakeTimers()`

Para código que usa `setTimeout`, `setInterval` o `Date`:

```javascript
// src/recordatorio.js
function programarRecordatorio(mensaje, delayMs, callback) {
  setTimeout(() => {
    callback(`RECORDATORIO: ${mensaje}`);
  }, delayMs);
}

module.exports = { programarRecordatorio };
```

```javascript
// tests/recordatorio.test.js
const { programarRecordatorio } = require('../src/recordatorio');

describe('programarRecordatorio', () => {
  beforeEach(() => {
    jest.useFakeTimers();   // reemplaza setTimeout con timers controlables
  });

  afterEach(() => {
    jest.useRealTimers();   // restaura los timers reales
  });

  test('llama al callback con el mensaje formateado', () => {
    const mockCallback = jest.fn();

    programarRecordatorio('Reunión a las 3pm', 5000, mockCallback);

    // El callback NO se ha llamado aún
    expect(mockCallback).not.toHaveBeenCalled();

    // Avanzar el tiempo 5 segundos
    jest.advanceTimersByTime(5000);

    // Ahora sí debe haberse llamado
    expect(mockCallback).toHaveBeenCalledTimes(1);
    expect(mockCallback).toHaveBeenCalledWith('RECORDATORIO: Reunión a las 3pm');
  });

  test('no se llama antes del delay', () => {
    const mockCallback = jest.fn();
    programarRecordatorio('Test', 10000, mockCallback);

    jest.advanceTimersByTime(9999);  // justo antes
    expect(mockCallback).not.toHaveBeenCalled();

    jest.advanceTimersByTime(1);     // el milisegundo que faltaba
    expect(mockCallback).toHaveBeenCalledTimes(1);
  });

  test('runAllTimers ejecuta todos los timers pendientes', () => {
    const mock1 = jest.fn();
    const mock2 = jest.fn();

    programarRecordatorio('Primero', 1000, mock1);
    programarRecordatorio('Segundo', 5000, mock2);

    jest.runAllTimers();  // ejecuta todos los timers inmediatamente

    expect(mock1).toHaveBeenCalledTimes(1);
    expect(mock2).toHaveBeenCalledTimes(1);
  });
});
```

---

## Parte 7 — `jest.config.js` completo

Para proyectos reales, centraliza la configuración en `jest.config.js`:

```javascript
// jest.config.js — en la raíz del proyecto

module.exports = {
  // Directorio raíz de los tests
  testMatch: ['**/tests/**/*.test.js', '**/__tests__/**/*.test.js'],

  // Entorno de ejecución
  testEnvironment: 'node',   // 'jsdom' para proyectos de frontend

  // Cobertura de código
  collectCoverage: false,    // true para siempre generar coverage
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/index.js',
    '!src/**/__mocks__/**',
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],

  // Umbrales mínimos de cobertura
  coverageThreshold: {
    global: {
      statements: 80,
      branches:   75,
      functions:  90,
      lines:      80,
    },
  },

  // Limpiar mocks automáticamente entre tests
  clearMocks:   true,   // limpia historial de llamadas
  resetMocks:   false,  // resetea implementaciones
  restoreMocks: true,   // restaura spies

  // Timeout por defecto para cada test (ms)
  testTimeout: 5000,

  // Variables de entorno para tests
  testEnvironmentOptions: {},
};
```

---

## Parte 8 — El proyecto completo de ejemplo

Estructura de un proyecto bien organizado:

```
mi-proyecto/
├── src/
│   ├── calculadora.js
│   ├── carrito.js
│   ├── registro.js
│   ├── emailService.js
│   └── __mocks__/
│       └── emailService.js   ← mock manual compartido
├── tests/
│   ├── calculadora.test.js
│   ├── carrito.test.js
│   ├── registro.test.js
│   └── emailService.test.js
├── coverage/                  ← generado por Jest
│   └── lcov-report/
│       └── index.html         ← reporte visual
├── jest.config.js
└── package.json
```

```json
{
  "scripts": {
    "test":           "jest",
    "test:watch":     "jest --watch",
    "test:coverage":  "jest --coverage",
    "test:verbose":   "jest --verbose"
  }
}
```

---

## Checklist: antes de hacer commit

Antes de subir tu código, verifica:

```
□ Todos los tests pasan:           npm test
□ La cobertura es aceptable:       npm run test:coverage
□ No hay test.only en el código    (busca "test.only" antes de commitear)
□ No hay console.log de debug en los tests
□ Los nombres de los tests son descriptivos y en español/idioma del proyecto
□ No hay tests que dependen del orden de ejecución
□ Los mocks se limpian en afterEach/beforeEach
□ Los tests asíncronos tienen await donde corresponde
```

---

## Antipatrones más comunes

```javascript
// ✗ Test sin assertions — siempre pasa, no prueba nada
test('procesar datos', async () => {
  await procesarDatos(datos);
  // no hay ningún expect → el test pasa aunque la función esté rota
});

// ✗ Assertion que siempre es verdadera
test('el array tiene elementos', () => {
  const lista = [1, 2, 3];
  expect(lista.length > 0).toBe(true);  // ← siempre pasa, sin valor
  // Mejor: expect(lista).toHaveLength(3)
});

// ✗ Mock que nunca se usa para verificar nada
test('enviar email', async () => {
  const mockEmail = jest.fn();
  await registrar(datos, mockEmail);
  // mockEmail se creó pero nunca se verificó
  // ¿Se llamó? ¿Con qué argumentos? No sabemos.
});

// ✗ Test demasiado atado a la implementación
test('usa Array.prototype.filter internamente', () => {
  const spy = jest.spyOn(Array.prototype, 'filter');
  buscarActivos(usuarios);
  expect(spy).toHaveBeenCalled();
  // Si cambiamos la implementación a un for-loop, el test falla
  // aunque el resultado sea correcto
});
```

---

## Ejercicios finales del curso

1. **Proyecto completo** — Crea un sistema de gestión de biblioteca con:
   - `Libro` (título, autor, isbn, disponible)
   - `Biblioteca` con métodos: `registrar`, `prestar`, `devolver`, `buscarPorAutor`
   - Tests con cobertura del 100% usando `describe` anidados, `beforeEach`,
     `test.each` para múltiples libros, y al menos un mock de `Date.now`.

2. **Alcanzar 100% de cobertura** — Toma el código de `validarContrasena`
   del módulo 1, ejecuta el reporte de cobertura y añade los tests
   necesarios para alcanzar el 100% en todas las métricas.

3. **Configuración real** — Crea un `jest.config.js` con umbrales del 80%
   y `clearMocks: true`. Ejecuta `npm run test:coverage` y verifica que
   el reporte HTML se genera correctamente. Luego borra un test para
   que la cobertura caiga por debajo del umbral y observa el mensaje de error.

4. **Tests de regresión** — Encuentra un bug real en este código, escribe
   primero el test que lo detecte (que fallará), luego arregla el código
   y verifica que el test pasa:
   ```javascript
   // ¿Qué devuelve filtrarPares([]) ?
   // ¿Qué devuelve filtrarPares([1, null, 3]) ?
   function filtrarPares(numeros) {
     return numeros.filter(n => n % 2 === 0);
   }
   ```

---

## Resumen de la página 8

- El **coverage** mide qué porcentaje del código se ejecuta durante los tests, no si los tests son buenos. Un 100% de cobertura no garantiza que el código sea correcto.
- Las cuatro métricas: **Statements** (instrucciones), **Branches** (ramas de decisión), **Functions** (funciones), **Lines** (líneas). Branches es la más difícil de llegar al 100%.
- `jest.config.js` centraliza la configuración: umbrales de cobertura, timeout, limpieza automática de mocks, y patrones de archivos de test.
- `jest.useFakeTimers()` permite controlar `setTimeout`, `setInterval` y `Date` sin esperar tiempos reales.
- **Buenas prácticas:** nombres descriptivos de comportamiento, un assert por test, tests independientes, no testear detalles de implementación.
- **Antipatrones a evitar:** tests sin assertions, assertions que siempre son verdaderas, mocks que no se verifican, tests atados a la implementación interna.
- La cobertura es una herramienta de diagnóstico, no un objetivo en sí misma. Un 80% con tests de calidad es mejor que un 100% con tests vacíos.

---

## Recapitulación del curso

| Módulo | Tema |
|---|---|
| 1 | Qué son las pruebas unitarias, `test()`, `expect()`, patrón AAA |
| 2 | Matchers: `toBe`, `toEqual`, `toContain`, `toThrow`, `objectContaining` |
| 3 | `describe`, `beforeEach`, `afterEach`, `beforeAll`, `afterAll` |
| 4 | Casos borde, `test.each` con tablas de datos |
| 5 | Testing asíncrono: `async/await`, `.resolves`, `.rejects` |
| 6 | Mocks con `jest.fn()`, `mockReturnValue`, `mockResolvedValue` |
| 7 | `jest.spyOn()`, `jest.mock()`, módulos `__mocks__` |
| 8 | Cobertura de código, `jest.config.js`, buenas prácticas |

---

> **¡Fin del tutorial!** Ya tienes todas las herramientas para escribir
> pruebas unitarias profesionales con Jest. El siguiente paso es aplicar
> estos conceptos en un proyecto real y practicar el ciclo TDD:
> **Red → Green → Refactor** (escribe el test que falla, haz que pase,
> luego mejora el código sin romper los tests).
