# Testing en React con Vitest — Página 3
## Módulo 3 · Queries de RTL
### Familias `getBy`/`queryBy`/`findBy`, variantes `*AllBy`, prioridad de queries y accesibilidad

---

## Cómo trabajar este módulo

Este módulo es **incremental**: construiremos la suite de queries de `LoginForm`
y `TodoList` **un caso a la vez**, no de un solo bloque. Mantén abierto el modo
watch para ver cada `it` reflejarse al guardar:

```bash
npm run test:watch
```

Iremos descubriendo las tres familias de queries probando una y prediciendo qué
pasa cuando el elemento existe, no existe o aparece más tarde. Resuelve los retos
**🔧 Tu turno** **antes** de mirar la solución colapsable.

```
Paso 1  →  getBy: el caso que SÍ existe
Paso 2  →  queryBy: afirmar que algo NO existe
Paso 3  →  getAllBy: cuando esperamos varios
Paso 4  →  tú eliges la query correcta
```

---

## ¿Qué es una *query*?

Una **query** es una función de Testing Library que busca elementos en el DOM renderizado. Es el corazón de cada test: para afirmar algo sobre un componente, primero hay que **encontrar** el elemento.

Todas las queries siguen el mismo patrón de nombre:

```
[ get | query | find ]  +  [ By | AllBy ]  +  [ Role | Text | LabelText | ... ]
   │                          │                  │
   variante                   ¿uno o varios?     criterio de búsqueda
```

Ejemplos: `getByRole`, `queryByText`, `findAllByLabelText`. Entender estas tres dimensiones —variante, cardinalidad y criterio— te da acceso a docenas de queries con solo aprender unas pocas reglas.

---

## Las tres familias: `getBy`, `queryBy`, `findBy`

La **variante** decide qué pasa cuando el elemento no existe y si la búsqueda es síncrona o asíncrona.

| Familia | Si encuentra 1 | Si no encuentra | Si encuentra varios | Sync/Async | Uso típico |
|---------|----------------|-----------------|---------------------|------------|------------|
| `getBy*` | Devuelve el nodo | **Lanza error** | Lanza error | Síncrono | Afirmar que algo **existe** |
| `queryBy*` | Devuelve el nodo | Devuelve **`null`** | Lanza error | Síncrono | Afirmar que algo **NO existe** |
| `findBy*` | Devuelve el nodo (Promise) | **Rechaza** tras *timeout* | Rechaza | Asíncrono | Esperar algo que **aparecerá** |

La regla mental:

```
¿El elemento DEBE estar ahora?           → getBy
¿Compruebo que el elemento NO está?      → queryBy   (... ).not.toBeInTheDocument()
¿El elemento APARECERÁ tras una espera?  → findBy   (await)
```

- **`getBy`** falla con un mensaje muy descriptivo si no encuentra el elemento. Por eso es la opción por defecto para "esto debería estar aquí".
- **`queryBy`** es la **única** que devuelve `null` en lugar de lanzar error. Es la correcta para aserciones negativas, porque `getBy` reventaría antes de poder afirmar el `.not`.
- **`findBy`** devuelve una *Promise* y reintenta durante un *timeout* (por defecto ~1000 ms). Se usa con `await` para esperar contenido asíncrono (lo explotaremos en el módulo 9).

```tsx
// Patrones característicos de cada familia
// Existe ahora:
expect(screen.getByText('Comprar pan')).toBeInTheDocument();
// No existe:
expect(screen.queryByText('No existe')).not.toBeInTheDocument();
// Aparecerá (asíncrono):
expect(await screen.findByText('Cargado')).toBeInTheDocument();
```

---

## El componente `LoginForm`

Vamos a practicar sobre un formulario de login. Esta es la firma que mantenemos en todo el curso. Créalo antes de testear:

```tsx
// src/components/LoginForm.tsx
import { useState } from 'react';

interface LoginFormProps {
  onLogin: (username: string, password: string) => void;
}

export function LoginForm({ onLogin }: LoginFormProps) {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    onLogin(username, password);
  }

  return (
    <form onSubmit={handleSubmit} aria-label="Formulario de acceso">
      {/* Label asociada por htmlFor + id → getByLabelText */}
      <label htmlFor="username">Usuario</label>
      <input
        id="username"
        value={username}
        placeholder="Tu usuario"
        onChange={(e) => setUsername(e.target.value)}
      />

      <label htmlFor="password">Contraseña</label>
      <input
        id="password"
        type="password"
        value={password}
        placeholder="Tu contraseña"
        onChange={(e) => setPassword(e.target.value)}
      />

      <button type="submit">Entrar</button>
    </form>
  );
}
```

---

## Paso 1 — `getBy`: el caso que SÍ existe

Empezamos por la familia más común. Crea el archivo de test con **un solo caso**: encontrar los campos por su `<label>`. Esta es la query preferida para formularios:

```tsx
// src/components/LoginForm.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { LoginForm } from './LoginForm';

describe('LoginForm · queries', () => {
  it('debería encontrar campos por su label (getByLabelText)', () => {
    // Arrange
    render(<LoginForm onLogin={() => {}} />);
    // Assert: la forma preferida para campos de formulario
    expect(screen.getByLabelText('Usuario')).toBeInTheDocument();
    expect(screen.getByLabelText('Contraseña')).toBeInTheDocument();
  });
});
```

Guarda y mira la terminal: **1 passed**. `getByLabelText` localiza el `<input>`
gracias al `<label htmlFor="username">` asociado por `id`. Si la asociación se
rompiera, esta query lanzaría error.

### 🔧 Tu turno #1

Sin mirar abajo: añade un segundo `it` que encuentre el **botón** por su rol y
nombre, y un tercero que encuentre el **formulario** por su rol con `name`.
Pista: el nombre accesible de un `<button>` es su texto; el del `<form>` viene
del `aria-label`.

<details><summary>✅ Solución</summary>

```tsx
  it('debería encontrar el botón por rol y nombre (getByRole)', () => {
    render(<LoginForm onLogin={() => {}} />);
    // El nombre accesible de un button es su texto.
    expect(
      screen.getByRole('button', { name: 'Entrar' }),
    ).toBeInTheDocument();
  });

  it('debería encontrar el formulario por su rol con name', () => {
    render(<LoginForm onLogin={() => {}} />);
    // aria-label define el nombre accesible del <form>.
    expect(
      screen.getByRole('form', { name: 'Formulario de acceso' }),
    ).toBeInTheDocument();
  });
```

Con esto la suite va por **3 passed**. Si quitaras el `aria-label` del `<form>`,
el último test fallaría: sin nombre accesible, el `<form>` no expone el rol
`form`.

</details>

---

## `getByRole` con `name`: la query estrella

`getByRole` busca por el **rol de accesibilidad** del elemento, lo que coincide con cómo lo perciben los lectores de pantalla. Es la query preferida porque **prueba accesibilidad de paso**.

Cada elemento HTML tiene un rol implícito:

```
<button>          → role="button"
<input type=text> → role="textbox"
<input type=checkbox> → role="checkbox"
<a href>          → role="link"
<li>              → role="listitem"
<ul>/<ol>         → role="list"
<h1>..<h6>        → role="heading"
```

El parámetro `name` filtra por el **nombre accesible** (el texto del botón, la label del input, etc.):

```tsx
// Encuentra el botón cuyo nombre accesible es "Eliminar"
screen.getByRole('button', { name: 'Eliminar' });

// Encuentra el checkbox por su aria-label
screen.getByRole('checkbox', { name: 'Marcar "Comprar pan"' });

// name acepta expresiones regulares (útil para coincidencias parciales)
screen.getByRole('button', { name: /eliminar/i });
```

También existe `getByPlaceholderText` para inputs sin label visible:

```tsx
// Encuentra un input por su atributo placeholder
screen.getByPlaceholderText('Tu usuario');
```

> Si tienes que recurrir a `getByTestId`, pregúntate primero si falta una `label`, un `aria-label` o un texto accesible. Mejorar la accesibilidad suele eliminar la necesidad del `testId`.

---

## Paso 2 — `queryBy`: afirmar que algo NO existe

`LoginForm` **no** muestra ningún mensaje de error al renderizarse. ¿Cómo afirmamos que algo *no* está sin que la query reviente? Con `queryBy`, la única familia que devuelve `null` en vez de lanzar error. Añade este caso al `describe`:

