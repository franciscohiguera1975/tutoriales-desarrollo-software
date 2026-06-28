# Testing en React con Vitest — Página 9

## Módulo 9 · Asíncrono y fetch con MSW

### Cómo testear componentes que cargan datos de una API sin mockear `fetch` a mano

---

## Cómo trabajar este módulo

Este módulo es **incremental**: vamos a montar la interceptación de red con MSW
**pieza a pieza** (handlers → servidor → setup) y luego testear los tres estados
de un componente que carga datos: **cargando**, **éxito** y **error**. No copies
todo de golpe — añade **un paso cada vez**, guarda y deja corriendo
`npm run test:watch`. Entre paso y paso encontrarás retos **🔧 Tu turno** que
debes resolver **antes** de mirar la solución.

```
Paso 1  →  instalamos y dibujamos la estructura de MSW
Paso 2  →  handlers.ts: qué responde cada endpoint
Paso 3  →  server.ts + setup.ts: conectamos al ciclo de vida
Paso 4  →  test del estado "cargando" (síncrono)
Paso 5  →  test del estado "éxito" con findBy* (Tu turno)
Paso 6  →  test del estado "error" con server.use()
Paso 7  →  esperar a que algo desaparezca (Tu turno)
```

---

En los módulos anteriores probamos componentes síncronos: les pasábamos props,
hacíamos clic y verificábamos el resultado de inmediato. Pero las aplicaciones
reales hablan con un servidor. Nuestra **Todo App** carga las tareas desde
`/api/todos`, y eso introduce **tiempo**: hay un instante de carga, luego datos
(o un error).

En este módulo aprenderás a testear ese flujo de forma realista usando **MSW**
(Mock Service Worker), la herramienta estándar para interceptar peticiones de
red en los tests.

---

## ¿Por qué no mockear `fetch` a mano?

La tentación es reemplazar `fetch` con un `vi.fn()`:

```ts
// ❌ Anti-patrón: mockear fetch directamente
globalThis.fetch = vi.fn().mockResolvedValue({
  ok: true,
  json: async () => [{ id: '1', text: 'Comprar pan', completed: false }],
} as Response);
```

Funciona… hasta que deja de hacerlo. Estos son los problemas:

| Problema | Mockear `fetch` a mano | Interceptar con MSW |
| --- | --- | --- |
| Fidelidad | Simulas un objeto `Response` incompleto | Petición HTTP real interceptada |
| Acoplamiento | Acoplado a que el código use `fetch` | Funciona con `fetch`, `axios`, etc. |
| Headers, status, body | Los reconstruyes tú mismo | MSW los maneja de forma nativa |
| Reutilización | Copias el mock en cada test | Handlers compartidos por toda la suite |
| Mantenimiento | Frágil ante refactors | Estable: solo importa la URL |

La idea clave de MSW es interceptar la petición **a nivel de red**, no a nivel
de función. Tu componente sigue llamando a `fetch('/api/todos')` exactamente
igual que en producción; MSW responde antes de que la petición salga.

```text
   Componente                 fetch('/api/todos')
       │                              │
       ▼                              ▼
  fetchTodos()  ───────────►  [ MSW intercepta aquí ]
                                      │
                                      ▼
                           handlers.ts → HttpResponse.json([...])
```

---

## El módulo de API y el componente que vamos a testear

Recordemos la firma de nuestra capa de datos:

```ts
// src/api/todos.ts
import type { Todo } from '../types';

// Obtiene las tareas desde el backend.
export async function fetchTodos(): Promise<Todo[]> {
  const res = await fetch('/api/todos');
  if (!res.ok) {
    throw new Error('No se pudieron cargar las tareas');
  }
  return res.json() as Promise<Todo[]>;
}
```

Y el componente que la consume, con sus tres estados (cargando, éxito, error):

```tsx
// src/components/TodoLoader.tsx
import { useEffect, useState } from 'react';
import type { Todo } from '../types';
import { fetchTodos } from '../api/todos';

// Componente de demostración: carga las tareas al montarse.
export function TodoLoader() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let cancelado = false;

    fetchTodos()
      .then((data) => {
        if (!cancelado) setTodos(data);
      })
      .catch((err: unknown) => {
        // Capturamos el mensaje de error para mostrarlo en pantalla.
        if (!cancelado) {
          setError(err instanceof Error ? err.message : 'Error desconocido');
        }
      })
      .finally(() => {
        if (!cancelado) setLoading(false);
      });

    return () => {
      cancelado = true;
    };
  }, []);

  if (loading) return <p role="status">Cargando tareas…</p>;
  if (error) return <p role="alert">{error}</p>;

  return (
    <ul aria-label="Lista de tareas">
      {todos.map((todo) => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}
```

