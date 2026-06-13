# Tutorial Jest — Página 5
## Módulo 5 · Testing asíncrono
### Cómo probar Promises, async/await y callbacks

---

## El problema con el código asíncrono

En JavaScript muchas operaciones son asíncronas: llamadas a APIs,
lectura de archivos, consultas a bases de datos. Si no le dices a Jest
que debe esperar, el test termina antes de que la operación acabe
y puede dar un falso positivo (el test pasa aunque la lógica esté mal).

```javascript
// Este test tiene un bug silencioso — Jest NO espera la Promise
test('MAL — Jest no espera la Promise', () => {
  obtenerUsuario(1).then(usuario => {
    expect(usuario.nombre).toBe('Ana');  // este expect nunca se evalúa
  });
  // Jest llega aquí, ve que no lanzó error y marca el test como pasado
  // aunque el expect de dentro nunca se ejecutó
});
```

Hay tres formas correctas de manejar esto en Jest:

```
1. Devolver la Promise    → return promise.then(...)
2. async/await            → test('...', async () => { await ... })
3. resolves / rejects     → expect(promise).resolves.toBe(...)
```

---

## Parte 1 — Funciones asíncronas de ejemplo

Primero el código que vamos a probar. Simularemos llamadas a una API
con `Promise` y pequeños delays para que sea realista.

```javascript
// src/api.js

// Simula una base de datos en memoria
const usuarios = [
  { id: 1, nombre: 'Ana García',  email: 'ana@ejemplo.com',  activo: true  },
  { id: 2, nombre: 'Luis Pérez',  email: 'luis@ejemplo.com', activo: false },
  { id: 3, nombre: 'María López', email: 'maria@ejemplo.com',activo: true  },
];

/**
 * Simula obtener un usuario por id desde una API.
 * Devuelve una Promise que resuelve con el usuario o rechaza si no existe.
 */
function obtenerUsuario(id) {
  return new Promise((resolve, reject) => {
    // Simula un delay de red de 10ms
    setTimeout(() => {
      const usuario = usuarios.find(u => u.id === id);
      if (usuario) {
        resolve(usuario);
      } else {
        reject(new Error(`Usuario con id ${id} no encontrado`));
      }
    }, 10);
  });
}

/**
 * Simula obtener todos los usuarios activos.
 */
async function obtenerUsuariosActivos() {
  // En una app real aquí harías: const resp = await fetch('/api/usuarios?activo=true')
  return usuarios.filter(u => u.activo);
}

/**
 * Simula guardar un usuario. Falla si el email ya existe.
 */
async function guardarUsuario(nuevoUsuario) {
  const existe = usuarios.find(u => u.email === nuevoUsuario.email);
  if (existe) {
    throw new Error(`El email ${nuevoUsuario.email} ya está registrado`);
  }
  const guardado = { ...nuevoUsuario, id: usuarios.length + 1 };
  usuarios.push(guardado);
  return guardado;
}

module.exports = { obtenerUsuario, obtenerUsuariosActivos, guardarUsuario };
```

---

## Parte 2 — Las tres formas de testear código asíncrono

### Forma 1 — Devolver la Promise

La forma más básica: hacer `return` de la Promise dentro del test.
Jest esperará a que la Promise se resuelva.

```javascript
const { obtenerUsuario } = require('../src/api');

test('Forma 1: devolver la Promise — obtener usuario existente', () => {
  // SIEMPRE haz return de la Promise — sin return Jest no espera
  return obtenerUsuario(1).then(usuario => {
    expect(usuario.nombre).toBe('Ana García');
    expect(usuario.email).toBe('ana@ejemplo.com');
  });
});

test('Forma 1: devolver la Promise — usuario inexistente rechaza', () => {
  return obtenerUsuario(99).catch(error => {
    expect(error.message).toBe('Usuario con id 99 no encontrado');
  });
});
```

### Forma 2 — async/await (la más recomendada)

Usando `async/await` el test se lee igual que código síncrono.
Es la forma más clara y la que deberías usar por defecto.

```javascript
const { obtenerUsuario, obtenerUsuariosActivos, guardarUsuario } = require('../src/api');

test('Forma 2: async/await — obtener usuario existente', async () => {
  // La función del test es async — Jest espera automáticamente
  const usuario = await obtenerUsuario(1);

  expect(usuario.nombre).toBe('Ana García');
  expect(usuario.activo).toBe(true);
});

test('Forma 2: async/await — obtener usuarios activos', async () => {
  const activos = await obtenerUsuariosActivos();

  expect(activos).toHaveLength(2);
  // Verifica que todos los devueltos están activos
  activos.forEach(u => expect(u.activo).toBe(true));
});

test('Forma 2: async/await — guardar usuario nuevo', async () => {
  const nuevo = { nombre: 'Carlos', email: 'carlos@nuevo.com', activo: true };
  const guardado = await guardarUsuario(nuevo);

  expect(guardado).toHaveProperty('id');
  expect(guardado.nombre).toBe('Carlos');
  expect(guardado.email).toBe('carlos@nuevo.com');
});
```

### Forma 3 — `.resolves` y `.rejects`

