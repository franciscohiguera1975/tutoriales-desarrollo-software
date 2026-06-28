# Testing en React con Vitest — Página 10

## Módulo 10 · Context y Router

### Testear componentes que consumen Context y navegan con react-router

---

## Cómo trabajar este módulo

Este módulo es **incremental**: partiremos del problema (un componente que falla
sin su provider) y subiremos peldaño a peldaño hasta tener un `render` helper que
envuelve **todos** los providers de la app y nos deja testear sesión, rutas y
redirecciones. No copies el helper final de golpe — añade **un paso cada vez**,
guarda y deja corriendo `npm run test:watch`. Entre paso y paso encontrarás retos
**🔧 Tu turno** que debes resolver **antes** de mirar la solución.

```
Paso 1  →  el problema: render sin provider falla
Paso 2  →  envolver a mano con AuthProvider
Paso 3  →  la opción wrapper de render
Paso 4  →  un render helper con TODOS los providers
Paso 5  →  probar "con usuario" disparando login (Tu turno)
Paso 6  →  MemoryRouter e initialEntries
Paso 7  →  navegación con user-event
Paso 8  →  redirección de ruta protegida (Tu turno)
```

---

Muchos componentes no funcionan aislados: dependen de un **Context** (como la
sesión del usuario) o de un **router** (para navegar entre rutas). Si los
renderizas tal cual en un test, fallarán con errores como
*"useAuth debe usarse dentro de AuthProvider"* o
*"useNavigate may be used only in the context of a Router"*.

En este módulo aprenderás a **envolver** tus componentes con los providers que
necesitan, a crear un *render helper* reutilizable y a testear navegación,
rutas y redirecciones.

---

## Recordatorio: el AuthContext de la app

Nuestra **Todo App** usa un contexto de autenticación con esta forma:

```tsx
// src/context/AuthContext.tsx
import { createContext, useContext, useState } from 'react';
import type { ReactNode } from 'react';
import type { User } from '../types';

interface AuthContextValue {
  user: User | null;
  login: (name: string) => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  // Simulamos un login: en una app real iría una petición al backend.
  const login = (name: string) => setUser({ id: 'u1', name });
  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// Hook de consumo: lanza si se usa fuera del provider.
export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) {
    throw new Error('useAuth debe usarse dentro de AuthProvider');
  }
  return ctx;
}
```

Un componente que muestra la sesión:

```tsx
// src/components/UserBadge.tsx
import { useAuth } from '../context/AuthContext';

// Muestra el nombre del usuario o invita a iniciar sesión.
export function UserBadge() {
  const { user, logout } = useAuth();

  if (!user) return <p>No has iniciado sesión</p>;

  return (
    <div>
      <span>Hola, {user.name}</span>
      <button onClick={logout}>Cerrar sesión</button>
    </div>
  );
}
```

---

## Paso 1 — El problema: renderizar sin provider falla

Antes de la solución, **provoca el error** para entenderlo. Escribe este test (que
fallará) y mira el mensaje:

```tsx
// ❌ Esto lanza: "useAuth debe usarse dentro de AuthProvider"
import { render } from '@testing-library/react';
import { UserBadge } from './UserBadge';

it('falla sin provider (a propósito)', () => {
  render(<UserBadge />);
});
```

El `useAuth` no encuentra el contexto (`useContext` devuelve `null`) y lanza
nuestro propio `Error`. Conclusión: el componente necesita estar **dentro** del
`AuthProvider`.

---

## Paso 2 — Envolver a mano con `AuthProvider`

La solución más directa es envolver el componente manualmente en el JSX del
render:

```tsx
// src/components/UserBadge.test.tsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { AuthProvider } from '../context/AuthContext';
import { UserBadge } from './UserBadge';

describe('UserBadge', () => {
  it('muestra el mensaje de invitado sin usuario', () => {
    render(
      <AuthProvider>
        <UserBadge />
      </AuthProvider>,
    );

    expect(screen.getByText('No has iniciado sesión')).toBeInTheDocument();
  });
});
```

Guarda: **1 passed**. Funciona, pero repetir ese envoltorio en cada test es
tedioso y propenso a errores. Vamos a mejorarlo.

---

## Paso 3 — La opción `wrapper` de `render`

`render` acepta una opción `wrapper`: un componente que envuelve a lo que
renderizas. Es ideal cuando varios tests comparten los mismos providers.