> Nota el uso de roles ARIA (`status`, `alert`). Nos permiten consultar el
> estado del componente con queries accesibles en lugar de `data-testid`.

---

## Paso 1 — Instalar MSW y dibujar la estructura

Instalamos MSW como dependencia de desarrollo:

```bash
npm i -D msw
```

> En tests con Node (jsdom) usamos `setupServer`, no el service worker del
> navegador. No necesitas ejecutar `npx msw init`; eso es solo para el modo
> navegador.

La estructura de archivos que vamos a crear, **uno por paso**:

```text
src/
└── test/
    ├── mocks/
    │   ├── handlers.ts   ← Paso 2: qué responde cada endpoint
    │   └── server.ts     ← Paso 3: el servidor de interceptación
    └── setup.ts          ← Paso 3: conecta el servidor al ciclo de vida
```

---

## Paso 2 — Definir los handlers

Un *handler* describe cómo responder a una petición. Empezamos con el caso feliz:
un solo handler que responde `GET /api/todos` con datos. En MSW 2 usamos el
namespace `http` y la clase `HttpResponse`:

```ts
// src/test/mocks/handlers.ts
import { http, HttpResponse } from 'msw';
import type { Todo } from '../../types';

// Datos por defecto que devolverá el endpoint en los tests "felices".
export const todosDeEjemplo: Todo[] = [
  { id: '1', text: 'Aprender Vitest', completed: false },
  { id: '2', text: 'Configurar MSW', completed: true },
];

export const handlers = [
  // Intercepta GET /api/todos y responde con JSON.
  http.get('/api/todos', () => {
    return HttpResponse.json(todosDeEjemplo);
  }),
];
```

Si vienes de MSW 1, esta es la traducción mental:

| Concepto MSW 1 (antiguo) | Concepto MSW 2 (actual) |
| --- | --- |
| `rest.get(url, (req, res, ctx) => ...)` | `http.get(url, () => ...)` |
| `res(ctx.json(data))` | `HttpResponse.json(data)` |
| `res(ctx.status(500))` | `new HttpResponse(null, { status: 500 })` |

---

## Paso 3 — Crear el servidor y conectarlo al ciclo de vida

Dos archivos pequeños. Primero el servidor que aplica nuestros handlers:

```ts
// src/test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

// El servidor intercepta peticiones en el entorno Node de los tests.
export const server = setupServer(...handlers);
```

Y el `setup.ts`, que se ejecuta una vez antes de toda la suite (referenciado en
`vite.config.ts` con `test.setupFiles`). El patrón de **tres ganchos** es esencial:

```ts
// src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { afterAll, afterEach, beforeAll } from 'vitest';
import { server } from './mocks/server';

// 1. Antes de todos los tests: empieza a interceptar.
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));

// 2. Después de CADA test: limpia los handlers añadidos puntualmente.
afterEach(() => server.resetHandlers());

// 3. Al terminar la suite: deja de interceptar y libera recursos.
afterAll(() => server.close());
```

| Gancho | Cuándo | Por qué importa |
| --- | --- | --- |
| `beforeAll(listen)` | Una vez, al inicio | Activa la interceptación |
| `afterEach(resetHandlers)` | Tras cada test | Aísla tests: deshace `server.use()` |
| `afterAll(close)` | Una vez, al final | Evita fugas entre suites |

> `onUnhandledRequest: 'error'` hace fallar el test si tu código pide una URL
> que no tiene handler. Es un seguro contra peticiones reales accidentales.

Y la referencia desde la configuración de Vitest:

```ts
// vite.config.ts
import { defineConfig } from 'vitest/config'; // ← desde vitest/config, NO desde 'vite'
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'], // ← aquí
  },
});
```

Con esto la infraestructura está lista. A partir de aquí solo escribimos tests.

---

## Paso 4 — Testear el estado de carga (síncrono)

Justo después de renderizar, antes de que la promesa se resuelva, el componente
muestra el mensaje de carga. Lo capturamos de forma **síncrona** con `getBy*`:

```tsx
// src/components/TodoLoader.test.tsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { TodoLoader } from './TodoLoader';

describe('TodoLoader', () => {
  it('muestra el estado de carga al montarse', () => {
    render(<TodoLoader />);

    // getBy* es síncrono: el mensaje existe en el primer render.
    expect(screen.getByRole('status')).toHaveTextContent('Cargando tareas…');
  });
});
```

Guarda: **1 passed**. Este es el único estado que existe **antes** de que la red
responda, por eso `getBy*` (síncrono) basta.

