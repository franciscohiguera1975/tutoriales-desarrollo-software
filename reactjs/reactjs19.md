# Curso React.js + TypeScript — Página 16
## Frameworks de estilos · React-Bootstrap v2
### LAB: 5 componentes aislados — HOME: landing de cursos con Router

---

## ¿Qué es React-Bootstrap?

React-Bootstrap es una reimplementación completa de los componentes Bootstrap 5
como componentes React nativos. Sin jQuery, sin manipulación directa del DOM —
cada componente es un componente React que acepta props tipadas.

```
Bootstrap 5 original        React-Bootstrap v2
─────────────────           ──────────────────────────
<button class="btn          <Button variant="primary">
  btn-primary">             Enviar
  Enviar                    </Button>
</button>
```

---

## Instalación

```bash
npm install react-bootstrap bootstrap
```

| Paquete | Versión actual | Descripción |
|---|---|---|
| `react-bootstrap` | 2.10.10 | Componentes React |
| `bootstrap` | 5.3.8 | CSS base requerido |

---

## `src/main.tsx`

El CSS de Bootstrap se importa **una sola vez** en el punto de entrada.
Todos los componentes de la app lo heredan automáticamente.

```tsx
// src/main.tsx

import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import 'bootstrap/dist/css/bootstrap.min.css'   // ← CSS global de Bootstrap
import App from './App.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

---

## Estructura del proyecto

```
src/
├── lab/
│   ├── LabRbButtons.tsx
│   ├── LabRbAlert.tsx
│   ├── LabRbCard.tsx
│   ├── LabRbForm.tsx
│   └── LabRbTable.tsx
├── components/rb/
│   ├── RBNavbar.tsx
│   ├── RBHero.tsx
│   ├── RBCourseGrid.tsx
│   ├── RBNewsletter.tsx
│   └── RBFooter.tsx
├── pages/
│   ├── HomeRB.tsx
│   └── AboutRB.tsx
├── App.tsx          ← alterna entre AppLab y AppHome
├── AppLab.tsx       ← fase 1: selector de componentes
├── AppHome.tsx      ← fase 2: landing con Router
└── main.tsx
```

---

## Fase 1 — Laboratorio

Cada componente del lab es independiente y se puede usar directamente
en `App.tsx` sin necesidad de Router ni layout. El selector permite
navegar entre ellos sin recargar la página.

---

### `src/lab/LabRbButtons.tsx`

```tsx
// src/lab/LabRbButtons.tsx

import { Container, Button, Stack } from 'react-bootstrap'

export default function LabRbButtons() {
  return (
    <Container className="py-4">
      <h2 className="h4 fw-bold mb-1">LAB: Buttons</h2>
      <p className="text-secondary mb-3">Variantes, outline y tamaños.</p>

      <Stack direction="horizontal" gap={2} className="flex-wrap mb-3">
        <Button variant="primary">Primary</Button>
        <Button variant="outline-primary">Outline</Button>
        <Button variant="success">Success</Button>
        <Button variant="danger">Danger</Button>
        <Button variant="warning">Warning</Button>
        <Button variant="secondary">Secondary</Button>
      </Stack>

      <Stack direction="horizontal" gap={2}>
        <Button variant="primary" size="lg">Large</Button>
        <Button variant="primary">Normal</Button>
        <Button variant="primary" size="sm">Small</Button>
      </Stack>
    </Container>
  )
}
```

### Prueba esto

- Cambia `variant="primary"` a `variant="success"` en el primer botón — el botón se vuelve verde; Bootstrap usa variables CSS para los colores de variante
- Cambia `variant="outline-primary"` a `variant="outline-danger"` — el borde y el texto cambian a rojo sin necesidad de CSS adicional
- Añade `disabled` como prop a cualquier botón (`<Button variant="primary" disabled>`) — el botón se atenúa y deja de responder a clics
- Cambia `size="lg"` a `size="sm"` en el botón Large — el botón encoge al tamaño pequeño; Bootstrap usa clases `.btn-sm` y `.btn-lg` internamente
- Añade `className="w-100"` a un botón — ocupa el 100% del ancho del contenedor
- Cambia `gap={2}` a `gap={4}` en el `Stack` — el espacio entre botones aumenta; Bootstrap usa su escala de espaciado (1 = 4px, 4 = 24px)
- Reemplaza `<Stack direction="horizontal"` por `<Stack direction="vertical"` — los botones se apilan verticalmente

---

### `src/lab/LabRbAlert.tsx`

```tsx
// src/lab/LabRbAlert.tsx

