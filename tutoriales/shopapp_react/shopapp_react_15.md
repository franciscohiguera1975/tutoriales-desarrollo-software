# ShopApp React Web — Módulo 15

## Build y Deploy — Optimización, Nginx y Docker

**Duración estimada:** 2–3 horas
**Prerequisitos:** Módulos 1–14 completados. Docker y Docker Compose instalados en la máquina de despliegue.

---

> **Objetivo**
> Preparar la aplicación React para producción: variables de entorno con Vite, optimización del bundle
> con Rollup, configuración de Nginx para SPA con React Router, empaquetado multi-etapa con Docker y
> orquestación con Docker Compose junto al backend Django.
>
> **Checkpoint final**
> - `npm run build` genera el directorio `dist/` sin errores de TypeScript.
> - `npm run preview` sirve la app correctamente desde `dist/`.
> - `docker build -t shopapp-react .` construye la imagen sin errores.
> - `docker run -p 3000:80 shopapp-react` sirve la app en `http://localhost:3000`.
> - Navegar directamente a `/catalog` en el navegador devuelve la app (no 404).
> - `docker-compose up -d --build` levanta el frontend junto al backend Django.

---

## 15.1 Variables de entorno

Vite expone variables de entorno al código de la aplicación bajo el objeto `import.meta.env`. Solo las variables cuyo nombre comienza con `VITE_` se incluyen en el bundle. Las demás variables del sistema operativo son ignoradas por seguridad.

### Archivos de entorno

**Archivo:** `.env` (desarrollo local — no se sube al repositorio)

```bash
# .env
VITE_API_BASE_URL=http://localhost:8000/api
```

**Archivo:** `.env.production` (producción — no se sube al repositorio)

```bash
# .env.production
VITE_API_BASE_URL=https://api.midominio.com/api
```

**Archivo:** `.env.example` (plantilla — sí se sube al repositorio como referencia)

```bash
# .env.example
# Copia este archivo como .env y completa los valores reales.
# Para producción cópialo como .env.production.

# URL base del backend Django REST API (sin barra final)
VITE_API_BASE_URL=http://localhost:8000/api
```

Agrega los archivos de entorno reales al `.gitignore`:

```bash
# .gitignore (agregar estas líneas)
.env
.env.production
.env.local
.env.*.local
```

### Uso en el código

En `src/core/config/env.ts`:

```typescript
// src/core/config/env.ts

/**
 * Centraliza el acceso a las variables de entorno de Vite.
 * Importar desde aquí en lugar de usar import.meta.env directamente
 * facilita el tipado y el mantenimiento.
 */
export const env = {
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL as string,
  isDevelopment: import.meta.env.DEV,
  isProduction: import.meta.env.PROD,
  mode: import.meta.env.MODE,
} as const;
```

En `src/data/api/api.client.ts` (ya configurado en el módulo 2, confirmar que usa `env.apiBaseUrl`):

```typescript
// src/data/api/api.client.ts

import axios from 'axios';
import { env } from '@/core/config/env';

export const apiClient = axios.create({
  baseURL: env.apiBaseUrl,
  headers: { 'Content-Type': 'application/json' },
  timeout: 15000,
});
```

**Comportamiento de Vite con entornos:**

| Comando | Archivo cargado | `import.meta.env.MODE` |
|---------|----------------|------------------------|
| `npm run dev` | `.env` → `.env.local` | `"development"` |
| `npm run build` | `.env` → `.env.production` | `"production"` |
| `npm run preview` | `.env` → `.env.production` | `"production"` |

`.env.local` tiene precedencia sobre `.env`. `.env.production` tiene precedencia sobre `.env` solo durante el build.

---

## 15.2 Build con Vite

```bash
# Genera el directorio dist/ con los archivos optimizados
npm run build

# Sirve el build de producción localmente para verificar antes de desplegar
npm run preview
```

`npm run build` ejecuta en secuencia: comprobación de tipos TypeScript (`tsc --noEmit`) y luego Rollup (el bundler interno de Vite) para empaquetar y minificar. El resultado queda en `dist/`.

`npm run preview` levanta un servidor HTTP estático que sirve `dist/` en `http://localhost:4173`. Es útil para detectar problemas que solo aparecen en el bundle de producción (rutas, variables de entorno, imports dinámicos).

