# Testing E2E en React con Cypress — Página 3
## Módulo 3 · Selectores
### Por qué `data-cy` es el estándar y cómo navegar el DOM con `.find()`, `.within()`, `.first()` y `.eq()`

---

## Cómo trabajar este módulo

Este módulo es **incremental**: empezaremos con un selector frágil que se rompe,
lo arreglaremos con `data-cy` y luego iremos sumando, **paso a paso**, las
herramientas para navegar el DOM (`.find`, `.within`, `.first`/`.eq`). Escribe
cada bloque, guarda y observa el cambio.

Antes de empezar, ten preparado:

- La app corriendo en `http://localhost:5173` (`npm run dev`).
- Cypress abierto (`npx cypress open`) con `selectors.cy.ts` seleccionado, para
  ver **en vivo** qué elemento resalta cada selector en la app de la derecha.

Resuelve los retos **🔧 Tu turno** **antes** de mirar la solución colapsable.

```
Paso 1  →  el selector frágil que se rompe
Paso 2  →  arreglarlo con data-cy
Paso 3  →  acotar el contexto con .within()
Paso 4  →  navegar con .find() y elegir por posición
Paso 5  →  tú combinas todo en un flujo real
```

---

## El problema: selectores frágiles

Un test E2E solo es útil si es **estable**: debe fallar cuando la app se rompe, no cuando alguien cambia un color o reordena una clase de CSS. El mayor enemigo de la estabilidad son los **malos selectores**.

Imagina este botón de login en React:

```tsx
// src/components/LoginForm.tsx
<button className="btn btn-primary btn-lg login__submit">
  Entrar
</button>
```

---

## Paso 1 — El selector frágil que se rompe

Vamos a sentir el problema en carne propia. Crea un spec y selecciona el botón **por su clase de estilo**:

```ts
// cypress/e2e/selectors.cy.ts
describe('Selectores estables', () => {
  it('FRÁGIL: selecciona por clase de estilo', () => {
    cy.visit('/login')
    // FRÁGIL: depende de clases de estilo que pueden cambiar
    cy.get('.btn-primary').click()
  })
})
```

Ahora **imagina** que el diseñador cambia `btn-primary` por `btn-cta`. El test se romperá... pero **la funcionalidad no cambió**: el botón sigue haciendo login. Un test que falla por motivos equivocados es ruido, y el ruido hace que el equipo deje de confiar en los tests.

| Selector | Problema |
|----------|----------|
| `.btn-primary` | Las clases de CSS existen para **estilizar**, no para identificar. Cambian a menudo. |
| `div > div > span` | El más frágil de todos: cualquier refactor de layout lo rompe. |
| `#root .list li:nth-child(2)` | Depende de la posición; si reordenas la lista, apunta a otra cosa. |
| `cy.contains('Entrar')` | Se rompe si traduces la UI o cambias el copy ("Entrar" → "Acceder"). |

> El texto **sí** sirve cuando ese texto es parte del contrato (un mensaje de error que el usuario debe ver). Pero para *identificar controles*, prefiere un atributo dedicado.

### 🔧 Tu turno #1

Mira estos cuatro selectores. **Ordénalos** de más frágil a más estable y explica con una frase por qué el primero es tan malo.

```
a) cy.get('#root div.container form .btn-primary')
b) cy.get('[data-cy=login-button]')
c) cy.get('[name=username]')
d) cy.contains('Entrar')
```

<details><summary>✅ Solución</summary>

De **más frágil** a **más estable**: `a → d → c → b`.

- **a)** es el peor: encadena `id`, clases de layout y clases de estilo. Cualquier refactor de la estructura o del CSS lo rompe, aunque el login siga funcionando.
- **d)** depende del texto visible: se rompe si traduces o cambias el copy.
- **c)** un atributo semántico (`name`) es razonable para formularios, pero no es exclusivo para tests.
- **b)** es el mejor: `data-cy` es un contrato dedicado al testing, inmune a cambios de estilo o de texto.

</details>

---

## Paso 2 — Arreglarlo con `data-cy`

