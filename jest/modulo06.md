# Tutorial Jest — Página 6
## Módulo 6 · Mocks y `jest.fn()`
### Simular funciones para aislar el código que pruebas

---

## ¿Por qué necesitamos mocks?

Imagina que quieres probar una función que envía un email al registrar
un usuario. Para hacer el test, ¿realmente quieres enviar un email real
cada vez? ¿O necesitas acceso a internet? ¿Y si el servidor de email está caído?

Un **mock** es una función simulada que reemplaza a la función real.
En lugar de llamar al servidor de email, el test llama a una función
falsa que tú controlas.

```
Sin mocks                        Con mocks
────────────────────────────     ────────────────────────────
Tu función llama a:              Tu función llama a:
  - servicio de email real  →      - función simulada
  - base de datos real      →      - función simulada
  - API externa real        →      - función simulada

Problemas:                       Ventajas:
  - Requiere internet              - Sin dependencias externas
  - Puede fallar por error         - Siempre el mismo resultado
    del servidor externo           - Test rapidísimo
  - Puede costar dinero            - Puedes simular errores
  - Difícil de reproducir          - Verificas que fue llamada
```

---

## Parte 1 — `jest.fn()` básico

`jest.fn()` crea una función simulada que registra cada vez que es llamada,
con qué argumentos, cuántas veces, y qué devolvió.

```javascript
test('jest.fn() básico', () => {
  // Crear una función mock vacía
  const mockFn = jest.fn();

  // Llamarla varias veces
  mockFn('hola');
  mockFn('mundo', 42);
  mockFn();

  // Verificar cuántas veces fue llamada
  expect(mockFn).toHaveBeenCalledTimes(3);

  // Verificar que fue llamada con ciertos argumentos
  expect(mockFn).toHaveBeenCalledWith('hola');
  expect(mockFn).toHaveBeenCalledWith('mundo', 42);

  // Verificar los argumentos de la última llamada
  expect(mockFn).toHaveBeenLastCalledWith();

  // Verificar que fue llamada al menos una vez
  expect(mockFn).toHaveBeenCalled();
});
```

### Configurar el valor que devuelve

```javascript
test('jest.fn() con valor de retorno', () => {
  // Devuelve siempre el mismo valor
  const mockSumar = jest.fn().mockReturnValue(42);

  expect(mockSumar(1, 2)).toBe(42);  // ignora los argumentos, devuelve 42
  expect(mockSumar(5, 5)).toBe(42);  // siempre 42

  // Devuelve distintos valores en llamadas sucesivas
  const mockSecuencia = jest.fn()
    .mockReturnValueOnce('primero')
    .mockReturnValueOnce('segundo')
    .mockReturnValue('por defecto');   // todas las siguientes

  expect(mockSecuencia()).toBe('primero');
  expect(mockSecuencia()).toBe('segundo');
  expect(mockSecuencia()).toBe('por defecto');
  expect(mockSecuencia()).toBe('por defecto');
});

test('jest.fn() con implementación personalizada', () => {
  // mockImplementation permite dar lógica propia
  const mockDividir = jest.fn().mockImplementation((a, b) => {
    if (b === 0) return null;
    return a / b;
  });

  expect(mockDividir(10, 2)).toBe(5);
  expect(mockDividir(10, 0)).toBeNull();
  expect(mockDividir).toHaveBeenCalledTimes(2);
});
```

---

## Parte 2 — Mocks para funciones asíncronas

```javascript
test('mock de función asíncrona', async () => {
  // mockResolvedValue → la Promise se resuelve con ese valor
  const mockObtenerDatos = jest.fn().mockResolvedValue({ id: 1, nombre: 'Ana' });

  const datos = await mockObtenerDatos();
  expect(datos.nombre).toBe('Ana');
  expect(mockObtenerDatos).toHaveBeenCalledTimes(1);

  // mockRejectedValue → la Promise se rechaza con ese error
  const mockFallar = jest.fn().mockRejectedValue(new Error('Sin conexión'));

  await expect(mockFallar()).rejects.toThrow('Sin conexión');

  // mockResolvedValueOnce → solo en la primera llamada
  const mockInestable = jest.fn()
    .mockRejectedValueOnce(new Error('Timeout'))    // 1ª llamada falla
    .mockResolvedValueOnce({ id: 2, nombre: 'Luis' }); // 2ª llamada funciona

  await expect(mockInestable()).rejects.toThrow('Timeout');
  await expect(mockInestable()).resolves.toEqual({ id: 2, nombre: 'Luis' });
});
```

---

## Parte 3 — Ejemplo real: servicio de notificaciones

Cuando queremos probar el comportamiento de una función que depende de
otra (el envío de email), pasamos el mock como argumento — técnica
conocida como **inyección de dependencias**.