```tsx
// src/components/LoginForm.test.tsx  (añade dentro del describe)
  it('NO debería mostrar un mensaje de error inicialmente (queryBy)', () => {
    render(<LoginForm onLogin={() => {}} />);
    // queryBy devuelve null → seguro para aserciones negativas.
    expect(screen.queryByText('Credenciales inválidas')).not.toBeInTheDocument();
  });
```

Si aquí hubieras usado `getByText('Credenciales inválidas')`, el test fallaría
con un error de "elemento no encontrado" **antes** de llegar al `.not`. Esa es la
razón de existir de `queryBy`.

### 🔧 Tu turno #2

Predice y luego comprueba: cambia el `queryByText` anterior por
`getByText('Credenciales inválidas')` (sin el `.not`). ¿Cuál es el mensaje de
error exacto y en qué línea apunta?

<details><summary>✅ Solución / qué observar</summary>

`getByText` lanza `Unable to find an element with the text: Credenciales
inválidas` y apunta a la línea del `getByText`. El test ni siquiera evalúa la
aserción: muere en la query. Conclusión: **para presencia negativa usa siempre
`queryBy`**. Devuelve `null`, y `null` encaja perfecto con
`.not.toBeInTheDocument()`.

</details>

---

## Variantes `*AllBy`: cuando esperas varios

Las queries `*By` exigen **exactamente un** resultado. Si esperas **varios** elementos, usa la versión `*AllBy`, que devuelve un **array**:

| Variante | Devuelve | Si no hay coincidencias |
|----------|----------|--------------------------|
| `getAllBy*` | Array (≥1) | Lanza error |
| `queryAllBy*` | Array (puede estar vacío) | `[]` |
| `findAllBy*` | Promise de array (≥1) | Rechaza |

```tsx
// En una lista con 3 tareas, esperamos 3 elementos de lista
const items = screen.getAllByRole('listitem');
expect(items).toHaveLength(3);
```

> Si un `getByRole('listitem')` falla diciendo *"found multiple elements"*, casi siempre la solución es cambiar a `getAllByRole` o acotar la búsqueda con un `name`/contexto más preciso.

---

## Paso 3 — `getAllBy` y el estado vacío sobre `TodoList`

`TodoList` renderiza varios `TodoItem` o un estado vacío. Su firma:

```tsx
// src/components/TodoList.tsx
import type { Todo } from '../types';
import { TodoItem } from './TodoItem';

interface TodoListProps {
  todos: Todo[];
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}

export function TodoList({ todos, onToggle, onDelete }: TodoListProps) {
  // Estado vacío: mensaje claro cuando no hay tareas.
  if (todos.length === 0) {
    return <p>No hay tareas pendientes</p>;
  }

  return (
    <ul aria-label="Lista de tareas">
      {todos.map((todo) => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={onToggle}
          onDelete={onDelete}
        />
      ))}
    </ul>
  );
}
```

Crea su test y añade **primero** el caso de varios elementos con `getAllByRole`:

```tsx
// src/components/TodoList.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { TodoList } from './TodoList';
import type { Todo } from '../types';

const tareas: Todo[] = [
  { id: '1', text: 'Comprar pan', completed: false },
  { id: '2', text: 'Lavar el coche', completed: true },
  { id: '3', text: 'Estudiar Vitest', completed: false },
];

describe('TodoList · queries', () => {
  it('debería renderizar un listitem por cada tarea (getAllByRole)', () => {
    // Arrange
    render(<TodoList todos={tareas} onToggle={() => {}} onDelete={() => {}} />);
    // Act + Assert: tres tareas → tres elementos de lista
    const items = screen.getAllByRole('listitem');
    expect(items).toHaveLength(3);
  });

  it('debería encontrar la lista por su aria-label', () => {
    render(<TodoList todos={tareas} onToggle={() => {}} onDelete={() => {}} />);
    expect(
      screen.getByRole('list', { name: 'Lista de tareas' }),
    ).toBeInTheDocument();
  });
});
```

Ahora el estado vacío. Aquí combinamos `getByText` (el mensaje **debe** estar) con `queryAllByRole` (que devuelve `[]` sin lanzar error):