### Estructura de dist/ después del build

```
dist/
├── index.html
├── assets/
│   ├── index-[hash].js       ← bundle principal
│   ├── vendor-[hash].js      ← React y React-DOM (si usas manualChunks)
│   ├── zustand-[hash].js     ← Zustand
│   ├── axios-[hash].js       ← Axios
│   ├── index-[hash].css      ← Tailwind compilado y purgeado
│   └── [page]-[hash].js      ← chunks de páginas (lazy import)
```

Los hashes en los nombres de archivo son derivados del contenido: si el archivo no cambia entre builds, el hash es idéntico y el navegador puede usar la versión en caché. Si cambia, el hash es distinto y el navegador descarga la versión nueva.

### Analizar el bundle

Para identificar qué módulos contribuyen más al tamaño del bundle:

```bash
npm install -D rollup-plugin-visualizer
```

Agrega el plugin en `vite.config.ts` (ver sección 15.3).

Después de ejecutar `npm run build`, se genera `stats.html` en la raíz del proyecto. Ábrelo en el navegador para ver el mapa de árbol interactivo.

---

## 15.3 Configuración optimizada de Vite

**Archivo:** `vite.config.ts` (versión completa para producción)

```typescript
// vite.config.ts

import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';
import { visualizer } from 'rollup-plugin-visualizer';
import { resolve } from 'path';

export default defineConfig(({ mode }) => ({
  plugins: [
    react(),
    tailwindcss(),
    // El visualizador solo se activa si se pasa ANALYZE=true al build.
    // Uso: ANALYZE=true npm run build
    mode === 'production' && process.env.ANALYZE === 'true'
      ? visualizer({
          filename: 'stats.html',
          open: true,       // Abre el archivo en el navegador al terminar el build
          gzipSize: true,   // Muestra el tamaño después de gzip además del tamaño raw
          brotliSize: true, // Muestra el tamaño después de brotli
        })
      : null,
  ].filter(Boolean),

  resolve: {
    alias: {
      // Alias @ apunta a src/, configurado desde el módulo 1
      '@': resolve(__dirname, './src'),
    },
  },

  build: {
    // No generar sourcemaps en producción para no exponer el código fuente
    sourcemap: false,

    // esbuild es el minificador por defecto de Vite y el más rápido
    minify: 'esbuild',

    // Tamaño máximo de un chunk antes de emitir advertencia (en KB)
    chunkSizeWarningLimit: 600,

    rollupOptions: {
      output: {
        /**
         * manualChunks divide el bundle en archivos separados por dominio lógico.
         *
         * Beneficio: si solo cambia el código de la app (no las librerías),
         * los chunks de vendor siguen siendo los mismos y el navegador los
         * mantiene en caché, reduciendo el tamaño de la actualización.
         */
        manualChunks(id: string) {
          // React y React DOM van juntos: siempre se usan simultáneamente
          if (id.includes('node_modules/react/') || id.includes('node_modules/react-dom/')) {
            return 'vendor-react';
          }
          // React Router en su propio chunk: cambia menos frecuentemente
          if (id.includes('node_modules/react-router')) {
            return 'vendor-router';
          }
          // Zustand es pequeño pero vale separarlo para análisis
          if (id.includes('node_modules/zustand')) {
            return 'vendor-zustand';
          }
          // Axios
          if (id.includes('node_modules/axios')) {
            return 'vendor-axios';
          }
          // Componentes de shadcn/ui (Radix UI)
          if (id.includes('node_modules/@radix-ui')) {
            return 'vendor-ui';
          }
          // Zod y react-hook-form juntos (siempre se usan juntos para formularios)
          if (
            id.includes('node_modules/zod') ||
            id.includes('node_modules/react-hook-form') ||
            id.includes('node_modules/@hookform')
          ) {
            return 'vendor-forms';
          }
          // Lucide (iconos): puede ser grande si se importan muchos iconos
          if (id.includes('node_modules/lucide-react')) {
            return 'vendor-icons';
          }
        },
      },
    },
  },

  // Configuración del servidor de desarrollo (no afecta al build de producción)
  server: {
    port: 5173,
    host: true, // Escucha en 0.0.0.0 para acceso desde otros dispositivos en la red local
  },

  // Configuración del servidor de preview (npm run preview)
  preview: {
    port: 4173,
    host: true,
  },
}));
```

