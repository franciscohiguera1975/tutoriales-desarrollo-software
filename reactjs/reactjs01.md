# Curso React.js + TypeScript — Página 1
## Módulo 1 · Fundamentos
### Introducción, entorno y filosofía de tipos — React 19 + Vite 8

---

## ¿Por qué React con TypeScript?

TypeScript no es un añadido opcional en proyectos serios: es el estándar de la industria. Incorporarlo desde el inicio evita la deuda técnica de migrar una base de código JavaScript ya existente, que es costosa y propensa a errores silenciosos.

| Sin TypeScript | Con TypeScript |
|---|---|
| Errores en tiempo de ejecución | Errores detectados en tiempo de escritura |
| Props sin contrato explícito | Interfaces que documentan y validan la API del componente |
| Refactorización manual y riesgosa | El compilador guía cada cambio |
| Autocompletado limitado | IntelliSense completo en el editor |

> Hoy más del **70 % de los proyectos React nuevos** arrancan directamente con TypeScript.
> React 19 es la versión estable actual — incluye mejoras en el compilador, nuevos hooks
> y cambios en la forma en que se manejan los tipos.

---

## Prerrequisitos

- **JavaScript ES6+**: destructuring, spread/rest, módulos, async/await, arrow functions.
- **Conceptos básicos de tipado**: qué es un tipo, una interfaz, un genérico.
- **Node.js LTS** instalado (`v20.x` o superior).

```bash
node --version   # v20.x o superior
npm --version    # 10.x o superior
```

---

## Crear el proyecto — React 19 + Vite 8

```bash
npm create vite@latest mi-proyecto -- --template react-ts
cd mi-proyecto
npm install
npm run dev
```

El servidor de desarrollo arranca en `http://localhost:5173`.

Al correr `npm install` verás en `package.json` las versiones actuales:

```json
{
  "dependencies": {
    "react":     "^19.2.4",
    "react-dom": "^19.2.4"
  },
  "devDependencies": {
    "@types/react":        "^19.2.14",
    "@types/react-dom":    "^19.2.3",
    "@vitejs/plugin-react": "^6.0.1",
    "typescript":          "~5.9.3",
    "vite":                "^8.0.1",
    "eslint":              "^9.39.4",
    "typescript-eslint":   "^8.57.0"
  }
}
```

> `@types/react` y `@types/react-dom` son las definiciones de tipos de React para TypeScript.
> Con el template `react-ts` ya vienen instaladas — no necesitas agregarlas manualmente.

---

## Estructura del proyecto generado

```
mi-proyecto/
├── public/
│   ├── favicon.svg
│   └── icons.svg
├── src/
│   ├── assets/
│   │   ├── hero.png
│   │   ├── react.svg
│   │   └── vite.svg
│   ├── App.css
│   ├── App.tsx          ← componente raíz
│   ├── index.css        ← estilos globales
│   └── main.tsx         ← punto de entrada
├── index.html
├── package.json
├── vite.config.ts
├── eslint.config.js     ← ESLint 9 con flat config (nuevo en React 19 template)
├── tsconfig.json        ← raíz: orquesta los otros dos
├── tsconfig.app.json    ← configuración para el código de la aplicación (src/)
└── tsconfig.node.json   ← configuración para archivos de herramientas (vite.config.ts)
```

### Diferencia crítica: `.tsx` vs `.ts`

| Extensión | Uso |
|---|---|
| `.tsx` | Archivos TypeScript **que contienen JSX** — componentes React |
| `.ts` | Archivos TypeScript sin JSX — lógica pura, tipos, utilidades |

---

## Los tres archivos `tsconfig`

React 19 con Vite separa la configuración de TypeScript en tres archivos con responsabilidades distintas. Esto es una mejora deliberada sobre versiones anteriores que usaban un solo archivo.

---

### `tsconfig.json` — el orquestador

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

No contiene `compilerOptions`. Su único rol es **declarar qué proyectos TypeScript existen**
usando referencias de proyecto (`references`). Esto le indica a `tsc` que el workspace
tiene dos contextos de compilación separados e independientes.

- `files: []` — el proyecto raíz no compila nada por sí solo.
- `references` — apunta a los dos proyectos reales.

---

### `tsconfig.app.json` — configuración de la aplicación

