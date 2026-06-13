# Tutorial Jest — Página 7
## Módulo 7 · `jest.spyOn()` y módulos mockeados
### Interceptar funciones existentes y reemplazar módulos completos

---

## La diferencia entre `jest.fn()` y `jest.spyOn()`

En el módulo anterior usamos `jest.fn()` para crear funciones mock desde cero
y pasarlas como argumentos. Pero ¿qué pasa cuando el código que queremos probar
importa directamente un módulo y llama a sus funciones? No podemos inyectarlas
como argumentos — necesitamos interceptarlas donde ya están.

```
jest.fn()           → crea una función mock de cero
                      ideal cuando puedes inyectar dependencias

jest.spyOn()        → intercepta una función que ya existe en un objeto
                      ideal cuando el código importa el módulo directamente
```

```javascript
// Con jest.fn() — necesitas inyección de dependencias
function enviarEmail(datos, mailer) {  // mailer es inyectado
  return mailer.send(datos);
}

// Con jest.spyOn() — puedes mockear mailer.send sin cambiar el código
const mailer = require('./mailer');
jest.spyOn(mailer, 'send').mockResolvedValue({ ok: true });
```

---

## Parte 1 — `jest.spyOn()` básico

`jest.spyOn(objeto, 'nombreMetodo')` crea un spy sobre un método
de un objeto. Por defecto, el método original sigue ejecutándose,
pero ahora puedes ver cuántas veces fue llamado y con qué argumentos.

```javascript
// src/calculadora.js
const calculadora = {
  sumar(a, b)      { return a + b; },
  restar(a, b)     { return a - b; },
  multiplicar(a, b){ return a * b; },
};

module.exports = calculadora;
```

```javascript
// tests/spy.test.js
const calculadora = require('../src/calculadora');

describe('jest.spyOn básico', () => {
  afterEach(() => {
    // Restaurar todos los spies al terminar cada test
    jest.restoreAllMocks();
  });

  test('espía una función sin cambiar su comportamiento', () => {
    // Crea un spy sobre calculadora.sumar
    const spySumar = jest.spyOn(calculadora, 'sumar');

    // El método original sigue funcionando
    const resultado = calculadora.sumar(3, 4);
    expect(resultado).toBe(7);

    // Pero ahora sabemos que fue llamado
    expect(spySumar).toHaveBeenCalledTimes(1);
    expect(spySumar).toHaveBeenCalledWith(3, 4);
  });

  test('un spy puede cambiar el comportamiento del método', () => {
    // Sobrescribe la implementación
    jest.spyOn(calculadora, 'sumar').mockReturnValue(999);

    expect(calculadora.sumar(1, 1)).toBe(999);  // ya no devuelve 2

    // restoreAllMocks restaura el método original
    jest.restoreAllMocks();
    expect(calculadora.sumar(1, 1)).toBe(2);  // vuelve a funcionar
  });
});
```

---

## Parte 2 — Spying en módulos de Node.js

Podemos espiar funciones de módulos built-in de Node como `console`,
`Date` o `Math`:

```javascript
describe('espiar console.log', () => {
  afterEach(() => jest.restoreAllMocks());

  test('verificar que se registra un mensaje de error', () => {
    // Silencia console.error durante el test y lo espía
    const spyError = jest.spyOn(console, 'error').mockImplementation(() => {});

    // Código que internamente llama a console.error
    function procesarDato(dato) {
      if (dato === null) {
        console.error('Dato nulo recibido');
        return false;
      }
      return true;
    }

    procesarDato(null);

    expect(spyError).toHaveBeenCalledTimes(1);
    expect(spyError).toHaveBeenCalledWith('Dato nulo recibido');
  });
});

describe('espiar Date.now para controlar el tiempo', () => {
  afterEach(() => jest.restoreAllMocks());

  test('función que usa la fecha actual', () => {
    // Fija la fecha a un valor conocido
    const fechaFija = new Date('2024-03-15T10:00:00Z').getTime();
    jest.spyOn(Date, 'now').mockReturnValue(fechaFija);

    function esDiaLaboral() {
      const dia = new Date(Date.now()).getDay(); // 0=Dom, 1=Lun...5=Vie, 6=Sab
      return dia >= 1 && dia <= 5;
    }

    // 2024-03-15 es viernes (día 5)
    expect(esDiaLaboral()).toBe(true);
  });
});
```

