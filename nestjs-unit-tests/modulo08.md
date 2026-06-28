# Tests Unitarios en NestJS — Página 8
## Módulo 8 · Buenas prácticas y cobertura
### Cómo medir y mejorar la calidad de los tests

---

## Estructura recomendada de cada spec

Seguir una estructura consistente hace los tests más legibles y
predecibles. El patrón usado en este proyecto:

```
describe('NombreClase', () => {           ← suite principal
  describe('nombreMetodo()', () => {      ← una suite por método
    it('should ... when ...', ...)        ← caso feliz
    it('should ... when ...', ...)        ← caso límite
    it('should ... when ...', ...)        ← caso de error
  });
});
```

Apuntar a **al menos 3 tests por método**: caso feliz, caso límite
(no encontrado, vacío) y caso de error (servicio lanza o devuelve
null).

---

## Nomenclatura de tests

El nombre del test debe describir comportamiento, no implementación.

```
// Mal: describe qué hace el código internamente
it('should call repository.save')

// Bien: describe qué devuelve o qué efecto produce
it('should return the created category')
it('should return null when repository throws')
it('should throw NotFoundException when category does not exist')
```

El patrón `should [resultado] when [condición]` hace que el reporte
de Jest sea legible sin abrir el código.

---

## Datos de prueba con UUID reales

Las entidades del proyecto usan `uuid` como tipo de columna. Usar
IDs con formato UUID en los mocks garantiza que los tests son
representativos del comportamiento real.

```typescript
// Mal: IDs que nunca existirían en la base de datos
const id = '1';
const id = '42';
const id = 'no-existe';

// Bien: constantes con nombre y formato UUID válido
const CATEGORY_ID   = '44444444-4444-4444-4444-444444444444';
const CATEGORY_ID_2 = '55555555-5555-5555-5555-555555555555';
const NOT_FOUND_ID  = '99999999-9999-9999-9999-999999999999';
```

Declarar las constantes fuera del `describe` para reutilizarlas en
toda la suite.

---

## Restaurar mocks en beforeEach

`jest.clearAllMocks()` borra el historial de llamadas (`mock.calls`,
`mock.results`) pero no restaura las implementaciones definidas con
`mockReturnThis()`. Los mocks de queryBuilder deben re-aplicarse
después de `clearAllMocks()`.

```typescript
beforeEach(async () => {
  jest.clearAllMocks();

  // Restaurar implementaciones del queryBuilder
  mockQueryBuilder.where.mockReturnThis();
  mockQueryBuilder.orderBy.mockReturnThis();

  // Reconectar el queryBuilder al repositorio
  mockRepository.createQueryBuilder.mockReturnValue(mockQueryBuilder);

  // ... crear el módulo de testing
});
```

---

## Ejecutar cobertura

```bash
npm run test:cov
```

Jest genera un reporte en consola y una carpeta `coverage/` con
un informe HTML detallado.

```
------------------------|---------|----------|---------|---------|
File                    | % Stmts | % Branch | % Funcs | % Lines |
------------------------|---------|----------|---------|---------|
categories.service.ts   |   92.30 |    83.33 |  100.00 |   91.66 |
categories.controller.ts|   95.65 |    66.66 |  100.00 |   95.23 |
------------------------|---------|----------|---------|---------|
```

---

## Significado de cada métrica

**Statements (sentencias):** porcentaje de líneas de código ejecutadas
al menos una vez. Es la métrica más general.

**Branches (ramas):** porcentaje de ramas de decisión cubiertas. Cada
`if / else`, operador ternario y `&&` introduce dos ramas. Un test
que solo prueba el caso feliz deja las ramas de error sin cubrir.

**Functions (funciones):** porcentaje de funciones llamadas al menos
una vez. Si un método existe pero ningún test lo llama, baja esta
métrica.

**Lines (líneas):** similar a statements pero cuenta líneas físicas
en lugar de nodos del AST. Suele coincidir con statements.

---

## Configurar umbrales mínimos

Para hacer fallar el build si la cobertura baja de un nivel aceptable,
añadir `coverageThreshold` al bloque `jest` en `package.json`:

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

Con esta configuración, `npm run test:cov` devuelve exit code 1 si
alguna métrica cae por debajo del umbral, lo que permite usarlo como
gate en CI/CD igual que `npm test`.

---

## Abrir el informe HTML

```bash
# Linux / macOS
xdg-open coverage/lcov-report/index.html
open coverage/lcov-report/index.html
```

El informe muestra cada archivo con las líneas cubiertas (verde),
no cubiertas (rojo) y parcialmente cubiertas (amarillo). Sirve para
identificar exactamente qué ramas faltan por testear.

---

## Checklist de calidad

Antes de considerar un módulo completo:

- [ ] Existe `NombreClase.spec.ts` con `describe('NombreClase', ...)`
- [ ] Cada método público tiene su propio `describe`
- [ ] Al menos 3 tests por método (feliz, límite, error)
- [ ] Los IDs usan formato UUID
- [ ] `jest.clearAllMocks()` en `beforeEach`
- [ ] Los mocks de queryBuilder se restauran en `beforeEach`
- [ ] `npm test` pasa con 0 fallos
- [ ] `npm run test:cov` supera los umbrales configurados

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
