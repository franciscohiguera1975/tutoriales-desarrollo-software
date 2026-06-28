# Tutorial TypeScript — Página 6
## Programación Orientada a Objetos
### Clases, modificadores de acceso, herencia y polimorfismo

---

> **Cómo usar esta página.** Cada bloque sigue el mismo ritmo:
> **1)** el concepto puro, **2)** un ejemplo aplicado real, **3)** un
> **mini-ejercicio** que se resuelve en clase en **5–8 minutos**.
> Escribe el código en el [TypeScript Playground](https://www.typescriptlang.org/play)
> o ejecútalo con `npx tsx archivo.ts`.

---

## Parte A · Clases y constructores

---

### A.1 · Clase básica: propiedades tipadas, `constructor` e instanciación

Una **clase** es una plantilla que describe propiedades y comportamientos.
Con `new` creas una **instancia** (objeto concreto) a partir de esa plantilla.

```ts
// Concepto puro
class Producto {
  nombre: string;
  precio: number;
  enStock: boolean;

  constructor(nombre: string, precio: number, enStock: boolean) {
    this.nombre = nombre;
    this.precio = precio;
    this.enStock = enStock;
  }

  // Método: acción que puede realizar la instancia
  describir(): string {
    const estado = this.enStock ? "disponible" : "agotado";
    return `${this.nombre} — $${this.precio} (${estado})`;
  }
}

const teclado = new Producto("Teclado mecánico", 120, true);
const monitor = new Producto("Monitor 4K", 450, false);

console.log(teclado.describir()); // Teclado mecánico — $120 (disponible)
console.log(monitor.describir()); // Monitor 4K — $450 (agotado)
```

#### Ejemplo aplicado — clase `Temperatura`

```ts
class Temperatura {
  valorCelsius: number;

  constructor(celsius: number) {
    this.valorCelsius = celsius;
  }

  aFahrenheit(): number {
    return this.valorCelsius * 9 / 5 + 32;
  }

  aKelvin(): number {
    return this.valorCelsius + 273.15;
  }

  describir(): string {
    return (
      `${this.valorCelsius}°C = ` +
      `${this.aFahrenheit()}°F = ` +
      `${this.aKelvin()}K`
    );
  }
}

const hervor = new Temperatura(100);
const congelacion = new Temperatura(0);

console.log(hervor.describir());     // 100°C = 212°F = 373.15K
console.log(congelacion.describir()); // 0°C = 32°F = 273.15K
```

> **Mini-ejercicio (5 min).** Crea la clase `Rectangulo` con propiedades
> `ancho: number` y `alto: number`. Agrega los métodos `area(): number`
> y `perimetro(): number`. Instancia dos rectángulos distintos e imprime
> sus resultados.

---

### A.2 · Atajo de parámetros del constructor (parameter properties)

TypeScript permite **declarar y asignar** la propiedad directamente en
la firma del constructor con un modificador de acceso. Elimina la
repetición de `this.x = x`.

```ts
// Concepto puro — forma larga (sin atajo)
class PuntoLargo {
  x: number;
  y: number;
  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }
}

// Forma corta con parameter properties (equivalente exacto)
class Punto {
  constructor(
    public x: number,
    public y: number
  ) {}

  distanciaAlOrigen(): number {
    return Math.sqrt(this.x ** 2 + this.y ** 2);
  }
}

const p = new Punto(3, 4);
console.log(p.x);                    // 3
console.log(p.distanciaAlOrigen());  // 5
```

#### Ejemplo aplicado — clase `Usuario` con atajo

```ts
class Usuario {
  constructor(
    public nombre: string,
    public email: string,
    public rol: "admin" | "editor" | "lector"
  ) {}

  saludo(): string {
    return `Hola, ${this.nombre}. Tienes rol "${this.rol}".`;
  }
}

const ana = new Usuario("Ana", "ana@ejemplo.com", "admin");
const luis = new Usuario("Luis", "luis@ejemplo.com", "lector");

console.log(ana.saludo());  // Hola, Ana. Tienes rol "admin".
console.log(luis.email);    // luis@ejemplo.com
```

> **Mini-ejercicio (5 min).** Convierte esta clase a la forma corta
> con parameter properties:
> ```ts
> class Libro {
>   titulo: string;
>   autor: string;
>   paginas: number;
>   constructor(titulo: string, autor: string, paginas: number) {
>     this.titulo = titulo;
>     this.autor = autor;
>     this.paginas = paginas;
>   }
> }
> ```
> Luego agrega un método `resumen(): string` que devuelva
> `"'Titulo' de Autor (N páginas)"`.

---

## Parte B · Modificadores de acceso y propiedades especiales

---

### B.1 · Modificadores de acceso: `public`, `private`, `protected`, `readonly`

Los modificadores controlan **desde dónde** se puede leer o escribir
una propiedad o método.

| Modificador   | Accesible desde...                              |
|---------------|-------------------------------------------------|
| `public`      | Cualquier lugar (por defecto)                   |
| `private`     | Solo dentro de la misma clase                   |
| `protected`   | La clase y sus subclases                        |
| `readonly`    | Solo lectura después del constructor            |

```ts
// Concepto puro
class CuentaBancaria {
  readonly id: string;           // no cambia tras la creación
  public titular: string;        // visible desde fuera
  private saldo: number;         // solo accesible dentro de la clase
  protected moneda: string;      // accesible también en subclases

  constructor(id: string, titular: string, saldoInicial: number) {
    this.id = id;
    this.titular = titular;
    this.saldo = saldoInicial;
    this.moneda = "MXN";
  }

  // Método público que expone el saldo de forma controlada
  obtenerSaldo(): number {
    return this.saldo;
  }

  depositar(monto: number): void {
    if (monto <= 0) throw new Error("Monto inválido");
    this.saldo += monto;
  }
}

const cuenta = new CuentaBancaria("CC-001", "Ana García", 1000);
console.log(cuenta.titular);         // Ana García
console.log(cuenta.id);              // CC-001
console.log(cuenta.obtenerSaldo());  // 1000
cuenta.depositar(500);
console.log(cuenta.obtenerSaldo());  // 1500

// cuenta.saldo = 9999;  // Error: 'saldo' is private
// cuenta.id = "otro";   // Error: 'id' is readonly
```

#### Ejemplo aplicado — clase `Configuracion` con propiedades privadas

```ts
class Configuracion {
  private readonly apiKey: string;
  private entorno: "desarrollo" | "produccion";
  public version: string;

  constructor(apiKey: string, entorno: "desarrollo" | "produccion") {
    this.apiKey = apiKey;
    this.entorno = entorno;
    this.version = "1.0.0";
  }

  esProduccion(): boolean {
    return this.entorno === "produccion";
  }

  // Expone parte del key de forma enmascarada
  keyResumida(): string {
    return `***${this.apiKey.slice(-4)}`;
  }
}

const cfg = new Configuracion("sk-ABCDE-12345", "produccion");
console.log(cfg.esProduccion()); // true
console.log(cfg.keyResumida());  // ***2345
console.log(cfg.version);        // 1.0.0
```

> **Mini-ejercicio (6 min).** Crea la clase `Contador` con una propiedad
> `private valor: number = 0`. Agrega los métodos públicos:
> - `incrementar(): void` — suma 1.
> - `decrementar(): void` — resta 1 pero no baja de 0.
> - `obtenerValor(): number` — devuelve el valor actual.
>
> Prueba que no se puede asignar `contador.valor = 99` directamente.

---

### B.2 · Getters y setters (`get` / `set`) con validación

Los `get`/`set` permiten acceder a una propiedad **con lógica integrada**,
manteniendo una sintaxis de acceso natural (sin paréntesis al leer).

```ts
// Concepto puro
class Circulo {
  private _radio: number;

  constructor(radio: number) {
    this._radio = radio;
  }

  get radio(): number {
    return this._radio;
  }

  set radio(valor: number) {
    if (valor <= 0) throw new Error("El radio debe ser positivo");
    this._radio = valor;
  }

  get area(): number {
    return Math.PI * this._radio ** 2;
  }
}

const c = new Circulo(5);
console.log(c.radio);          // 5   ← usa el getter
console.log(c.area.toFixed(2)); // 78.54

c.radio = 10;                  // usa el setter
console.log(c.area.toFixed(2)); // 314.16

// c.radio = -3;  // Error: El radio debe ser positivo
```

> ⚠️ **Convención:** la propiedad privada lleva prefijo `_` (por ejemplo
> `_radio`) y el getter/setter se llama sin guion bajo (`radio`).
> Así el setter puede validar antes de asignar.

#### Ejemplo aplicado — clase `Porcentaje`

```ts
class Porcentaje {
  private _valor: number;

  constructor(valor: number) {
    this._valor = 0;
    this.valor = valor; // pasa por el setter para validar
  }

  get valor(): number {
    return this._valor;
  }

  set valor(v: number) {
    if (v < 0 || v > 100) {
      throw new Error(`Porcentaje inválido: ${v}. Debe estar entre 0 y 100.`);
    }
    this._valor = v;
  }

  get complemento(): number {
    return 100 - this._valor;
  }

  toString(): string {
    return `${this._valor}% (complemento: ${this.complemento}%)`;
  }
}

const descuento = new Porcentaje(25);
console.log(descuento.toString()); // 25% (complemento: 75%)
descuento.valor = 40;
console.log(descuento.toString()); // 40% (complemento: 60%)

// new Porcentaje(150); // Error: Porcentaje inválido: 150
```

> **Mini-ejercicio (7 min).** Crea la clase `Temperatura` (nueva versión)
> con una propiedad privada `_celsius: number`. Agrega:
> - `get celsius()` y `set celsius(v)` (valida que no baje de -273.15).
> - `get fahrenheit()` — calculado: `celsius * 9/5 + 32`.
> - `set fahrenheit(f)` — convierte y almacena en `_celsius`.
>
> Prueba asignar grados Fahrenheit y leer en Celsius.

---

## Parte C · Herencia y clases abstractas

---

### C.1 · Herencia: `extends`, `super()` y sobrescritura de métodos

Una subclase **hereda** todas las propiedades y métodos de su clase base
y puede añadir los suyos propios o **sobrescribir** los heredados.

```ts
// Concepto puro
class Animal {
  constructor(public nombre: string) {}

  hablar(): string {
    return `${this.nombre} hace un sonido.`;
  }
}

class Perro extends Animal {
  constructor(nombre: string, public raza: string) {
    super(nombre); // llama al constructor del padre
  }

  // override sobrescribe el método del padre
  override hablar(): string {
    return `${this.nombre} ladra: ¡Guau!`;
  }

  buscar(objeto: string): string {
    return `${this.nombre} busca el ${objeto}.`;
  }
}

const a = new Animal("Criatura");
const d = new Perro("Rex", "Labrador");

console.log(a.hablar());       // Criatura hace un sonido.
console.log(d.hablar());       // Rex ladra: ¡Guau!
console.log(d.buscar("palo")); // Rex busca el palo.
console.log(d.raza);           // Labrador
```

> **Truco** La palabra clave `override` (TypeScript 4.3+) es
> opcional pero muy recomendable: si el método base no existe o cambia
> de firma, el compilador te avisa en lugar de crear un método "huérfano".

#### Ejemplo aplicado — jerarquía de empleados

```ts
class Empleado {
  constructor(
    public nombre: string,
    protected salarioBase: number
  ) {}

  calcularSalario(): number {
    return this.salarioBase;
  }

  infoLaboral(): string {
    return `${this.nombre} — Salario: $${this.calcularSalario()}`;
  }
}

class Gerente extends Empleado {
  constructor(
    nombre: string,
    salarioBase: number,
    private bonificacion: number
  ) {
    super(nombre, salarioBase);
  }

  override calcularSalario(): number {
    return this.salarioBase + this.bonificacion;
  }
}

class Vendedor extends Empleado {
  constructor(
    nombre: string,
    salarioBase: number,
    private comision: number,
    private ventasMes: number
  ) {
    super(nombre, salarioBase);
  }

  override calcularSalario(): number {
    return this.salarioBase + this.comision * this.ventasMes;
  }
}

const emp = new Empleado("Carlos", 2000);
const ger = new Gerente("Laura", 3000, 1500);
const vend = new Vendedor("Pedro", 1500, 50, 30);

console.log(emp.infoLaboral());  // Carlos — Salario: $2000
console.log(ger.infoLaboral());  // Laura — Salario: $4500
console.log(vend.infoLaboral()); // Pedro — Salario: $3000
```

> **Mini-ejercicio (7 min).** Crea la clase base `Vehiculo` con propiedades
> `marca: string` y `velocidadMax: number`, y el método
> `describir(): string`. Luego crea `Automovil extends Vehiculo` que añada
> `numeroPuertas: number` y sobrescriba `describir()` incluyendo las puertas.
> Crea también `Motocicleta extends Vehiculo` que añada `tieneSidecar: boolean`.
> Instancia uno de cada tipo y llama a `describir()`.

---

### C.2 · Clases abstractas (`abstract`) y métodos abstractos

Una clase **abstracta** no puede instanciarse directamente. Sirve como
contrato que obliga a las subclases a implementar ciertos métodos.

```ts
// Concepto puro
abstract class Figura {
  abstract area(): number;       // sin implementación — las subclases DEBEN implementarlo
  abstract perimetro(): number;

  // Los métodos concretos SÍ tienen implementación
  describir(): string {
    return (
      `Área: ${this.area().toFixed(2)} | ` +
      `Perímetro: ${this.perimetro().toFixed(2)}`
    );
  }
}

class Circulo2 extends Figura {
  constructor(private radio: number) {
    super();
  }

  override area(): number {
    return Math.PI * this.radio ** 2;
  }

  override perimetro(): number {
    return 2 * Math.PI * this.radio;
  }
}

class Rectangulo2 extends Figura {
  constructor(private ancho: number, private alto: number) {
    super();
  }

  override area(): number {
    return this.ancho * this.alto;
  }

  override perimetro(): number {
    return 2 * (this.ancho + this.alto);
  }
}

// const f = new Figura(); // Error: Cannot create an instance of an abstract class.

const circulo = new Circulo2(5);
const rect = new Rectangulo2(4, 6);

console.log(circulo.describir()); // Área: 78.54 | Perímetro: 31.42
console.log(rect.describir());    // Área: 24.00 | Perímetro: 20.00
```

> ⚠️ Si una subclase **no implementa** todos los métodos abstractos del
> padre, TypeScript lanza un error en tiempo de compilación. No hay forma
> de "olvidarse" accidentalmente.

#### Ejemplo aplicado — sistema de pagos

```ts
abstract class MetodoPago {
  constructor(protected titular: string) {}

  abstract procesar(monto: number): string;

  abstract validar(): boolean;

  // Método concreto reutilizable
  resumen(monto: number): string {
    if (!this.validar()) return `[${this.titular}] Pago rechazado: datos inválidos.`;
    return this.procesar(monto);
  }
}

class TarjetaCredito extends MetodoPago {
  constructor(
    titular: string,
    private ultimos4: string,
    private saldoDisponible: number
  ) {
    super(titular);
  }

  override validar(): boolean {
    return this.ultimos4.length === 4 && this.saldoDisponible > 0;
  }

  override procesar(monto: number): string {
    if (monto > this.saldoDisponible) return "Fondos insuficientes en tarjeta.";
    this.saldoDisponible -= monto;
    return `Tarjeta ****${this.ultimos4}: $${monto} aprobado. Saldo restante: $${this.saldoDisponible}`;
  }
}

class TransferenciaBancaria extends MetodoPago {
  constructor(
    titular: string,
    private clabe: string
  ) {
    super(titular);
  }

  override validar(): boolean {
    return this.clabe.length === 18;
  }

  override procesar(monto: number): string {
    return `Transferencia de $${monto} para ${this.titular} a CLABE ${this.clabe.slice(-4).padStart(18, "*")}.`;
  }
}

const tarjeta = new TarjetaCredito("Ana", "4321", 500);
const transferencia = new TransferenciaBancaria("Luis", "123456789012345678");

console.log(tarjeta.resumen(200));
// Tarjeta ****4321: $200 aprobado. Saldo restante: $300

console.log(transferencia.resumen(1000));
// Transferencia de $1000 para Luis a CLABE **************5678.
```

> **Mini-ejercicio (8 min).** Define la clase abstracta `Exportador` con
> el método abstracto `exportar(datos: string[]): string` y el método
> concreto `encabezado(): string` que devuelva `"=== Exportación ==="`.
> Implementa `ExportadorCSV` (une con comas) y `ExportadorJSON`
> (usa `JSON.stringify`). Prueba ambos con el mismo array de strings.

---

## Parte D · Interfaces, miembros estáticos y polimorfismo

---

### D.1 · Implementar interfaces: `class X implements Y`

Una clase puede **comprometerse** a cumplir el contrato de una interfaz
usando `implements`. Puede implementar **varias** interfaces a la vez.

```ts
// Concepto puro
interface Serializable {
  serializar(): string;
}

interface Validable {
  esValido(): boolean;
}

class Pedido implements Serializable, Validable {
  constructor(
    public id: string,
    public productos: string[],
    public total: number
  ) {}

  serializar(): string {
    return JSON.stringify({ id: this.id, productos: this.productos, total: this.total });
  }

  esValido(): boolean {
    return this.productos.length > 0 && this.total > 0;
  }
}

const pedido = new Pedido("P-001", ["Mouse", "Teclado"], 150);
console.log(pedido.esValido());    // true
console.log(pedido.serializar());
// {"id":"P-001","productos":["Mouse","Teclado"],"total":150}
```

> **Truco** La diferencia clave entre `implements` y `extends`:
> `extends` hereda **implementación** (código); `implements` solo hereda
> el **contrato** (firmas). Una clase puede `extends` una sola clase pero
> `implements` múltiples interfaces.

#### Ejemplo aplicado — repositorio de usuarios

```ts
interface RepositorioLectura<T> {
  buscarPorId(id: number): T | undefined;
  listarTodos(): T[];
}

interface RepositorioEscritura<T> {
  guardar(entidad: T): void;
  eliminar(id: number): boolean;
}

interface Repositorio<T> extends RepositorioLectura<T>, RepositorioEscritura<T> {}

interface UsuarioEntidad {
  id: number;
  nombre: string;
}

class RepositorioUsuarios implements Repositorio<UsuarioEntidad> {
  private datos: UsuarioEntidad[] = [];

  guardar(u: UsuarioEntidad): void {
    this.datos.push(u);
  }

  eliminar(id: number): boolean {
    const idx = this.datos.findIndex((u) => u.id === id);
    if (idx === -1) return false;
    this.datos.splice(idx, 1);
    return true;
  }

  buscarPorId(id: number): UsuarioEntidad | undefined {
    return this.datos.find((u) => u.id === id);
  }

  listarTodos(): UsuarioEntidad[] {
    return [...this.datos];
  }
}

const repo = new RepositorioUsuarios();
repo.guardar({ id: 1, nombre: "Ana" });
repo.guardar({ id: 2, nombre: "Luis" });
console.log(repo.buscarPorId(1));  // { id: 1, nombre: 'Ana' }
console.log(repo.listarTodos().length); // 2
repo.eliminar(1);
console.log(repo.listarTodos().length); // 1
```

> **Mini-ejercicio (7 min).** Define la interfaz `Imprimible` con el
> método `imprimir(): void`. Luego crea las clases `Factura` y `Recibo`,
> ambas implementando `Imprimible`. Cada una debe imprimir un formato
> distinto con sus propios datos. Crea instancias y llama a `imprimir()`.

---

### D.2 · Miembros estáticos (`static`)

Los miembros `static` pertenecen a la **clase** misma, no a las
instancias. Se accede con el nombre de la clase, no con `this`.

```ts
// Concepto puro
class Matematica {
  static readonly PI: number = 3.14159265;

  static circunferencia(radio: number): number {
    return 2 * Matematica.PI * radio;
  }

  static potencia(base: number, exp: number): number {
    return base ** exp;
  }
}

// No necesitas instanciar la clase
console.log(Matematica.PI);                   // 3.14159265
console.log(Matematica.circunferencia(5));    // 31.4159265
console.log(Matematica.potencia(2, 10));      // 1024
```

#### Ejemplo aplicado — generador de IDs y contador de instancias

```ts
class Conexion {
  private static totalConexiones: number = 0;
  private static readonly MAX_CONEXIONES: number = 5;

  public readonly idConexion: number;
  public readonly host: string;

  constructor(host: string) {
    if (Conexion.totalConexiones >= Conexion.MAX_CONEXIONES) {
      throw new Error(`Límite alcanzado: máximo ${Conexion.MAX_CONEXIONES} conexiones.`);
    }
    Conexion.totalConexiones++;
    this.idConexion = Conexion.totalConexiones;
    this.host = host;
  }

  static cuantasActivas(): number {
    return Conexion.totalConexiones;
  }

  static resetear(): void {
    Conexion.totalConexiones = 0;
  }

  cerrar(): void {
    Conexion.totalConexiones--;
    console.log(`Conexión #${this.idConexion} a "${this.host}" cerrada.`);
  }
}

