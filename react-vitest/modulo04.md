# Testing en React con Vitest — Página 4
## Módulo 4 · Matchers y jest-dom
### Matchers nativos de Vitest, matchers de jest-dom, aserciones negadas y `expect.assertions`

---

## Cómo trabajar este módulo

Este módulo es **incremental**: iremos sumando matchers de uno en uno sobre
`TodoItem` y `AddTodoForm`, prediciendo el resultado antes de guardar. Mantén el
modo watch en marcha para ver cada aserción pasar (o fallar) al instante:

```bash
npm run test:watch
```

La gracia de los matchers se entiende **rompiéndolos**: cambia un dato, observa el
mensaje de error y compáralo con el de un matcher peor elegido. Resuelve los retos
**🔧 Tu turno** **antes** de mirar la solución.

```
Paso 1  →  matchers nativos: toBe vs toEqual
Paso 2  →  jest-dom sobre TodoItem (toBeChecked)
Paso 3  →  jest-dom sobre AddTodoForm (toBeDisabled, toHaveValue)
Paso 4  →  tú escribes una aserción negada
```

---

## ¿Qué es un *matcher*?

Un **matcher** es la función que se encadena a `expect(...)` para definir *qué* condición debe cumplir el valor:

```
expect(  valorRecibido  ).  matcher(  valorEsperado  )
   │            │                │            │
 wrapper   lo que pruebas    la condición   lo que esperas
```

Hay dos grandes grupos de matchers en nuestro stack:

1. **Nativos de Vitest** — comparaciones genéricas sobre cualquier valor (números, strings, arrays, objetos).
2. **De jest-dom** — comprobaciones específicas sobre **nodos del DOM** (visibilidad, atributos, estado de un checkbox...). Los registramos en el módulo 1 con `import '@testing-library/jest-dom/vitest'`.

---

## Paso 1 — Matchers nativos: `toBe` vs `toEqual`

Estos sirven para cualquier valor de JavaScript, no solo para el DOM:

| Matcher | Comprueba | Ejemplo |
|---------|-----------|---------|
| `toBe(v)` | Igualdad **estricta** (`===`) | `expect(2 + 2).toBe(4)` |
| `toEqual(v)` | Igualdad **profunda** (estructura) | `expect(obj).toEqual({ id: '1' })` |
| `toStrictEqual(v)` | Igualdad profunda + tipos | `expect(a).toStrictEqual(b)` |
| `toContain(v)` | Que un array/string incluya algo | `expect([1, 2]).toContain(2)` |
| `toHaveLength(n)` | La longitud de array/string | `expect(todos).toHaveLength(3)` |
| `toBeTruthy()` / `toBeFalsy()` | Veracidad | `expect(user).toBeTruthy()` |
| `toBeNull()` / `toBeUndefined()` | Valores vacíos | `expect(found).toBeNull()` |
| `toBeGreaterThan(n)` | Comparación numérica | `expect(count).toBeGreaterThan(0)` |
| `toThrow()` | Que una función lance error | `expect(fn).toThrow()` |

La distinción más importante es **`toBe` vs `toEqual`**. Escribe este pequeño test de exploración (no prueba un componente, solo el concepto):

```ts
// src/matchers.test.ts
import { describe, it, expect } from 'vitest';

describe('toBe vs toEqual', () => {
  it('toBe usa === : ideal para primitivos', () => {
    expect('hola').toBe('hola');   // ✓
    expect(42).toBe(42);            // ✓
  });

  it('toEqual compara la estructura recursivamente', () => {
    // Para objetos/arrays, === compararía REFERENCIAS, no contenido.
    expect({ id: '1' }).toEqual({ id: '1' }); // ✓ mismo contenido
  });
});
```

> Regla práctica: **`toBe` para primitivos**, **`toEqual` para objetos y arrays**. Confundirlos es la causa #1 de tests que fallan "sin motivo aparente".

### 🔧 Tu turno #1

Predice y comprueba: añade un `it` que haga `expect({ id: '1' }).toBe({ id: '1' })`. ¿Pasa o falla? ¿Por qué?

<details><summary>✅ Solución / qué observar</summary>

**Falla.** `toBe` usa `===`, y dos objetos literales distintos son **referencias
diferentes** aunque tengan el mismo contenido. El mensaje será algo como
`expected { id: '1' } to be { id: '1' } // Object.is equality`. Cámbialo a
`toEqual` y pasa: `toEqual` compara la estructura, no la identidad. Por eso para
objetos/arrays casi siempre quieres `toEqual`.

</details>

---

## Matchers de jest-dom

Estos solo tienen sentido sobre **nodos del DOM** (lo que devuelven las queries de RTL). Hacen los tests legibles y expresivos:

| Matcher | Comprueba que el elemento... | Aplicable a |
|---------|------------------------------|-------------|
| `toBeInTheDocument()` | Está presente en el documento | Cualquiera |
| `toBeVisible()` | Es visible (no `display:none`, etc.) | Cualquiera |
| `toHaveTextContent(t)` | Contiene cierto texto | Cualquiera |
| `toBeDisabled()` | Está deshabilitado | Inputs, botones |
| `toBeEnabled()` | Está habilitado | Inputs, botones |
| `toBeChecked()` | Está marcado | Checkbox, radio |
| `toHaveValue(v)` | Tiene cierto valor | Inputs, selects |
| `toHaveAttribute(n, v)` | Tiene un atributo (y valor) | Cualquiera |
| `toHaveClass(c)` | Tiene una clase CSS | Cualquiera |
| `toHaveFocus()` | Tiene el foco | Cualquiera |

> Sin jest-dom tendrías que escribir `expect(el.checked).toBe(true)`. Con jest-dom escribes `expect(el).toBeChecked()`: más claro, y con mensajes de error mucho mejores.

---

## Paso 2 — jest-dom sobre `TodoItem`: el checkbox según `completed`

Recordemos que `TodoItem` muestra un checkbox cuyo estado refleja `todo.completed`. Empezamos por el caso "completada" con `toBeChecked()`. Crea el archivo con **un solo `it`**:

```tsx
// src/components/TodoItem.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { TodoItem } from './TodoItem';
import type { Todo } from '../types';

function crearTodo(overrides: Partial<Todo> = {}): Todo {
  return { id: '1', text: 'Comprar pan', completed: false, ...overrides };
}

describe('TodoItem · matchers', () => {
  it('debería marcar el checkbox cuando la tarea está completada', () => {
    // Arrange: tarea completada
    const todo = crearTodo({ completed: true });
    // Act
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    // Assert: el checkbox debe estar marcado
    expect(screen.getByRole('checkbox')).toBeChecked();
  });
});
```

Guarda: **1 passed**. El helper `crearTodo` (del módulo 2) nos deja sobreescribir
solo el campo que importa, aquí `completed: true`.

### 🔧 Tu turno #2

Sin mirar abajo: añade un segundo `it` para el caso contrario —tarea **activa**,
checkbox **sin marcar**— usando la **negación** del matcher. Y un tercero que
verifique el texto exacto del `<li>` con `toHaveTextContent`.

<details><summary>✅ Solución</summary>

```tsx
  it('NO debería marcar el checkbox cuando la tarea está activa', () => {
    // Arrange: tarea pendiente
    const todo = crearTodo({ completed: false });
    // Act
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    // Assert: matcher negado con .not
    expect(screen.getByRole('checkbox')).not.toBeChecked();
  });

  it('debería mostrar el texto exacto de la tarea', () => {
    const todo = crearTodo({ text: 'Estudiar matchers' });
    render(<TodoItem todo={todo} onToggle={() => {}} onDelete={() => {}} />);
    // toHaveTextContent comprueba el texto contenido en el <li>
    expect(screen.getByRole('listitem')).toHaveTextContent('Estudiar matchers');
  });
```

La suite llega a **3 passed**. Fíjate en `.not.toBeChecked()`: cualquier matcher
se invierte anteponiendo `.not`.

</details>

---

## Paso 3 — jest-dom sobre `AddTodoForm`: `toBeDisabled` y `toHaveValue`

`AddTodoForm` es un formulario con un input y un botón que solo se habilita cuando hay texto. Su firma:

```tsx
// src/components/AddTodoForm.tsx
import { useState } from 'react';

interface AddTodoFormProps {
  onAdd: (text: string) => void;
}

export function AddTodoForm({ onAdd }: AddTodoFormProps) {
  const [text, setText] = useState('');

  // El botón se deshabilita si el input está vacío (sin contar espacios).
  const disabled = text.trim() === '';

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (disabled) return;
    onAdd(text.trim());
    setText('');
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="new-todo">Nueva tarea</label>
      <input
        id="new-todo"
        value={text}
        placeholder="¿Qué hay que hacer?"
        onChange={(e) => setText(e.target.value)}
      />
      <button type="submit" disabled={disabled}>
        Añadir
      </button>
    </form>
  );
}
```

Probamos el **estado inicial** (sin interacción todavía). Crea el test añadiendo un caso a la vez. Primero, el botón deshabilitado:

```tsx
// src/components/AddTodoForm.test.tsx
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { AddTodoForm } from './AddTodoForm';

describe('AddTodoForm · matchers', () => {
  it('debería deshabilitar el botón cuando el input está vacío', () => {
    // Arrange + Act
    render(<AddTodoForm onAdd={() => {}} />);
    // Assert: con input vacío, el botón está deshabilitado
    expect(screen.getByRole('button', { name: 'Añadir' })).toBeDisabled();
  });
});
```

Ahora sumamos el valor inicial del input, su atributo `placeholder` y su visibilidad:

```tsx
// src/components/AddTodoForm.test.tsx  (añade dentro del describe)
  it('debería empezar con el input vacío', () => {
    render(<AddTodoForm onAdd={() => {}} />);
    // toHaveValue('') confirma que el campo arranca vacío
    expect(screen.getByLabelText('Nueva tarea')).toHaveValue('');
  });

  it('debería exponer el placeholder esperado (toHaveAttribute)', () => {
    render(<AddTodoForm onAdd={() => {}} />);
    // Comprobamos un atributo concreto con su valor
    expect(screen.getByLabelText('Nueva tarea')).toHaveAttribute(
      'placeholder',
      '¿Qué hay que hacer?',
    );
  });

  it('el input debería ser visible', () => {
    render(<AddTodoForm onAdd={() => {}} />);
    expect(screen.getByLabelText('Nueva tarea')).toBeVisible();
  });
```

> En el módulo 5 escribiremos texto en el input con `user-event` y comprobaremos cómo el botón pasa de `toBeDisabled()` a `toBeEnabled()`. Aquí solo verificamos el estado **inicial**, sin interacción.

---

## Matchers negados con `.not`

Cualquier matcher puede invertirse anteponiendo **`.not`**. Esto cubre la otra mitad de los casos: comprobar que algo **no** ocurre.

```tsx
// El elemento NO está presente
expect(screen.queryByText('Error')).not.toBeInTheDocument();

// El checkbox NO está marcado
expect(screen.getByRole('checkbox')).not.toBeChecked();

// El botón NO está deshabilitado (equivale a toBeEnabled)
expect(screen.getByRole('button')).not.toBeDisabled();

// El array NO contiene el valor
expect([1, 2, 3]).not.toContain(4);
```

| Aserción positiva | Equivalente negado |
|-------------------|---------------------|
| `toBeEnabled()` | `.not.toBeDisabled()` |
| `toBeChecked()` (falso) | `.not.toBeChecked()` |
| presencia | `.not.toBeInTheDocument()` |

> Para aserciones negativas de **presencia**, recuerda usar `queryBy` (no `getBy`): `getBy` lanzaría un error antes de llegar al `.not`.

---

## Paso 4 — 🔧 Tu turno #3: una aserción negada

Sobre `AddTodoForm` recién renderizado, escribe tú mismo un `it` que afirme,
**con un matcher negado**, que el botón "Añadir" **no** está habilitado. Hay al
menos dos formas de expresarlo: encuéntralas.

<details><summary>✅ Solución</summary>

Dos equivalencias válidas:

```tsx
  it('el botón NO debería estar habilitado al inicio (negación)', () => {
    render(<AddTodoForm onAdd={() => {}} />);
    const boton = screen.getByRole('button', { name: 'Añadir' });
    // Forma 1: negar toBeEnabled
    expect(boton).not.toBeEnabled();
    // Forma 2 (equivalente): afirmar toBeDisabled
    expect(boton).toBeDisabled();
  });
```

`not.toBeEnabled()` y `toBeDisabled()` describen el mismo estado. Elige la que se
lea más clara según lo que quieras *comunicar* en el nombre del test.

</details>

---

## `expect.assertions`: garantizar que se ejecutaron las aserciones

En tests asíncronos o con *callbacks*, existe el riesgo de que un `expect` **nunca se ejecute** (por una promesa que no se resuelve, una rama que no se entra...) y el test pase "en falso". Para protegerte, declara cuántas aserciones esperas:

```tsx
// Fragmento ilustrativo (lo explotaremos con asíncrono en el módulo 9)
import { it, expect } from 'vitest';

it('debería ejecutar exactamente una aserción', () => {
  // Declaramos que este test DEBE correr 1 aserción.
  expect.assertions(1);

  const todo = { id: '1', text: 'Comprar pan', completed: false };
  // Si esta línea no se ejecutara, el test fallaría por nº de aserciones.
  expect(todo.text).toBe('Comprar pan');
});
```

También existe `expect.hasAssertions()`, que solo exige **al menos una** aserción sin fijar el número exacto:

| Forma | Garantiza |
|-------|-----------|
| `expect.assertions(n)` | Exactamente `n` aserciones ejecutadas |
| `expect.hasAssertions()` | Al menos una aserción ejecutada |