import { useState } from 'react'
import { Container, Alert, Button } from 'react-bootstrap'

export default function LabRbAlert() {
  const [show, setShow] = useState(true)

  return (
    <Container className="py-4">
      <h2 className="h4 fw-bold mb-1">LAB: Alert</h2>
      <p className="text-secondary mb-3">Con dismissible y control de visibilidad.</p>

      {show
        ? (
          <Alert variant="success" onClose={() => setShow(false)} dismissible>
            <Alert.Heading>Operación exitosa</Alert.Heading>
            <p className="mb-0">El formulario se envió correctamente (demo).</p>
          </Alert>
        )
        : <Button onClick={() => setShow(true)}>Mostrar alerta</Button>
      }
    </Container>
  )
}
```

### Prueba esto

- Cambia `variant="success"` a `variant="danger"` — el alert cambia de verde a rojo; Bootstrap aplica un color de fondo, borde y texto coherentes con la variante
- Cambia `variant="success"` a `variant="info"` — el alert adopta un tono azul claro, útil para mensajes informativos no críticos
- Elimina la prop `dismissible` y `onClose` — el botón de cierre (✕) desaparece; el estado `show` ya no se necesita
- Cambia `<Alert.Heading>` por texto plano — la tipografía del encabezado vuelve al estilo normal sin el peso extra de Bootstrap
- Añade `className="mt-3"` al `Alert` — aparece un margen superior; Bootstrap respeta las clases utilitarias junto a los componentes
- Cambia el `variant` del `Button` (mostrar alerta) a `"outline-secondary"` — el botón adopta estilo de contorno gris en lugar del sólido azul

---

### `src/lab/LabRbCard.tsx`

```tsx
// src/lab/LabRbCard.tsx

import { Container, Card, Button, Row, Col, Badge } from 'react-bootstrap'

interface ProductCardProps {
  title:    string
  price:    string
  category: string
  active:   boolean
}

function ProductCard({ title, price, category, active }: ProductCardProps) {
  return (
    <Card className="h-100 shadow-sm">
      <Card.Body>
        <Card.Title className="fw-bold">{title}</Card.Title>
        <Card.Subtitle className="mb-2 text-muted">{category}</Card.Subtitle>
        <Card.Text>
          <Badge bg={active ? 'success' : 'secondary'} className="me-2">
            {active ? 'Activo' : 'Inactivo'}
          </Badge>
          <strong>{price}</strong>
        </Card.Text>
        <Button variant="primary" size="sm">Ver detalle</Button>
      </Card.Body>
    </Card>
  )
}

export default function LabRbCard() {
  const products = [
    { title: 'Teclado mecánico',  price: '$89.99',  category: 'Periféricos', active: true  },
    { title: 'Monitor 27"',       price: '$349.99', category: 'Pantallas',   active: true  },
    { title: 'Mouse inalámbrico', price: '$29.99',  category: 'Periféricos', active: false },
  ]

  return (
    <Container className="py-4">
      <h2 className="h4 fw-bold mb-1">LAB: Cards</h2>
      <p className="text-secondary mb-3">Grid responsivo con cards tipadas.</p>
      <Row className="g-3">
        {products.map(p => (
          <Col key={p.title} xs={12} sm={6} md={4}>
            <ProductCard {...p} />
          </Col>
        ))}
      </Row>
    </Container>
  )
}
```

### Prueba esto

- Cambia `md={4}` a `md={6}` en `Col` — cada tarjeta ocupa la mitad del ancho en pantallas medianas; el grid pasa de 3 a 2 columnas
- Cambia `md={4}` a `md={12}` — todas las tarjetas ocupan el ancho completo, apiladas verticalmente incluso en pantallas medianas
- Redimensiona el navegador a ancho móvil — con `xs={12}` las columnas ya están a 100%; el grid se adapta sin media queries adicionales
- Cambia `className="shadow-sm"` a `className="shadow"` en `Card` — la sombra se hace más pronunciada; Bootstrap tiene tres niveles: `shadow-sm`, `shadow` y `shadow-lg`
- Cambia `bg={active ? 'success' : 'secondary'}` a `bg={active ? 'primary' : 'warning'}` — el color del `Badge` cambia según el estado; Bootstrap tiene 8 variantes de color predefinidas
- Añade `className="border-0"` a `Card` — desaparece el borde de la tarjeta; el `shadow-sm` queda como única separación visual
- Cambia `g-3` a `g-5` en `Row` — el espacio entre tarjetas (gutter) aumenta; Bootstrap usa la misma escala de espaciado del 1 al 5

---

### `src/lab/LabRbForm.tsx`

```tsx
// src/lab/LabRbForm.tsx