---

## Paso 5 — 🔧 Tu turno #1: testear el estado de éxito con `findBy*`

Aquí está el corazón del módulo. Cuando esperamos algo que aparecerá **en el
futuro**, NO usamos `getBy*` (lanza error de inmediato si no existe). Usamos
`findBy*`, que devuelve una promesa y reintenta hasta encontrar el elemento o
agotar el timeout.

**Tu reto:** añade un `it` (async) que renderice `TodoLoader`, espere con
`findByRole` a que aparezca la lista (`role="list"`, nombre "Lista de tareas") y
verifique que se ven los dos textos de `todosDeEjemplo`.

<details><summary>✅ Solución</summary>

```tsx
// src/components/TodoLoader.test.tsx (continúa)
it('muestra las tareas cuando la petición tiene éxito', async () => {
  render(<TodoLoader />);

  // findBy* = getBy* + waitFor. Espera a que aparezca el elemento.
  const lista = await screen.findByRole('list', { name: 'Lista de tareas' });

  expect(lista).toBeInTheDocument();
  expect(screen.getByText('Aprender Vitest')).toBeInTheDocument();
  expect(screen.getByText('Configurar MSW')).toBeInTheDocument();
});
```

La clave es el `await screen.findByRole(...)`. Una vez la lista existe, los textos
ya están presentes, así que para ellos basta `getByText` (síncrono). Si usaras
`getByRole('list')` directamente, fallaría: en el primer render aún se ve
"Cargando…".

</details>

### `findBy*` vs `waitFor`

Ambos esperan, pero se usan en situaciones distintas:

| API | Devuelve | Úsalo para |
| --- | --- | --- |
| `findByRole(...)` | Promise del elemento | Esperar a que **aparezca** un elemento |
| `waitFor(cb)` | Promise<void> | Esperar a que una **aserción** se cumpla |

`waitFor` es más general: reintenta el callback hasta que no lance error.

```tsx
// src/components/TodoLoader.test.tsx (variante con waitFor)
import { waitFor } from '@testing-library/react';

it('renderiza dos elementos de tarea (con waitFor)', async () => {
  render(<TodoLoader />);

  await waitFor(() => {
    // El callback se reintenta hasta que la lista tenga 2 elementos.
    expect(screen.getAllByRole('listitem')).toHaveLength(2);
  });
});
```

> Regla práctica: si esperas un **elemento**, usa `findBy*`. Si esperas que una
> **condición** se vuelva cierta, usa `waitFor`. Evita meter varias aserciones
> dentro de un mismo `waitFor`; la primera que falle ocultará a las demás.

---

## Paso 6 — Testear el estado de error con `server.use()`

Para probar el camino de error, sobrescribimos el handler **solo en este test**
con `server.use()`. Gracias a `resetHandlers()` en `afterEach`, este override
no contamina los demás tests.

```tsx
// src/components/TodoLoader.test.tsx (continúa)
import { http, HttpResponse } from 'msw';
import { server } from '../test/mocks/server';

it('muestra un mensaje de error si la petición falla', async () => {
  // Override puntual: este endpoint responderá 500 solo en este test.
  server.use(
    http.get('/api/todos', () => {
      return new HttpResponse(null, { status: 500 });
    }),
  );

  render(<TodoLoader />);

  // role="alert" aparece cuando setError se ejecuta.
  const alerta = await screen.findByRole('alert');
  expect(alerta).toHaveTextContent('No se pudieron cargar las tareas');
});
```

```text
  Test "success"            Test "error"
  ───────────              ───────────
  handler base             server.use(handler 500)
  responde 200       →     responde 500
       │                        │
       ▼                        ▼
  afterEach: resetHandlers()  ← borra el override 500
       │
       ▼
  el siguiente test vuelve a tener el handler base
```

El override vive **solo** mientras dura el test; el `afterEach(resetHandlers)` del
Paso 3 lo borra automáticamente. Por eso no contaminamos los demás casos.

---

## Paso 7 — 🔧 Tu turno #2: esperar a que algo desaparezca

Un caso muy común: el spinner de carga debe **desaparecer** cuando llegan los
datos. Para verificar una desaparición usamos `waitForElementToBeRemoved`.

**Tu reto:** escribe un `it` que renderice `TodoLoader`, capture el elemento con
`role="status"` mientras está presente, espere a que se elimine del DOM y luego
verifique que ya no está y que en su lugar aparece la lista.

<details><summary>✅ Solución</summary>