```tsx
// src/components/UserBadge.test.tsx (continúa)
import type { ReactNode } from 'react';

// Wrapper que provee el contexto a cualquier hijo.
function ConAuth({ children }: { children: ReactNode }) {
  return <AuthProvider>{children}</AuthProvider>;
}

it('usa la opción wrapper para no repetir el provider', () => {
  render(<UserBadge />, { wrapper: ConAuth });

  expect(screen.getByText('No has iniciado sesión')).toBeInTheDocument();
});
```

Ahora el test ya no anida el provider en su JSX. Compara las tres formas que
llevamos hasta aquí:

| Forma | Ventaja | Inconveniente |
| --- | --- | --- |
| Envolver a mano (JSX) | Explícito, sin abstracción | Repetitivo en cada test |
| Opción `wrapper` | Reutiliza un componente | Definido por archivo de test |
| `render` helper propio | Centralizado en todo el proyecto | Requiere crearlo una vez |

La última fila es nuestro destino: un helper para **todo el proyecto**.

---

## Paso 4 — Un `render` helper con TODOS los providers (recomendado)

La práctica recomendada por Testing Library es crear un `render`
**personalizado** que ya incluya todos los providers de la app (auth, router,
tema, i18n…). Lo colocamos en `src/test/utils.tsx` y re-exportamos todo lo de
Testing Library.

```tsx
// src/test/utils.tsx
import type { ReactElement, ReactNode } from 'react';
import { render } from '@testing-library/react';
import type { RenderOptions } from '@testing-library/react';
import { MemoryRouter } from 'react-router';
import { AuthProvider } from '../context/AuthContext';

// Opciones extra que admite nuestro render: rutas iniciales del router.
interface OpcionesCustom extends Omit<RenderOptions, 'wrapper'> {
  rutasIniciales?: string[];
}

// Envoltorio con TODOS los providers de la app.
function TodosLosProviders({
  children,
  rutasIniciales = ['/'],
}: {
  children: ReactNode;
  rutasIniciales?: string[];
}) {
  return (
    <MemoryRouter initialEntries={rutasIniciales}>
      <AuthProvider>{children}</AuthProvider>
    </MemoryRouter>
  );
}

// render personalizado: úsalo en lugar del de Testing Library.
function renderConProviders(
  ui: ReactElement,
  { rutasIniciales, ...options }: OpcionesCustom = {},
) {
  return render(ui, {
    wrapper: ({ children }) => (
      <TodosLosProviders rutasIniciales={rutasIniciales}>
        {children}
      </TodosLosProviders>
    ),
    ...options,
  });
}

// Re-exportamos todo y sobrescribimos render con el nuestro.
export * from '@testing-library/react';
export { renderConProviders as render };
```

Ahora los tests importan `render` desde `src/test/utils`, no desde Testing
Library, y obtienen los providers gratis:

```tsx
// src/components/UserBadge.test.tsx (con el helper)
import { render, screen } from '../test/utils';
import { UserBadge } from './UserBadge';

it('renderiza con todos los providers sin configuración extra', () => {
  render(<UserBadge />);
  expect(screen.getByText('No has iniciado sesión')).toBeInTheDocument();
});
```

```text
   render(<UserBadge />)            ← test
        │
        ▼
   <MemoryRouter>                   ← provider de rutas
     <AuthProvider>                 ← provider de sesión
       <UserBadge />                ← componente bajo prueba
     </AuthProvider>
   </MemoryRouter>
```

Este helper es la pieza que reutilizaremos en todos los pasos siguientes.

---

## Paso 5 — 🔧 Tu turno #1: probar el estado "con usuario"

Hasta ahora probamos el estado de invitado. A veces quieres testear el componente
como si **ya hubiera** sesión. Como nuestro `login` es simulado, podemos disparar
el login con interacción y luego verificar el resultado.

**Tu reto:** crea un componente auxiliar `PantallaConLogin` (solo para el test)
que pinte un botón "Entrar" que llame a `login('Ada')` y debajo el `UserBadge`.
Luego escribe un `it` (async) que renderice, haga clic en "Entrar" y verifique
que aparece "Hola, Ada" y el botón "Cerrar sesión".

<details><summary>✅ Solución</summary>

```tsx
// src/components/UserBadge.test.tsx (continúa)
import { render, screen } from '../test/utils';
import userEvent from '@testing-library/user-event';
import { useAuth } from '../context/AuthContext';

// Componente auxiliar solo para el test: añade un botón de login.
function PantallaConLogin() {
  const { login } = useAuth();
  return (
    <>
      <button onClick={() => login('Ada')}>Entrar</button>
      <UserBadge />
    </>
  );
}

it('muestra el nombre tras iniciar sesión', async () => {
  const user = userEvent.setup();
  render(<PantallaConLogin />);

  await user.click(screen.getByRole('button', { name: 'Entrar' }));

  expect(screen.getByText('Hola, Ada')).toBeInTheDocument();
  expect(
    screen.getByRole('button', { name: 'Cerrar sesión' }),
  ).toBeInTheDocument();
});
```