Es el archivo que aplica a todo el código dentro de `src/`. Es donde viven tus componentes,
hooks y lógica de negocio.

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "target": "ES2023",
    "useDefineForClassFields": true,
    "lib": ["ES2023", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "types": ["vite/client"],
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["src"]
}
```

Las opciones más importantes explicadas:

| Opción | Valor | Qué hace |
|---|---|---|
| `target` | `ES2023` | A qué versión de JS se compila el output |
| `lib` | `["ES2023","DOM","DOM.Iterable"]` | Qué APIs del entorno están disponibles |
| `jsx` | `"react-jsx"` | Transforma JSX sin necesidad de `import React` en cada archivo |
| `strict` | `true` | Activa **todas** las comprobaciones estrictas — innegociable |
| `moduleResolution` | `"bundler"` | Resolución de módulos optimizada para bundlers como Vite |
| `verbatimModuleSyntax` | `true` | Fuerza usar `import type` para importar solo tipos — React 19 |
| `noEmit` | `true` | TypeScript solo verifica tipos; Vite se encarga del bundle |
| `erasableSyntaxOnly` | `true` | Nuevo en TS 5.5 — previene sintaxis que no puede eliminarse sin ejecutar |
| `noUncheckedSideEffectImports` | `true` | Nuevo — advierte sobre imports con efectos secundarios sin verificar |
| `include` | `["src"]` | Solo compila archivos dentro de `src/` |

> **`"strict": true` es innegociable.** Actívalo desde el inicio.
> Añadirlo después a un proyecto grande es extremadamente doloroso.

> **`"verbatimModuleSyntax": true`** es nuevo en el template de React 19.
> Significa que cuando importas solo un tipo, debes usar `import type`:
>
> ```ts
> // ❌ Con verbatimModuleSyntax activo, esto causa un warning
> import { FC } from 'react'
>
> // ✅ Correcto — le dice explícitamente que es solo un tipo
> import type { FC } from 'react'
>
> // ✅ También válido — mezcla valores y tipos en un solo import
> import React, { type FC } from 'react'
> ```

### Prueba esto

- Cambia `"target": "ES2023"` a `"ES2020"` y corre `npm run build` — la compilación sigue funcionando; ES2023 solo añade algunas APIs de arrays nuevas
- Elimina temporalmente `"strict": true` — observa cómo TypeScript deja de detectar múltiples categorías de errores (luego vuélvelo a activar)
- Añade `"noImplicitReturns": true` bajo `/* Linting */` — TypeScript marcará error si una función puede retornar sin un `return` explícito
- Cambia `"jsx": "react-jsx"` a `"react"` — los archivos `.tsx` que no importan React empezarán a dar error `React is not defined`
- Añade `"paths": { "@/*": ["./src/*"] }` bajo `compilerOptions` — habilita alias de importación; también deberás configurarlo en `vite.config.ts`
- Revisa que `"verbatimModuleSyntax": true` está activo — escribe `import { FC } from 'react'` en un componente y observa el error que produce

---

### `tsconfig.node.json` — configuración de herramientas

Aplica **únicamente** a `vite.config.ts` y otros archivos de configuración del tooling.
Tiene su propio contexto porque se ejecuta en Node.js, no en el navegador.

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",
    "target": "ES2023",
    "lib": ["ES2023"],
    "module": "ESNext",
    "types": ["node"],
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["vite.config.ts"]
}
```

Diferencias clave respecto a `tsconfig.app.json`:

| Diferencia | `tsconfig.app.json` | `tsconfig.node.json` |
|---|---|---|
| `lib` | `["ES2023", "DOM", "DOM.Iterable"]` | `["ES2023"]` — sin DOM |
| `types` | `["vite/client"]` | `["node"]` — tipos de Node.js |
| `jsx` | `"react-jsx"` | ausente — no hay JSX aquí |
| `include` | `["src"]` | `["vite.config.ts"]` |

No hay `"DOM"` en `lib` porque `vite.config.ts` corre en Node.js, no en el navegador.

---

### ¿Por qué tres archivos en lugar de uno?

| Razón | Explicación |
|---|---|
| **Contextos distintos** | `src/` corre en el navegador; `vite.config.ts` corre en Node.js. Mezclarlos en un solo `tsconfig` contamina los tipos disponibles |
| **Compilación incremental** | Con referencias de proyecto, `tsc` puede compilar cada parte por separado y cachear el resultado (`tsBuildInfoFile`) |
| **Errores más precisos** | Si usas una API del DOM en `vite.config.ts`, TypeScript lo detecta porque ese archivo no tiene `"DOM"` en su `lib` |
| **Escalabilidad** | En proyectos más grandes se pueden añadir más referencias (tests, scripts, etc.) sin tocar la configuración de la app |

---

## `vite.config.ts`

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

