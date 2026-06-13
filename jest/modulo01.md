# Tutorial Jest — Página 1
## Módulo 1 · Introducción a las pruebas unitarias
### ¿Qué son las pruebas unitarias?, instalación de Jest y tu primer test

---

## ¿Qué es una prueba unitaria?

Una **prueba unitaria** verifica que una pequeña pieza de código
(una función, un método) funciona exactamente como se espera,
de forma aislada del resto del sistema.

```
Sin pruebas                     Con pruebas unitarias
─────────────────────────────   ─────────────────────────────
Escribes código                 Escribes el código
Abres el navegador              Ejecutas: npm test
Haces clic en todas las         Jest verifica todo en segundos
pantallas manualmente           Te dice exactamente qué falló
```

La idea central es simple: **si una función recibe cierta entrada,
debería producir una salida predecible**. Los tests verifican eso
de forma automática, cada vez que cambias algo.

### ¿Por qué usar pruebas?

Imagina que tienes una función que calcula el total de una compra.
Funciona hoy. Mañana tu compañero añade descuentos. ¿Sigue funcionando
para carritos vacíos? ¿Para precios negativos? ¿Para cantidades decimales?

Con pruebas unitarias, sabes la respuesta **al instante** sin abrir el navegador.

---

## Versiones del curso

| Herramienta | Versión |
|---|---|
| Node.js | **20 LTS** o superior |
| npm | **10+** (incluido con Node) |
| Jest | **29.x** |
| JavaScript | ES Modules o CommonJS |

---

## Parte 1 — Instalación del entorno

### Instalar Node.js

```bash
# Verifica si ya lo tienes instalado
node --version    # debe ser v20 o superior
npm --version     # debe ser 10 o superior

# Si no lo tienes, descárgalo desde nodejs.org
# o usa un gestor de versiones:
# macOS / Linux
brew install node        # con Homebrew
# o
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 20

# Windows — descargar el instalador desde nodejs.org
```

### Crear un proyecto e instalar Jest

```bash
# Crear la carpeta del proyecto
mkdir mi-proyecto-jest
cd mi-proyecto-jest

# Inicializar el proyecto (crea package.json)
npm init -y

# Instalar Jest como dependencia de desarrollo
npm install --save-dev jest

# Verificar instalación
npx jest --version   # debe mostrar 29.x.x
```

### Configurar el script de test

Abre el archivo `package.json` y ajusta la sección `"scripts"`:

```json
{
  "name": "mi-proyecto-jest",
  "version": "1.0.0",
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch"
  },
  "devDependencies": {
    "jest": "^29.0.0"
  }
}
```

Ahora puedes ejecutar los tests con:

```bash
npm test              # ejecuta todos los tests una vez
npm run test:watch    # re-ejecuta tests automáticamente al guardar
```

### Estructura del proyecto

```
mi-proyecto-jest/
├── src/
│   └── calculadora.js      ← código que vas a probar
├── tests/
│   └── calculadora.test.js ← los tests
├── package.json
└── package-lock.json
```

> **Convención de nombres:** Jest detecta automáticamente archivos que
> terminen en `.test.js` o `.spec.js`, o que estén dentro de una carpeta
> llamada `__tests__`. Puedes usar cualquiera de los tres estilos.

---

## Parte 2 — Tu primera función y tu primer test

### El código a probar

Crea el archivo `src/calculadora.js`:

```javascript
// src/calculadora.js

// Suma dos números y devuelve el resultado
function sumar(a, b) {
  return a + b;
}

// Resta el segundo número al primero
function restar(a, b) {
  return a - b;
}

// Multiplica dos números
function multiplicar(a, b) {
  return a * b;
}

// Divide el primer número entre el segundo
// Lanza un error si el divisor es cero
function dividir(a, b) {
  if (b === 0) {
    throw new Error('No se puede dividir entre cero');
  }
  return a / b;
}

module.exports = { sumar, restar, multiplicar, dividir };
```

### El primer test

Crea el archivo `tests/calculadora.test.js`:

```javascript
// tests/calculadora.test.js

// Importamos las funciones que vamos a probar
const { sumar, restar, multiplicar, dividir } = require('../src/calculadora');

// test() recibe dos cosas:
//   1. Un string que describe qué se está probando
//   2. Una función que ejecuta la prueba

test('sumar 2 + 3 devuelve 5', () => {
  const resultado = sumar(2, 3);
  expect(resultado).toBe(5);
  //     ↑ valor real    ↑ valor esperado
});

test('restar 10 - 4 devuelve 6', () => {
  expect(restar(10, 4)).toBe(6);
});

test('multiplicar 3 × 4 devuelve 12', () => {
  expect(multiplicar(3, 4)).toBe(12);
});

test('dividir 10 ÷ 2 devuelve 5', () => {
  expect(dividir(10, 2)).toBe(5);
});

test('dividir entre cero lanza un error', () => {
  // Cuando una función debe lanzar un error,
  // hay que envolverla en una función flecha
  expect(() => dividir(10, 0)).toThrow('No se puede dividir entre cero');
});
```