const c1 = new Conexion("db.local");
const c2 = new Conexion("cache.local");
const c3 = new Conexion("api.local");

console.log(`Activas: ${Conexion.cuantasActivas()}`); // Activas: 3
c2.cerrar();                                          // Conexión #2 a "cache.local" cerrada.
console.log(`Activas: ${Conexion.cuantasActivas()}`); // Activas: 2
```

> **Mini-ejercicio (6 min).** Crea la clase `IdUnico` con un miembro
> estático privado `private static ultimo: number = 0` y el método
> estático `static generar(): number` que incremente y devuelva el
> siguiente ID. Crea 5 IDs distintos usando el método estático sin
> instanciar la clase.

---

### D.3 · Polimorfismo: tratar subclases por su tipo base

El **polimorfismo** permite tratar objetos de distintas subclases de
forma uniforme a través del tipo de la clase (o interfaz) base.
TypeScript invocará la **versión correcta** de cada método en tiempo
de ejecución.

```ts
// Concepto puro
class Forma {
  nombre(): string { return "Forma"; }
  area(): number { return 0; }
}

class Circulo3 extends Forma {
  constructor(private r: number) { super(); }
  override nombre(): string { return "Círculo"; }
  override area(): number { return Math.PI * this.r ** 2; }
}

class Triangulo extends Forma {
  constructor(private base: number, private altura: number) { super(); }
  override nombre(): string { return "Triángulo"; }
  override area(): number { return (this.base * this.altura) / 2; }
}