```javascript
// src/registro.js

/**
 * Registra un nuevo usuario y le envía un email de bienvenida.
 *
 * @param {object}   datos         - { nombre, email, password }
 * @param {Function} enviarEmail   - función para enviar email (inyectada)
 * @param {Function} guardarEnBD   - función para guardar en BD (inyectada)
 */
async function registrarUsuario(datos, enviarEmail, guardarEnBD) {
  // Validar datos
  if (!datos.nombre || datos.nombre.trim() === '') {
    throw new Error('El nombre es obligatorio');
  }
  if (!datos.email || !datos.email.includes('@')) {
    throw new Error('El email no es válido');
  }
  if (!datos.password || datos.password.length < 8) {
    throw new Error('La contraseña debe tener al menos 8 caracteres');
  }

  // Guardar en la base de datos
  const usuario = await guardarEnBD({
    nombre:   datos.nombre.trim(),
    email:    datos.email.toLowerCase(),
    password: datos.password,
  });

  // Enviar email de bienvenida
  await enviarEmail({
    para:    usuario.email,
    asunto:  'Bienvenido/a',
    cuerpo:  `Hola ${usuario.nombre}, tu cuenta ha sido creada.`,
  });

  return usuario;
}

module.exports = { registrarUsuario };
```

```javascript
// tests/registro.test.js
const { registrarUsuario } = require('../src/registro');

describe('registrarUsuario', () => {
  let mockEnviarEmail;
  let mockGuardarEnBD;

  beforeEach(() => {
    // Crear mocks frescos antes de cada test
    mockEnviarEmail = jest.fn().mockResolvedValue({ enviado: true });
    mockGuardarEnBD = jest.fn().mockResolvedValue({
      id:     42,
      nombre: 'Ana García',
      email:  'ana@ejemplo.com',
    });
  });

  test('registra al usuario y envía email de bienvenida', async () => {
    const datos = {
      nombre:   'Ana García',
      email:    'ANA@EJEMPLO.COM',
      password: 'contraseña123',
    };

    const usuario = await registrarUsuario(datos, mockEnviarEmail, mockGuardarEnBD);

    // Verifica que se guardó en la BD con el email en minúsculas
    expect(mockGuardarEnBD).toHaveBeenCalledWith({
      nombre:   'Ana García',
      email:    'ana@ejemplo.com',   // minúsculas
      password: 'contraseña123',
    });

    // Verifica que se envió el email correctamente
    expect(mockEnviarEmail).toHaveBeenCalledWith({
      para:   'ana@ejemplo.com',
      asunto: 'Bienvenido/a',
      cuerpo: 'Hola Ana García, tu cuenta ha sido creada.',
    });

    // Verifica el resultado
    expect(usuario.id).toBe(42);
    expect(usuario.nombre).toBe('Ana García');
  });

  test('ambas operaciones se realizan exactamente una vez', async () => {
    await registrarUsuario(
      { nombre: 'Luis', email: 'luis@ej.com', password: 'password123' },
      mockEnviarEmail,
      mockGuardarEnBD
    );

    expect(mockGuardarEnBD).toHaveBeenCalledTimes(1);
    expect(mockEnviarEmail).toHaveBeenCalledTimes(1);
  });

  test('si la BD falla, NO se envía el email', async () => {
    mockGuardarEnBD.mockRejectedValue(new Error('Conexión rechazada'));

    await expect(
      registrarUsuario(
        { nombre: 'Luis', email: 'luis@ej.com', password: 'password123' },
        mockEnviarEmail,
        mockGuardarEnBD
      )
    ).rejects.toThrow('Conexión rechazada');

    // El email no debe enviarse si la BD falló
    expect(mockEnviarEmail).not.toHaveBeenCalled();
  });

  describe('validaciones de datos', () => {
    test('nombre vacío lanza error sin llamar a la BD ni al email', async () => {
      await expect(
        registrarUsuario(
          { nombre: '', email: 'ana@ej.com', password: 'password123' },
          mockEnviarEmail, mockGuardarEnBD
        )
      ).rejects.toThrow('El nombre es obligatorio');

      expect(mockGuardarEnBD).not.toHaveBeenCalled();
      expect(mockEnviarEmail).not.toHaveBeenCalled();
    });

    test('email inválido lanza error sin guardar', async () => {
      await expect(
        registrarUsuario(
          { nombre: 'Ana', email: 'sinArroba', password: 'password123' },
          mockEnviarEmail, mockGuardarEnBD
        )
      ).rejects.toThrow('El email no es válido');

      expect(mockGuardarEnBD).not.toHaveBeenCalled();
    });

    test('contraseña corta lanza error', async () => {
      await expect(
        registrarUsuario(
          { nombre: 'Ana', email: 'ana@ej.com', password: 'corta' },
          mockEnviarEmail, mockGuardarEnBD
        )
      ).rejects.toThrow('al menos 8 caracteres');
    });
  });
});
```