```tsx
// src/components/TodoLoader.test.tsx (continúa)
import { waitForElementToBeRemoved } from '@testing-library/react';

it('quita el mensaje de carga cuando llegan los datos', async () => {
  render(<TodoLoader />);

  const cargando = screen.getByRole('status');
  expect(cargando).toBeInTheDocument();

  // Espera hasta que el nodo se elimine del DOM.
  await waitForElementToBeRemoved(cargando);

  expect(screen.queryByRole('status')).not.toBeInTheDocument();
  expect(screen.getByRole('list')).toBeInTheDocument();
});
```

> Importante: pásale el **elemento** ya capturado (o una función que lo
> devuelva). Si el elemento nunca existió, `waitForElementToBeRemoved` lanzará
> un error, lo cual es útil para detectar tests mal planteados. Fíjate también en
> `queryByRole`: a diferencia de `getByRole`, devuelve `null` en vez de lanzar,
> por eso es el que usamos para afirmar **ausencia**.

</details>

---

## Simular latencia con `delay`

A veces quieres asegurarte de que el spinner se ve durante la espera. MSW
incluye `delay`:

```ts
// src/test/mocks/handlers.ts (handler alternativo)
import { http, HttpResponse, delay } from 'msw';

export const handlerConRetraso = http.get('/api/todos', async () => {
  await delay(150); // simula 150 ms de red
  return HttpResponse.json(todosDeEjemplo);
});
```

```tsx
// uso en un test concreto
server.use(handlerConRetraso);
render(<TodoLoader />);
expect(screen.getByRole('status')).toBeInTheDocument(); // aún cargando
await screen.findByRole('list'); // ya llegaron los datos
```

---

### Prueba esto

- Añade un tercer todo en `todosDeEjemplo` y comprueba que `getAllByRole('listitem')` devuelve 3.
- Crea un test que use `server.use()` para devolver una **lista vacía** y verifica que se muestra "No hay tareas" (tendrás que añadir ese caso al componente).
- Cambia `onUnhandledRequest` a `'warn'` y observa la diferencia cuando pides una URL sin handler.
- Reemplaza un `findByRole` por `getByRole` en el test de éxito y observa cómo falla; entiende por qué.
- Usa `delay('infinite')` en un handler para dejar el componente eternamente cargando y verifica que el spinner permanece.
- Sobrescribe el handler para responder un status `404` y comprueba que también entra por la rama de error.

---

## Ejercicios propuestos

### Básico

**B1 — Texto del spinner.** Escribe un `it` que verifique que el `role="status"`
contiene exactamente el texto "Cargando tareas…" en el primer render.

**B2 — Conteo de items.** Con el handler base, escribe un `it` (async) que espere
la lista y verifique con `getAllByRole('listitem')` que hay exactamente 2 items.

### Intermedio

**I1 — Lista vacía.** Usa `server.use()` para que el endpoint responda `[]` y
verifica que la lista existe pero no contiene ningún `listitem`. (Decide qué
query usar para "cero elementos".)

**I2 — Otro status de error.** Crea un test que sobrescriba el handler para
responder `404` y comprueba que el componente entra igualmente por la rama de
error mostrando el `role="alert"`.

### Avanzado

**A1 — Latencia y spinner garantizado.** Usa un handler con `delay(200)` para
afirmar **primero** (síncrono) que el spinner está visible y **después** (con
`findBy*`) que la lista aparece. Demuestra que ambos estados coexisten en el tiempo.

**A2 — Leer el cuerpo de la petición.** Añade un handler `http.post('/api/todos',
...)` que lea el body con `await request.json()` y devuelva el todo creado con un
`id`. Testea un componente que envíe una nueva tarea y verifica que aparece en la
lista con el id asignado por el "servidor".

---

## Resumen del módulo 9

- Mockear `fetch` a mano es frágil; **MSW** intercepta a nivel de red y es agnóstico al cliente HTTP.
- En MSW 2 se usan `http.get/post/...` y `HttpResponse.json(...)`, junto con `setupServer` para el entorno Node.
- Montamos la infraestructura **por pasos**: handlers → servidor → setup con los tres ganchos `beforeAll(listen)` + `afterEach(resetHandlers)` + `afterAll(close)`.
- Usa `getBy*` para el estado de carga (síncrono), `findBy*` para esperar a que aparezca un elemento y `waitFor` para esperar a que se cumpla una aserción.
- Para probar errores, sobrescribe el handler con `server.use()` devolviendo un status 500; `resetHandlers()` lo limpia entre tests.
- `waitForElementToBeRemoved` verifica que un elemento (como un spinner) desaparece del DOM.

---

> **Siguiente página →** Módulo 10: testear componentes que dependen de Context (AuthContext) y de react-router con `MemoryRouter`, además de crear un `render` helper reutilizable.
