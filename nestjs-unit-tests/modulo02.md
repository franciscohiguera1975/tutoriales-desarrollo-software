# Tests Unitarios en NestJS — Página 2
## Módulo 2 · Mocks: tipos, patrones y configuración
### Todo lo que necesitas saber para aislar dependencias en los tests

---

## ¿Qué es un mock?

Un mock es un objeto falso que reemplaza a una dependencia real
(repositorio, servicio, librería) durante el test. Su propósito es
aislar la clase bajo prueba para que el test no dependa de la base
de datos, del sistema de archivos ni de ningún servicio externo.

```
Sin mock                             Con mock
─────────────────────────────────    ─────────────────────────────────
PostsService → Repository<Post>      PostsService → mockPostsRepository
Repository   → Base de datos         mockPostsRepository → jest.fn()
Test falla si la BD está caída       Test siempre predecible y rápido
```

---

## `jest.fn()` — la función mock básica

`jest.fn()` crea una función vacía que no hace nada por defecto,
pero que Jest puede observar: cuántas veces fue llamada, con qué
argumentos, y qué devolvió.

```typescript
const mockFn = jest.fn();
mockFn('hola');
expect(mockFn).toHaveBeenCalledWith('hola');
expect(mockFn).toHaveBeenCalledTimes(1);
```

---

## Configurar valores de retorno

Hay tres métodos principales para controlar lo que devuelve un mock:

`mockReturnValue(valor)` — devuelve un valor síncrono inmediatamente.

`mockResolvedValue(valor)` — devuelve una Promesa resuelta con el valor.
Equivale a `mockReturnValue(Promise.resolve(valor))`. Se usa para
funciones `async`.

`mockRejectedValue(error)` — devuelve una Promesa rechazada. Se usa
para simular errores.

```typescript
const mockSave = jest.fn();

mockSave.mockReturnValue({ id: '11111111-1111-1111-1111-111111111111' });
mockSave.mockResolvedValue({ id: '11111111-1111-1111-1111-111111111111' });
mockSave.mockRejectedValue(new Error('Duplicate key'));
```

---

## `mockReturnThis()` — mocks encadenables (QueryBuilder)

Los QueryBuilder de TypeORM encadenan métodos: `.where().orderBy()`.
Cada método debe devolver el propio objeto para que la cadena funcione.
`mockReturnThis()` hace exactamente eso.

```typescript
const mockQueryBuilder = {
  leftJoinAndSelect: jest.fn().mockReturnThis(),
  where:             jest.fn().mockReturnThis(),
  orderBy:           jest.fn().mockReturnThis(),
};
```

---

## Mockear un repositorio de TypeORM

Para inyectar un repositorio falso en el módulo de testing se usa
`getRepositoryToken(Entidad)` como token de inyección y un objeto
plano con las funciones que usa el servicio como valor.

```typescript
import { getRepositoryToken } from '@nestjs/typeorm';
import { Post } from './post.entity';

const mockPostsRepository = {
  create:             jest.fn(),
  save:               jest.fn(),
  findOne:            jest.fn(),
  delete:             jest.fn(),
  createQueryBuilder: jest.fn(),
};

// En el módulo de testing:
{ provide: getRepositoryToken(Post), useValue: mockPostsRepository }
```

Solo se definen los métodos que el servicio realmente usa.

---

## Mockear un servicio

Para mockear otro servicio (dependencia de un controlador o servicio)
se proporciona directamente la clase como token:

```typescript
const mockPostsService = {
  create:  jest.fn(),
  findAll: jest.fn(),
  findOne: jest.fn(),
  update:  jest.fn(),
  remove:  jest.fn(),
};

{ provide: PostsService, useValue: mockPostsService }
```

---

## `jest.mock()` — mockear módulos completos

Cuando la librería no permite usar `jest.spyOn` (por ejemplo bcrypt v6
define sus propiedades como no-configurables), se usa `jest.mock()`
al inicio del archivo. Jest lo eleva (hoisting) antes de los imports.

