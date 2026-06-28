# Tests Unitarios en NestJS — Página 1
## Módulo 1 · Introducción y configuración
### ¿Qué son los tests unitarios en NestJS?, estructura y comandos esenciales

---

## ¿Por qué testear en NestJS?

En un proyecto como `higuera-blog-backend`, el código está distribuido
en servicios, controladores y módulos. Sin tests, cada cambio exige
verificar manualmente que el login sigue funcionando, que los posts
se crean bien, que los errores se manejan correctamente.

```
Sin tests                           Con tests unitarios
─────────────────────────────────   ─────────────────────────────────
Cambias AuthService.login()         Cambias AuthService.login()
Ejecutas el servidor                Ejecutas: npm test
Abres Postman                       Jest verifica todos los casos
Pruebas caso a caso                 en segundos — sin servidor
```

Los **tests unitarios** verifican una clase (servicio, controlador)
de forma aislada, sin base de datos real ni servidor HTTP.
Las dependencias externas se reemplazan por **mocks** — objetos
falsos que simulan el comportamiento necesario.

---

## Diferencia entre tipos de test

| Tipo | Qué prueba | Base de datos | Velocidad |
|---|---|---|---|
| **Unitario** | Una clase aislada | No (mock) | Muy rápido |
| **Integración** | Módulo completo | Puede usarla | Medio |
| **E2E** | Flujo HTTP completo | Sí | Lento |

Este tutorial se enfoca en **tests unitarios** — los más rápidos,
los más fáciles de mantener y los más útiles para detectar errores
de lógica.

---

## Configuración del proyecto

El proyecto ya tiene Jest configurado. Revisa el `package.json`:

```json
// package.json (fragmento)
"scripts": {
  "test":       "jest",
  "test:watch": "jest --watch",
  "test:cov":   "jest --coverage"
},
"jest": {
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": "src",
  "testRegex": ".*\\.spec\\.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  },
  "testEnvironment": "node"
}
```

Puntos clave:
- Jest busca archivos que terminen en **`.spec.ts`** dentro de `src/`
- Usa **`ts-jest`** para compilar TypeScript en tiempo de test
- No necesita servidor ni base de datos corriendo

### Comandos disponibles

```bash
# Ejecuta todos los tests una vez
npm test

# Re-ejecuta solo los tests afectados al guardar un archivo
npm run test:watch

# Genera reporte de cobertura en /coverage
npm run test:cov
```

---

## Estructura de archivos `.spec.ts`

NestJS genera archivos de test automáticamente al crear un módulo.
En este proyecto ya existen:

```
src/
├── posts/
│   ├── posts.service.ts
│   ├── posts.service.spec.ts     ← test del servicio
│   ├── posts.controller.ts
│   └── posts.controller.spec.ts  ← test del controlador
├── auth/
│   ├── auth.service.ts
│   └── auth.service.spec.ts
└── users/
    ├── users.service.ts
    └── users.service.spec.ts
```

> **Convención:** el archivo de test vive junto al código que prueba,
> con el mismo nombre y extensión `.spec.ts`. Así es fácil encontrar
> qué tests corresponden a cada archivo.

---

## Parte 1 — Anatomía de un test en NestJS

Abre `src/posts/posts.service.spec.ts`. Este es el archivo generado
automáticamente por NestJS:

```typescript
// src/posts/posts.service.spec.ts (generado por NestJS)

import { Test, TestingModule } from '@nestjs/testing';
import { PostsService } from './posts.service';

describe('PostsService', () => {
  let service: PostsService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [PostsService],
    }).compile();

    service = module.get<PostsService>(PostsService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

Este test **todavía no funciona** porque `PostsService` necesita
dos repositorios de TypeORM que no están disponibles. Lo veremos
en el módulo 2. Por ahora, analicemos su estructura:

```
describe('PostsService', () => {
    ↑ Agrupa todos los tests de este servicio

  beforeEach(async () => { ... });
    ↑ Crea una instancia fresca del módulo antes de cada test

  it('should be defined', () => { ... });
    ↑ Un test individual — "it" y "test" son sinónimos en Jest
});
```

### Las tres piezas del test de NestJS

```typescript
// 1. CREACIÓN DEL MÓDULO DE TEST
//    Simula el módulo de NestJS pero sin servidor ni base de datos
const module: TestingModule = await Test.createTestingModule({
  providers: [PostsService],
}).compile();