---

## Parte 3 — `jest.mock()` para módulos completos

`jest.mock('modulo')` reemplaza automáticamente **todo** un módulo
con mocks vacíos. Es la solución cuando el código importa módulos
directamente con `require()`.

```javascript
// src/emailService.js
const nodemailer = require('nodemailer');  // módulo externo real

async function enviarBienvenida(email, nombre) {
  const transporter = nodemailer.createTransport({
    host: 'smtp.ejemplo.com',
    port: 587,
  });

  return transporter.sendMail({
    from:    'noreply@ejemplo.com',
    to:      email,
    subject: 'Bienvenido',
    text:    `Hola ${nombre}, bienvenido a nuestra plataforma.`,
  });
}

module.exports = { enviarBienvenida };
```

```javascript
// tests/emailService.test.js

// jest.mock('modulo') — debe llamarse ANTES del require
jest.mock('nodemailer');

const nodemailer = require('nodemailer');
const { enviarBienvenida } = require('../src/emailService');

describe('enviarBienvenida', () => {
  let mockSendMail;

  beforeEach(() => {
    // Configurar qué devuelve createTransport y sendMail
    mockSendMail = jest.fn().mockResolvedValue({ messageId: 'msg-001' });

    nodemailer.createTransport.mockReturnValue({
      sendMail: mockSendMail,
    });
  });

  afterEach(() => jest.clearAllMocks());

  test('envía un email con los datos correctos', async () => {
    await enviarBienvenida('ana@ejemplo.com', 'Ana García');

    expect(mockSendMail).toHaveBeenCalledWith({
      from:    'noreply@ejemplo.com',
      to:      'ana@ejemplo.com',
      subject: 'Bienvenido',
      text:    'Hola Ana García, bienvenido a nuestra plataforma.',
    });
  });

  test('crea el transporter una sola vez', async () => {
    await enviarBienvenida('luis@ejemplo.com', 'Luis');
    expect(nodemailer.createTransport).toHaveBeenCalledTimes(1);
  });

  test('propaga el error si sendMail falla', async () => {
    mockSendMail.mockRejectedValue(new Error('Servidor SMTP caído'));

    await expect(enviarBienvenida('ana@ejemplo.com', 'Ana'))
      .rejects.toThrow('Servidor SMTP caído');
  });
});
```

---

## Parte 4 — `jest.mock()` con implementación manual

A veces quieres controlar exactamente cómo se comporta el módulo mockeado:

```javascript
// src/bd.js — simula una capa de acceso a datos
const { Pool } = require('pg');  // módulo de PostgreSQL

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function obtenerUsuarioPorEmail(email) {
  const resultado = await pool.query(
    'SELECT * FROM usuarios WHERE email = $1',
    [email]
  );
  return resultado.rows[0] ?? null;
}

async function crearUsuario(datos) {
  const resultado = await pool.query(
    'INSERT INTO usuarios (nombre, email) VALUES ($1, $2) RETURNING *',
    [datos.nombre, datos.email]
  );
  return resultado.rows[0];
}

module.exports = { obtenerUsuarioPorEmail, crearUsuario };
```