**Por qué separar vendors:** el código de las librerías de `node_modules` cambia solo cuando actualizas dependencias (evento poco frecuente). El código de la app (`src/`) cambia con cada feature. Al separarlos en chunks distintos, una nueva versión de la app solo invalida el cache del chunk de app, mientras que los chunks de vendor siguen siendo servidos desde el caché del navegador.

---

## 15.4 Configuración de Nginx

**Archivo:** `nginx.conf`

```nginx
# nginx.conf

server {
    listen 80;
    server_name _;          # Acepta cualquier nombre de host; ajusta en producción real
    root /usr/share/nginx/html;
    index index.html;

    # ─── SPA Routing ────────────────────────────────────────────────────────────
    # React Router maneja las rutas en el cliente (JavaScript).
    # Sin esta configuración, una petición directa a /catalog devolvería 404
    # porque Nginx buscaría un archivo llamado "catalog" en el sistema de archivos
    # y no lo encontraría.
    #
    # try_files intenta en orden:
    #   1. $uri          → archivo exacto (ej: /assets/index-abc123.js sí existe)
    #   2. $uri/         → directorio (no aplica para SPA)
    #   3. /index.html   → fallback: sirve index.html y React Router toma el control
    location / {
        try_files $uri $uri/ /index.html;
    }

    # ─── Cache agresivo para assets estáticos ───────────────────────────────────
    # Los archivos en /assets/ tienen hashes en el nombre (vendor-abc123.js).
    # Si el contenido cambia, el nombre cambia y el hash es distinto.
    # Por lo tanto, es seguro cachearlos durante 1 año.
    #
    # Cache-Control: immutable indica al navegador que no re-valide el archivo
    # durante el periodo de cache, ni siquiera con una petición condicional If-None-Match.
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # ─── Seguridad ───────────────────────────────────────────────────────────────
    # X-Frame-Options: DENY previene que la app sea embebida en un iframe
    # (protección contra clickjacking).
    add_header X-Frame-Options DENY;

    # X-Content-Type-Options: nosniff previene que el navegador "adivine" el tipo
    # MIME de una respuesta (protección contra ataques MIME sniffing).
    add_header X-Content-Type-Options nosniff;

    # Referrer-Policy: controla qué información se envía en la cabecera Referer
    # al navegar a otros dominios. strict-origin-when-cross-origin envía el origen
    # completo en peticiones al mismo origen, y solo el dominio en peticiones cross-origin.
    add_header Referrer-Policy strict-origin-when-cross-origin;

    # ─── Compresión gzip ─────────────────────────────────────────────────────────
    # Reduce el tamaño de transferencia de archivos de texto antes de enviarlos al cliente.
    # El navegador descomprime automáticamente si envía Accept-Encoding: gzip.
    gzip on;
    gzip_vary on;          # Agrega Vary: Accept-Encoding para que los proxies cacheen correctamente
    gzip_proxied any;      # Comprime también las respuestas hacia proxies
    gzip_comp_level 6;     # Nivel de compresión: 6 es el balance recomendado velocidad/ratio
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;
    gzip_min_length 1000;  # No comprimir archivos menores a 1 KB (overhead no justificado)
}
```

**El punto más importante:** la directiva `try_files $uri $uri/ /index.html` en el bloque `location /` es lo que hace que React Router funcione en producción. Sin ella, recargar la página en cualquier ruta distinta de `/` (como `/catalog` o `/admin/products`) devolvería un 404 de Nginx porque esos directorios no existen en el filesystem.

---

## 15.5 Dockerfile multi-etapa

**Archivo:** `Dockerfile`

