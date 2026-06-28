# Testing en React con Vitest — Página 11

## Módulo 11 · Cobertura y buenas prácticas

### Mide lo que importa, evita los anti-patrones y escribe tests que duren

---

## Cómo trabajar este módulo

Este módulo es **incremental**, pero distinto a los anteriores: en vez de
construir un componente, vamos a **configurar la cobertura** paso a paso y luego
afinar el criterio de qué testear. No copies el bloque de config de golpe — añade
**una opción cada vez**, ejecuta `vitest run --coverage` y observa cómo cambia la
tabla. Entre paso y paso encontrarás retos **🔧 Tu turno** que debes resolver
**antes** de mirar la solución.

```
Paso 1  →  instalar el provider y la primera medición
Paso 2  →  configurar coverage en vite.config.ts
Paso 3  →  interpretar el reporte
Paso 4  →  añadir thresholds y hacerlos fallar (Tu turno)
Paso 5  →  decidir qué testear y qué NO
Paso 6  →  cazar y arreglar anti-patrones (Tu turno)
Apéndice →  equivalencias con Jest
```

---

Has aprendido a renderizar, consultar, interactuar, mockear, esperar peticiones
y envolver providers. En este último módulo damos un paso atrás para mirar la
suite como un todo: **cuánto** del código estamos probando (cobertura), **qué**
merece la pena testear y **cómo** mantener los tests legibles y robustos a lo
largo del tiempo.

Cerramos también con un **apéndice de equivalencias con Jest** para quien llegue
desde proyectos que usaban ese framework.

---

## Paso 1 — Instalar el provider y la primera medición

La **cobertura** mide qué porcentaje de tu código se ejecuta durante los tests.
Vitest la calcula con el provider **v8** (rápido, nativo de Node, sin
instrumentar el código). Instálalo:

```bash
npm i -D @vitest/coverage-v8
```

Y lanza la suite midiendo cobertura:

```bash
vitest run --coverage
```

Se imprime una tabla en consola y se genera un reporte HTML navegable en
`coverage/index.html`. Ábrelo: verás cada archivo coloreado por líneas cubiertas
(verde) y sin cubrir (rojo). Esa primera foto es nuestro punto de partida.

---

## Paso 2 — Configurar `coverage` en `vite.config.ts`

Pasar flags a mano cansa. Define la configuración **una sola vez**. Empieza con lo
mínimo (`provider` y `reporter`) y luego añade `include`/`exclude`:

```ts
// vite.config.ts
import { defineConfig } from 'vitest/config'; // ← desde vitest/config, NO desde 'vite'
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      // Formatos del reporte: texto en consola, JSON y HTML navegable.
      reporter: ['text', 'json', 'html'],
      // Archivos que SÍ cuentan para la cobertura.
      include: ['src/**/*.{ts,tsx}'],
      // Archivos que NO tiene sentido medir.
      exclude: [
        'src/**/*.test.{ts,tsx}',
        'src/test/**',
        'src/**/*.d.ts',
        'src/main.tsx',
      ],
    },
  },
});
```

| Opción | Qué hace |
| --- | --- |
| `provider: 'v8'` | Motor de cobertura nativo de V8 |
| `reporter` | Formatos de salida (`text`, `html`, `json`, `lcov`…) |
| `include` / `exclude` | Define el universo de archivos medidos |

Vuelve a ejecutar `vitest run --coverage` sin flags extra: la config ya manda.
Fíjate cómo `exclude` quita los `*.test.tsx` y la infraestructura de `src/test/`
del cálculo (no tiene sentido medir el código de pruebas a sí mismo).

---

## Paso 3 — Interpretar el reporte

La tabla de consola tiene cuatro métricas por archivo:

```text
 % Coverage report from v8
-----------------|---------|----------|---------|---------|-------------------
File             | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
-----------------|---------|----------|---------|---------|-------------------
 components/      |   92.10 |    85.71 |   88.88 |   92.10 |
  TodoItem.tsx    |  100.00 |   100.00 |  100.00 |  100.00 |
  TodoLoader.tsx  |   84.00 |    66.66 |   80.00 |   84.00 | 22-25
-----------------|---------|----------|---------|---------|-------------------
```

| Métrica | Significado | Ejemplo |
| --- | --- | --- |
| **Statements** | Sentencias ejecutadas | `const x = 1;` |
| **Branches** | Ramas de decisiones cubiertas | ambos lados de un `if`/`?:` |
| **Functions** | Funciones invocadas al menos una vez | callbacks, handlers |
| **Lines** | Líneas de código ejecutadas | similar a statements |