### Ejecutar los tests

```bash
npm test
```

Salida esperada:

```
 PASS  tests/calculadora.test.js
  ✓ sumar 2 + 3 devuelve 5 (3 ms)
  ✓ restar 10 - 4 devuelve 6 (1 ms)
  ✓ multiplicar 3 × 4 devuelve 12 (1 ms)
  ✓ dividir 10 ÷ 2 devuelve 5 (1 ms)
  ✓ dividir entre cero lanza un error (2 ms)

Test Suites: 1 passed, 1 total
Tests:       5 passed, 5 total
Time:        0.8 s
```

Si un test falla, Jest te muestra exactamente qué salió mal:

```
  ✕ sumar 2 + 3 devuelve 5

  ● sumar 2 + 3 devuelve 5

    expect(received).toBe(expected)

    Expected: 5
    Received: 6     ← esto es lo que devolvió tu función
```

---

## Parte 3 — Anatomía de un test

```javascript
test('descripción clara de qué se está probando', () => {
  // ── ARRANGE (preparar) ───────────────────────────────────────
  // Aquí preparas los datos de entrada y el estado inicial
  const precio   = 100;
  const cantidad = 3;

  // ── ACT (actuar) ─────────────────────────────────────────────
  // Aquí llamas a la función que estás probando
  const total = calcularTotal(precio, cantidad);

  // ── ASSERT (verificar) ───────────────────────────────────────
  // Aquí verificas que el resultado es el esperado
  expect(total).toBe(300);
});
```

Este patrón se llama **AAA** (Arrange, Act, Assert) y es el estándar
para escribir tests claros y mantenibles. Separa visualmente cada etapa
para que cualquier persona pueda entender el test de un vistazo.

### `test` vs `it` — son idénticos

Jest acepta ambas formas. `it` es más legible en inglés
(`it should return 5`), `test` es más directo en español:

```javascript
// Estas dos formas son exactamente iguales en Jest
test('sumar 2 + 3 devuelve 5', () => { ... });
it('sumar 2 + 3 devuelve 5',   () => { ... });
```

---

## Parte 4 — Ejemplo aplicado: validador de contraseñas

Vamos a probar algo más realista: una función que valida contraseñas.

### El código

```javascript
// src/validador.js

/**
 * Valida si una contraseña cumple los requisitos mínimos de seguridad.
 * Devuelve un objeto con { valida, errores }
 */
function validarContrasena(contrasena) {
  const errores = [];

  if (typeof contrasena !== 'string') {
    return { valida: false, errores: ['La contraseña debe ser un texto'] };
  }

  if (contrasena.length < 8) {
    errores.push('Debe tener al menos 8 caracteres');
  }

  if (!/[A-Z]/.test(contrasena)) {
    errores.push('Debe tener al menos una letra mayúscula');
  }

  if (!/[0-9]/.test(contrasena)) {
    errores.push('Debe tener al menos un número');
  }

  if (!/[!@#$%^&*]/.test(contrasena)) {
    errores.push('Debe tener al menos un símbolo especial (!@#$%^&*)');
  }

  return {
    valida: errores.length === 0,
    errores,
  };
}

module.exports = { validarContrasena };
```

### Los tests

```javascript
// tests/validador.test.js

const { validarContrasena } = require('../src/validador');

test('contraseña válida devuelve valida=true y errores vacíos', () => {
  // Arrange
  const contrasena = 'Segura@123';

  // Act
  const resultado = validarContrasena(contrasena);

  // Assert
  expect(resultado.valida).toBe(true);
  expect(resultado.errores).toHaveLength(0);
});

test('contraseña muy corta devuelve el error correspondiente', () => {
  const resultado = validarContrasena('Ab1!');

  expect(resultado.valida).toBe(false);
  expect(resultado.errores).toContain('Debe tener al menos 8 caracteres');
});

test('contraseña sin mayúsculas devuelve error', () => {
  const resultado = validarContrasena('sinmayus123!');

  expect(resultado.valida).toBe(false);
  expect(resultado.errores).toContain('Debe tener al menos una letra mayúscula');
});

test('contraseña sin números devuelve error', () => {
  const resultado = validarContrasena('SinNumero!');

  expect(resultado.valida).toBe(false);
  expect(resultado.errores).toContain('Debe tener al menos un número');
});

test('contraseña sin símbolo especial devuelve error', () => {
  const resultado = validarContrasena('SinSimbolo1');

  expect(resultado.valida).toBe(false);
  expect(resultado.errores).toContain('Debe tener al menos un símbolo especial (!@#$%^&*)');
});

test('contraseña con todos los problemas acumula varios errores', () => {
  const resultado = validarContrasena('abc');

  expect(resultado.valida).toBe(false);
  expect(resultado.errores.length).toBeGreaterThan(1);
});

test('contraseña que no es texto devuelve error especial', () => {
  const resultado = validarContrasena(12345678);

  expect(resultado.valida).toBe(false);
  expect(resultado.errores[0]).toBe('La contraseña debe ser un texto');
});
```