---

## Parte 4 — Matchers específicos para mocks

```javascript
test('matchers de mock', () => {
  const mock = jest.fn();

  // toHaveBeenCalled — fue llamada al menos una vez
  mock('a');
  expect(mock).toHaveBeenCalled();

  // toHaveBeenCalledTimes — número exacto de llamadas
  mock('b');
  expect(mock).toHaveBeenCalledTimes(2);

  // toHaveBeenCalledWith — verificar argumentos de alguna llamada
  expect(mock).toHaveBeenCalledWith('a');
  expect(mock).toHaveBeenCalledWith('b');

  // toHaveBeenLastCalledWith — argumentos de la ÚLTIMA llamada
  mock('último');
  expect(mock).toHaveBeenLastCalledWith('último');

  // toHaveBeenNthCalledWith — argumentos de la llamada N (1-indexado)
  expect(mock).toHaveBeenNthCalledWith(1, 'a');
  expect(mock).toHaveBeenNthCalledWith(2, 'b');
  expect(mock).toHaveBeenNthCalledWith(3, 'último');

  // not.toHaveBeenCalled — NO fue llamada
  const otraMock = jest.fn();
  expect(otraMock).not.toHaveBeenCalled();
});
```

### Acceder a las llamadas directamente

```javascript
test('acceder al historial de llamadas', () => {
  const mock = jest.fn();

  mock(10, 'a');
  mock(20, 'b');
  mock(30, 'c');

  // mock.calls — array de arrays con los argumentos de cada llamada
  expect(mock.calls).toHaveLength(3);
  expect(mock.calls[0]).toEqual([10, 'a']);  // primera llamada
  expect(mock.calls[1]).toEqual([20, 'b']);  // segunda
  expect(mock.calls[2]).toEqual([30, 'c']);  // tercera

  // Primera llamada, primer argumento
  expect(mock.calls[0][0]).toBe(10);

  // mock.results — array con los resultados de cada llamada
  const mockRetorno = jest.fn(x => x * 2);
  mockRetorno(5);
  mockRetorno(10);

  expect(mockRetorno.results[0].value).toBe(10);
  expect(mockRetorno.results[1].value).toBe(20);
});
```

---

## Parte 5 — Limpiar mocks entre tests

Los mocks guardan el historial de llamadas. Si no los limpias,
la información de un test contamina el siguiente.

```javascript
describe('limpieza de mocks', () => {
  let mockFn;

  beforeEach(() => {
    mockFn = jest.fn();
  });

  // Opción 1: crear un mock nuevo en cada beforeEach (lo más simple)
  // Opción 2: limpiar el mock existente

  afterEach(() => {
    // mockClear — limpia historial de llamadas (calls, results)
    // pero conserva la implementación
    mockFn.mockClear();

    // mockReset — limpia historial Y resetea la implementación
    // mockFn.mockReset();

    // mockRestore — solo para jest.spyOn(), restaura la función original
    // mockFn.mockRestore();
  });
});

// Alternativa: configurar en jest.config.js para limpiar automáticamente
// module.exports = { clearMocks: true }   // mockClear en cada test
// module.exports = { resetMocks: true }   // mockReset en cada test
// module.exports = { restoreMocks: true } // mockRestore en cada test
```

---

## Parte 6 — Ejemplo completo: carrito con descuentos

```javascript
// src/pedido.js
async function procesarPedido(carrito, calcularDescuento, cobrar, notificar) {
  if (carrito.items.length === 0) {
    throw new Error('El carrito está vacío');
  }

  const subtotal = carrito.items.reduce((s, i) => s + i.precio * i.cantidad, 0);
  const descuento = await calcularDescuento(carrito.usuarioId, subtotal);
  const total = subtotal - descuento;

  if (total <= 0) {
    throw new Error('El total no puede ser cero o negativo');
  }

  const pago = await cobrar(carrito.usuarioId, total);

  await notificar(carrito.usuarioId, {
    pedidoId: pago.pedidoId,
    total,
    items:    carrito.items.length,
  });

  return { pedidoId: pago.pedidoId, subtotal, descuento, total };
}

module.exports = { procesarPedido };
```