class Cuadrado extends Forma {
  constructor(private lado: number) { super(); }
  override nombre(): string { return "Cuadrado"; }
  override area(): number { return this.lado ** 2; }
}

// Array de tipo base — el polimorfismo en acción
const formas: Forma[] = [
  new Circulo3(3),
  new Triangulo(6, 4),
  new Cuadrado(5),
];

for (const f of formas) {
  // TypeScript llama la versión correcta de area() en cada iteración
  console.log(`${f.nombre()}: área = ${f.area().toFixed(2)}`);
}
// Círculo: área = 28.27
// Triángulo: área = 12.00
// Cuadrado: área = 25.00
```

#### Ejemplo aplicado — sistema de notificaciones polimórfico

```ts
abstract class Notificacion {
  constructor(protected destinatario: string, protected mensaje: string) {}

  abstract enviar(): string;

  resumen(): string {
    return `[${this.constructor.name}] → ${this.destinatario}`;
  }
}

class NotificacionEmail extends Notificacion {
  constructor(destinatario: string, mensaje: string, private asunto: string) {
    super(destinatario, mensaje);
  }

  override enviar(): string {
    return `Email a <${this.destinatario}> | Asunto: "${this.asunto}" | Cuerpo: ${this.mensaje}`;
  }
}

class NotificacionSMS extends Notificacion {
  constructor(destinatario: string, mensaje: string, private telefono: string) {
    super(destinatario, mensaje);
  }