```dockerfile
# Dockerfile

# ─── Etapa 1: Build ──────────────────────────────────────────────────────────
# Usa Node 20 Alpine: imagen pequeña con Node.js para compilar la app.
FROM node:20-alpine AS build

# Establece el directorio de trabajo dentro del contenedor.
WORKDIR /app

# Copia primero los archivos de dependencias.
# Docker cachea esta capa: si package.json y package-lock.json no cambian,
# npm ci no se vuelve a ejecutar aunque el código fuente cambie.
COPY package.json package-lock.json ./

# npm ci (clean install) es más estricto y reproducible que npm install:
# - Usa package-lock.json como fuente de verdad.
# - Elimina node_modules antes de instalar.
# - Falla si hay discrepancias entre package.json y package-lock.json.
RUN npm ci

# Copia el resto del código fuente.
COPY . .

# Copia el archivo de entorno de producción para el build.
# Si prefieres pasar la URL como ARG de Docker, ver la nota al pie.
COPY .env.production .env.production

# Genera el bundle de producción en /app/dist/.
RUN npm run build

# ─── Etapa 2: Serve ──────────────────────────────────────────────────────────
# Usa la imagen oficial de Nginx Alpine: extremadamente ligera (~7 MB).
FROM nginx:alpine AS serve

# Elimina la configuración por defecto de Nginx.
RUN rm /etc/nginx/conf.d/default.conf

# Copia la configuración personalizada.
COPY nginx.conf /etc/nginx/conf.d/shopapp.conf

# Copia los archivos del build desde la etapa anterior.
# Solo se copian los archivos de dist/, no node_modules ni código fuente.
COPY --from=build /app/dist /usr/share/nginx/html

# Nginx escucha en el puerto 80.
EXPOSE 80

# CMD por defecto de la imagen nginx:alpine es iniciar nginx en primer plano.
# No es necesario definirlo explícitamente.
```

**Por qué multi-etapa:** la imagen final no contiene Node.js, `node_modules`, código fuente, ni herramientas de build. Solo contiene Nginx y los archivos estáticos de `dist/`. Esto reduce el tamaño de la imagen de ~1 GB (con Node y node_modules) a ~30 MB, disminuye la superficie de ataque y acelera los despliegues.

**Alternativa con ARG para la URL de la API en build time:**

```dockerfile
# Alternativa: pasar la URL como argumento de build (no requiere .env.production en el repo)
ARG VITE_API_BASE_URL=http://localhost:8000/api
ENV VITE_API_BASE_URL=$VITE_API_BASE_URL
RUN npm run build

# Uso al construir:
# docker build --build-arg VITE_API_BASE_URL=https://api.midominio.com/api -t shopapp-react .
```

---

## 15.6 Docker Compose

**Archivo:** `docker-compose.yml`

```yaml
# docker-compose.yml

version: '3.9'

services:
  # ─── Frontend React ───────────────────────────────────────────────────────────
  web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: shopapp-web
    ports:
      # Puerto host:puerto contenedor
      # El contenedor expone Nginx en el puerto 80.
      # Se mapea al 3000 del host para no requerir privilegios root en desarrollo.
      - "3000:80"
    env_file:
      # Variables de entorno adicionales (si se usa window.__RUNTIME_CONFIG__, ver 15.8)
      - .env.production
    depends_on:
      - api
    networks:
      - shopapp-network
    restart: unless-stopped

  # ─── Backend Django REST ──────────────────────────────────────────────────────
  # Referencia al servicio del backend (definido en shopapi/docker-compose.yml).
  # Aquí se muestra cómo incluir el backend en el mismo archivo compose.
  api:
    # Sustituye esta sección por la configuración real de tu shopapi docker-compose.
    # Si el backend corre en un docker-compose separado, usa `external: true` en la red.
    image: shopapi:latest            # Imagen del backend (debe estar construida)
    container_name: shopapp-api
    ports:
      - "8000:8000"
    environment:
      - DJANGO_SETTINGS_MODULE=shopapi.settings.production
      - DATABASE_URL=sqlite:///db.sqlite3   # Reemplaza con PostgreSQL en producción real
      - ALLOWED_HOSTS=localhost,api
      - CORS_ALLOWED_ORIGINS=http://localhost:3000,https://midominio.com
    networks:
      - shopapp-network
    restart: unless-stopped

# ─── Red compartida ──────────────────────────────────────────────────────────
# Los servicios en la misma red pueden comunicarse por nombre de servicio.
# Desde el frontend (Nginx) no es necesario: el browser hace las peticiones
# directamente a la URL pública de la API.
networks:
  shopapp-network:
    driver: bridge
```

**Si el backend está en un docker-compose separado** (el caso más común en este tutorial):

```yaml
# docker-compose.yml — solo el frontend, conectado a red externa del backend

version: '3.9'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: shopapp-web
    ports:
      - "3000:80"
    restart: unless-stopped

# No se necesita red compartida si el browser accede a la API por URL pública.
```