La práctica recomendada por el equipo de Cypress es añadir un atributo dedicado al testing: `data-cy`. Es un contrato explícito entre el código y los tests.

```tsx
// src/components/LoginForm.tsx
export function LoginForm() {
  return (
    <form data-cy="login-form">
      <input data-cy="username" name="username" placeholder="Usuario" />
      <input data-cy="password" name="password" type="password" />
      <button data-cy="login-button" type="submit">Entrar</button>
    </form>
  )
}
```

Y reescribe el test del paso 1 para que use el atributo estable:

```ts
// cypress/e2e/selectors.cy.ts
it('ESTABLE: selecciona por data-cy', () => {
  cy.visit('/login')
  // ESTABLE: data-cy no cambia por motivos de estilo ni de copy
  cy.get('[data-cy=login-button]').click()
})
```

Ventajas:

- **Inmune al estilo**: cambiar clases o el texto no afecta al test.
- **Explícito**: cualquiera que lea el JSX sabe que ese elemento se testea.
- **Buscable**: un `grep` de `data-cy=` lista todo lo que está bajo prueba.

> Convención: usa siempre `data-cy` (no `id`, no `class`). Es legible y reservado exclusivamente para tests.

### Tabla de prioridad de selectores

El equipo de Cypress recomienda este orden, de **mejor** a **peor**:

| Prioridad | Selector | Ejemplo | Recomendación |
|-----------|----------|---------|---------------|
| 1 (mejor) | `data-cy` dedicado | `[data-cy=login-button]` | Úsalo siempre que puedas |
| 2 | `data-test` / `data-testid` | `[data-testid=submit]` | Aceptable si ya existe |
| 3 | Texto visible | `cy.contains('Entrar')` | Solo si el texto es contrato |
| 4 | Atributos semánticos | `[name=username]` | Útil para formularios |
| 5 (peor) | Clases / tags / posición | `.btn`, `div span`, `:nth-child` | Evítalo |

```
  MEJOR  ┌──────────────────────────────┐
    ▲    │ [data-cy=...]                  │  estable, explícito
    │    ├──────────────────────────────┤
    │    │ [data-testid=...]             │
    │    ├──────────────────────────────┤
    │    │ cy.contains('texto')          │  ok si es contrato visual
    │    ├──────────────────────────────┤
    │    │ [name=...] [type=...]         │
    │    ├──────────────────────────────┤
    ▼    │ .clase  tag  :nth-child       │  frágil, evitar
  PEOR   └──────────────────────────────┘
```

Para nuestra Todo App, estos son **todos** los selectores estables que usaremos en el curso:

| Pantalla | Elemento | Selector |
|----------|----------|----------|
| Login | Input usuario | `[data-cy=username]` |
| Login | Input contraseña | `[data-cy=password]` |
| Login | Botón Entrar | `[data-cy=login-button]` |
| Todos | Input nueva tarea | `[data-cy=new-todo-input]` |
| Todos | Botón Agregar | `[data-cy=add-button]` |
| Todos | Cada tarea | `[data-cy=todo-item]` |
| Todos | Checkbox de tarea | `[data-cy=todo-checkbox]` |
| Todos | Botón eliminar | `[data-cy=delete-button]` |
| Todos | Filtro Todas | `[data-cy=filter-all]` |
| Todos | Filtro Activas | `[data-cy=filter-active]` |
| Todos | Filtro Completadas | `[data-cy=filter-completed]` |

La sintaxis es un selector de atributo CSS estándar; las comillas internas son opcionales si el valor no tiene espacios:

```ts
// cypress/e2e/selectors.cy.ts
cy.get('[data-cy=login-button]')
cy.get('[data-cy="login-button"]') // equivalente, más explícito
```

---

## Paso 3 — `.within()`: acotar el contexto

`.within()` ejecuta un bloque de comandos **limitados** al elemento actual. Dentro del bloque, cualquier `cy.get()` solo busca dentro de ese contenedor. Añade un test que use `.within()` para rellenar el login sin ambigüedades:

```ts
// cypress/e2e/selectors.cy.ts
it('login acotado con .within()', () => {
  cy.visit('/login')
  // Todos los cy.get() de dentro se acotan al formulario de login
  cy.get('[data-cy=login-form]').within(() => {
    cy.get('[data-cy=username]').type('admin')
    cy.get('[data-cy=password]').type('1234')
    cy.get('[data-cy=login-button]').click()
  })
})
```

Aplicado a un todo concreto, evita ambigüedades cuando hay muchos en la lista:

```ts
// cypress/e2e/todos.cy.ts
cy.get('[data-cy=todo-item]').first().within(() => {
  // Aquí 'checkbox' y 'delete-button' son los DE ESTE todo
  cy.get('[data-cy=todo-checkbox]').check()
  cy.get('[data-cy=delete-button]').click()
})
```

```
Lista de todos
┌──────────────────────────────────────┐
│ todo-item ◄── .within() acota aquí     │
│   ├─ [todo-checkbox]                    │
│   ├─ "Comprar pan"                      │
│   └─ [delete-button]                    │
├──────────────────────────────────────┤
│ todo-item (ignorado dentro del within) │
└──────────────────────────────────────┘
```

---

## Paso 4 — `.find()` y elegir por posición

`.find()` busca descendientes **dentro** del elemento actual de la cadena. Es como un `querySelectorAll` acotado a un subárbol.

```ts
// cypress/e2e/todos.cy.ts
// Dentro del primer todo, encontrar su checkbox y su botón eliminar
cy.get('[data-cy=todo-item]')
  .first()
  .find('[data-cy=todo-checkbox]')
  .should('exist')
```

Diferencia clave con `cy.get()`:

| Comando | Busca en |
|---------|----------|
| `cy.get(sel)` | Todo el documento |
| `.find(sel)` | Solo dentro del elemento anterior de la cadena |

Cuando un selector devuelve **varios** elementos (como la lista de todos), eliges uno concreto por su posición:

```ts
// cypress/e2e/todos.cy.ts
cy.get('[data-cy=todo-item]').first() // el primero
cy.get('[data-cy=todo-item]').last()  // el último
cy.get('[data-cy=todo-item]').eq(2)   // el tercero (índice base 0)
```

| Comando | Devuelve |
|---------|----------|
| `.first()` | El primer elemento de la colección |
| `.last()` | El último elemento |
| `.eq(n)` | El elemento en el índice `n` (empieza en 0) |

También puedes combinar estructura y contenido con `cy.contains(selector, texto)`:

```ts
// cypress/e2e/todos.cy.ts
// Buscar un elemento [data-cy=todo-item] CUYO TEXTO sea "Comprar pan"
cy.contains('[data-cy=todo-item]', 'Comprar pan').should('be.visible')
```

> Úsalos con criterio: si necesitas un todo **específico**, suele ser mejor buscarlo por su texto con `cy.contains('[data-cy=todo-item]', '...')` que depender de su posición.

### 🔧 Tu turno #2

Escribe una sola cadena de comandos que: tome **todos** los `[data-cy=todo-item]`, afirme que hay **3**, tome el **primero**, encuentre **dentro** de él su `[data-cy=todo-checkbox]` y lo **marque** (`.check()`).

**Pista:** encadena `cy.get` → `.should('have.length', 3)` → `.first()` → `.find(...)` → `.check()`.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/todos.cy.ts
cy.get('[data-cy=todo-item]')   // obtén todos los items
  .should('have.length', 3)     // deben ser 3
  .first()                      // toma el primero
  .find('[data-cy=todo-checkbox]') // dentro, su checkbox
  .check()                      // márcalo
```

Cada eslabón opera sobre el resultado del anterior. Si un comando "rompe" la cadena (por ejemplo, `cy.visit()`), empieza una cadena nueva.

```
cy.get(items) ──► 3 elementos
      │
   .first() ──► 1 elemento (el primero)
      │
   .find(checkbox) ──► 1 elemento (descendiente)
      │
   .check() ──► acción sobre ese checkbox