  override enviar(): string {
    return `SMS a ${this.telefono} (${this.destinatario}): ${this.mensaje}`;
  }
}

class NotificacionPush extends Notificacion {
  constructor(destinatario: string, mensaje: string, private dispositivoId: string) {
    super(destinatario, mensaje);
  }

  override enviar(): string {
    return `Push → dispositivo ${this.dispositivoId} (${this.destinatario}): ${this.mensaje}`;
  }
}

// Polimorfismo: un solo array, tipos distintos
const notificaciones: Notificacion[] = [
  new NotificacionEmail("ana@mail.com", "Tu pedido llegó.", "Entrega completada"),
  new NotificacionSMS("Luis", "Tu cita es mañana.", "+52-555-0001"),
  new NotificacionPush("Carlos", "¡Oferta especial!", "dev-abc-123"),
];

function despacharTodas(lista: Notificacion[]): void {
  for (const n of lista) {
    console.log(n.resumen());
    console.log("  >", n.enviar());
  }
}

despacharTodas(notificaciones);
```

Salida:
```
[NotificacionEmail] → ana@mail.com
  > Email a <ana@mail.com> | Asunto: "Entrega completada" | Cuerpo: Tu pedido llegó.
[NotificacionSMS] → Luis
  > SMS a +52-555-0001 (Luis): Tu cita es mañana.