En ese caso, `VITE_API_BASE_URL` debe apuntar a la URL pública o IP del servidor donde corre Django, no a `localhost`.

---

## 15.7 Comandos de deploy

### Build y ejecución manual

```bash
# Construir la imagen Docker etiquetada como shopapp-react
docker build -t shopapp-react .

# Ejecutar el contenedor, mapeando el puerto 3000 del host al 80 del contenedor
docker run -p 3000:80 shopapp-react

# Ejecutar en segundo plano (detached mode)
docker run -d -p 3000:80 --name shopapp-web shopapp-react

# Ver los logs del contenedor
docker logs shopapp-web

# Detener y eliminar el contenedor
docker stop shopapp-web && docker rm shopapp-web
```

### Con Docker Compose

```bash
# Levantar todos los servicios en segundo plano y reconstruir las imágenes
docker-compose up -d --build

# Ver el estado de los servicios
docker-compose ps

# Ver logs de todos los servicios (Ctrl+C para salir)
docker-compose logs -f

# Ver logs solo del frontend
docker-compose logs -f web

# Detener y eliminar contenedores (los volúmenes se conservan)
docker-compose down

# Detener y eliminar contenedores y volúmenes (cuidado en producción)
docker-compose down -v
```

### Verificar que el deploy funciona

```bash
# La página principal debe responder
curl -I http://localhost:3000

# React Router: rutas internas deben devolver index.html (no 404)
curl -I http://localhost:3000/catalog
curl -I http://localhost:3000/admin/products

# Los assets deben tener Cache-Control correcto
curl -I http://localhost:3000/assets/index-[hash].js
# Esperado: Cache-Control: public, immutable
```

---

## 15.8 URL de la API en runtime vs build time

### El problema: VITE_ es build-time, no runtime

Vite reemplaza `import.meta.env.VITE_API_BASE_URL` por el valor literal de la cadena de texto **durante el build**. El valor queda "quemado" en el JavaScript generado:

```javascript
// Lo que escribe el desarrollador:
const BASE = import.meta.env.VITE_API_BASE_URL;

// Lo que genera Vite en el bundle:
const BASE = "https://api.midominio.com/api";
```

Esto significa que **no puedes cambiar la URL de la API cambiando una variable de entorno en el contenedor Docker** sin reconstruir la imagen. Para muchos proyectos esto es aceptable, pero si necesitas una imagen Docker que funcione en múltiples entornos (staging, producción, QA) con la misma imagen, necesitas el patrón de configuración en runtime.

### Patrón window.__RUNTIME_CONFIG__

**1. Crea el script de configuración en runtime:**

```html
<!-- public/config.js — será copiado a dist/ por Vite sin procesar -->
<!-- IMPORTANT: Este archivo es reemplazado por el script de entrypoint de Docker -->
window.__RUNTIME_CONFIG__ = {
  apiBaseUrl: "http://localhost:8000/api"
};
```

**2. Carga el script antes de la app en index.html:**

```html
<!-- index.html -->
<!doctype html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>ShopApp</title>
    <!-- Se carga antes de los módulos JS de la app -->
    <script src="/config.js"></script>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**3. Actualiza env.ts para leer de window primero:**

```typescript
// src/core/config/env.ts — versión con soporte runtime

declare global {
  interface Window {
    __RUNTIME_CONFIG__?: {
      apiBaseUrl?: string;
    };
  }
}

export const env = {
  // En runtime: lee de window.__RUNTIME_CONFIG__ si está disponible.
  // Fallback: lee de import.meta.env (build time).
  // Esto permite que la misma imagen funcione en múltiples entornos.
  apiBaseUrl:
    window.__RUNTIME_CONFIG__?.apiBaseUrl ??
    import.meta.env.VITE_API_BASE_URL ??
    'http://localhost:8000/api',
  isDevelopment: import.meta.env.DEV,
  isProduction: import.meta.env.PROD,
} as const;
```

**4. Entrypoint de Docker que genera config.js dinámicamente:**

```bash
#!/bin/sh
# docker-entrypoint.sh

# Genera /usr/share/nginx/html/config.js con las variables de entorno actuales del contenedor.
# Este script se ejecuta antes de iniciar Nginx.
cat > /usr/share/nginx/html/config.js << EOF
window.__RUNTIME_CONFIG__ = {
  apiBaseUrl: "${API_BASE_URL:-http://localhost:8000/api}"
};
EOF