```tsx
// src/components/TodoList.test.tsx  (añade dentro del describe)
  it('debería mostrar el estado vacío cuando no hay tareas', () => {
    // Arrange: lista vacía
    render(<TodoList todos={[]} onToggle={() => {}} onDelete={() => {}} />);
    // Assert: aparece el mensaje y NO hay listitems
    expect(screen.getByText('No hay tareas pendientes')).toBeInTheDocument();
    expect(screen.queryAllByRole('listitem')).toHaveLength(0);
  });
```

Observa el uso combinado: `getAllByRole` cuando esperamos varios, y `queryAllByRole` (que devuelve `[]`) para afirmar que **no hay** ninguno sin que la query lance error.

---

## Paso 4 — 🔧 Tu turno #3: elige la query correcta

Sobre `TodoList` con las tres `tareas`, escribe tú mismo un `it` que verifique
que el texto **"Lavar el coche"** está en el documento. La pregunta clave:
¿`getByText`, `getAllByText` o `queryByText`? Justifícalo y escríbelo.

<details><summary>✅ Solución</summary>

Hay **exactamente un** elemento con ese texto, y **debe** existir, así que la
familia correcta es `getByText` (singular, lanza error si falta):

```tsx
  it('debería mostrar el texto de una tarea concreta', () => {
    render(<TodoList todos={tareas} onToggle={() => {}} onDelete={() => {}} />);
    expect(screen.getByText('Lavar el coche')).toBeInTheDocument();
  });
```

`getAllByText` sería innecesario (no esperamos varios) y `queryByText` solo se
justifica si quisiéramos afirmar **ausencia**. Regla: *un elemento que debe
estar* → `getBy`.

</details>

---

## Prioridad recomendada de queries

Testing Library define un **orden de preferencia** pensado para que tus tests se parezcan a cómo un usuario (incluyendo quien usa lector de pantalla) interactúa con la app:

```
1. Accesibles para todos (lo mejor)
   ├─ getByRole          ← preferida casi siempre
   ├─ getByLabelText     ← campos de formulario
   ├─ getByPlaceholderText
   └─ getByText          ← contenido visible
2. Semánticas
   ├─ getByDisplayValue
   ├─ getByAltText
   └─ getByTitle
3. Último recurso
   └─ getByTestId        ← solo si nada de lo anterior aplica
```

La idea: **cuanto más arriba en la lista, más se acerca tu test a la experiencia real del usuario** y más accesibilidad fomenta. Reserva `getByTestId` para casos donde el elemento no tiene rol ni texto significativos.

| Query | Busca por | Ejemplo de elemento |
|-------|-----------|---------------------|
| `getByRole` | Rol ARIA (+ `name` opcional) | `<button>`, `<input>`, `<li>` |
| `getByLabelText` | Etiqueta `<label>` asociada | Campos de formulario |
| `getByPlaceholderText` | Atributo `placeholder` | Inputs sin label |
| `getByText` | Texto visible | Párrafos, spans, botones |
| `getByDisplayValue` | Valor actual de un input | Inputs rellenados |
| `getByAltText` | Atributo `alt` | Imágenes |
| `getByTitle` | Atributo `title` | Cualquiera con `title` |
| `getByTestId` | Atributo `data-testid` | Último recurso |

---

## Depurar queries: `screen.debug()` y `logRoles`

Cuando una query no encuentra lo que esperas, dos herramientas son oro puro:

- **`screen.debug()`** imprime el HTML actual (todo el documento o un nodo concreto).
- **`logRoles(container)`** lista todos los roles accesibles disponibles, ideal cuando no sabes qué `role`/`name` usar.

```tsx
// Fragmento ilustrativo dentro de un test
import { render, screen, logRoles } from '@testing-library/react';
import { LoginForm } from './LoginForm';

it('inspecciona el DOM y los roles', () => {
  const { container } = render(<LoginForm onLogin={() => {}} />);

  // Imprime TODO el HTML renderizado en consola
  screen.debug();

  // Imprime solo un nodo (más enfocado)
  screen.debug(screen.getByRole('button', { name: 'Entrar' }));

  // Lista todos los roles y sus nombres accesibles
  logRoles(container);
});
```