import { useState } from 'react'
import { Container, Form, Button, Alert, Row, Col } from 'react-bootstrap'

interface FormValues {
  name:  string
  email: string
  role:  string
}

export default function LabRbForm() {
  const [values,  setValues]  = useState<FormValues>({ name: '', email: '', role: 'viewer' })
  const [success, setSuccess] = useState(false)
  const [errors,  setErrors]  = useState<Partial<FormValues>>({})

  function validate(): boolean {
    const e: Partial<FormValues> = {}
    if (!values.name.trim())         e.name  = 'El nombre es requerido'
    if (!values.email.includes('@'))  e.email = 'Email inválido'
    setErrors(e)
    return Object.keys(e).length === 0
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    if (!validate()) return
    setSuccess(true)
    setValues({ name: '', email: '', role: 'viewer' })
    setTimeout(() => setSuccess(false), 3000)
  }

  return (
    <Container className="py-4">
      <h2 className="h4 fw-bold mb-1">LAB: Form</h2>
      <p className="text-secondary mb-3">Con validación manual y feedback visual.</p>

      {success && <Alert variant="success">✅ Registro enviado correctamente</Alert>}

      <Form onSubmit={handleSubmit} style={{ maxWidth: 480 }}>
        <Row className="g-3">
          <Col xs={12}>
            <Form.Group>
              <Form.Label>Nombre completo</Form.Label>
              <Form.Control
                type="text"
                value={values.name}
                onChange={e => setValues(v => ({ ...v, name: e.target.value }))}
                isInvalid={!!errors.name}
                placeholder="Ana García"
              />
              <Form.Control.Feedback type="invalid">
                {errors.name}
              </Form.Control.Feedback>
            </Form.Group>
          </Col>

          <Col xs={12}>
            <Form.Group>
              <Form.Label>Correo electrónico</Form.Label>
              <Form.Control
                type="email"
                value={values.email}
                onChange={e => setValues(v => ({ ...v, email: e.target.value }))}
                isInvalid={!!errors.email}
                placeholder="ana@ejemplo.com"
              />
              <Form.Control.Feedback type="invalid">
                {errors.email}
              </Form.Control.Feedback>
            </Form.Group>
          </Col>

          <Col xs={12}>
            <Form.Group>
              <Form.Label>Rol</Form.Label>
              <Form.Select
                value={values.role}
                onChange={e => setValues(v => ({ ...v, role: e.target.value }))}
              >
                <option value="viewer">Viewer</option>
                <option value="editor">Editor</option>
                <option value="admin">Admin</option>
              </Form.Select>
            </Form.Group>
          </Col>

          <Col xs={12}>
            <Button type="submit" variant="primary">Registrar</Button>
          </Col>
        </Row>
      </Form>
    </Container>
  )
}
```

> `isInvalid` en `Form.Control` activa el estilo rojo de Bootstrap y muestra
> `Form.Control.Feedback type="invalid"` automáticamente — sin CSS adicional.

### Prueba esto

- Envía el formulario con campos vacíos — los bordes de los campos inválidos se vuelven rojos y aparecen los mensajes de error; Bootstrap lo gestiona con `isInvalid` sin CSS extra
- Rellena el campo de email con un texto sin `@` y envía — solo ese campo muestra error; la validación es por campo independiente
- Cambia `variant="primary"` del botón submit a `variant="success"` — el botón de envío se vuelve verde; útil para indicar una acción positiva
- Añade `size="lg"` a `Form.Control` del campo de nombre — el input crece; Bootstrap ajusta altura y tamaño de fuente coherentemente
- Cambia `type="submit"` por `type="button"` en el botón — el formulario ya no se envía al hacer clic; `handleSubmit` no se llama
- Cambia el `style={{ maxWidth: 480 }}` del `Form` a `style={{ maxWidth: 720 }}` — el formulario se ensancha; los campos ocupan más ancho en pantallas grandes

---

### `src/lab/LabRbTable.tsx`

```tsx
// src/lab/LabRbTable.tsx

