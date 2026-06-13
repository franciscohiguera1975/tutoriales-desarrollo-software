# Tutorial Jest — Página 3
## Módulo 3 · Organización de tests
### `describe`, `beforeEach`, `afterEach`, `beforeAll` y `afterAll`

---

## El problema de los tests sin organización

Cuando un archivo de tests crece, sin organización se convierte en una lista
interminable de tests sin contexto claro:

```
✓ devuelve 0 cuando el carrito está vacío
✓ añade un producto correctamente
✓ lanza error si el email está vacío
✓ calcula el total con descuento
✓ lanza error si el email no tiene arroba
✓ elimina un producto por id
```

¿Qué tests son del carrito? ¿Cuáles son del email?
Con `describe` los agrupamos y el reporte queda claro:

```
Carrito de compras
  ✓ devuelve 0 cuando el carrito está vacío
  ✓ añade un producto correctamente
  ✓ calcula el total con descuento
  ✓ elimina un producto por id

Validación de email
  ✓ lanza error si el email está vacío
  ✓ lanza error si el email no tiene arroba
```

---

## `describe` — agrupar tests relacionados

`describe` crea un bloque que agrupa tests que prueban el mismo concepto.
Puede anidarse para crear jerarquías más detalladas.

```javascript
// src/usuario.js
function crearUsuario(nombre, email, edad) {
  if (!nombre || nombre.trim() === '') throw new Error('El nombre es obligatorio');
  if (!email || !email.includes('@'))  throw new Error('El email no es válido');
  if (edad < 0 || edad > 120)          throw new Error('La edad no es válida');

  return {
    id:       Math.floor(Math.random() * 10000),
    nombre:   nombre.trim(),
    email:    email.toLowerCase(),
    edad,
    activo:   true,
    creadoEn: new Date().toISOString(),
  };
}

function desactivarUsuario(usuario) {
  return { ...usuario, activo: false };
}

function cambiarEmail(usuario, nuevoEmail) {
  if (!nuevoEmail.includes('@')) throw new Error('Email inválido');
  return { ...usuario, email: nuevoEmail.toLowerCase() };
}

module.exports = { crearUsuario, desactivarUsuario, cambiarEmail };
```

```javascript
// tests/usuario.test.js
const { crearUsuario, desactivarUsuario, cambiarEmail } = require('../src/usuario');

// describe() agrupa tests relacionados bajo una descripción común
describe('crearUsuario', () => {

  test('crea un usuario con los datos correctos', () => {
    const usuario = crearUsuario('Ana García', 'ana@ejemplo.com', 28);

    expect(usuario).toHaveProperty('nombre', 'Ana García');
    expect(usuario).toHaveProperty('email', 'ana@ejemplo.com');
    expect(usuario).toHaveProperty('edad', 28);
    expect(usuario).toHaveProperty('activo', true);
  });

  test('convierte el email a minúsculas', () => {
    const usuario = crearUsuario('Luis', 'LUIS@EJEMPLO.COM', 35);
    expect(usuario.email).toBe('luis@ejemplo.com');
  });

  test('elimina espacios del nombre', () => {
    const usuario = crearUsuario('  Ana  ', 'ana@ejemplo.com', 28);
    expect(usuario.nombre).toBe('Ana');
  });

  // Describe anidado para los casos de error
  describe('cuando los datos son inválidos', () => {
    test('lanza error si el nombre está vacío', () => {
      expect(() => crearUsuario('', 'ana@ejemplo.com', 28))
        .toThrow('El nombre es obligatorio');
    });

    test('lanza error si el nombre es solo espacios', () => {
      expect(() => crearUsuario('   ', 'ana@ejemplo.com', 28))
        .toThrow('El nombre es obligatorio');
    });

    test('lanza error si el email no tiene arroba', () => {
      expect(() => crearUsuario('Ana', 'sinArroba.com', 28))
        .toThrow('El email no es válido');
    });

    test('lanza error si la edad es negativa', () => {
      expect(() => crearUsuario('Ana', 'ana@ejemplo.com', -1))
        .toThrow('La edad no es válida');
    });

    test('lanza error si la edad supera 120', () => {
      expect(() => crearUsuario('Ana', 'ana@ejemplo.com', 150))
        .toThrow('La edad no es válida');
    });
  });
});

describe('desactivarUsuario', () => {
  test('marca al usuario como inactivo', () => {
    const usuario = crearUsuario('Luis', 'luis@ejemplo.com', 30);
    const inactivo = desactivarUsuario(usuario);

    expect(inactivo.activo).toBe(false);
  });

  test('no modifica el usuario original', () => {
    const usuario = crearUsuario('Luis', 'luis@ejemplo.com', 30);
    desactivarUsuario(usuario);

    // El original debe seguir activo
    expect(usuario.activo).toBe(true);
  });
});

describe('cambiarEmail', () => {
  test('actualiza el email correctamente', () => {
    const usuario = crearUsuario('Ana', 'ana@viejo.com', 28);
    const actualizado = cambiarEmail(usuario, 'ANA@NUEVO.COM');

    expect(actualizado.email).toBe('ana@nuevo.com');
  });

  test('lanza error si el nuevo email es inválido', () => {
    const usuario = crearUsuario('Ana', 'ana@viejo.com', 28);
    expect(() => cambiarEmail(usuario, 'sinArroba')).toThrow('Email inválido');
  });
});
```