[NotificacionPush] → Carlos
  > Push → dispositivo dev-abc-123 (Carlos): ¡Oferta especial!
```

> **Mini-ejercicio (8 min).** Crea la clase abstracta `Descuento` con el
> método abstracto `aplicar(precio: number): number` y el método concreto
> `etiqueta(precio: number): string` que devuelva
> `"$precio → $precioFinal"`. Implementa:
> - `DescuentoPorcentaje(porcentaje: number)` — reduce el precio en ese %.
> - `DescuentoFijo(cantidad: number)` — resta una cantidad fija (mínimo $0).
> - `SinDescuento` — devuelve el precio sin cambios.
>
> Guárdalos en un array `Descuento[]` y recórrelo imprimiendo `etiqueta(100)`.

---

## Ejemplo combinado — Sistema de biblioteca

Une todos los conceptos de la página en un caso real:
clases base, herencia, `abstract`, `implements`, `static` y polimorfismo.

```ts
// ── Interfaz ──────────────────────────────────────────────────────────
interface Prestable {
  prestar(usuario: string): boolean;
  devolver(): void;
  estaDisponible(): boolean;
}

// ── Clase abstracta base ──────────────────────────────────────────────
abstract class RecursoBiblioteca implements Prestable {
  private static totalRecursos: number = 0;
  readonly id: number;
  protected _usuarioActual: string | null = null;