import { useState } from 'react'
import { Container, Table, Badge, Form, InputGroup } from 'react-bootstrap'

interface Product {
  id:       number
  name:     string
  category: string
  price:    number
  active:   boolean
}

const PRODUCTS: Product[] = [
  { id: 1, name: 'Teclado mecánico',   category: 'Periféricos', price: 89.99,  active: true  },
  { id: 2, name: 'Monitor 27"',        category: 'Pantallas',   price: 349.99, active: true  },
  { id: 3, name: 'Mouse inalámbrico',  category: 'Periféricos', price: 29.99,  active: false },
  { id: 4, name: 'Webcam HD',          category: 'Cámaras',     price: 59.99,  active: true  },
  { id: 5, name: 'Auriculares BT',     category: 'Audio',       price: 149.99, active: false },
]

export default function LabRbTable() {
  const [search, setSearch] = useState('')

  const filtered = PRODUCTS.filter(p =>
    p.name.toLowerCase().includes(search.toLowerCase()) ||
    p.category.toLowerCase().includes(search.toLowerCase())
  )

  return (
    <Container className="py-4">
      <h2 className="h4 fw-bold mb-1">LAB: Table</h2>
      <p className="text-secondary mb-3">
        Striped, hover, responsive y búsqueda en tiempo real.
      </p>

      <InputGroup className="mb-3" style={{ maxWidth: 340 }}>
        <Form.Control
          placeholder="Buscar producto o categoría..."
          value={search}
          onChange={e => setSearch(e.target.value)}
        />
      </InputGroup>

      <Table striped bordered hover responsive>
        <thead className="table-dark">
          <tr>
            <th>#</th>
            <th>Nombre</th>
            <th>Categoría</th>
            <th className="text-end">Precio</th>
            <th>Estado</th>
          </tr>
        </thead>
        <tbody>
          {filtered.map(p => (
            <tr key={p.id}>
              <td>{p.id}</td>
              <td className="fw-semibold">{p.name}</td>
              <td>{p.category}</td>
              <td className="text-end">${p.price.toFixed(2)}</td>
              <td>
                <Badge bg={p.active ? 'success' : 'secondary'}>
                  {p.active ? 'Activo' : 'Inactivo'}
                </Badge>
              </td>
            </tr>
          ))}
          {filtered.length === 0 && (
            <tr>
              <td colSpan={5} className="text-center text-muted">
                Sin resultados.
              </td>
            </tr>
          )}
        </tbody>
      </Table>
    </Container>
  )
}
```

### Prueba esto

- Escribe "peri" en el buscador — la tabla filtra en tiempo real mostrando solo los productos de la categoría Periféricos; el estado de React actualiza `filtered` en cada pulsación
- Escribe una cadena sin coincidencias como "xyz" — aparece la fila "Sin resultados." con `colSpan={5}` que ocupa toda la fila
- Quita la prop `striped` de `Table` — las filas dejan de alternar color de fondo; Bootstrap aplica el rayado con `table-striped`
- Quita la prop `hover` de `Table` — las filas ya no cambian de color al pasar el cursor; Bootstrap lo gestiona con `table-hover`
- Cambia `thead className="table-dark"` a `className="table-light"` — el encabezado cambia de fondo oscuro a gris claro
- Cambia `bg={p.active ? 'success' : 'secondary'}` a `bg={p.active ? 'primary' : 'danger'}` — los badges cambian a azul/rojo en lugar de verde/gris
- Redimensiona el navegador a ancho estrecho — la tabla no desborda gracias a la prop `responsive`; aparece scroll horizontal en el contenedor

---

### `src/AppLab.tsx` — selector del lab

```tsx
// src/AppLab.tsx

import { useState } from 'react'
import LabRbButtons from './lab/LabRbButtons'
import LabRbAlert   from './lab/LabRbAlert'
import LabRbCard    from './lab/LabRbCard'
import LabRbForm    from './lab/LabRbForm'
import LabRbTable   from './lab/LabRbTable'

type LabKey = 'buttons' | 'alert' | 'card' | 'form' | 'table'