```typescript
// Debe ir ANTES de los imports
jest.mock('bcrypt', () => ({
  hash:    jest.fn(),
  compare: jest.fn(),
}));

import * as bcrypt from 'bcrypt';

// En los tests:
(bcrypt.hash as jest.Mock).mockResolvedValue('$2b$10$hash_simulado');
(bcrypt.compare as jest.Mock).mockResolvedValue(true);
```

El mismo patrón se aplica para `nestjs-typeorm-paginate`:

```typescript
jest.mock('nestjs-typeorm-paginate', () => ({
  paginate: jest.fn(),
}));

import { paginate } from 'nestjs-typeorm-paginate';
const mockPaginate = paginate as jest.Mock;

// En los tests:
mockPaginate.mockResolvedValue({ items: [], meta: {} });
```

---

## `overrideGuard()` — ignorar guards de NestJS

Cuando un controlador tiene `@UseGuards(JwtAuthGuard)`, el módulo de
testing fallará al compilar si no se configura el guard. La solución
es sobrescribirlo con un objeto que siempre devuelva `true`:

```typescript
const module = await Test.createTestingModule({
  controllers: [UsersController],
  providers: [{ provide: UsersService, useValue: mockUsersService }],
})
  .overrideGuard(JwtAuthGuard)
  .useValue({ canActivate: () => true })
  .compile();
```

---

## `jest.clearAllMocks()` en `beforeEach`

Llamar `jest.clearAllMocks()` al inicio de cada test borra el historial
de llamadas (`.mock.calls`) de todos los mocks, pero no resetea las
implementaciones definidas con `mockReturnValue`. Esto garantiza que
las aserciones como `toHaveBeenCalledTimes(1)` no acumulen llamadas
de tests anteriores.

```typescript
beforeEach(async () => {
  jest.clearAllMocks();
  // ...configurar el módulo
});
```

---

## `moduleNameMapper` — resolver imports con rutas absolutas

NestJS permite importar con rutas `src/...` que funcionan en
producción gracias a `tsconfig-paths`, pero Jest no las resuelve
por defecto. Para arreglarlo se añade `moduleNameMapper` en
`package.json`:

```json
"jest": {
  "rootDir": "src",
  "moduleNameMapper": {
    "^src/(.*)$": "<rootDir>/$1"
  }
}
```

Esto mapea `src/auth/guards/jwt-auth.guard` →
`<rootDir>/auth/guards/jwt-auth.guard` = `src/auth/guards/jwt-auth.guard`.

---

## UUIDs en los mocks

Los IDs en el proyecto son UUIDs generados por TypeORM. Los mocks
deben reflejar ese formato para ser realistas. Se declaran como
constantes al inicio del archivo de test:

```typescript
const POST_ID     = '11111111-1111-1111-1111-111111111111';
const USER_ID     = '33333333-3333-3333-3333-333333333333';
const CATEGORY_ID = '44444444-4444-4444-4444-444444444444';

// ID que simula un recurso que no existe:
const NOT_FOUND_ID = '99999999-9999-9999-9999-999999999999';
```

---

## Tabla de referencia rápida

| Patrón                              | Cuándo usarlo                                      |
|-------------------------------------|----------------------------------------------------|
| `jest.fn()`                         | Crear cualquier mock de función                    |
| `mockResolvedValue(x)`              | Mock de función async que resuelve con x           |
| `mockRejectedValue(err)`            | Mock que simula error / excepción                  |
| `mockReturnThis()`                  | Métodos encadenables (QueryBuilder)                |
| `jest.mock('módulo', factory)`      | Módulos con propiedades no-configurables           |
| `getRepositoryToken(Entity)`        | Inyectar repositorios TypeORM falsos               |
| `overrideGuard(Guard)`              | Bypass de guards JwtAuthGuard en controladores     |
| `jest.clearAllMocks()`              | Limpiar historial de llamadas entre tests          |
| `moduleNameMapper`                  | Resolver imports `src/...` en Jest                 |