Para tests cortos y directos. Combina `expect` con la Promise directamente:

```javascript
test('Forma 3: resolves — el usuario 1 tiene nombre "Ana García"', () => {
  // IMPORTANTE: hace falta return (o await) aquí también
  return expect(obtenerUsuario(1)).resolves.toHaveProperty('nombre', 'Ana García');
});

test('Forma 3: resolves con async/await', async () => {
  await expect(obtenerUsuario(1)).resolves.toEqual(
    expect.objectContaining({ nombre: 'Ana García', activo: true })
  );
});

test('Forma 3: rejects — usuario inexistente lanza error', async () => {
  await expect(obtenerUsuario(99)).rejects.toThrow('Usuario con id 99 no encontrado');
});

test('Forma 3: rejects — email duplicado lanza error', async () => {
  const duplicado = { nombre: 'Otro', email: 'ana@ejemplo.com', activo: true };
  await expect(guardarUsuario(duplicado)).rejects.toThrow('ya está registrado');
});
```

---

## Parte 3 — Verificar errores con async/await

Para verificar que una función asíncrona lanza un error,
usa `try/catch` o `.rejects.toThrow()`:

```javascript
// Forma A: try/catch explícito
test('usuario inexistente — try/catch', async () => {
  try {
    await obtenerUsuario(999);
    // Si llegamos aquí, el test debe fallar — no lanzó el error esperado
    expect(true).toBe(false);  // fuerza el fallo
    // O usa: fail('Debería haber lanzado un error') — pero requiere configuración extra
  } catch (error) {
    expect(error.message).toBe('Usuario con id 999 no encontrado');
    expect(error).toBeInstanceOf(Error);
  }
});

// Forma B: .rejects (más concisa — la preferida)
test('usuario inexistente — rejects', async () => {
  await expect(obtenerUsuario(999)).rejects.toThrow('no encontrado');
});
```

> **Prefiere `.rejects.toThrow()`** sobre `try/catch` para código más limpio.
> El `try/catch` puede tener el bug silencioso de olvidar el fallo forzado.

---

## Parte 4 — Ejemplo aplicado: servicio de clima

Un ejemplo más completo que simula un servicio real:

```javascript
// src/clima.js

const datosClima = {
  Madrid:    { temperatura: 22, humedad: 45, descripcion: 'Soleado' },
  Barcelona: { temperatura: 26, humedad: 70, descripcion: 'Parcialmente nublado' },
  Sevilla:   { temperatura: 35, humedad: 30, descripcion: 'Muy caluroso' },
};

async function obtenerClima(ciudad) {
  if (!ciudad || ciudad.trim() === '') {
    throw new Error('La ciudad no puede estar vacía');
  }

  const datos = datosClima[ciudad];

  if (!datos) {
    throw new Error(`No hay datos de clima para "${ciudad}"`);
  }

  // Simula el delay de una llamada HTTP
  await new Promise(resolve => setTimeout(resolve, 5));

  return {
    ciudad,
    ...datos,
    consultadoEn: new Date().toISOString(),
  };
}

async function obtenerAlertas(ciudad) {
  const clima = await obtenerClima(ciudad);
  const alertas = [];

  if (clima.temperatura > 30) {
    alertas.push({ nivel: 'alta', mensaje: 'Temperatura extrema — mantenerse hidratado' });
  }
  if (clima.humedad > 80) {
    alertas.push({ nivel: 'media', mensaje: 'Humedad elevada — posibles lluvias' });
  }

  return alertas;
}

module.exports = { obtenerClima, obtenerAlertas };
```

```javascript
// tests/clima.test.js
const { obtenerClima, obtenerAlertas } = require('../src/clima');

describe('obtenerClima', () => {
  test('devuelve los datos de una ciudad existente', async () => {
    const clima = await obtenerClima('Madrid');

    expect(clima.ciudad).toBe('Madrid');
    expect(clima.temperatura).toBe(22);
    expect(clima.humedad).toBe(45);
    expect(clima.descripcion).toBe('Soleado');
    expect(clima).toHaveProperty('consultadoEn'); // campo dinámico — solo que existe
  });

  test('el campo consultadoEn tiene formato ISO', async () => {
    const clima = await obtenerClima('Barcelona');
    // Verifica que es una fecha ISO válida
    expect(clima.consultadoEn).toMatch(/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/);
  });

  test('ciudad inexistente rechaza con error', async () => {
    await expect(obtenerClima('Atlantis')).rejects.toThrow(
      'No hay datos de clima para "Atlantis"'
    );
  });

  test('ciudad vacía rechaza con error', async () => {
    await expect(obtenerClima('')).rejects.toThrow('La ciudad no puede estar vacía');
  });

  test('ciudad con solo espacios rechaza con error', async () => {
    await expect(obtenerClima('   ')).rejects.toThrow('La ciudad no puede estar vacía');
  });

  test.each([
    ['Madrid',    22, 'Soleado'],
    ['Barcelona', 26, 'Parcialmente nublado'],
    ['Sevilla',   35, 'Muy caluroso'],
  ])('clima de %s: temperatura=%i, descripción="%s"', async (ciudad, temp, desc) => {
    const clima = await obtenerClima(ciudad);
    expect(clima.temperatura).toBe(temp);
    expect(clima.descripcion).toBe(desc);
  });
});

describe('obtenerAlertas', () => {
  test('Sevilla genera alerta de temperatura extrema', async () => {
    const alertas = await obtenerAlertas('Sevilla');
    expect(alertas).toContainEqual(
      expect.objectContaining({ nivel: 'alta' })
    );
  });

  test('Madrid no genera alertas', async () => {
    const alertas = await obtenerAlertas('Madrid');
    expect(alertas).toHaveLength(0);
  });

  test('ciudad inexistente propaga el error', async () => {
    await expect(obtenerAlertas('Ciudad Invisible')).rejects.toThrow();
  });
});
```