export default function AppLab() {
  const [lab, setLab] = useState<LabKey>('buttons')

  return (
    <div>
      <div className="border-bottom bg-light px-3 py-2 d-flex align-items-center gap-3">
        <span className="fw-bold">React-Bootstrap LAB</span>
        <select
          className="form-select form-select-sm"
          style={{ maxWidth: 180 }}
          value={lab}
          onChange={e => setLab(e.target.value as LabKey)}
        >
          <option value="buttons">Buttons</option>
          <option value="alert">Alert</option>
          <option value="card">Cards</option>
          <option value="form">Form</option>
          <option value="table">Table</option>
        </select>
      </div>

      {lab === 'buttons' && <LabRbButtons />}
      {lab === 'alert'   && <LabRbAlert />}
      {lab === 'card'    && <LabRbCard />}
      {lab === 'form'    && <LabRbForm />}
      {lab === 'table'   && <LabRbTable />}
    </div>
  )
}
```

### Prueba esto

- Cambia el valor inicial de `useState<LabKey>('buttons')` a `'table'` — la tabla se muestra al cargar en lugar de los botones
- Cambia `className="bg-light"` a `className="bg-dark"` en la barra superior — el fondo se vuelve oscuro; ajusta también `className="fw-bold"` a `className="fw-bold text-white"` para que el texto sea visible
- Añade una nueva opción al `select`: `<option value="alert">Alert 2</option>` — aparece en el selector pero no renderiza nada porque no hay rama condicional para ese valor
- Cambia `style={{ maxWidth: 180 }}` a `style={{ maxWidth: 300 }}` — el select se ensancha; útil si los nombres de los labs son más largos
- Cambia `className="form-select form-select-sm"` a `className="form-select"` — el select adopta el tamaño estándar de Bootstrap en lugar del pequeño

---

## Fase 2 — Home: landing de cursos con Router

---

### `src/components/rb/RBNavbar.tsx`

```tsx
// src/components/rb/RBNavbar.tsx

import { NavLink }                       from 'react-router-dom'
import { Navbar, Container, Nav }        from 'react-bootstrap'

export default function RBNavbar() {
  return (
    <Navbar bg="dark" data-bs-theme="dark" expand="lg" className="border-bottom border-secondary">
      <Container>
        <Navbar.Brand as={NavLink} to="/" className="fw-bold">
          DevCursos
        </Navbar.Brand>
        <Navbar.Toggle aria-controls="main-nav" />
        <Navbar.Collapse id="main-nav">
          <Nav className="ms-auto gap-1">
            <Nav.Link as={NavLink} to="/" end>Inicio</Nav.Link>
            <Nav.Link as={NavLink} to="/about">Acerca de</Nav.Link>
          </Nav>
        </Navbar.Collapse>
      </Container>
    </Navbar>
  )
}
```

> `data-bs-theme="dark"` activa el tema oscuro de Bootstrap 5 en el Navbar.
> `as={NavLink}` delega la navegación a React Router — sin recargar la página.

### Prueba esto

- Cambia `bg="dark"` a `bg="light"` y elimina `data-bs-theme="dark"` — el navbar adopta fondo claro con texto oscuro automáticamente
- Cambia `expand="lg"` a `expand="md"` — el menú hamburguesa aparece antes; en pantallas medianas el navbar ya colapsa
- Cambia `expand="lg"` a `expand={false}` — el navbar nunca expande en horizontal; el botón hamburguesa siempre está visible
- Añade `sticky="top"` a `Navbar` — el navbar queda fijo en la parte superior al hacer scroll
- Cambia `className="ms-auto gap-1"` del `Nav` a `className="mx-auto gap-3"` — los enlaces se centran en lugar de alinearse a la derecha
- Añade otro `Nav.Link as={NavLink}` con `to="/courses"` — aparece un tercer enlace en el navbar; React Router lo gestiona sin configuración extra

---

### `src/components/rb/RBHero.tsx`

```tsx
// src/components/rb/RBHero.tsx

import { Container, Button, Stack, Badge } from 'react-bootstrap'