  constructor(public titulo: string, public autor: string) {
    RecursoBiblioteca.totalRecursos++;
    this.id = RecursoBiblioteca.totalRecursos;
  }

  static contarRecursos(): number {
    return RecursoBiblioteca.totalRecursos;
  }

  // Contrato abstracto — cada subclase define su tipo
  abstract tipo(): string;

  // Implementación del contrato Prestable
  prestar(usuario: string): boolean {
    if (!this.estaDisponible()) return false;
    this._usuarioActual = usuario;
    return true;
  }

  devolver(): void {
    this._usuarioActual = null;
  }

  estaDisponible(): boolean {
    return this._usuarioActual === null;
  }

  // Método concreto reutilizable
  ficha(): string {
    const estado = this.estaDisponible()
      ? "Disponible"
      : `Prestado a: ${this._usuarioActual}`;
    return `[${this.tipo()}] #${this.id} "${this.titulo}" — ${this.autor} | ${estado}`;
  }
}

// ── Subclases concretas ───────────────────────────────────────────────
class Libro2 extends RecursoBiblioteca {
  constructor(
    titulo: string,
    autor: string,
    public readonly paginas: number
  ) {
    super(titulo, autor);
  }

  override tipo(): string { return "Libro"; }
}

class Revista extends RecursoBiblioteca {
  constructor(
    titulo: string,
    autor: string,
    public readonly edicion: number
  ) {
    super(titulo, autor);
  }