La columna **Uncovered Line #s** te dice exactamente qué líneas faltan. En el
ejemplo, `TodoLoader.tsx` tiene un 66% de branches: probablemente falta cubrir
la rama de error o el caso de lista vacía (¿recuerdas el módulo 9?).

> La cobertura mide **qué se ejecuta**, no **qué se verifica**. Un test sin
> aserciones puede dar 100% de cobertura y no probar nada. Busca cobertura
> alta *y* aserciones significativas. No persigas el 100% a ciegas.

---

## Paso 4 — 🔧 Tu turno #1: añadir thresholds y hacerlos fallar

Los `thresholds` son tu red de seguridad en CI: garantizan que la cobertura no
baje con el tiempo. Tu reto tiene dos partes:

1. Añade un bloque `thresholds` razonable a la config del Paso 2.
2. Sube **a propósito** un umbral a un valor imposible (p. ej. `statements: 99`)
   y ejecuta `vitest run --coverage`. Predice qué exit code devolverá.

<details><summary>✅ Solución</summary>

Bloque `thresholds` dentro de `coverage`:

```ts
// vite.config.ts (dentro de coverage)
      // Umbrales mínimos: si no se alcanzan, el comando falla (exit code 1).
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80,
      },
```

| Opción | Qué hace |
| --- | --- |
| `thresholds` | Mínimos exigidos; por debajo, el comando falla |

Al subir `statements: 99` (más de lo que tu suite cubre), el comando **falla con
exit code 1** y muestra un mensaje del estilo
`ERROR: Coverage for statements (XX%) does not meet global threshold (99%)`. Eso
es justo lo que rompe el pipeline de CI y evita que se mezcle código sin probar.

> Empieza con valores realistas (los que ya cumples) y súbelos poco a poco a
> medida que añades tests. Un umbral que nunca se cumple solo entrena al equipo a
> ignorar el rojo.

</details>

---

## Paso 5 — Decidir qué testear y qué NO

Configurar la herramienta es lo fácil. La decisión más importante es **qué** vale
la pena testear. El objetivo es probar **comportamiento observable por el
usuario**, no detalles internos.

| ✅ Sí merece test | ❌ No (o con cuidado) |
| --- | --- |
| Lógica de negocio (reducers, hooks como `useTodos`) | Detalles de implementación (nombre de un `useState`) |
| Qué ve el usuario (texto, roles, estados) | Estilos CSS exactos o píxeles |
| Ramas condicionales (loading/éxito/error) | Librerías de terceros ya testeadas |
| Interacciones (clic, escribir, enviar) | Getters/setters triviales |
| Casos límite (lista vacía, error de red) | Tipos de TypeScript (los valida el compilador) |
| Accesibilidad (roles, labels) | Logs, comentarios |

Regla mental: *"Si refactorizo la implementación sin cambiar lo que ve el
usuario, ¿debería romperse este test?"* Si la respuesta es sí, probablemente
estás probando un detalle de implementación.

---

## Paso 6 — 🔧 Tu turno #2: cazar y arreglar anti-patrones

Aquí tienes cuatro fragmentos con olor a problema. **Antes de mirar la solución**,
identifica el anti-patrón en cada uno y reescríbelo bien.

```tsx
// (a)
expect(component.state.contador).toBe(1);

// (b)
expect(container).toMatchSnapshot();

// (c)
screen.getByTestId('boton-guardar');

// (d)
await new Promise((r) => setTimeout(r, 1000));
```

<details><summary>✅ Solución</summary>

**(a) Testear detalles de implementación.** Acoplar el test al estado interno
hace que cualquier refactor lo rompa, aunque la UI funcione.

```tsx
// ✅ Bien: comprueba lo que el usuario ve
expect(screen.getByText('Total: 1')).toBeInTheDocument();
```

**(b) Snapshots frágiles y enormes.** Un snapshot de todo el árbol se rompe ante
cualquier cambio cosmético y el equipo lo "actualiza a ciegas".

| Snapshot frágil | Aserción explícita |
| --- | --- |
| `toMatchSnapshot()` de todo el DOM | `getByRole('heading')` concreto |
| Se rompe ante cambios cosméticos | Solo se rompe si cambia el comportamiento |
| Difícil de revisar en el PR | El intento queda claro al leer el test |

Si usas snapshots, que sean **pequeños y deliberados** (p. ej. el resultado de una
función de formato), no árboles DOM completos.

**(c) Abusar de `getByTestId`.** Recurrir a `data-testid` cuando hay una query
accesible oculta problemas de marcado.