```javascript
// tests/pedido.test.js
const { procesarPedido } = require('../src/pedido');

describe('procesarPedido', () => {
  let mockCalcularDescuento;
  let mockCobrar;
  let mockNotificar;
  let carrito;

  beforeEach(() => {
    mockCalcularDescuento = jest.fn().mockResolvedValue(10);      // 10€ de descuento
    mockCobrar            = jest.fn().mockResolvedValue({ pedidoId: 'PED-001' });
    mockNotificar         = jest.fn().mockResolvedValue({ enviado: true });

    carrito = {
      usuarioId: 1,
      items: [
        { nombre: 'Teclado', precio: 89.99, cantidad: 1 },
        { nombre: 'Ratón',   precio: 29.99, cantidad: 2 },
      ],
    };
  });

  test('procesa el pedido correctamente y devuelve el resumen', async () => {
    const resultado = await procesarPedido(
      carrito, mockCalcularDescuento, mockCobrar, mockNotificar
    );

    // subtotal: 89.99 + 29.99*2 = 149.97
    expect(resultado.subtotal).toBeCloseTo(149.97, 2);
    expect(resultado.descuento).toBe(10);
    expect(resultado.total).toBeCloseTo(139.97, 2);
    expect(resultado.pedidoId).toBe('PED-001');
  });

  test('calcula el descuento con el id de usuario correcto', async () => {
    await procesarPedido(carrito, mockCalcularDescuento, mockCobrar, mockNotificar);

    expect(mockCalcularDescuento).toHaveBeenCalledWith(1, expect.any(Number));
    //                                                   ↑ usuarioId
  });

  test('cobra el total después de aplicar el descuento', async () => {
    await procesarPedido(carrito, mockCalcularDescuento, mockCobrar, mockNotificar);

    expect(mockCobrar).toHaveBeenCalledWith(1, expect.closeTo(139.97, 2));
  });

  test('notifica al usuario con el pedidoId correcto', async () => {
    await procesarPedido(carrito, mockCalcularDescuento, mockCobrar, mockNotificar);

    expect(mockNotificar).toHaveBeenCalledWith(
      1,
      expect.objectContaining({ pedidoId: 'PED-001', items: 2 })
    );
  });

  test('carrito vacío lanza error sin llamar a cobrar', async () => {
    carrito.items = [];

    await expect(
      procesarPedido(carrito, mockCalcularDescuento, mockCobrar, mockNotificar)
    ).rejects.toThrow('El carrito está vacío');

    expect(mockCobrar).not.toHaveBeenCalled();
    expect(mockNotificar).not.toHaveBeenCalled();
  });

  test('si el cobro falla, no se envía la notificación', async () => {
    mockCobrar.mockRejectedValue(new Error('Tarjeta rechazada'));

    await expect(
      procesarPedido(carrito, mockCalcularDescuento, mockCobrar, mockNotificar)
    ).rejects.toThrow('Tarjeta rechazada');

    expect(mockNotificar).not.toHaveBeenCalled();
  });
});
```

---

## Ejercicios propuestos

1. **Logger de eventos** — Crea `src/eventos.js` con la función
   `registrarEvento(tipo, datos, logger)`. Verifica con mocks que:
   `logger` se llama exactamente una vez, con el tipo correcto,
   y que si `logger` lanza un error, la función lo propaga.

2. **Retry automático** — Crea `async function conReintento(fn, maxIntentos)`.
   Usa `jest.fn()` con `mockRejectedValueOnce` para simular fallos en
   las primeras llamadas y éxito en la última. Verifica con
   `toHaveBeenCalledTimes` que se llama el número correcto de veces.

3. **Pipeline de datos** — Crea `procesarDatos(datos, validar, transformar, guardar)`.
   Usa mocks para verificar el orden de llamadas: primero `validar`, luego
   `transformar` con el resultado validado, luego `guardar` con el resultado
   transformado. Verifica con `toHaveBeenNthCalledWith`.

4. **Simular distintos escenarios** — Para la función `registrarUsuario` del
   ejemplo, escribe un test que use `mockReturnValueOnce` para simular que
   el primer intento de guardar en la BD falla, y un segundo test donde
   el servicio de email falla después de guardar exitosamente. Verifica
   que el usuario no queda en un estado inconsistente.

---

## Resumen de la página 6

- Un **mock** es una función simulada que reemplaza a la función real — elimina la dependencia de servicios externos, bases de datos o APIs.
- `jest.fn()` crea un mock que registra cada llamada: argumentos, veces, resultados.
- `mockReturnValue(val)` configura el valor de retorno. `mockReturnValueOnce` lo hace solo en la siguiente llamada.
- `mockResolvedValue(val)` y `mockRejectedValue(err)` son equivalentes para funciones asíncronas.
- Los matchers de mock más importantes: `toHaveBeenCalled`, `toHaveBeenCalledTimes`, `toHaveBeenCalledWith`, `toHaveBeenLastCalledWith`.
- La **inyección de dependencias** (pasar las funciones dependientes como argumentos) hace que el código sea naturalmente testeable con mocks.
- Limpia los mocks en `beforeEach` o con `jest.clearAllMocks()` para evitar que el historial de un test contamine el siguiente.

---

> **Siguiente página →** Página 7: `jest.spyOn()` y módulos mockeados —
> cómo interceptar llamadas a funciones ya existentes y mockear módulos
> completos sin cambiar el código de producción.