Salida:

```
 PASS  tests/usuario.test.js
  crearUsuario
    ✓ crea un usuario con los datos correctos (3 ms)
    ✓ convierte el email a minúsculas (1 ms)
    ✓ elimina espacios del nombre (1 ms)
    cuando los datos son inválidos
      ✓ lanza error si el nombre está vacío (1 ms)
      ✓ lanza error si el nombre es solo espacios (1 ms)
      ✓ lanza error si el email no tiene arroba (1 ms)
      ✓ lanza error si la edad es negativa (1 ms)
      ✓ lanza error si la edad supera 120 (1 ms)
  desactivarUsuario
    ✓ marca al usuario como inactivo (1 ms)
    ✓ no modifica el usuario original (1 ms)
  cambiarEmail
    ✓ actualiza el email correctamente (1 ms)
    ✓ lanza error si el nuevo email es inválido (1 ms)
```

---

## `beforeEach` — preparar el estado antes de cada test

`beforeEach` ejecuta una función **antes de cada test** dentro del bloque
`describe`. Evita repetir el mismo código de preparación en cada test.

```javascript
// src/carrito.js
class Carrito {
  constructor() {
    this.items = [];
  }

  agregar(producto) {
    const existente = this.items.find(i => i.id === producto.id);
    if (existente) {
      existente.cantidad += producto.cantidad;
    } else {
      this.items.push({ ...producto });
    }
  }

  eliminar(id) {
    const indice = this.items.findIndex(i => i.id === id);
    if (indice === -1) throw new Error(`Producto con id ${id} no encontrado`);
    this.items.splice(indice, 1);
  }

  calcularTotal() {
    return this.items.reduce((total, item) => {
      return total + item.precio * item.cantidad;
    }, 0);
  }

  limpiar() {
    this.items = [];
  }

  get cantidadTotal() {
    return this.items.reduce((sum, item) => sum + item.cantidad, 0);
  }
}

module.exports = { Carrito };
```