Salida típica de `logRoles`:

```
textbox:
  Name "Usuario": <input id="username" />
  Name "Tu contraseña": <input id="password" />
button:
  Name "Entrar": <button type="submit" />
form:
  Name "Formulario de acceso": <form />
```

> `logRoles` es la forma más rápida de saber **qué argumentos pasar a `getByRole`**. Úsalo cuando dudes entre `name` exacto o regex.

---

### Prueba esto

- Cambia `getByText` por `queryByText` buscando un texto inexistente y comprueba que devuelve `null` en lugar de lanzar error.
- Provoca un fallo con `getByRole('button', { name: 'Salir' })` y lee cómo el error sugiere los roles disponibles.
- Usa `logRoles(container)` sobre `TodoList` y localiza el rol `list` y los `listitem`.
- Sustituye un `getByLabelText` por `getByTestId` y razona por qué la primera versión es preferible para accesibilidad.
- Cambia el `name: 'Entrar'` por la regex `/entrar/i` y verifica que sigue encontrando el botón sin importar mayúsculas.
- Elimina el `aria-label` del `<form>` y observa que `getByRole('form', ...)` deja de encontrarlo.

---

## Ejercicios propuestos

### Básico

**B1 — Placeholder.** Escribe un `it` sobre `LoginForm` que encuentre el input de contraseña por su `placeholder` (`'Tu contraseña'`) con `getByPlaceholderText` y afirme que está en el documento.

**B2 — Conteo dinámico.** Crea un array de **cinco** tareas y un `it` sobre `TodoList` que verifique con `getAllByRole('listitem')` que hay exactamente cinco elementos. Cambia luego a cuatro y observa el fallo.

### Intermedio

**I1 — Regex en `name`.** Reescribe el test del botón "Entrar" usando `getByRole('button', { name: /entrar/i })`. Después cambia el texto del botón a "ENTRAR AHORA" en el componente y comprueba que la regex sigue encontrándolo mientras que `name: 'Entrar'` exacto fallaría.

**I2 — Ausencia condicional.** En `TodoList`, escribe un `it` que con la lista vacía afirme **a la vez** dos cosas: que aparece el mensaje de estado vacío (`getByText`) y que `queryByRole('list')` devuelve `null` (la `<ul>` no se renderiza).

### Avanzado

**A1 — `findBy` anticipado.** Investiga `findByText` leyendo su firma. Sin implementar todavía la carga asíncrona (módulo 9), explica por escrito por qué `await screen.findByText('Cargado')` necesita `await` y `getByText` no, y qué `timeout` aplica por defecto.

**A2 — Auditoría de queries.** Toma la suite de `LoginForm` y reescribe cada query para que use la **prioridad recomendada** más alta posible. Documenta en comentarios por qué cada elección (rol, label, texto) supera a un hipotético `getByTestId`.

---

## Resumen del módulo 3

- Una *query* combina **variante** (`get`/`query`/`find`), **cardinalidad** (`By`/`AllBy`) y **criterio** (`Role`, `Text`, `LabelText`...).
- `getBy` lanza error si no encuentra (afirmar existencia); `queryBy` devuelve `null` (afirmar inexistencia); `findBy` es asíncrona y espera con *timeout*.
- Las variantes `*AllBy` devuelven arrays cuando esperas varios elementos; `queryAllBy` devuelve `[]` sin lanzar error.
- `getByRole` con `name` es la query preferida porque prueba accesibilidad a la vez que localiza.
- Construimos las suites **por pasos**: del `getBy` que existe, al `queryBy` para la ausencia, al `getAllBy` para varios y el estado vacío, hasta elegir tú mismo la query correcta.
- La **prioridad de queries** va de las accesibles (`getByRole`, `getByLabelText`) a las semánticas y, en último lugar, `getByTestId`.
- `screen.debug()` imprime el HTML y `logRoles()` lista los roles disponibles para depurar queries.

> **Siguiente página →** Módulo 4: Matchers y jest-dom — los matchers nativos de Vitest y los de jest-dom (`toBeChecked`, `toBeDisabled`, `toHaveValue`...) aplicados a `TodoItem` y `AddTodoForm`.