```tsx
// ✅ Preferir queries accesibles que reflejan cómo usa la app un humano
screen.getByRole('button', { name: 'Guardar' });
```

Orden de preferencia (de mejor a peor): `getByRole` → `getByLabelText` →
`getByPlaceholderText` → `getByText` → `getByDisplayValue` → … → `getByTestId`
(último recurso).

**(d) Esperas frágiles con timers fijos.** Esperar un tiempo arbitrario es lento y
propenso a fallos intermitentes.

```tsx
// ✅ Esperar a una condición real
await screen.findByRole('list');
```

</details>

---

## Tests legibles: el patrón AAA

Estructura cada test en tres bloques: **Arrange** (preparar), **Act** (actuar),
**Assert** (verificar).

```tsx
// src/components/AddTodoForm.test.tsx
import { render, screen } from '../test/utils';
import userEvent from '@testing-library/user-event';
import { vi } from 'vitest';
import { AddTodoForm } from './AddTodoForm';

it('llama a onAdd con el texto introducido', async () => {
  // Arrange: preparar el escenario
  const onAdd = vi.fn();
  const user = userEvent.setup();
  render(<AddTodoForm onAdd={onAdd} />);

  // Act: ejecutar la interacción
  await user.type(screen.getByRole('textbox'), 'Comprar pan');
  await user.click(screen.getByRole('button', { name: /añadir/i }));

  // Assert: verificar el resultado
  expect(onAdd).toHaveBeenCalledWith('Comprar pan');
});
```

Buenas prácticas de legibilidad:

- Un comportamiento por test; el nombre describe el **qué**, no el **cómo**.
- `describe` agrupa por componente o función; `it`/`test` describe el caso.
- Evita lógica (bucles, condicionales) dentro del test: si lo necesitas, el test
  es demasiado complejo.
- Usa `userEvent.setup()` una vez por test, no eventos `fireEvent` salvo casos
  puntuales.

---

## Organización de carpetas

Dos convenciones habituales; elige una y sé consistente:

```text
# Opción A: tests junto al código (colocados)
src/
├── components/
│   ├── TodoItem.tsx
│   ├── TodoItem.test.tsx      ← junto al componente
│   └── TodoList.tsx
├── hooks/
│   ├── useTodos.ts
│   └── useTodos.test.ts
└── test/
    ├── setup.ts
    ├── utils.tsx
    └── mocks/
        ├── handlers.ts
        └── server.ts

# Opción B: carpeta __tests__ por módulo
src/
└── components/
    ├── TodoItem.tsx
    └── __tests__/
        └── TodoItem.test.tsx
```

| Estrategia | Ventaja | Inconveniente |
| --- | --- | --- |
| Colocados (`*.test.tsx` al lado) | Fácil de encontrar y mover | Carpetas más pobladas |
| `__tests__/` | Separa producción de pruebas | Un salto más para navegar |

La infraestructura compartida (setup, helpers, mocks de MSW) vive siempre en
`src/test/`, fuera de la cobertura de producción.

---

## Apéndice — Equivalencias con Jest

Si vienes de Jest, casi todo te resultará familiar: la API de aserciones y de
mocking es deliberadamente parecida. La diferencia principal es el namespace
global `vi` en lugar de `jest`.

| Tarea | Jest | Vitest |
| --- | --- | --- |
| Función mock | `jest.fn()` | `vi.fn()` |
| Mockear módulo | `jest.mock('./api')` | `vi.mock('./api')` |
| Espiar método | `jest.spyOn(obj, 'm')` | `vi.spyOn(obj, 'm')` |
| Restaurar mocks | `jest.restoreAllMocks()` | `vi.restoreAllMocks()` |
| Limpiar mocks | `jest.clearAllMocks()` | `vi.clearAllMocks()` |
| Resetear mocks | `jest.resetAllMocks()` | `vi.resetAllMocks()` |
| Timers falsos | `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| Avanzar timers | `jest.advanceTimersByTime(n)` | `vi.advanceTimersByTime(n)` |
| Mock por defecto | `jest.requireActual` | `vi.importActual` |
| Globals (`describe`, `it`) | Implícitos | `globals: true` o `import` desde `vitest` |
| Config | `jest.config.js` | `test` en `vite.config.ts` |
| Setup | `setupFilesAfterEach` | `test.setupFiles` |
| Matchers DOM | `@testing-library/jest-dom` | `@testing-library/jest-dom/vitest` |
| Snapshot inline | `toMatchInlineSnapshot()` | `toMatchInlineSnapshot()` (igual) |
| Cobertura | `--coverage` (istanbul/babel) | `--coverage` (provider v8) |

Diferencias clave a recordar:

```ts
// Jest: importas el actual con requireActual
jest.mock('./api', () => ({
  ...jest.requireActual('./api'),
  fetchTodos: jest.fn(),
}));

