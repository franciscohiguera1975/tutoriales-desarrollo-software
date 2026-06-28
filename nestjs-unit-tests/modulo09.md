# Tests Unitarios en NestJS — Página 9
## Módulo 9 · Integración con CI/CD (GitHub Actions)
### Ejecutar tests automáticamente antes de cada despliegue

---

## ¿Por qué ejecutar tests en CI/CD?

Ejecutar los tests localmente está bien, pero no es suficiente:
alguien puede olvidar correrlos o hacer un push desde otra máquina.
La solución es integrarlos en el pipeline de CI/CD para que se
ejecuten automáticamente en cada push, antes de que el código
llegue al servidor.

```
Sin CI/CD                           Con CI/CD + tests
─────────────────────────────────   ─────────────────────────────────
git push → deploy directo           git push → tests en Actions
Sin validación automática           Si tests fallan → deploy bloqueado
Un bug puede llegar a producción    Solo código verde llega al VPS
```

---

## Estructura del workflow original

El archivo `.github/workflows/deploy.yml` original tenía un único job
`deploy` que copiaba el código al VPS y lo levantaba con PM2.
El problema: si el código tenía bugs, igual se desplegaba.

---

## Separar en dos jobs: `test` y `deploy`

La estrategia correcta es dividir el workflow en dos jobs:

`test` — instala dependencias y ejecuta `npm test`

`deploy` — solo se ejecuta si `test` pasó gracias a `needs: test`

```
push → job "test"
           ↓ (pasa)
       job "deploy"
           ↓
       VPS actualizado

push → job "test"
           ↓ (falla)
       job "deploy" ← NO SE EJECUTA
       VPS sin cambios
```

---

## `.github/workflows/deploy.yml` actualizado

```yaml
name: 🚀 Deploy NestJS Blog Backend API to VPS

on:
  push:
    branches: [main]

jobs:
  test:
    name: 🧪 Unit Tests
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'

    - name: Install dependencies
      run: npm install

    - name: Run unit tests
      run: npm test -- --forceExit

  deploy:
    name: 🚀 Deploy to VPS
    needs: test
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Copy project to VPS
      uses: appleboy/scp-action@v0.1.3
      with:
        host: ${{ secrets.VPS_HOST }}
        username: root
        key: ${{ secrets.VPS_KEY }}
        source: "."
        target: "/opt/higuera-blog-backend"

    - name: Run deploy commands on VPS
      uses: appleboy/ssh-action@v1.0.3
      with:
        host: ${{ secrets.VPS_HOST }}
        username: root
        key: ${{ secrets.VPS_KEY }}
        script: |
          set -e
          cd /opt/higuera-blog-backend

          sudo lsof -t -i:3000 | xargs -r sudo kill -9 || true

          printf "%s" "${{ secrets.ENV_FILE }}" > .env

          npm install
          npm run build

          pm2 stop higuera-blog-backend || true
          pm2 delete higuera-blog-backend || true
          pm2 start dist/main.js --name higuera-blog-backend
          pm2 save
```

---

## Puntos clave del workflow

`needs: test` declara dependencia entre jobs. El job `deploy` solo
arranca si `test` terminó con exit code 0. Si algún test falla, Jest
devuelve exit code 1 y GitHub Actions cancela el deploy automáticamente.

`--forceExit` fuerza a Jest a terminar el proceso aunque haya handles
asíncronos abiertos (conexiones TypeORM, timers, etc.). Sin él el job
de CI puede quedarse colgado indefinidamente.

`actions/setup-node@v4` configura Node 20 en el runner de GitHub
Actions, garantizando compatibilidad con NestJS 11.

`moduleNameMapper` en `package.json` es necesario para que Jest
resuelva los imports con ruta `src/...` que usa el proyecto.

---

## `package.json` — configuración de Jest requerida

```json
"jest": {
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": "src",
  "testRegex": ".*\\.spec\\.ts$",
  "transform": { "^.+\\.(t|j)s$": "ts-jest" },
  "collectCoverageFrom": ["**/*.(t|j)s"],
  "coverageDirectory": "../coverage",
  "testEnvironment": "node",
  "moduleNameMapper": {
    "^src/(.*)$": "<rootDir>/$1"
  }
}
```

---

## Opcional: cobertura en CI

Para ver los porcentajes de cobertura en los logs de Actions, cambia
el comando de test:

```yaml
- name: Run unit tests with coverage
  run: npm run test:cov -- --forceExit
```

Para hacer fallar el build si la cobertura baja de cierto umbral,
añade `coverageThreshold` al bloque `jest` en `package.json`:

```json
"coverageThreshold": {
  "global": {
    "statements": 80,
    "branches":   75,
    "functions":  80,
    "lines":      80
  }
}
```

---

## Resumen del tutorial

| Página | Contenido                                              |
|--------|--------------------------------------------------------|
| 01     | Introducción: tipos de test, Jest config, AAA          |
| 02     | Mocks: jest.fn(), jest.mock(), overrideGuard, UUIDs    |
| 03     | Módulo Auth — AuthService y AuthController             |
| 04     | Módulo Users — UsersService y UsersController          |
| 05     | Módulo Categories — service y controller               |
| 06     | Módulo Posts — service y controller                    |
| 07     | Módulo Cursos — service y controller                   |
| 08     | Buenas prácticas y cobertura                           |
| 09     | CI/CD — GitHub Actions con test gate antes del deploy  |