export default function RBHero() {
  return (
    <section className="py-5 bg-dark text-white">
      <Container>
        <Badge bg="primary" className="mb-3">React-Bootstrap v2</Badge>
        <h1 className="display-5 fw-bold mb-3">
          Aprende React con componentes probados
        </h1>
        <p className="lead text-white-50 mb-4" style={{ maxWidth: 560 }}>
          Componentes Bootstrap nativos en React. Sin clases manuales,
          sin jQuery, sin conflictos.
        </p>
        <Stack direction="horizontal" gap={2} className="flex-wrap">
          <Button variant="primary"      size="lg">Ver cursos</Button>
          <Button variant="outline-light" size="lg">Plan de estudio</Button>
        </Stack>
      </Container>
    </section>
  )
}
```

### Prueba esto

- Cambia `className="py-5 bg-dark text-white"` a `className="py-5 bg-primary text-white"` — el hero adopta el fondo azul de Bootstrap en lugar del oscuro
- Cambia `bg="primary"` del `Badge` a `bg="warning"` — la etiqueta de versión se vuelve amarilla; útil para resaltar contenido nuevo o en beta
- Cambia `className="display-5 fw-bold mb-3"` a `className="display-3 fw-bold mb-3"` — el título crece; Bootstrap tiene clases `display-1` a `display-6`
- Cambia `variant="outline-light"` a `variant="outline-warning"` en el segundo botón — el borde y el texto se vuelven amarillos sobre el fondo oscuro
- Cambia `size="lg"` a `size="sm"` en ambos botones — los botones del hero se vuelven pequeños; el contraste visual con el encabezado cambia
- Añade `className="py-5 py-md-6"` — en pantallas medianas y mayores el padding vertical aumenta; Bootstrap no tiene `py-6` por defecto, pero puedes añadirlo con variables CSS

---

### `src/components/rb/RBCourseGrid.tsx`

```tsx
// src/components/rb/RBCourseGrid.tsx

import { Container, Row, Col, Card, Button, Badge } from 'react-bootstrap'

interface Course {
  id:       number
  title:    string
  level:    'Básico' | 'Intermedio' | 'Avanzado'
  duration: string
  tag:      string
}

const COURSES: Course[] = [
  { id: 1, title: 'React + TypeScript',   level: 'Básico',     duration: '12h', tag: 'Nuevo'   },
  { id: 2, title: 'Hooks avanzados',      level: 'Intermedio', duration: '8h',  tag: 'Popular' },
  { id: 3, title: 'Testing con Vitest',   level: 'Avanzado',   duration: '6h',  tag: ''        },
  { id: 4, title: 'TanStack Query',       level: 'Intermedio', duration: '5h',  tag: 'Nuevo'   },
  { id: 5, title: 'React Router v7',      level: 'Básico',     duration: '4h',  tag: ''        },
  { id: 6, title: 'Performance patterns', level: 'Avanzado',   duration: '7h',  tag: 'Popular' },
]

const LEVEL_COLOR: Record<Course['level'], string> = {
  Básico:     'success',
  Intermedio: 'warning',
  Avanzado:   'danger',
}

export default function RBCourseGrid() {
  return (
    <section className="py-5">
      <Container>
        <h2 className="fw-bold mb-1">Cursos disponibles</h2>
        <p className="text-muted mb-4">Selecciona el nivel que mejor se adapte a ti.</p>

        <Row className="g-3">
          {COURSES.map(course => (
            <Col key={course.id} xs={12} sm={6} lg={4}>
              <Card className="h-100 shadow-sm">
                <Card.Body className="d-flex flex-column">
                  <div className="d-flex justify-content-between align-items-start mb-2">
                    <Badge bg={LEVEL_COLOR[course.level]}>{course.level}</Badge>
                    {course.tag && <Badge bg="dark">{course.tag}</Badge>}
                  </div>
                  <Card.Title className="fw-bold">{course.title}</Card.Title>
                  <Card.Text className="text-muted flex-grow-1">
                    Duración: {course.duration}
                  </Card.Text>
                  <Button variant="outline-primary" size="sm" className="mt-auto">
                    Ver curso
                  </Button>
                </Card.Body>
              </Card>
            </Col>
          ))}
        </Row>
      </Container>
    </section>
  )
}
```

### Prueba esto

- Cambia `lg={4}` a `lg={3}` en `Col` — el grid pasa de 3 a 4 columnas en pantallas grandes; con 6 cursos quedan 2 filas de 3 y una fila de 0 (Bootstrap ajusta el wrap automáticamente)
- Cambia `sm={6}` a `sm={12}` — en pantallas medianas las tarjetas se apilan en lugar de ir de 2 en 2
- Cambia `LEVEL_COLOR.Básico` de `'success'` a `'info'` — los cursos básicos se muestran con badge azul claro en lugar de verde
- Cambia `variant="outline-primary"` del botón a `variant="primary"` — el botón pasa de contorno a relleno sólido
- Cambia `g-3` a `g-4` en `Row` — los gutters entre tarjetas aumentan; los cursos tienen más aire
- Cambia `className="h-100 shadow-sm"` a `className="h-100 shadow border-0"` — las tarjetas pierden el borde Bootstrap y solo se separan con sombra
- Añade `className="bg-light"` a `<section>` — el fondo de la sección cambia a gris claro de Bootstrap; contrasta con el hero oscuro

---

### `src/components/rb/RBNewsletter.tsx`

```tsx
// src/components/rb/RBNewsletter.tsx