# Inicia Nginx en primer plano (requerido por Docker)
exec nginx -g "daemon off;"
```

**5. Actualiza el Dockerfile para usar el entrypoint:**

```dockerfile
# Agregar al Dockerfile después de COPY --from=build:
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
```

**6. Uso con Docker:**

```bash
# Ahora puedes pasar la URL en runtime sin reconstruir la imagen
docker run -p 3000:80 -e API_BASE_URL=https://api.staging.com/api shopapp-react
docker run -p 3001:80 -e API_BASE_URL=https://api.produccion.com/api shopapp-react
```

**Cuándo usar cada enfoque:**

| Enfoque | Cuándo usarlo |
|---------|--------------|
| Build-time (`VITE_`) | Proyectos simples, un solo entorno de producción, CI/CD que construye una imagen por entorno |
| Runtime (`window.__RUNTIME_CONFIG__`) | Misma imagen en múltiples entornos, configuración que cambia sin reconstruir la imagen |

---

## 15.9 Configuración de CORS en el backend

Cuando el frontend pasa a producción en un dominio diferente al del backend, Django debe aceptar peticiones de ese dominio. Esto se configuró en `shopapi_01` con `django-cors-headers`.

Verifica en `settings/production.py` del backend:

```python
# shopapi/settings/production.py

CORS_ALLOWED_ORIGINS = [
    "https://midominio.com",
    "https://www.midominio.com",
    # En staging:
    "https://staging.midominio.com",
]

# Alternativa para desarrollo (nunca usar en producción):
# CORS_ALLOW_ALL_ORIGINS = True

# Si el frontend envía cookies de sesión (no es el caso con JWT en Authorization header,
# pero sí si usas cookies HttpOnly):
CORS_ALLOW_CREDENTIALS = True
```

**Por qué CORS es necesario:** el navegador implementa la política Same-Origin. Cuando el JavaScript de `https://midominio.com` hace una petición a `https://api.midominio.com`, el navegador primero envía una petición `OPTIONS` (preflight) y solo procede si el servidor responde con los encabezados CORS correctos. Si el backend no está configurado, la petición falla en el navegador aunque funcione con curl.

**JWT en el header Authorization no requiere `CORS_ALLOW_CREDENTIALS`**: las credenciales de CORS hacen referencia a cookies. Los tokens JWT en el header `Authorization: Bearer <token>` son tratados como una cabecera simple y no activan la restricción de credenciales.

---

## 15.10 Checklist final del tutorial

Esta lista cubre todos los módulos del tutorial ShopApp React Web. Úsala para verificar que el proyecto está completo antes de la entrega o demostración.

### Configuración base (Módulos 1–3)

- [ ] Proyecto Vite creado con TypeScript (`npm create vite@latest shopapp-react -- --template react-ts`)
- [ ] Tailwind CSS instalado y configurado con `@tailwindcss/vite`
- [ ] shadcn/ui inicializado (`npx shadcn@latest init`)
- [ ] Alias `@` apuntando a `src/` en `vite.config.ts` y `tsconfig.json`
- [ ] Clean Architecture implementada: `data/api/`, `domain/model/`, `domain/store/`, `presentation/pages/`, `presentation/components/`, `core/`
- [ ] React Router v6 configurado con rutas lazy y `Suspense`
- [ ] Axios client (`api.client.ts`) con interceptor de request (JWT) e interceptor de response (redirect 401)
- [ ] Autenticación completa: login, registro, logout, persistencia con `localStorage`
- [ ] `AuthGuard` protegiendo rutas privadas

### Catálogo (Módulo 4–6)

- [ ] `catalog.types.ts` con `Product`, `Category`, `PaginatedResponse`, `CatalogFilters`
- [ ] `catalog.api.ts` con `fetchProducts`, `fetchCategories`, `fetchProductById`
- [ ] `catalog.store.ts` (Zustand) con estado, acciones y selectors
- [ ] `CatalogPage` con grilla responsive, skeleton loading y estado vacío
- [ ] `ProductCard` con imagen, nombre, precio, badge de categoría y botón de carrito
- [ ] `ProductDetailPage` con imagen grande, descripción y botón de agregar al carrito
- [ ] Filtro por categoría funcional sin recarga de página
- [ ] Búsqueda con debounce
- [ ] Paginación prev/next