```javascript
// tests/carrito.test.js
const { Carrito } = require('../src/carrito');

describe('Carrito de compras', () => {
  let carrito;  // variable compartida entre los tests

  // beforeEach se ejecuta ANTES de cada test
  // Crea un carrito fresco para cada test — sin contaminación entre tests
  beforeEach(() => {
    carrito = new Carrito();
  });

  test('empieza vacío', () => {
    expect(carrito.items).toHaveLength(0);
    expect(carrito.calcularTotal()).toBe(0);
  });

  test('agregar un producto lo añade a items', () => {
    carrito.agregar({ id: 1, nombre: 'Ratón', precio: 29.99, cantidad: 1 });

    expect(carrito.items).toHaveLength(1);
    expect(carrito.items[0].nombre).toBe('Ratón');
  });

  test('agregar el mismo producto incrementa su cantidad', () => {
    carrito.agregar({ id: 1, nombre: 'Ratón', precio: 29.99, cantidad: 1 });
    carrito.agregar({ id: 1, nombre: 'Ratón', precio: 29.99, cantidad: 2 });

    expect(carrito.items).toHaveLength(1);           // sigue siendo 1 producto
    expect(carrito.items[0].cantidad).toBe(3);        // pero con cantidad 3
  });

  test('calcularTotal suma precio × cantidad de todos los items', () => {
    carrito.agregar({ id: 1, nombre: 'Ratón',    precio: 29.99, cantidad: 2 });
    carrito.agregar({ id: 2, nombre: 'Teclado',  precio: 89.99, cantidad: 1 });

    // 29.99*2 + 89.99*1 = 59.98 + 89.99 = 149.97
    expect(carrito.calcularTotal()).toBeCloseTo(149.97, 2);
  });

  test('eliminar un producto lo quita de items', () => {
    carrito.agregar({ id: 1, nombre: 'Ratón',   precio: 29.99, cantidad: 1 });
    carrito.agregar({ id: 2, nombre: 'Teclado', precio: 89.99, cantidad: 1 });

    carrito.eliminar(1);

    expect(carrito.items).toHaveLength(1);
    expect(carrito.items[0].id).toBe(2);
  });

  test('eliminar un id inexistente lanza error', () => {
    expect(() => carrito.eliminar(99)).toThrow('Producto con id 99 no encontrado');
  });

  test('cantidadTotal suma todas las cantidades', () => {
    carrito.agregar({ id: 1, nombre: 'Ratón',   precio: 29.99, cantidad: 3 });
    carrito.agregar({ id: 2, nombre: 'Teclado', precio: 89.99, cantidad: 2 });

    expect(carrito.cantidadTotal).toBe(5);
  });

  // Cada test empieza con el carrito vacío gracias a beforeEach
  // Este test NO está afectado por los productos añadidos en los tests anteriores
  test('el carrito sigue vacío al inicio de este test', () => {
    expect(carrito.items).toHaveLength(0);
  });
});
```

---

## `afterEach` — limpiar después de cada test

`afterEach` ejecuta código **después de cada test**. Útil para limpiar
recursos, resetear estados globales o liberar conexiones.

```javascript
describe('con afterEach', () => {
  let registros = [];  // simula un log global

  afterEach(() => {
    // Se ejecuta después de cada test
    // Limpia el array para no contaminar el siguiente test
    registros = [];
    // console.log('Test terminado — estado limpiado');
  });

  test('añade entradas al registro', () => {
    registros.push('entrada 1');
    registros.push('entrada 2');
    expect(registros).toHaveLength(2);
  });

  test('el registro está vacío al inicio de este test', () => {
    // afterEach del test anterior limpió registros
    expect(registros).toHaveLength(0);
  });
});
```

---

## `beforeAll` y `afterAll` — una sola vez por bloque

A diferencia de `beforeEach/afterEach` que se ejecutan en cada test,
`beforeAll` y `afterAll` se ejecutan **una sola vez** por bloque `describe`.

```
beforeAll     ← una vez al inicio del bloque
  beforeEach  ← antes de CADA test
    test 1
  afterEach   ← después de CADA test
  beforeEach
    test 2
  afterEach
  beforeEach
    test 3
  afterEach
afterAll      ← una vez al final del bloque
```

### Cuándo usar cada uno

```javascript
describe('ejemplo con todos los hooks', () => {

  let conexion;    // recurso costoso — solo se crea una vez
  let usuario;     // estado fresco — se recrea en cada test

  // beforeAll — recurso que tarda en crearse (BD, cliente HTTP, etc.)
  beforeAll(() => {
    // Simula abrir una conexión costosa
    conexion = { host: 'localhost', puerto: 5432, activa: true };
    console.log('Conexión establecida');
  });

  // afterAll — cierra el recurso al terminar todos los tests
  afterAll(() => {
    conexion.activa = false;
    console.log('Conexión cerrada');
  });

  // beforeEach — estado que debe ser fresco en cada test
  beforeEach(() => {
    usuario = { id: 1, nombre: 'Ana', activo: true };
  });

  // afterEach — limpia efectos secundarios de cada test
  afterEach(() => {
    usuario = null;
  });

  test('la conexión está activa', () => {
    expect(conexion.activa).toBe(true);
  });

  test('el usuario empieza activo', () => {
    expect(usuario.activo).toBe(true);
    usuario.activo = false;  // modifica el usuario en este test
  });

  test('el usuario está activo de nuevo (beforeEach lo resetea)', () => {
    expect(usuario.activo).toBe(true);  // nuevo objeto gracias a beforeEach
  });
});
```