import { useState } from 'react'
import { Container, Form, Button, Alert, Row, Col } from 'react-bootstrap'

export default function RBNewsletter() {
  const [email,   setEmail]   = useState('')
  const [success, setSuccess] = useState(false)

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    setSuccess(true)
    setEmail('')
    setTimeout(() => setSuccess(false), 3000)
  }

  return (
    <section className="py-5 bg-light border-top">
      <Container>
        <Row className="justify-content-center">
          <Col xs={12} md={8} lg={6} className="text-center">
            <h2 className="fw-bold mb-2">Mantente actualizado</h2>
            <p className="text-muted mb-4">
              Recibe novedades sobre nuevos cursos y contenido exclusivo.
            </p>
            {success && (
              <Alert variant="success">✅ Suscripción registrada (demo)</Alert>
            )}
            <Form onSubmit={handleSubmit}>
              <Row className="g-2">
                <Col>
                  <Form.Control
                    type="email"
                    placeholder="tu@correo.com"
                    value={email}
                    onChange={e => setEmail(e.target.value)}
                    required
                  />
                </Col>
                <Col xs="auto">
                  <Button type="submit" variant="primary">Suscribirme</Button>
                </Col>
              </Row>
            </Form>
          </Col>
        </Row>
      </Container>
    </section>
  )
}
```

### Prueba esto

- Envía el formulario con un email válido — el alert de éxito aparece durante 3 segundos y desaparece; el `setTimeout` limpia el estado automáticamente
- Cambia `variant="success"` del `Alert` a `variant="info"` — el mensaje de confirmación cambia de verde a azul
- Cambia `lg={6}` a `lg={8}` en `Col` — la sección de newsletter se ensancha en pantallas grandes
- Cambia `className="py-5 bg-light border-top"` a `className="py-5 bg-dark text-white border-top"` — la sección adopta fondo oscuro; ajusta también `className="text-muted mb-4"` a `className="text-white-50 mb-4"` para que el subtítulo sea visible
- Cambia `variant="primary"` del botón a `variant="success"` — el botón de suscripción se vuelve verde, sugiriendo una acción positiva
- Elimina el atributo `required` del `Form.Control` — el formulario se envía aunque el campo esté vacío; el navegador ya no bloquea el submit

---

### `src/components/rb/RBFooter.tsx`

```tsx
// src/components/rb/RBFooter.tsx

import { Container, Row, Col } from 'react-bootstrap'

export default function RBFooter() {
  return (
    <footer className="py-4 bg-dark text-white border-top border-secondary">
      <Container>
        <Row className="align-items-center g-2">
          <Col xs={12} md="auto">
            <span className="fw-bold">DevCursos</span>
          </Col>
          <Col className="text-md-center text-muted small">
            Construido con React-Bootstrap v2 + Bootstrap 5
          </Col>
          <Col xs={12} md="auto" className="d-flex gap-3 justify-content-end">
            <a href="#" className="text-white-50 text-decoration-none small">Docs</a>
            <a href="#" className="text-white-50 text-decoration-none small">GitHub</a>
          </Col>
        </Row>
      </Container>
    </footer>
  )
}
```

---

### `src/pages/HomeRB.tsx`

```tsx
// src/pages/HomeRB.tsx

import RBHero       from '../components/rb/RBHero'
import RBCourseGrid from '../components/rb/RBCourseGrid'
import RBNewsletter from '../components/rb/RBNewsletter'