Minimalista por diseño. El plugin `@vitejs/plugin-react` hace tres cosas:
- Transforma JSX usando el runtime automático de React (`react-jsx`).
- Habilita Fast Refresh (HMR sin perder el estado de los componentes).
- Integra Babel para transformaciones adicionales si se necesitan.

### Prueba esto

- Añade `server: { port: 3000 }` dentro de `defineConfig({})` — el servidor cambiará de `5173` a `3000` al reiniciar con `npm run dev`
- Añade `server: { open: true }` — el navegador se abrirá automáticamente al iniciar el servidor de desarrollo
- Añade `build: { outDir: 'build' }` — la carpeta de producción cambiará de `dist` a `build`
- Añade `resolve: { alias: { '@': '/src' } }` — configura el alias `@` para imports absolutos; añade el `paths` correspondiente en `tsconfig.app.json`
- Observa que `defineConfig` es una función helper que solo aporta tipos; el objeto de configuración funciona igual sin ella
- Revisa la lista de plugins en la documentación de Vite — `@vitejs/plugin-react-swc` es la alternativa con SWC (Rust) en lugar de Babel, compilación más rápida pero sin soporte de Babel plugins

---

## `eslint.config.js` — ESLint 9 con flat config

React 19 adopta el nuevo formato de configuración de ESLint 9 (`flat config`),
reemplazando el antiguo `.eslintrc.json`:

```js
// eslint.config.js
import js from '@eslint/js'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import tseslint from 'typescript-eslint'
import { defineConfig, globalIgnores } from 'eslint/config'

export default defineConfig([
  globalIgnores(['dist']),
  {
    files: ['**/*.{ts,tsx}'],
    extends: [
      js.configs.recommended,
      tseslint.configs.recommended,
      reactHooks.configs.flat.recommended,
      reactRefresh.configs.vite,
    ],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
    },
  },
])
```

Reglas activas por defecto:

| Plugin | Qué revisa |
|---|---|
| `@eslint/js` | Reglas base de JavaScript |
| `typescript-eslint` | Reglas específicas de TypeScript |
| `eslint-plugin-react-hooks` | Uso correcto de hooks (orden, dependencias) |
| `eslint-plugin-react-refresh` | Compatibilidad con HMR de Vite |

---

## `main.tsx` — punto de entrada

```tsx
// src/main.tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

- `createRoot` — API de React 18+ que habilita el modo concurrente. Sin cambios en React 19.
- `document.getElementById('root')!` — el `!` es una non-null assertion: le dice a TypeScript que ese elemento siempre existe en el DOM.
- `<StrictMode>` — activo solo en desarrollo. En React 19 detecta más problemas que en versiones anteriores, incluyendo el uso de APIs obsoletas del nuevo compilador.
- `import App from './App.tsx'` — en React 19 con `verbatimModuleSyntax` se importa el archivo con extensión explícita.

---

## Tu primer componente limpio

El `App.tsx` que genera el template viene con código de demostración. Reemplázalo completamente:

```tsx
// src/App.tsx

export default function App() {
  return (
    <main style={{ maxWidth: 480, margin: '40px auto', fontFamily: 'sans-serif' }}>
      <h1>Hola desde React 19 + TypeScript</h1>
      <p>Proyecto configurado con Vite 8.</p>
    </main>
  )
}
```

Este componente no usa tipos `React.*` directamente — no necesita `import React`.
Con `"jsx": "react-jsx"` en `tsconfig.app.json` el JSX se transforma automáticamente.

### Prueba esto

- Cambia `maxWidth: 480` a `600` y guarda — observa el HMR en acción: el navegador actualiza sin recargar la página completa
- Añade `color: '#1a6b4a'` al objeto de estilo del `<h1>` — los estilos inline en JSX usan camelCase, no kebab-case como en CSS
- Agrega un `<p>Versión: 1.0.0</p>` debajo del primer párrafo — JSX admite múltiples elementos hermanos dentro del mismo contenedor padre
- Convierte `<main>` en `<section>` — los estilos inline siguen funcionando; las propiedades de estilo no dependen del elemento HTML
- Renombra la función `App` a `MiApp` y actualiza el `export default` — TypeScript no fuerza el nombre del componente, pero `eslint-plugin-react-refresh` puede advertirte si no coincide con el nombre del archivo
- Extrae el contenido del `<main>` a un componente `Bienvenida` dentro del mismo archivo y úsalo: `<main><Bienvenida /></main>`

---

## Extensiones recomendadas (VS Code)

| Extensión | Propósito |
|---|---|
| **ESLint** | Muestra errores de linting en el editor en tiempo real |
| **Prettier** | Formateo automático al guardar |
| **Error Lens** | Muestra errores de TypeScript inline junto al código |
| **Auto Rename Tag** | Renombra etiquetas JSX de apertura y cierre simultáneamente |

---


## Tipo de retorno en componentes — React 19

En versiones anteriores de React era común anotar el tipo de retorno de un componente como `JSX.Element`. En React 19 con `"moduleDetection": "force"` y `"types": ["vite/client"]` en `tsconfig.app.json`, **esto produce un error**:

```
Cannot find namespace 'JSX'
```

Esto ocurre porque `moduleDetection: force` trata cada archivo como un módulo ES aislado, y el namespace global `JSX` no está disponible sin un import explícito que lo traiga al scope.

### Las tres formas correctas en React 19

```tsx
// Forma 1 — React.JSX.Element (recomendada cuando ya importas React)
import React from 'react'