---

## Ejemplo aplicado — sistema de inventario

```javascript
// src/inventario.js
class Inventario {
  constructor() {
    this.productos = new Map();
  }

  agregar(id, nombre, cantidad) {
    if (cantidad < 0) throw new Error('La cantidad no puede ser negativa');
    this.productos.set(id, { id, nombre, cantidad });
  }

  retirar(id, cantidad) {
    const producto = this.productos.get(id);
    if (!producto)             throw new Error(`Producto ${id} no existe`);
    if (producto.cantidad < cantidad) throw new Error('Stock insuficiente');
    producto.cantidad -= cantidad;
    return producto;
  }

  existencias(id) {
    return this.productos.get(id)?.cantidad ?? 0;
  }

  totalProductos() {
    return this.productos.size;
  }
}

module.exports = { Inventario };
```

```javascript
// tests/inventario.test.js
const { Inventario } = require('../src/inventario');

describe('Inventario', () => {
  let inv;

  // Estado base antes de cada test: inventario con 3 productos
  beforeEach(() => {
    inv = new Inventario();
    inv.agregar('A001', 'Teclado', 50);
    inv.agregar('A002', 'Ratón',   30);
    inv.agregar('A003', 'Monitor', 10);
  });

  describe('agregar', () => {
    test('añade un producto nuevo correctamente', () => {
      inv.agregar('A004', 'Auriculares', 20);
      expect(inv.totalProductos()).toBe(4);
      expect(inv.existencias('A004')).toBe(20);
    });

    test('cantidad negativa lanza error', () => {
      expect(() => inv.agregar('A005', 'Cable', -5))
        .toThrow('La cantidad no puede ser negativa');
    });
  });

  describe('retirar', () => {
    test('descuenta la cantidad correctamente', () => {
      inv.retirar('A001', 10);
      expect(inv.existencias('A001')).toBe(40);
    });

    test('retirar todo deja en cero', () => {
      inv.retirar('A003', 10);
      expect(inv.existencias('A003')).toBe(0);
    });

    test('producto inexistente lanza error', () => {
      expect(() => inv.retirar('X999', 1)).toThrow('Producto X999 no existe');
    });

    test('stock insuficiente lanza error', () => {
      expect(() => inv.retirar('A002', 100)).toThrow('Stock insuficiente');
    });
  });

  describe('existencias', () => {
    test('devuelve la cantidad de un producto existente', () => {
      expect(inv.existencias('A001')).toBe(50);
      expect(inv.existencias('A002')).toBe(30);
    });

    test('devuelve 0 para un producto que no existe', () => {
      expect(inv.existencias('XXXXX')).toBe(0);
    });
  });
});
```

---

## Hooks en describes anidados — orden de ejecución

```javascript
describe('externo', () => {
  beforeAll(() => console.log('beforeAll EXTERNO'));
  afterAll(() => console.log('afterAll EXTERNO'));
  beforeEach(() => console.log('beforeEach EXTERNO'));
  afterEach(() => console.log('afterEach EXTERNO'));

  test('test externo', () => {
    console.log('→ test externo');
  });

  describe('interno', () => {
    beforeAll(() => console.log('  beforeAll INTERNO'));
    afterAll(() => console.log('  afterAll INTERNO'));
    beforeEach(() => console.log('  beforeEach INTERNO'));
    afterEach(() => console.log('  afterEach INTERNO'));

    test('test interno', () => {
      console.log('  → test interno');
    });
  });
});
```

Salida (orden de ejecución):

```
beforeAll EXTERNO
  beforeEach EXTERNO          ← para "test externo"
  → test externo
  afterEach EXTERNO
  beforeAll INTERNO
    beforeEach EXTERNO        ← para "test interno"
    beforeEach INTERNO
    → test interno
    afterEach INTERNO
    afterEach EXTERNO
  afterAll INTERNO
afterAll EXTERNO
```