export default function HomeRB() {
  return (
    <>
      <RBHero />
      <RBCourseGrid />
      <RBNewsletter />
    </>
  )
}
```

---

### `src/pages/AboutRB.tsx`

```tsx
// src/pages/AboutRB.tsx

import { Container, Card, ListGroup } from 'react-bootstrap'

export default function AboutRB() {
  return (
    <Container className="py-5" style={{ maxWidth: 600 }}>
      <h1 className="h3 fw-bold mb-4">Acerca de este proyecto</h1>
      <Card className="shadow-sm">
        <Card.Header className="fw-semibold">Stack utilizado</Card.Header>
        <ListGroup variant="flush">
          <ListGroup.Item>React 19 + TypeScript</ListGroup.Item>
          <ListGroup.Item>React-Bootstrap v2 + Bootstrap 5</ListGroup.Item>
          <ListGroup.Item>React Router v7</ListGroup.Item>
          <ListGroup.Item>Vite 8</ListGroup.Item>
        </ListGroup>
      </Card>
    </Container>
  )
}
```

---

### `src/AppHome.tsx`

```tsx
// src/AppHome.tsx

import { BrowserRouter, Routes, Route } from 'react-router-dom'
import RBNavbar from './components/rb/RBNavbar'
import RBFooter from './components/rb/RBFooter'
import HomeRB   from './pages/HomeRB'
import AboutRB  from './pages/AboutRB'

export default function AppHome() {
  return (
    <BrowserRouter>
      <RBNavbar />
      <Routes>
        <Route path="/"      element={<HomeRB />} />
        <Route path="/about" element={<AboutRB />} />
      </Routes>
      <RBFooter />
    </BrowserRouter>
  )
}
```

---

### `src/App.tsx` — alterna entre fases

```tsx
// src/App.tsx

import AppLab  from './AppLab'
// import AppHome from './AppHome'

export default function App() {
  return <AppLab />
  // Fase 2 — descomenta y comenta la línea anterior:
  // return <AppHome />
}
```

### Prueba esto

- Comienza con `return <AppLab />` — usa el dropdown de `AppLab` para explorar los 5 componentes Bootstrap
- Cuando termines el lab, comenta `return <AppLab />` y descomenta `return <AppHome />` — ve la landing con Router
- Navega entre "/" y "/about" para verificar que React Router y el navbar de Bootstrap funcionan juntos
- Vuelve a la Fase 1 comentando `AppHome` y descomentando `AppLab` — el patrón funciona en ambas direcciones

---

## Componentes más utilizados — referencia rápida

| Componente | Import | Uso típico |
|---|---|---|
| `Button` | `react-bootstrap` | Acciones, variantes, tamaños |
| `Alert` | `react-bootstrap` | Mensajes con `dismissible` |
| `Card` | `react-bootstrap` | Contenedores de contenido |
| `Form`, `Form.Control` | `react-bootstrap` | Inputs con `isInvalid` |
| `Form.Select` | `react-bootstrap` | Select nativo Bootstrap |
| `Table` | `react-bootstrap` | Tablas con `striped`, `hover` |
| `Badge` | `react-bootstrap` | Etiquetas de estado |
| `Row`, `Col` | `react-bootstrap` | Grid responsive de 12 columnas |
| `Container` | `react-bootstrap` | Contenedor centrado con máx. ancho |
| `Stack` | `react-bootstrap` | Flex horizontal o vertical con `gap` |
| `Navbar`, `Nav` | `react-bootstrap` | Barra de navegación responsive |

---

## Resumen de la página 16

- El CSS de Bootstrap se importa **una vez** en `main.tsx` — no en cada componente.
- Los componentes usan props en lugar de clases: `variant="primary"` en vez de `className="btn-primary"`.
- `as={NavLink}` en `Nav.Link` y `Navbar.Brand` delega la navegación a React Router sin perder el estado de la app.
- `data-bs-theme="dark"` en `Navbar` activa el tema oscuro nativo de Bootstrap 5.
- `Form.Control` con `isInvalid` + `Form.Control.Feedback` maneja la validación visualmente sin CSS adicional.
- `Row` + `Col` con `xs`, `sm`, `md`, `lg` implementan el grid responsive de 12 columnas de Bootstrap.
- El patrón LAB → HOME: primero practica cada componente en aislamiento, luego los ensambla en una página real.

---

> **Siguiente página →** Página 17: Tailwind CSS v4 — LAB + landing de cursos.