---

## Parte 5 — Timeouts en tests asíncronos

Por defecto Jest espera **5 segundos** por cada test. Si tu código tarda más,
el test falla con "Timeout exceeded". Puedes cambiar el timeout:

```javascript
// Cambiar el timeout de un test específico — tercer parámetro de test()
test('operación lenta', async () => {
  const resultado = await operacionMuyLenta();
  expect(resultado).toBeDefined();
}, 10000);  // ← 10 segundos para este test

// Cambiar el timeout global en un describe
describe('tests lentos', () => {
  jest.setTimeout(15000);  // 15 segundos para todos los tests de este describe

  test('test 1', async () => { ... });
  test('test 2', async () => { ... });
});

// Cambiar el timeout global en jest.config.js
// module.exports = { testTimeout: 10000 };
```

> **En la práctica:** si un test necesita más de 5 segundos es una señal
> de que deberías usar **mocks** para no llamar al servicio real.
> Los tests lentos son los que menos se ejecutan. Lo veremos en el siguiente módulo.

---

## Parte 6 — El error más común: olvidar `await`

```javascript
// Este test siempre pasa, ¡aunque la función esté rota!
test('MAL — falta await', async () => {
  const usuario = obtenerUsuario(1);  // ← falta await
  // usuario es una Promise, no un objeto
  // expect(usuario.nombre) es undefined — pero el test pasa sin error
  expect(usuario.nombre).toBeUndefined(); // pasa por las razones equivocadas
});

// La versión correcta
test('BIEN — con await', async () => {
  const usuario = await obtenerUsuario(1);
  expect(usuario.nombre).toBe('Ana García');
});
```

Cuando un test asíncrono no hace lo que esperas, verifica siempre:

```
1. ¿La función del test es async?
2. ¿Hay await delante de cada llamada asíncrona?
3. ¿Hay return si usas .then()/.catch() en lugar de async/await?
4. ¿Hay await antes de expect(...).rejects.toThrow()?
```

---

## Ejercicios propuestos

1. **Servicio de productos** — Crea `src/productos.js` con las funciones
   `async obtenerProducto(id)` y `async listarProductos(categoria)`.
   Usa un array de productos en memoria. Escribe tests con `async/await`
   para: producto existente, producto inexistente (debe rechazar),
   lista de categoría válida, lista de categoría sin resultados (array vacío).

2. **Reintentos** — Crea `async function conReintento(fn, maxIntentos)` que
   llame a `fn()` y, si lanza error, la reintente hasta `maxIntentos` veces.
   Devuelve el resultado si algún intento tiene éxito, lanza el último error
   si todos fallan. Usa `jest.fn()` (lo estudiarás en la siguiente página,
   pero puedes usar un contador manual con un array de funciones).

3. **Promise.all** — Crea `async function obtenerMultiple(ids)` que use
   `Promise.all` para obtener varios usuarios en paralelo. Verifica que:
   todos los ids válidos devuelven los usuarios correctos, y que si un solo
   id es inválido, toda la operación falla.

4. **El bug del timeout** — Identifica qué está mal en este test y corrígelo:
   ```javascript
   test('guarda y recupera un usuario', async () => {
     guardarUsuario({ nombre: 'Test', email: 'test@test.com', activo: true });
     const guardado = obtenerUsuario(4);
     expect(guardado.nombre).toBe('Test');
   });
   ```

---

## Resumen de la página 5

- Jest no espera automáticamente el código asíncrono — debes indicárselo explícitamente.
- **Tres formas correctas:** `return promise`, función `async` con `await`, o `.resolves/.rejects`.
- La forma recomendada es `async/await` — hace que el test se lea como código síncrono.
- Para verificar errores asíncronos, prefiere `await expect(fn()).rejects.toThrow()` sobre `try/catch`.
- **Siempre** añade `await` antes de `expect(...).resolves/rejects` — si lo olvidas el test pasa aunque la lógica esté mal.
- El timeout por defecto es 5 segundos. Cambiarlo es una señal de que deberías usar mocks en su lugar.
- El error más común es olvidar `await` — el test pasa silenciosamente aunque el comportamiento sea incorrecto.

---

> **Siguiente página →** Página 6: Mocks y funciones simuladas — `jest.fn()`
> para controlar el comportamiento de funciones y verificar que fueron llamadas
> correctamente, sin depender de servicios externos.