```javascript
// tests/bd.test.js

// Mockear pg antes de importar bd.js
jest.mock('pg', () => {
  // La función factory devuelve el módulo mockeado
  const mockQuery = jest.fn();
  const mockPool = { query: mockQuery };

  return {
    Pool: jest.fn().mockImplementation(() => mockPool),
    // Guardamos mockQuery para poder acceder desde los tests
    __mockQuery: mockQuery,
  };
});

const pg = require('pg');
const { obtenerUsuarioPorEmail, crearUsuario } = require('../src/bd');

describe('capa de acceso a datos', () => {
  // Accedemos al mock a través de la propiedad especial que creamos
  const mockQuery = pg.__mockQuery;

  afterEach(() => mockQuery.mockClear());

  describe('obtenerUsuarioPorEmail', () => {
    test('devuelve el usuario cuando existe', async () => {
      const usuarioBD = { id: 1, nombre: 'Ana', email: 'ana@ej.com' };
      mockQuery.mockResolvedValue({ rows: [usuarioBD] });

      const usuario = await obtenerUsuarioPorEmail('ana@ej.com');

      expect(usuario).toEqual(usuarioBD);
      expect(mockQuery).toHaveBeenCalledWith(
        expect.stringContaining('SELECT'),
        ['ana@ej.com']
      );
    });

    test('devuelve null cuando no existe', async () => {
      mockQuery.mockResolvedValue({ rows: [] });

      const usuario = await obtenerUsuarioPorEmail('noexiste@ej.com');
      expect(usuario).toBeNull();
    });
  });

  describe('crearUsuario', () => {
    test('ejecuta el INSERT y devuelve el usuario creado', async () => {
      const nuevoBD = { id: 99, nombre: 'Luis', email: 'luis@ej.com' };
      mockQuery.mockResolvedValue({ rows: [nuevoBD] });

      const creado = await crearUsuario({ nombre: 'Luis', email: 'luis@ej.com' });

      expect(creado).toEqual(nuevoBD);
      expect(mockQuery).toHaveBeenCalledWith(
        expect.stringContaining('INSERT'),
        ['Luis', 'luis@ej.com']
      );
    });
  });
});
```

---

## Parte 5 — Mockear módulos propios con `__mocks__`

Para módulos que se mockean en muchos tests, puedes crear un archivo
de mock manual en una carpeta `__mocks__`:

```
src/
├── emailService.js         ← módulo real
└── __mocks__/
    └── emailService.js     ← mock manual (mismo nombre)

tests/
└── registro.test.js
```

```javascript
// src/__mocks__/emailService.js — mock manual compartido

const enviarBienvenida = jest.fn().mockResolvedValue({ enviado: true });
const enviarRecuperacion = jest.fn().mockResolvedValue({ enviado: true });

module.exports = { enviarBienvenida, enviarRecuperacion };
```

```javascript
// tests/registro.test.js — usa el mock automáticamente
jest.mock('../src/emailService');  // carga src/__mocks__/emailService.js

const { enviarBienvenida } = require('../src/emailService');
const { registrar } = require('../src/registro');

test('registrar usuario llama a enviarBienvenida', async () => {
  await registrar({ nombre: 'Ana', email: 'ana@ej.com' });
  expect(enviarBienvenida).toHaveBeenCalled();
});
```

---

## Parte 6 — Ejemplo completo: servicio con dependencias

```javascript
// src/clima.js — versión que hace fetch real
const axios = require('axios');

async function obtenerClimaReal(ciudad) {
  const respuesta = await axios.get(`https://api.clima.com/v1/${ciudad}`);
  return {
    ciudad,
    temperatura: respuesta.data.main.temp,
    descripcion: respuesta.data.weather[0].description,
  };
}

module.exports = { obtenerClimaReal };
```

```javascript
// tests/climaReal.test.js
jest.mock('axios');

const axios  = require('axios');
const { obtenerClimaReal } = require('../src/clima');