// Vitest: usas vi.importActual (¡es async!) y la factory async
vi.mock('./api', async () => {
  const actual = await vi.importActual<typeof import('./api')>('./api');
  return { ...actual, fetchTodos: vi.fn() };
});
```

```ts
// Importar el namespace explícitamente (recomendado con verbatimModuleSyntax):
import { describe, it, expect, vi, beforeEach } from 'vitest';
import type { Mock } from 'vitest';
```

> Vitest comparte la transformación de Vite, así que respeta tu `tsconfig`
> (incluido `verbatimModuleSyntax`): usa `import type` para los tipos. Migrar
> una suite de Jest suele reducirse a un *buscar y reemplazar* de `jest.` por
> `vi.` y ajustar la configuración.

---

### Prueba esto

- Ejecuta `vitest run --coverage` y abre `coverage/index.html`; localiza el archivo con menos branches cubiertos.
- Baja un threshold a un valor imposible (p. ej. `statements: 99`) y comprueba que el comando falla en CI.
- Reescribe un test que use `getByTestId` para que use `getByRole` y verifica que sigue pasando.
- Añade un test para la rama de error de `TodoLoader` y observa cómo sube la cobertura de branches.
- Toma una suite de Jest de otro proyecto y aplica las equivalencias de la tabla para migrarla a Vitest.
- Refactoriza un componente sin cambiar su salida y confirma que ningún test bien escrito se rompe.

---

## Ejercicios propuestos

### Básico

**B1 — Primer reporte.** Ejecuta `vitest run --coverage` y anota los cuatro
porcentajes globales (statements, branches, functions, lines). Identifica el
archivo con menor cobertura.

**B2 — Excluir un archivo.** Añade `src/main.tsx` (o cualquier entrypoint) a la
lista `exclude` y comprueba en la tabla que ya no aparece medido.

### Intermedio

**I1 — Subir branches.** Localiza un archivo con branches por debajo del 100% y
escribe el test que falta para cubrir la rama no ejecutada (típicamente un `else`
o un caso de error). Confirma que el porcentaje sube.

**I2 — Anti-patrón a query accesible.** Toma un test (real o inventado) que use
`getByTestId` y reescríbelo con la query accesible más adecuada según el orden de
preferencia. Verifica que sigue pasando.

### Avanzado

**A1 — Threshold por archivo.** Investiga los thresholds por patrón de archivo
(glob) en la config de Vitest y exige, por ejemplo, 100% a `src/hooks/**`
manteniendo umbrales globales más bajos. Demuestra que falla si un hook baja de 100%.

**A2 — Migración desde Jest.** Toma una mini-suite escrita con `jest.*` (puedes
adaptar un ejemplo del apéndice) y migra cada llamada a su equivalente `vi.*`,
incluyendo un `vi.mock` con `vi.importActual` async. Verifica que toda la suite
pasa bajo Vitest.

---

## Resumen del módulo 11

- La cobertura se mide con `vitest run --coverage` usando el provider **v8** (`@vitest/coverage-v8`).
- Configuramos `coverage` **por pasos**: primero `provider` y `reporter`, luego `include`/`exclude`, y por último `thresholds` que hacen fallar CI si la cobertura baja.
- El reporte distingue statements, branches, functions y lines; la cobertura indica qué se ejecuta, no qué se verifica.
- Testea el comportamiento observable (roles, texto, interacciones) y evita detalles de implementación, snapshots gigantes y abuso de `getByTestId`.
- Escribe tests legibles con el patrón AAA y organiza la infraestructura de pruebas en `src/test/`.
- Migrar desde Jest es directo: cambia `jest.` por `vi.`, usa `vi.importActual` (async) y mueve la config a `vite.config.ts`.

---

> **Has terminado el tutorial de Testing en React con Vitest.** Ya sabes probar
> componentes, hooks, peticiones asíncronas con MSW, Context y rutas, y mantener
> una suite saludable. El siguiente paso natural son las pruebas **end-to-end**:
> validar la aplicación completa en un navegador real, tal como la usa una
> persona. Continúa con el tutorial de **Cypress (E2E)** que encontrarás en la
> carpeta `react-cypress/`. ¡Nos vemos allí!