Salida:

```
 PASS  tests/validador.test.js
  ✓ contraseña válida devuelve valida=true y errores vacíos (2 ms)
  ✓ contraseña muy corta devuelve el error correspondiente (1 ms)
  ✓ contraseña sin mayúsculas devuelve error (1 ms)
  ✓ contraseña sin números devuelve error (1 ms)
  ✓ contraseña sin símbolo especial devuelve error (1 ms)
  ✓ contraseña con todos los problemas acumula varios errores (1 ms)
  ✓ contraseña que no es texto devuelve error especial (1 ms)
```

---

## Parte 5 — Skipping tests temporalmente

A veces necesitas saltar un test mientras arreglas algo sin borrar el código:

```javascript
// test.skip — salta este test (aparece como "skipped" en el reporte)
test.skip('funcionalidad aún no implementada', () => {
  expect(calcularImpuesto(100)).toBe(21);
});

// test.only — ejecuta SOLO este test, omite todos los demás
// Útil para enfocarte en un problema específico
test.only('este es el único test que quiero ejecutar ahora', () => {
  expect(sumar(1, 1)).toBe(2);
});
```

> **Cuidado con `test.only`:** nunca lo dejes en el código final.
> Haría que los demás tests no se ejecuten en el CI/CD y pasarían
> desapercibidos errores.

---

## Ejercicios propuestos

1. **Calculadora completa** — Añade dos funciones a `calculadora.js`:
   `potencia(base, exponente)` y `modulo(a, b)` (el resto de la división).
   Escribe al menos 3 tests para cada una, incluyendo casos con números
   negativos y con cero.

2. **Conversor de temperatura** — Crea `src/conversor.js` con las funciones
   `celsiusAFahrenheit(c)` y `fahrenheitACelsius(f)`. Recuerda que
   0°C = 32°F y 100°C = 212°F. Escribe tests que verifiquen estos
   valores exactos y el punto de congelación del agua en ambas escalas.

3. **Validador de email** — Crea `src/email.js` con la función
   `esEmailValido(email)` que devuelva `true` si el email tiene
   formato correcto (contiene `@`, tiene dominio con punto, no tiene espacios).
   Escribe tests para: email válido, sin arroba, sin dominio, con espacios,
   vacío, y un valor que no sea string.

4. **Detectar el bug** — La siguiente función tiene un bug.
   Escribe primero el test que lo detecte, luego corrígela:
   ```javascript
   function esPalindromo(texto) {
     const limpio = texto.toLowerCase().replace(/ /g, '');
     return limpio === limpio.reverse(); // ← pista: los strings no tienen .reverse()
   }
   ```

---

## Resumen de la página 1

- Una **prueba unitaria** verifica que una función devuelve el resultado esperado para una entrada dada, de forma aislada y automática.
- Jest se instala con `npm install --save-dev jest` y se ejecuta con `npm test`.
- Cada test usa `test('descripción', () => { ... })` — la descripción debe explicar qué comportamiento se verifica.
- `expect(valor).toBe(esperado)` es el matcher más básico — verifica igualdad estricta (`===`).
- El patrón **AAA** (Arrange, Act, Assert) organiza el test en tres secciones claras: preparar, actuar, verificar.
- `expect(() => funcion()).toThrow('mensaje')` verifica que una función lance un error — siempre hay que envolverla en una función flecha.
- `test.skip` omite un test temporalmente. `test.only` ejecuta solo ese test — nunca dejarlo en producción.

---

> **Siguiente página →** Página 2: Matchers de Jest — `toEqual`, `toBeNull`,
> `toBeTruthy`, `toContain`, `toMatch` y todos los comparadores que necesitas
> para verificar cualquier tipo de dato.