// 2. OBTENCIÓN DE LA INSTANCIA
//    Extrae la instancia del servicio que creó NestJS
service = module.get<PostsService>(PostsService);

// 3. VERIFICACIÓN
//    Confirma que la instancia existe y está inicializada
expect(service).toBeDefined();
```

---

## Parte 2 — `describe` e `it` en NestJS

`describe` agrupa los tests de una clase. `it` describe
un comportamiento específico. Juntos forman una oración legible:

```typescript
describe('PostsService', () => {
  it('should be defined', () => { ... });
  //   ↑ "PostsService should be defined"

  it('should return null when category does not exist', () => { ... });
  //   ↑ "PostsService should return null when category does not exist"

  it('should create a post successfully', () => { ... });
  //   ↑ "PostsService should create a post successfully"
});
```

Cuando ejecutas `npm test`, la salida refleja esta estructura:

```
PostsService
  ✓ should be defined
  ✓ should return null when category does not exist
  ✓ should create a post successfully

Test Suites: 1 passed
Tests:       3 passed
```

### `describe` anidados para organizar mejor

Cuando un servicio tiene muchos métodos, puedes anidar `describe`:

```typescript
describe('PostsService', () => {

  describe('create()', () => {
    it('should create a post when category exists', () => { ... });
    it('should return null when category does not exist', () => { ... });
  });

  describe('findOne()', () => {
    it('should return a post by id', () => { ... });
    it('should return null when post does not exist', () => { ... });
  });

  describe('remove()', () => {
    it('should return true when post is deleted', () => { ... });
    it('should return false when post does not exist', () => { ... });
  });

});
```

Salida en consola:

```
PostsService
  create()
    ✓ should create a post when category exists
    ✓ should return null when category does not exist
  findOne()
    ✓ should return a post by id
    ✓ should return null when post does not exist
  remove()
    ✓ should return true when post is deleted
    ✓ should return false when post does not exist
```

---

## Parte 3 — El patrón AAA en NestJS

El mismo patrón Arrange / Act / Assert se aplica en NestJS,
pero las etapas tienen contenido específico:

```typescript
it('should return null when category does not exist', async () => {

  // ── ARRANGE (preparar) ───────────────────────────────────────
  // Configuramos el mock para simular que la categoría no existe
  mockCategoriesRepository.findOne.mockResolvedValue(null);

  const dto = { title: 'Mi post', content: 'Contenido', categoryId: '99' };

  // ── ACT (actuar) ─────────────────────────────────────────────
  // Llamamos al método real del servicio
  const result = await service.create(dto);

  // ── ASSERT (verificar) ───────────────────────────────────────
  // Verificamos el resultado esperado
  expect(result).toBeNull();
});
```

> **Nota:** En NestJS la mayoría de los métodos son `async`,
> por lo tanto el callback del test debe ser `async`
> y usamos `await` al llamar al servicio.

---

## Parte 4 — Ejecutar los tests del proyecto

```bash
# Desde la raíz del proyecto
cd higuera-blog-backend

# Ejecutar todos los tests
npm test
```

En este momento, los tests generados automáticamente fallará
porque los servicios necesitan sus dependencias (repositorios).
En el módulo 2 aprenderemos a reemplazarlas con mocks.

```
FAIL  src/posts/posts.service.spec.ts
  ● Test suite failed to run
    Nest can't resolve dependencies of the PostsService (?).
    Please make sure that the argument Repository<Post> at
    index [0] is available in the RootTestModule context.
```

Este error es esperado y tiene solución. El módulo 2 lo resuelve.

---

## Resumen del módulo 1

- Los **tests unitarios** verifican clases aisladas — sin servidor, sin base de datos real.
- NestJS usa **Jest** + **ts-jest** configurados en `package.json`. Los tests se ejecutan con `npm test`.
- Los archivos de test tienen extensión `.spec.ts` y viven junto al código que prueban.
- `Test.createTestingModule()` crea un módulo de prueba que simula el sistema de inyección de dependencias de NestJS.
- `describe` agrupa los tests de una clase. `it` describe un comportamiento. Juntos forman una oración legible.
- Los tests son `async` porque los servicios de NestJS trabajan con Promises.
- El patrón **AAA** (Arrange, Act, Assert) sigue siendo la guía: preparar mocks, llamar al método, verificar el resultado.

---

> **Siguiente página →** Módulo 2: Mocks y el Testing Module —
> cómo reemplazar repositorios de TypeORM con objetos falsos
> y escribir los primeros tests reales de `PostsService`.
