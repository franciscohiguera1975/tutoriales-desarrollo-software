# Testing en React con Vitest — Página 2
## Módulo 2 · Anatomía de un test
### `describe`, `it`/`test`, `expect`, el patrón AAA y nuestro primer test real

---

## Cómo trabajar este módulo

Este módulo es **incremental**: construiremos la suite de tests de `TodoItem`
paso a paso. No copies el archivo final de golpe — añade **un bloque por paso**,
guarda y ejecuta `npm run test:watch` para ver cómo cada cambio se refleja al
instante. Entre paso y paso encontrarás retos **🔧 Tu turno** que debes resolver
**antes** de mirar la solución.

```
Paso 1  →  el test mínimo que pasa
Paso 2  →  añadimos un segundo caso
Paso 3  →  refactor con un helper
Paso 4  →  tú escribes el siguiente caso
```

---

## La estructura de un archivo de test

Un test en Vitest tiene tres piezas que se repiten una y otra vez:

```
describe(...)   ← agrupa tests relacionados (una "suite")
   │
   ├── it(...)  ← un caso de prueba concreto
   │     │
   │     └── expect(...)  ← una aserción: lo que debe cumplirse
   │
   └── it(...)
```

- **`describe(nombre, fn)`** agrupa varios casos bajo un mismo tema. No es obligatorio, pero organiza la salida y permite anidar.
- **`it(nombre, fn)`** (o su alias **`test`**) declara un caso concreto. El nombre debe leerse como una frase: *"debería mostrar el texto de la tarea"*.
- **`expect(valor)`** crea una aserción sobre un valor; se encadena con un *matcher* (`toBe`, `toBeInTheDocument`, etc.).

### `it` vs `test`: ¿cuál uso?

Son **exactamente lo mismo**; `test` es un alias de `it`. La elección es estilística. En este curso usaremos **`it`** con descripciones en español que empiezan por "debería", porque produce una salida muy legible:

```
TodoItem
  ✓ debería mostrar el texto de la tarea
  ✓ debería renderizar un checkbox
```

> Elige una convención y sé consistente. Mezclar `it` y `test` no rompe nada, pero distrae.

---

## El patrón AAA: Arrange, Act, Assert

Todo buen test sigue tres fases, conocidas como **AAA**:

```
┌──────────────────────────────────────────────┐
│ ARRANGE  Preparar: datos, render, mocks        │
├──────────────────────────────────────────────┤
│ ACT      Actuar: la acción que se prueba        │
├──────────────────────────────────────────────┤
│ ASSERT   Afirmar: comprobar el resultado        │
└──────────────────────────────────────────────┘
```

- **Arrange (preparar):** montas el escenario. Creas los datos de entrada, renderizas el componente y configuras los *mocks*.
- **Act (actuar):** ejecutas la acción bajo prueba. Puede ser un clic, escribir en un input o, en el caso más simple, el propio render.
- **Assert (afirmar):** verificas que el resultado es el esperado con uno o varios `expect`.

Separar estas fases con comentarios hace que cualquiera entienda el test de un vistazo.

---

## El componente que vamos a probar: `TodoItem`

Antes de testear, necesitamos el componente. Crea este archivo — es la firma que mantendremos durante todo el curso:

```tsx
// src/components/TodoItem.tsx
import type { Todo } from '../types';

interface TodoItemProps {
  todo: Todo;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}

export function TodoItem({ todo, onToggle, onDelete }: TodoItemProps) {
  return (
    <li>
      {/* El checkbox refleja si la tarea está completada */}
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
        aria-label={`Marcar "${todo.text}"`}
      />
      {/* El texto de la tarea */}
      <span>{todo.text}</span>
      {/* Botón para eliminar la tarea */}
      <button onClick={() => onDelete(todo.id)}>Eliminar</button>
    </li>
  );
}
```

> Fíjate en `import type { Todo }`: como el proyecto usa `verbatimModuleSyntax`, los tipos se importan con `type`.

Y el tipo, si aún no lo tienes:

```ts
// src/types.ts
export interface Todo {
  id: string;
  text: string;
  completed: boolean;
}
```

---

## `render` y `screen` de Testing Library

Para probar un componente necesitamos **montarlo en el DOM (jsdom)** y luego **consultarlo**:

- **`render(<Componente />)`** monta el componente en un contenedor real de jsdom. Es la fase *Arrange*.
- **`screen`** es un objeto global con *queries* (`getByText`, `getByRole`...) que buscan en **todo el documento** renderizado. Es la fase *Assert*.

```
render(<TodoItem ... />)
        │  monta el componente en el document de jsdom
        ▼
   document.body
        │  ahora screen puede consultarlo
        ▼
screen.getByText('Comprar pan')  ──► devuelve el nodo o lanza error
```

---

## Paso 1 — El test mínimo que pasa

Empezamos por lo más pequeño posible: comprobar que `TodoItem` **renderiza el texto de la tarea**. Crea el archivo de test con **solo este caso**:

```tsx
// src/components/TodoItem.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { TodoItem } from './TodoItem';
import type { Todo } from '../types';

describe('TodoItem', () => {
  it('debería mostrar el texto de la tarea', () => {
    // Arrange: preparamos una tarea de ejemplo y funciones vacías.
    const todo: Todo = { id: '1', text: 'Comprar pan', completed: false };

    // Act: renderizamos el componente (aquí el "acto" es el propio render).
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);

    // Assert: el texto de la tarea debe estar en el documento.
    expect(screen.getByText('Comprar pan')).toBeInTheDocument();
  });
});
```

Guarda y mira la terminal. Deberías ver **1 passed**. Desglose:

- `screen.getByText('Comprar pan')` busca un elemento cuyo texto sea exactamente "Comprar pan". Si no lo encuentra, **lanza un error** y el test falla con un mensaje claro.
- `.toBeInTheDocument()` es un *matcher* de **jest-dom** que confirma que el nodo está presente. Por eso registramos jest-dom en `setup.ts` en el módulo 1.

### 🔧 Tu turno #1

Sin mirar abajo: **rompe el test a propósito**. Cambia el `text` de la tarea a `'Comprar leche'` pero **deja** el `expect` buscando `'Comprar pan'`. Antes de guardar, predice qué dirá el error.

<details><summary>✅ Solución / qué observar</summary>

El test falla con `Unable to find an element with the text: Comprar pan` y, muy útil, **lista los textos que sí encontró** (`Comprar leche`). Aprende a leer este mensaje: te dice *qué buscó* y *qué había*. Vuelve a dejar ambos en `'Comprar pan'` para que pase.

</details>

---

## Paso 2 — Añadimos un segundo caso

Ahora verificamos que existe el **checkbox**. Añade un segundo `it` dentro del mismo `describe` (no borres el anterior):

```tsx
// src/components/TodoItem.test.tsx  (añade este it dentro del describe)
  it('debería renderizar un checkbox', () => {
    // Arrange
    const todo: Todo = { id: '1', text: 'Comprar pan', completed: false };
    // Act
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    // Assert: el rol "checkbox" identifica el <input type="checkbox">
    expect(screen.getByRole('checkbox')).toBeInTheDocument();
  });
```

Guarda. Ahora la salida muestra **2 passed**. Nota que volvimos a escribir el `const todo` y el `render` — empieza a oler a repetición. Lo arreglamos en el paso 3.

---

## Paso 3 — Refactor: un helper para no repetirnos

Repetir el `const todo` en cada test es ruido. Introducimos un *helper* con **datos por defecto sobreescribibles** mediante `Partial<Todo>`:

```tsx
// src/components/TodoItem.test.tsx  (añade arriba, tras los imports)
// Helper local: crea una tarea con valores por defecto sobreescribibles.
function crearTodo(overrides: Partial<Todo> = {}): Todo {
  return { id: '1', text: 'Comprar pan', completed: false, ...overrides };
}
```

Y reescribe los dos tests para usarlo:

```tsx
  it('debería mostrar el texto de la tarea', () => {
    const todo = crearTodo({ text: 'Estudiar Vitest' });
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    expect(screen.getByText('Estudiar Vitest')).toBeInTheDocument();
  });

  it('debería renderizar un checkbox', () => {
    const todo = crearTodo();
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    expect(screen.getByRole('checkbox')).toBeInTheDocument();
  });
```

Cada test ahora declara **solo lo que le importa**. Esta es una de las buenas prácticas más rentables del testing.

---

## Paso 4 — 🔧 Tu turno #2: escribe el tercer caso