### Carrito y órdenes (Módulos 7–8)

- [ ] `cart.store.ts` con `addItem`, `removeItem`, `updateQuantity`, `clearCart`, persistencia
- [ ] `CartPage` con lista de items, subtotales y total
- [ ] `CheckoutPage` con formulario de dirección validado con Zod
- [ ] `orders.api.ts` con `createOrder`, `fetchOrders`, `fetchOrderById`
- [ ] `OrdersPage` con historial de órdenes
- [ ] `OrderDetailPage` con detalle de líneas e items

### Perfil de usuario (Módulo 9)

- [ ] `user.types.ts` con `UserProfile`
- [ ] `profile.api.ts` con `fetchProfile`, `updateProfile`
- [ ] `profile.store.ts` (Zustand)
- [ ] `ProfilePage` con formulario editable validado con Zod
- [ ] `UserAvatar` en el header con inicial o imagen

### Panel de administración (Módulos 10–13)

- [ ] Rutas admin protegidas con guard de `is_staff`
- [ ] `AdminDashboardPage` con métricas (productos, categorías, órdenes, usuarios)
- [ ] `AdminCategoriesPage` con CRUD completo (tabla, diálogo crear/editar, confirmación de borrado)
- [ ] `AdminProductsPage` con CRUD completo y paginación
- [ ] `AdminOrdersPage` con listado y cambio de estado de órdenes
- [ ] `AdminUsersPage` con listado y activación/desactivación de usuarios

### Imágenes (Módulo 14)

- [ ] `images.api.ts` con `uploadProductImage` y `uploadUserAvatar`
- [ ] `ImageUploader` component con vista previa, validación de tamaño y estados de carga
- [ ] Imagen de producto integrada en `AdminProductsPage` (ProductDialog)
- [ ] `ProductCard` muestra imagen con lazy loading y fallback `onError`
- [ ] `ProductDetailPage` muestra imagen grande con fallback
- [ ] Avatar circular en `ProfilePage` con actualización reactiva del store

### Deploy (Módulo 15)

- [ ] `.env`, `.env.production` y `.env.example` creados
- [ ] `.env` y `.env.production` en `.gitignore`
- [ ] `vite.config.ts` con `manualChunks` para vendors, `sourcemap: false`, `minify: 'esbuild'`
- [ ] `nginx.conf` con `try_files /index.html`, cache de assets y cabeceras de seguridad
- [ ] `Dockerfile` multi-etapa (node:20-alpine para build, nginx:alpine para serve)
- [ ] `docker-compose.yml` con servicio `web` (frontend) y referencia al backend
- [ ] `npm run build` ejecuta sin errores de TypeScript
- [ ] `docker build -t shopapp-react .` construye sin errores
- [ ] Navegación directa a `/catalog` devuelve la app (no 404 de Nginx)
- [ ] CORS configurado en el backend para el dominio de producción

---

## 15.11 Tabla resumen de los 15 módulos