> Alternativa: si necesitas inyectar un usuario directamente, crea un
> `AuthProvider` de pruebas que acepte un `valorInicial`. Mantén la **misma
> firma** del contexto para no divergir de producción.

</details>

---

## Paso 6 — Navegación: `MemoryRouter` e `initialEntries`

En los tests no hay barra de direcciones del navegador. Usamos `MemoryRouter`,
que mantiene el historial **en memoria** y acepta `initialEntries` para decidir
en qué ruta arranca la app. Nuestro helper ya lo incluye, así que solo pasamos
`rutasIniciales`.

Supongamos estas rutas:

```tsx
// src/App.tsx
import { Routes, Route, Link } from 'react-router';
import { LoginPage } from './pages/LoginPage';
import { TodosPage } from './pages/TodosPage';

export function App() {
  return (
    <>
      <nav>
        <Link to="/login">Login</Link>
        <Link to="/todos">Tareas</Link>
      </nav>
      <Routes>
        <Route path="/login" element={<LoginPage />} />
        <Route path="/todos" element={<TodosPage />} />
      </Routes>
    </>
  );
}
```

Verificamos que arranca en la ruta correcta sin necesidad de navegar:

```tsx
// src/App.test.tsx
import { render, screen } from './test/utils';
import { App } from './App';

it('renderiza la página de login en la ruta /login', () => {
  // initialEntries decide la ruta inicial sin necesidad de navegar.
  render(<App />, { rutasIniciales: ['/login'] });

  expect(
    screen.getByRole('heading', { name: /iniciar sesión/i }),
  ).toBeInTheDocument();
});

it('renderiza la página de tareas en la ruta /todos', () => {
  render(<App />, { rutasIniciales: ['/todos'] });

  expect(
    screen.getByRole('heading', { name: /mis tareas/i }),
  ).toBeInTheDocument();
});
```

| Router | Cuándo usarlo |
| --- | --- |
| `MemoryRouter` | Tests: historial en memoria, controlas `initialEntries` |
| `BrowserRouter` | Producción en navegador: usa la URL real |
| `createMemoryRouter` | Tests del data router (loaders/actions) de react-router 7 |

> Nuestro `render` helper ya incluye `MemoryRouter`, así que pasamos
> `rutasIniciales` como opción. Evita anidar dos routers (el del helper y otro
> dentro del test): provocaría comportamientos confusos.

---

## Paso 7 — Verificar navegación al hacer clic

Arrancamos en una ruta y simulamos un clic en un enlace para comprobar que el
contenido de la nueva ruta aparece:

```tsx
// src/App.test.tsx (continúa)
import userEvent from '@testing-library/user-event';

it('navega de login a tareas al pulsar el enlace', async () => {
  const user = userEvent.setup();
  render(<App />, { rutasIniciales: ['/login'] });

  await user.click(screen.getByRole('link', { name: 'Tareas' }));

  // Tras navegar, el contenido de /todos está en pantalla.
  expect(
    screen.getByRole('heading', { name: /mis tareas/i }),
  ).toBeInTheDocument();
});
```

No verificamos la URL directamente: comprobamos **lo que ve el usuario** tras
navegar. Eso es testear comportamiento, no implementación.

---

## Paso 8 — 🔧 Tu turno #2: testear una redirección

Un patrón habitual: una ruta protegida que redirige a `/login` si no hay sesión.
Implementación con `<Navigate>`:

```tsx
// src/components/RutaProtegida.tsx
import { Navigate } from 'react-router';
import type { ReactNode } from 'react';
import { useAuth } from '../context/AuthContext';

// Si no hay usuario, redirige a /login; si lo hay, muestra el contenido.
export function RutaProtegida({ children }: { children: ReactNode }) {
  const { user } = useAuth();
  if (!user) return <Navigate to="/login" replace />;
  return <>{children}</>;
}
```

Y su uso en las rutas:

```tsx
// src/App.tsx (fragmento)
<Route
  path="/todos"
  element={
    <RutaProtegida>
      <TodosPage />
    </RutaProtegida>
  }
/>
```