El componente también pinta un **botón "Eliminar"**. Escribe tú mismo un tercer `it` que verifique que ese botón está en el documento.

**Pistas:**
- Usa `crearTodo()` para la tarea.
- El botón es un `role="button"`. Puedes filtrarlo por su nombre accesible con `getByRole('button', { name: 'Eliminar' })`.

Inténtalo antes de mirar.

<details><summary>✅ Solución</summary>

```tsx
  it('debería mostrar un botón de eliminar', () => {
    const todo = crearTodo();
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    expect(
      screen.getByRole('button', { name: 'Eliminar' }),
    ).toBeInTheDocument();
  });
```

Con esto la suite llega a **3 passed**. Si pusiste otro nombre que no existe, el error de `getByRole` te listará los roles disponibles.

</details>

---

## El archivo completo (referencia)

Tras los cuatro pasos, tu archivo debería verse así. Úsalo solo para **comparar**, no para copiar:

```tsx
// src/components/TodoItem.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { TodoItem } from './TodoItem';
import type { Todo } from '../types';

function crearTodo(overrides: Partial<Todo> = {}): Todo {
  return { id: '1', text: 'Comprar pan', completed: false, ...overrides };
}

describe('TodoItem', () => {
  it('debería mostrar el texto de la tarea', () => {
    const todo = crearTodo({ text: 'Estudiar Vitest' });
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    expect(screen.getByText('Estudiar Vitest')).toBeInTheDocument();
  });

  it('debería renderizar un checkbox', () => {
    const todo = crearTodo();
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    expect(screen.getByRole('checkbox')).toBeInTheDocument();
  });

  it('debería mostrar un botón de eliminar', () => {
    const todo = crearTodo();
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    expect(screen.getByRole('button', { name: 'Eliminar' })).toBeInTheDocument();
  });
});
```

---

## Cleanup automático

Cada test renderiza el componente en el DOM. ¿Qué pasa con el render anterior? Si no se limpiara, el segundo test encontraría **dos** checkboxes y `getByRole('checkbox')` fallaría por ambigüedad.

Aquí entra el **cleanup automático**. Con jest-dom registrado (o `globals: true`), Testing Library desmonta los componentes y vacía el DOM **después de cada `it`**:

```
it #1  render → DOM tiene 1 TodoItem
        └─ cleanup automático → DOM vacío
it #2  render → DOM tiene 1 TodoItem (limpio)
        └─ cleanup automático → DOM vacío
```

> No necesitas llamar a `cleanup()` manualmente. Eso era obligatorio en configuraciones antiguas; con Vitest moderno viene gratis.

---

## Hooks de ciclo de vida (vistazo rápido)

A veces quieres preparar o limpiar algo común a varios tests. Vitest ofrece *hooks* alrededor de los `it`:

```ts
// Fragmento ilustrativo
import { beforeEach, afterEach, beforeAll, afterAll } from 'vitest';

beforeAll(() => {/* una vez, antes de toda la suite */});
beforeEach(() => {/* antes de cada it */});
afterEach(() => {/* después de cada it (p. ej. restaurar mocks) */});
afterAll(() => {/* una vez, al terminar la suite */});
```

| Hook | Frecuencia | Uso típico |
|------|-----------|------------|
| `beforeAll` | Una vez | Levantar un servidor de mocks |
| `beforeEach` | Antes de cada `it` | Reiniciar datos compartidos |
| `afterEach` | Después de cada `it` | `vi.restoreAllMocks()` |
| `afterAll` | Una vez | Apagar recursos |

No los necesitamos todavía, pero en el módulo 8 (mocking) y el 9 (MSW) serán protagonistas.

---

## Ejecutar en modo watch y con `--ui`

El modo *watch* re-ejecuta solo los tests afectados al guardar:

```bash
# Modo watch: se queda escuchando cambios
npm run test:watch
```

En la terminal puedes pulsar teclas para filtrar:

```
 PASS  Waiting for file changes...
       press h to show help, press q to quit
       press p to filter by filename
       press t to filter by test name
```

Y la interfaz visual, ideal para suites grandes:

```bash
# Abre un panel en el navegador con el estado de cada test
npm run test:ui
```

---

## Cómo se lee la salida en consola

Así se ve nuestra suite de `TodoItem` al pasar:

```
 ✓ src/components/TodoItem.test.tsx (3 tests) 12ms
   ✓ TodoItem > debería mostrar el texto de la tarea
   ✓ TodoItem > debería renderizar un checkbox
   ✓ TodoItem > debería mostrar un botón de eliminar

 Test Files  1 passed (1)
      Tests  3 passed (3)
```

Y cuando algo falla, el mensaje señala el `expect` culpable:

```
 FAIL  src/components/TodoItem.test.tsx > TodoItem > debería mostrar el texto
 Error: Unable to find an element with the text: Comprar pan.

  ❯ src/components/TodoItem.test.tsx:14:18
```

> Casi siempre el error te dice *qué* buscó y *qué* encontró. `getByText` incluso sugiere texto similar.

---

### Prueba esto

- Marca un `it` con `it.only(...)` y comprueba que Vitest ejecuta **solo** ese caso. Luego usa `it.skip(...)` para saltarlo.
- Renombra un `it` a `test` y confirma que el comportamiento es idéntico.
- Añade un segundo `render` dentro del mismo `it` y observa cómo `getByRole('checkbox')` falla por encontrar dos elementos.
- Quita el *helper* `crearTodo` y repite los datos a mano; aprecia el ruido que añade.
- Cambia `getByText` por `getByText('comprar pan')` (minúsculas) y observa que falla: por defecto distingue mayúsculas.
- Añade `screen.debug()` justo después del `render` y mira el HTML que imprime en consola.

---

## Ejercicios propuestos

### Básico

**B1 — Caso "completada".** Escribe un `it` que cree una tarea con `completed: true` (usa `crearTodo({ completed: true })`) y verifique que el texto sigue mostrándose. *(Aún no comprobamos el estado del checkbox; eso llega en el módulo 4.)*

**B2 — Texto dinámico.** Parametriza el texto: crea tres tareas con textos distintos y un `it` por cada una que verifique su texto. Observa cómo se ven los tres casos en la salida.

### Intermedio

**I1 — `describe` anidado.** Reorganiza la suite con dos `describe` internos: `describe('contenido visible', ...)` (texto, checkbox, botón) y `describe('accesibilidad', ...)` con un test que verifique que el checkbox tiene el `aria-label` correcto usando `getByRole('checkbox', { name: /Marcar/ })`.

**I2 — Helper de render.** Crea un helper `renderItem(overrides?)` que combine `crearTodo` + `render` y devuelva nada (solo monta). Reescribe los tests para que cada uno sea de **dos líneas**: `renderItem(...)` y el `expect`.

### Avanzado

**A1 — `it.each`.** Investiga `it.each` de Vitest y úsalo para generar, desde una tabla de `[texto]`, varios tests que verifiquen el render del texto. Objetivo: una sola declaración que produzca N casos en la salida.

**A2 — Test que documenta un bug.** El componente no muestra ninguna marca visual de "completada" más allá del checkbox. Escribe un `it` (que **fallará**, márcalo con `it.fails(...)` o `it.todo`) describiendo el comportamiento deseado: *"debería tachar el texto cuando la tarea está completada"*. Esto introduce el TDD: el test primero, la implementación después (la harás en el módulo 6).

---

## Resumen del módulo 2

- Un test se estructura con `describe` (suite), `it`/`test` (caso) y `expect` (aserción).
- `it` y `test` son alias; elige uno. Usamos `it` con frases que empiezan por "debería".
- El patrón **AAA** (Arrange, Act, Assert) organiza cada test en preparación, acción y verificación.
- `render` monta el componente en jsdom; `screen` ofrece las queries para consultar todo el documento.
- Construimos la suite de `TodoItem` **por pasos**: del test mínimo, a un segundo caso, a un refactor con helper, hasta escribir tú mismo el tercero.
- El **cleanup automático** desmonta el componente tras cada `it`, evitando contaminación entre tests.
- Los hooks `beforeEach`/`afterEach`/`beforeAll`/`afterAll` preparan y limpian estado compartido.
- El modo *watch* y la UI aceleran el desarrollo; la salida de consola señala con precisión los fallos.

---

> **Siguiente página →** Módulo 3: Queries de RTL — las familias `getBy`/`queryBy`/`findBy`, la prioridad recomendada de queries y un enfoque centrado en accesibilidad.