  override tipo(): string { return "Revista"; }
}

class AudioLibro extends RecursoBiblioteca {
  constructor(
    titulo: string,
    autor: string,
    public readonly duracionMin: number
  ) {
    super(titulo, autor);
  }

  override tipo(): string { return "Audiolibro"; }

  // Método extra solo de AudioLibro
  duracionFormateada(): string {
    const h = Math.floor(this.duracionMin / 60);
    const m = this.duracionMin % 60;
    return `${h}h ${m}m`;
  }
}

// ── Polimorfismo: catálogo unificado ──────────────────────────────────
const catalogo: RecursoBiblioteca[] = [
  new Libro2("El señor de los anillos", "Tolkien", 1178),
  new Revista("National Geographic", "Varios", 312),
  new AudioLibro("Sapiens", "Harari", 683),
  new Libro2("Clean Code", "Martin", 431),
];

console.log("=== Catálogo de la Biblioteca ===");
for (const recurso of catalogo) {
  console.log(recurso.ficha());
}

console.log(`\nTotal de recursos registrados: ${RecursoBiblioteca.contarRecursos()}`);

// Simulamos préstamos
catalogo[0].prestar("Ana");
catalogo[2].prestar("Luis");

console.log("\n=== Estado tras préstamos ===");
for (const recurso of catalogo) {
  console.log(recurso.ficha());
}