**Tu reto:** escribe un `it` que arranque en `/todos` **sin** sesión y verifique
que (1) **sí** se ve el heading de "Iniciar sesión" y (2) **no** se ve el de "Mis
tareas". Pista: para afirmar ausencia, usa `queryByRole` (no `getByRole`).

<details><summary>✅ Solución</summary>

```tsx
// src/components/RutaProtegida.test.tsx
import { render, screen } from '../test/utils';
import { App } from '../App';

it('redirige a /login si se entra a /todos sin sesión', () => {
  // Sin login previo: AuthProvider arranca con user = null.
  render(<App />, { rutasIniciales: ['/todos'] });

  // No vemos las tareas; vemos el login porque hubo redirección.
  expect(
    screen.getByRole('heading', { name: /iniciar sesión/i }),
  ).toBeInTheDocument();
  expect(
    screen.queryByRole('heading', { name: /mis tareas/i }),
  ).not.toBeInTheDocument();
});
```

```text
  initialEntries=['/todos']
        │
        ▼
  RutaProtegida  →  user === null  →  <Navigate to="/login" />
        │
        ▼
  Se renderiza LoginPage  →  el test ve "Iniciar sesión"
```

> Como `<Navigate>` ocurre durante el render, no necesitas `waitFor`: la
> aserción es síncrona. Si tu guard hiciera la redirección en un `useEffect`,
> entonces sí usarías `findBy*` o `waitFor`.

</details>

---

### Prueba esto

- Crea un test que arranque en `/todos` **con** sesión (haciendo login antes) y verifica que NO redirige.
- Añade una ruta `*` (404) y testea que una ruta inexistente muestra "Página no encontrada".
- Extrae el `AuthProvider` de pruebas a `utils.tsx` con un parámetro `usuarioInicial` para inyectar sesión directamente.
- Comprueba que tras `logout()` el `UserBadge` vuelve a mostrar "No has iniciado sesión".
- Usa `rutasIniciales: ['/login', '/todos']` con un índice y observa cómo cambia la ruta inicial del historial.
- Verifica que el enlace activo recibe la clase correcta usando `NavLink` y `toHaveClass`.

---

## Ejercicios propuestos

### Básico

**B1 — Invitado por defecto.** Usando el `render` helper, escribe un `it` que
verifique que `UserBadge` muestra "No has iniciado sesión" sin ningún login previo.

**B2 — Arranque en /login.** Escribe un `it` que renderice `App` con
`rutasIniciales: ['/login']` y verifique que se ve el heading de la página de login.

### Intermedio

**I1 — Logout vuelve a invitado.** Parte del estado con sesión (haz login con un
botón), pulsa "Cerrar sesión" y verifica que vuelve a aparecer "No has iniciado
sesión".

**I2 — Navegación de ida y vuelta.** Arranca en `/login`, navega a `/todos` por el
enlace, vuelve a `/login` por su enlace y comprueba en cada paso que se ve el
heading correcto.

### Avanzado

**A1 — Provider de pruebas con `usuarioInicial`.** Amplía `utils.tsx` con un
`AuthProvider` de pruebas que acepte un `usuarioInicial?: User`, manteniendo la
firma del contexto. Úsalo para testear la ruta protegida **con** sesión sin
disparar `login` por interacción.

**A2 — Ruta 404.** Añade una `<Route path="*">` que muestre "Página no
encontrada". Escribe un test que arranque en `/no-existe` y verifique que aparece
ese texto, y otro que confirme que `/todos` (con sesión) no lo muestra.

---

## Resumen del módulo 10

- Los componentes que usan Context deben renderizarse **dentro** de su provider, o fallarán (lo comprobamos provocando el error en el Paso 1).
- Subimos por peldaños: envolver a mano → opción `wrapper` → un `render` helper centralizado en `src/test/utils.tsx` que envuelve todos los providers y re-exporta Testing Library.
- El estado "con usuario" se prueba disparando `login` por interacción (o con un provider de pruebas que reciba un valor inicial).
- Para testear rutas se usa `MemoryRouter` con `initialEntries`, controlando en qué ruta arranca la app.
- La navegación se prueba con `user-event` (clics en enlaces) y verificando el contenido de la nueva ruta.
- Las redirecciones con `<Navigate>` se comprueban arrancando en la ruta protegida sin sesión y verificando que aparece el login (de forma síncrona).

---

> **Siguiente página →** Módulo 11: medir cobertura con el provider v8, definir umbrales, distinguir qué testear de qué no, evitar anti-patrones y un apéndice de equivalencias con Jest.