export default function MyComponent() {
  return <div>Hola</div>
}
```

```tsx
// Forma 2 — ReactElement con import type (semánticamente precisa)
import type { ReactElement } from 'react'

export default function MyComponent(): ReactElement {
  return <div>Hola</div>
}
```

```tsx
// Forma 3 — sin anotación (TypeScript infiere el tipo correctamente)
// Válido cuando el componente no necesita ningún import de React
export default function MyComponent() {
  return <div>Hola</div>
}
```

> **Recomendación del curso**: si el componente ya importa `React` por otras razones
> (como usar `React.ReactNode` o `React.CSSProperties`), usa `React.JSX.Element`.
> Si el componente no necesita importar React para nada más, omite la anotación
> y deja que TypeScript infiera — es la opción más limpia.

| Forma | Import necesario | Cuándo usarla |
|---|---|---|
| `React.JSX.Element` | `import React from 'react'` | Cuando ya usas otros tipos `React.*` |
| `ReactElement` | `import type { ReactElement } from 'react'` | Cuando no necesitas el namespace completo |
| Sin anotación | ninguno | Componentes simples sin otras dependencias de React |

---
## `src/App.tsx`

El proyecto usa una constante `PASO` para navegar entre componentes sin cambiar la URL — idéntica al patrón `const int paso = 1` del módulo Flutter. Cambia `PASO` y guarda (`Ctrl+S`) para ver cada componente en el navegador.

```tsx
// src/App.tsx
// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  PrimerComponente  — componente de bienvenida                    │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

function PrimerComponente() {
  return (
    <main style={{ maxWidth: 480, margin: '40px auto', fontFamily: 'sans-serif' }}>
      <h1>Hola desde React 19 + TypeScript</h1>
      <p>Proyecto configurado con Vite 8.</p>
    </main>
  )
}

export default function App() {
  const content =
    PASO === 1 ? <PrimerComponente /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return content
}
```

### Prueba esto

- Cambia `PASO` a `2` — aparece el mensaje de error en rojo porque no existe un componente para ese paso
- Añade una línea al ternario: `PASO === 2 ? <p>Componente 2 pendiente</p> :` — ahora `PASO = 2` muestra ese placeholder
- Mueve `PrimerComponente` a `src/components/PrimerComponente.tsx` e impórtalo en `App.tsx` — observa que TypeScript valida el import automáticamente
- Agrega una segunda entrada al índice del comentario — el índice documenta en el propio archivo qué hay en cada paso
- Observa que `const content` no necesita anotación de tipo — TypeScript infiere `JSX.Element` desde las ramas del ternario

---

## Resumen de la página 1

- React 19.2 es la versión estable actual. Se instala automáticamente con el template `react-ts` de Vite 8.
- El proyecto genera **tres archivos tsconfig** con responsabilidades separadas:
  - `tsconfig.json` — orquestador, solo declara referencias.
  - `tsconfig.app.json` — configuración para `src/` (navegador).
  - `tsconfig.node.json` — configuración para `vite.config.ts` (Node.js).
- `"strict": true` en `tsconfig.app.json` es innegociable.
- `"verbatimModuleSyntax": true` — novedad de React 19: fuerza `import type` para importar solo tipos.
- `eslint.config.js` usa el nuevo formato flat config de ESLint 9.
- El `App.tsx` del template tiene código de demostración — se reemplaza al iniciar el proyecto real.

---

> **Siguiente página →** JSX: qué es, cómo funciona, reglas fundamentales,
> diferencias con HTML y componentes funcionales tipados.