// Polimorfismo con tipo narrowing
const audio = catalogo[2] as AudioLibro;
console.log(`\nDuración de "${audio.titulo}": ${audio.duracionFormateada()}`);

// Devolver y verificar
catalogo[0].devolver();
console.log(`\n"${catalogo[0].titulo}" disponible: ${catalogo[0].estaDisponible()}`);
```

Salida esperada:
```
=== Catálogo de la Biblioteca ===
[Libro] #1 "El señor de los anillos" — Tolkien | Disponible
[Revista] #2 "National Geographic" — Varios | Disponible
[Audiolibro] #3 "Sapiens" — Harari | Disponible
[Libro] #4 "Clean Code" — Martin | Disponible

Total de recursos registrados: 4

=== Estado tras préstamos ===
[Libro] #1 "El señor de los anillos" — Tolkien | Prestado a: Ana
[Revista] #2 "National Geographic" — Varios | Disponible
[Audiolibro] #3 "Sapiens" — Harari | Prestado a: Luis
[Libro] #4 "Clean Code" — Martin | Disponible

Duración de "Sapiens": 11h 23m

"El señor de los anillos" disponible: true
```

---

## Resumen de la página 6

- **Clase básica:** `class Nombre { propiedades; constructor(); métodos; }` — se instancia con `new`.
- **Parameter properties:** `constructor(public x: number)` declara y asigna la propiedad en un solo paso.
- **Modificadores de acceso:** `public` (todos), `private` (solo la clase), `protected` (clase + subclases), `readonly` (no reasignable tras el constructor).
- **Getters / setters:** `get propiedad()` y `set propiedad(v)` añaden lógica de lectura/escritura manteniendo sintaxis de acceso natural. Convención: campo privado `_nombre`, acceso público `nombre`.
- **Herencia:** `class Hijo extends Padre` — el constructor hijo debe llamar `super()`. La palabra clave `override` hace explícita la sobrescritura y el compilador avisa si el método base no existe.
- **Clases abstractas:** `abstract class` no puede instanciarse; los métodos `abstract` obligan a las subclases a implementarlos. Los métodos concretos sí se heredan.
- **Interfaces con `implements`:** una clase puede cumplir varios contratos simultáneamente con `implements A, B`. A diferencia de `extends`, no hereda implementación sino solo la forma.
- **Miembros estáticos:** `static` pertenece a la clase, no a la instancia. Se accede como `NombreClase.miembro`. Útil para contadores, fábricas y constantes compartidas.
- **Polimorfismo:** almacenar subclases en un array del tipo base y recorrerlo invoca el método correcto de cada objeto en tiempo de ejecución, sin condicionales adicionales.

---

> **Siguiente página →** Página 7: Genéricos — funciones y clases genéricas,
> restricciones (`extends`), `keyof` y utility types.