> El `beforeEach` del describe externo se ejecuta también para los tests
> del describe interno — los hooks se heredan hacia adentro.

---

## Regla de oro: independencia entre tests

Cada test debe poder ejecutarse solo, en cualquier orden, y producir
el mismo resultado. Si un test depende del resultado de otro, hay un
problema de diseño.

```javascript
// MAL — los tests dependen entre sí
describe('MAL ejemplo', () => {
  let carrito;

  test('1: inicializa el carrito', () => {
    carrito = new Carrito();         // si este test no corre, los demás fallan
    expect(carrito.items).toHaveLength(0);
  });

  test('2: añade un producto', () => {
    carrito.agregar({ id: 1, nombre: 'Ratón', precio: 29.99, cantidad: 1 }); // depende del test 1
    expect(carrito.items).toHaveLength(1);
  });
});

// BIEN — cada test es independiente gracias a beforeEach
describe('BIEN ejemplo', () => {
  let carrito;

  beforeEach(() => {
    carrito = new Carrito();  // cada test tiene su propio carrito
  });

  test('empieza vacío', () => {
    expect(carrito.items).toHaveLength(0);
  });

  test('añade un producto', () => {
    carrito.agregar({ id: 1, nombre: 'Ratón', precio: 29.99, cantidad: 1 });
    expect(carrito.items).toHaveLength(1);
  });
});
```

---

## Ejercicios propuestos

1. **Biblioteca de libros** — Crea una clase `Biblioteca` con métodos
   `registrar(libro)`, `prestar(isbn)` y `devolver(isbn)`. Organiza los
   tests en `describe` anidados: uno para `registrar`, uno para `prestar`
   y uno para `devolver`. Usa `beforeEach` para crear una biblioteca
   con 3 libros ya registrados antes de cada test.

2. **Orden de hooks** — Sin ejecutar el código, predice el orden exacto
   de los `console.log` para este bloque de tests con dos describes anidados
   y dos tests en cada nivel. Luego ejecútalo para verificar tu predicción.

3. **Cuenta bancaria** — Crea `src/cuenta.js` con una clase `Cuenta` que
   tenga `depositar(monto)`, `retirar(monto)` y `saldo`. Usa `beforeAll`
   para crear la cuenta una sola vez y `beforeEach` para depositar 1000
   antes de cada test. Verifica que `retirar` más del saldo disponible
   lanza un error, y que el saldo es siempre 1000 al inicio de cada test.

4. **`describe.skip` y `describe.only`** — Igual que `test.skip` y `test.only`,
   `describe` también acepta `.skip` y `.only`. Prueba su comportamiento:
   añade `describe.skip('bloque omitido', ...)` a uno de tus archivos de
   test y verifica en la salida de Jest que todos los tests de ese bloque
   aparecen como "skipped".

---

## Resumen de la página 3

- `describe('nombre', () => { ... })` agrupa tests relacionados bajo una descripción. Puede anidarse para crear jerarquías legibles.
- `beforeEach` se ejecuta antes de **cada test** — ideal para crear un estado fresco (nuevo objeto, array vacío) que evite contaminación entre tests.
- `afterEach` se ejecuta después de **cada test** — ideal para limpiar efectos secundarios (logs, timers, conexiones).
- `beforeAll` se ejecuta **una sola vez** al inicio del bloque — ideal para recursos costosos de crear (conexión a BD, carga de datos grandes).
- `afterAll` se ejecuta **una sola vez** al final del bloque — ideal para cerrar recursos abiertos en `beforeAll`.
- Los hooks del `describe` externo se heredan a los `describe` internos — el orden es: `beforeEach externo → beforeEach interno → test → afterEach interno → afterEach externo`.
- **Regla de oro:** cada test debe ser independiente. Si el orden de ejecución importa, hay un error de diseño.

---

> **Siguiente página →** Página 4: Testing de funciones puras — casos borde,
> tablas de datos con `test.each` y cómo identificar todos los escenarios
> que debes cubrir.