> Estas comprobaciones son especialmente valiosas en bloques `try/catch` y en código asíncrono, donde una aserción dentro de un `.catch` podría no llegar a correr nunca.

---

## Errores legibles: por qué importan los matchers correctos

Elegir el matcher adecuado no solo es elegante: produce **mensajes de error mucho mejores**. Compara:

```
// Con un booleano genérico:
expect(el.checked).toBe(true)
  → AssertionError: expected false to be true   (¿qué elemento? ¿por qué?)

// Con jest-dom:
expect(el).toBeChecked()
  → expected element to be checked:
    <input type="checkbox" />   (te muestra el nodo exacto)
```

El segundo te dice *qué nodo* falló y *en qué estado* estaba. Por eso preferimos siempre los matchers de jest-dom sobre las comprobaciones manuales de propiedades.

---

### Prueba esto

- Cambia `completed: true` a `false` en el test del checkbox y observa cómo el mensaje de `toBeChecked()` describe el nodo.
- Sustituye un `toEqual` por `toBe` al comparar dos objetos iguales y comprueba que falla por referencia.
- Escribe (temporalmente) texto en el input por defecto y verifica que `toBeDisabled()` empieza a fallar.
- Reemplaza `expect(el).toBeChecked()` por `expect(el.checked).toBe(true)` y compara la calidad de los mensajes de error.
- Añade `expect.assertions(2)` a un test que solo tiene 1 aserción y mira el fallo por número de aserciones.
- Usa `toHaveAttribute('placeholder')` sin el segundo argumento para comprobar solo la *existencia* del atributo, sin su valor.

---

## Ejercicios propuestos

### Básico

**B1 — Conteo con `toHaveLength`.** Crea un array de tareas y un `it` que use el matcher nativo `toHaveLength` para verificar su tamaño. No renderices nada: practica matchers nativos sobre datos.

**B2 — Atributo `type`.** Sobre `AddTodoForm`, escribe un `it` que verifique con `toHaveAttribute('type', 'submit')` que el botón "Añadir" es de tipo submit.

### Intermedio

**I1 — `toBeEnabled` vs `toBeDisabled`.** Escribe dos tests gemelos sobre el botón de `AddTodoForm`: uno con `not.toBeEnabled()` y otro con `toBeDisabled()`. Comenta en el código por qué describen el mismo estado y cuándo elegir cada redacción.

**I2 — `toContain` sobre textos.** Renderiza `TodoItem` con `text: 'Comprar pan integral'` y, obteniendo el `textContent` del `<li>`, usa el matcher nativo `toContain` para verificar que incluye la subcadena `'pan'`. Compara con `toHaveTextContent('pan')` de jest-dom.

### Avanzado

**A1 — `expect.assertions` que atrapa un bug.** Escribe un test con un `if (false) { expect(...).toBe(...) }` (rama que nunca se entra) y **sin** `expect.assertions`. Comprueba que pasa "en falso". Añade `expect.assertions(1)` al inicio y verifica que ahora el test falla por no ejecutar ninguna aserción. Explica el valor de esta guarda.

**A2 — Matcher peor vs mejor.** Toma el test del checkbox y escríbelo de dos maneras: `expect(checkbox.checked).toBe(true)` y `expect(checkbox).toBeChecked()`. Rompe ambos (pasa `completed: false`) y pega en un comentario los dos mensajes de error. Argumenta por qué el de jest-dom es superior para depurar.

---

## Resumen del módulo 4

- Un **matcher** define la condición que debe cumplir el valor pasado a `expect(...)`.
- Los matchers **nativos de Vitest** (`toBe`, `toEqual`, `toContain`, `toHaveLength`...) sirven para cualquier valor; usa `toBe` para primitivos y `toEqual` para objetos/arrays.
- Los matchers de **jest-dom** (`toBeInTheDocument`, `toBeChecked`, `toBeDisabled`, `toHaveValue`, `toHaveAttribute`, `toHaveClass`, `toBeVisible`...) operan sobre nodos del DOM y dan errores muy legibles.
- Construimos las suites **por pasos**: primero `toBe` vs `toEqual`, luego `TodoItem` con el checkbox según `completed`, después `AddTodoForm` con el botón `disabled` y el input vacío, y por fin una aserción negada escrita por ti.
- Cualquier matcher se invierte con **`.not`**; para presencia negativa usa siempre `queryBy`.
- `expect.assertions(n)` y `expect.hasAssertions()` garantizan que las aserciones esperadas realmente se ejecutaron, clave en código asíncrono.

> **Siguiente página →** Módulo 5: Interacción del usuario — simular clics y escritura con `@testing-library/user-event`, verificar que `onAdd`/`onToggle`/`onDelete` se llaman y ver el botón pasar de deshabilitado a habilitado.