describe('obtenerClimaReal', () => {
  afterEach(() => jest.clearAllMocks());

  test('procesa correctamente la respuesta de la API', async () => {
    // Simular la respuesta que devolvería axios.get
    axios.get.mockResolvedValue({
      data: {
        main: { temp: 24 },
        weather: [{ description: 'cielo despejado' }],
      },
    });

    const clima = await obtenerClimaReal('Madrid');

    expect(clima).toEqual({
      ciudad:      'Madrid',
      temperatura: 24,
      descripcion: 'cielo despejado',
    });

    expect(axios.get).toHaveBeenCalledWith('https://api.clima.com/v1/Madrid');
  });

  test('propaga el error si la API falla', async () => {
    axios.get.mockRejectedValue(new Error('Network Error'));

    await expect(obtenerClimaReal('Madrid')).rejects.toThrow('Network Error');
  });

  test('simula respuesta 404 de ciudad desconocida', async () => {
    const error = new Error('Request failed with status code 404');
    error.response = { status: 404, data: { message: 'City not found' } };
    axios.get.mockRejectedValue(error);

    await expect(obtenerClimaReal('Ciudad Invisible')).rejects.toThrow('404');
  });
});
```

---

## Tabla resumen: ¿cuándo usar qué?

| Situación | Herramienta |
|---|---|
| Crear una función simulada desde cero | `jest.fn()` |
| Interceptar un método de un objeto existente | `jest.spyOn(obj, 'metodo')` |
| Silenciar/controlar `console.log` o `console.error` | `jest.spyOn(console, 'log')` |
| Controlar la fecha/tiempo actual | `jest.spyOn(Date, 'now')` |
| Mockear un módulo completo (npm o propio) | `jest.mock('modulo')` |
| Mock manual reutilizable entre varios tests | `__mocks__/modulo.js` |
| Restaurar el método original después del spy | `mockFn.mockRestore()` o `jest.restoreAllMocks()` |
| Limpiar historial de llamadas | `mockFn.mockClear()` o `jest.clearAllMocks()` |

---

## Ejercicios propuestos

1. **Spy en Math.random** — Usa `jest.spyOn(Math, 'random')` para controlar
   el resultado de una función `generarCodigoPromocion()` que usa
   `Math.random()`. Fija el valor a 0.5 y verifica que el código generado
   es determinista. Restaura el método al final con `jest.restoreAllMocks()`.

2. **Módulo de archivos** — Mockea el módulo `fs` de Node.js para probar
   una función `leerConfiguracion(ruta)` que lee un archivo JSON.
   Simula que `fs.readFileSync` devuelve `'{"puerto": 3000}'` y verifica
   que la función devuelve `{ puerto: 3000 }`. Luego simula que el archivo
   no existe (lanza un error) y verifica que la función devuelve un objeto
   de configuración por defecto.

3. **Encadenamiento de mocks** — Tienes una función `generarReporte(datos, bd, mailer)`
   que: (1) guarda los datos en BD, (2) genera un PDF, (3) envía el PDF por email.
   Usa `jest.fn()` y `jest.spyOn()` para verificar que si la BD falla,
   no se genera el PDF ni se envía el email, y que si el email falla,
   los datos sí quedaron guardados en BD.

4. **Spy vs Mock** — Refactoriza uno de los tests del módulo 6 que usaba
   `jest.fn()` con inyección de dependencias para que en su lugar use
   `jest.mock()` sin cambiar el código de producción. Compara ambas
   aproximaciones y escribe un comentario en el test explicando cuándo
   preferirías cada una.

---

## Resumen de la página 7

- `jest.spyOn(objeto, 'metodo')` crea un spy sobre un método existente — puede observarlo sin cambiar su comportamiento, o sobreescribirlo con `mockReturnValue/mockImplementation`.
- `jest.mock('modulo')` reemplaza automáticamente un módulo completo — todas sus exportaciones se convierten en `jest.fn()` vacías.
- Con `jest.mock('modulo', () => ({ ... }))` puedes definir exactamente qué devuelve el módulo mockeado.
- La carpeta `__mocks__` permite crear mocks manuales compartidos entre varios archivos de test.
- `jest.restoreAllMocks()` en `afterEach` es imprescindible con `spyOn` — restaura las funciones originales para no contaminar otros tests.
- `jest.clearAllMocks()` limpia el historial de llamadas pero no restaura implementaciones. `jest.resetAllMocks()` también resetea la implementación. `jest.restoreAllMocks()` restaura las funciones originales (solo para spies).
- Mockear dependencias externas (axios, nodemailer, pg) hace tus tests independientes de servicios de red o bases de datos reales.

---

> **Siguiente página →** Página 8: Cobertura de código y buenas prácticas —
> cómo generar reportes de coverage, qué métricas importan y cómo escribir
> tests que aporten valor real a tu proyecto.