```

</details>

---

## Paso 5 — Combinar todo en un flujo real

Junta lo aprendido en un único spec que ejercita login y lista de todos con selectores estables:

```ts
// cypress/e2e/selectors.cy.ts
describe('Selectores estables', () => {
  it('login usando within', () => {
    cy.visit('/login')
    cy.get('[data-cy=login-form]').within(() => {
      cy.get('[data-cy=username]').type('admin')
      cy.get('[data-cy=password]').type('1234')
      cy.get('[data-cy=login-button]').click()
    })
    // Tras el login, deberíamos estar en /todos
    cy.url().should('include', '/todos')
  })

  it('opera sobre un todo concreto', () => {
    cy.visit('/todos')
    // Buscar por texto el item, y dentro de él tocar su checkbox
    cy.contains('[data-cy=todo-item]', 'Comprar pan')
      .find('[data-cy=todo-checkbox]')
      .check()
  })
})
```

---

### Prueba esto

- Cambia un selector de `[data-cy=login-button]` a `.btn-primary` y observa cómo el test se vuelve frágil.
- Reescribe el login usando `.within()` sobre `[data-cy=login-form]`.
- Usa `.first()`, `.eq(1)` y `.last()` para apuntar a distintos `[data-cy=todo-item]`.
- Selecciona un todo por su texto con `cy.contains('[data-cy=todo-item]', 'Comprar pan')`.
- Dentro de un todo, usa `.find('[data-cy=delete-button]')` para llegar a su botón de borrar.
- Encadena `cy.get(...).should('have.length', N).first()` y verifica que la cadena fluye.

---

## Ejercicios propuestos

### Básico

**B1 — Selector explícito.** Reescribe `cy.get('[data-cy=username]')` con comillas internas (`[data-cy="username"]`) y comprueba que el comportamiento es idéntico.

**B2 — `.last()` en la lista.** Visita `/todos` y usa `.last()` sobre `[data-cy=todo-item]` para marcar el checkbox del **último** todo.

### Intermedio

**I1 — `.within()` por cada filtro.** Acota con `.within()` la zona de filtros y verifica que `[data-cy=filter-all]`, `[data-cy=filter-active]` y `[data-cy=filter-completed]` existen sin tener que repetir el contenedor.

**I2 — Buscar por texto vs posición.** Escribe dos tests que marquen el todo "Comprar pan": uno con `.eq(n)` (posición) y otro con `cy.contains('[data-cy=todo-item]', 'Comprar pan')`. Reordena la lista en la app y observa cuál sigue funcionando.

### Avanzado

**A1 — Cadena larga y legible.** Construye una sola cadena que tome la lista, afirme su longitud, entre con `.eq(1)`, use `.within()` para marcar el checkbox y luego eliminar ese todo con su `[data-cy=delete-button]`.

**A2 — Selector configurable.** Investiga la opción de configuración para personalizar el atributo de selección (de `data-cy` a `data-test`). Documenta qué línea cambiarías en `cypress.config.ts` o cómo lo harías con un comando personalizado, sin romper los selectores actuales.

---

## Resumen del módulo 3

- Los selectores por **clase CSS, tag o posición** son frágiles: cambian por motivos de estilo.
- Lo vimos **por pasos**: primero un selector frágil que se rompe, luego su arreglo con `data-cy`, después `.within()`, `.find()` y la selección por posición, y al final un flujo combinado.
- El atributo **`data-cy`** es el estándar recomendado: estable, explícito y reservado para tests.
- La **prioridad** de selectores va de `data-cy` (mejor) a clases/posición (peor).
- `cy.get('[data-cy=...]')` selecciona por atributo; `cy.contains(sel, texto)` combina estructura y contenido.
- `.find()` busca descendientes; `.within()` acota todos los `cy.get()` a un contenedor.
- `.first()`, `.last()` y `.eq(n)` eligen por posición dentro de una colección.
- El **encadenamiento** opera sobre el resultado del comando anterior, formando frases legibles.

> **Siguiente página →** Módulo 4: Interacción con la UI — `.click()`, `.type()`, `.clear()`, `.check()`, `.select()` y `.submit()` para rellenar el login y manejar la lista de todos.