| Módulo | Título | Duración | Archivos principales |
|--------|--------|----------|----------------------|
| 01 | Setup — Vite, TypeScript, Tailwind y Clean Architecture | 2–3 h | `vite.config.ts`, `tsconfig.json`, estructura de carpetas |
| 02 | Auth — Login, registro y JWT | 3–4 h | `auth.types.ts`, `auth.api.ts`, `auth.store.ts`, `LoginPage`, `RegisterPage` |
| 03 | Routing — React Router v6, guards y lazy loading | 2–3 h | `router.tsx`, `AuthGuard`, `AdminGuard`, layouts |
| 04 | Catálogo — Listado, categorías y CatalogStore | 3–4 h | `catalog.types.ts`, `catalog.api.ts`, `catalog.store.ts`, `CatalogPage`, `ProductCard` |
| 05 | Búsqueda y filtros — Debounce y filtrado por categoría | 2–3 h | `SearchBar`, `CategoryFilter`, `useDebounce` |
| 06 | Detalle de producto — ProductDetailPage y paginación | 2–3 h | `ProductDetailPage`, `Pagination` |
| 07 | Carrito — CartStore con persistencia | 3–4 h | `cart.types.ts`, `cart.store.ts`, `CartPage`, `CartDrawer` |
| 08 | Órdenes — Checkout y historial | 3–4 h | `order.types.ts`, `orders.api.ts`, `CheckoutPage`, `OrdersPage`, `OrderDetailPage` |
| 09 | Perfil — Edición de datos y UserAvatar | 2–3 h | `user.types.ts`, `profile.api.ts`, `profile.store.ts`, `ProfilePage`, `UserAvatar` |
| 10 | Admin Dashboard — Panel de métricas | 2–3 h | `admin.api.ts`, `AdminDashboardPage`, tarjetas de métricas |
| 11 | Admin Categorías — CRUD completo | 2–3 h | `AdminCategoriesPage`, `CategoryDialog`, `DeleteConfirmDialog` |
| 12 | Admin Productos — CRUD con filtros | 3–4 h | `AdminProductsPage`, `ProductDialog`, `ProductsTable` |
| 13 | Admin Órdenes y Usuarios | 3–4 h | `AdminOrdersPage`, `AdminUsersPage`, `OrderStatusBadge` |
| 14 | Imágenes — Subida de fotos y avatar | 3–4 h | `images.api.ts`, `ImageUploader`, integraciones en ProductCard y ProfilePage |
| 15 | Build y Deploy — Nginx y Docker | 2–3 h | `nginx.conf`, `Dockerfile`, `docker-compose.yml`, `vite.config.ts` optimizado |

**Total estimado:** 37–52 horas de desarrollo guiado.

### Arquitectura final implementada

```
src/
├── core/
│   ├── config/
│   │   └── env.ts                    ← Variables de entorno centralizadas
│   └── utils/
│       ├── cn.ts                     ← Utilidad clsx + tailwind-merge
│       └── format.ts                 ← Formateo de fechas y moneda
│
├── data/
│   └── api/
│       ├── api.client.ts             ← Instancia Axios con interceptors JWT
│       ├── auth.api.ts               ← Login, register, refresh, me
│       ├── catalog.api.ts            ← Products, categories
│       ├── orders.api.ts             ← Create, list, detail, update status
│       ├── profile.api.ts            ← Fetch y update perfil
│       ├── images.api.ts             ← Upload product image y avatar
│       └── admin.api.ts              ← Dashboard metrics, user management
│
├── domain/
│   ├── model/
│   │   ├── auth.types.ts             ← User, Tokens, LoginCredentials
│   │   ├── catalog.types.ts          ← Product, Category, PaginatedResponse
│   │   ├── order.types.ts            ← Order, OrderItem, OrderStatus
│   │   └── user.types.ts             ← UserProfile
│   └── store/
│       ├── auth.store.ts             ← Zustand: user, tokens, login, logout
│       ├── catalog.store.ts          ← Zustand: products, categories, filters
│       ├── cart.store.ts             ← Zustand: items, persist middleware
│       └── profile.store.ts          ← Zustand: profile, setProfile
│
└── presentation/
    ├── components/
    │   ├── ImageUploader.tsx          ← Subida de imágenes reutilizable
    │   ├── ProductCard.tsx            ← Tarjeta de producto con lazy image
    │   ├── UserAvatar.tsx             ← Avatar con inicial o foto
    │   ├── SearchBar.tsx              ← Búsqueda con debounce
    │   ├── CategoryFilter.tsx         ← Filtro de categorías
    │   ├── Pagination.tsx             ← Paginación prev/next
    │   └── OrderStatusBadge.tsx       ← Badge de estado de orden
    └── pages/
        ├── auth/
        │   ├── LoginPage.tsx
        │   └── RegisterPage.tsx
        ├── catalog/
        │   ├── CatalogPage.tsx
        │   └── ProductDetailPage.tsx
        ├── cart/
        │   └── CartPage.tsx
        ├── orders/
        │   ├── CheckoutPage.tsx
        │   ├── OrdersPage.tsx
        │   └── OrderDetailPage.tsx
        ├── profile/
        │   └── ProfilePage.tsx
        └── admin/
            ├── AdminDashboardPage.tsx
            ├── AdminCategoriesPage.tsx
            ├── AdminProductsPage.tsx
            ├── AdminOrdersPage.tsx
            └── AdminUsersPage.tsx
```
