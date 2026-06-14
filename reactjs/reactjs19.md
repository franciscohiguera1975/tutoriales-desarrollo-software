# Curso React.js + TypeScript — Página 18
## Frameworks de estilos · Ant Design v6
### LAB: 5 componentes aislados — HOME: dashboard de ventas con Router

---

## ¿Qué es Ant Design?

Ant Design es un sistema de diseño empresarial de Alibaba con componentes
React completos y listos para producción. Su fortaleza son los componentes
de datos: tablas con sorting y filtrado, formularios con validación
integrada, estadísticas y gráficos.

```
Bootstrap / Tailwind          Ant Design
─────────────────             ────────────────────────────────────────
Estilo → tú lo construyes     Componentes completos out-of-the-box
Table → HTML + clases         Table → sorting, filtering, paginación incluidos
Form → inputs + validación    Form → Form.useForm(), rules, feedback automático
```

---

## Instalación

```bash
npm install antd
```

| Paquete | Versión actual |
|---|---|
| `antd` | 6.3.3 |

No necesitas instalar `@ant-design/icons` por separado — los iconos
están incluidos en el paquete principal de antd v6.

---

## `src/main.tsx`

```tsx
// src/main.tsx

import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import 'antd/dist/reset.css'   // ← reset CSS de Ant Design — importar una sola vez
import App from './App.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

> `antd/dist/reset.css` normaliza los estilos base del navegador.
> Se importa **una sola vez** en `main.tsx`. No en cada componente.

---

## Estructura del proyecto

```
src/
├── lab/
│   ├── LabAntButtons.tsx
│   ├── LabAntAlert.tsx
│   ├── LabAntCard.tsx
│   ├── LabAntForm.tsx
│   └── LabAntTable.tsx
├── components/antd/
│   ├── AntNavbar.tsx
│   ├── AntKpis.tsx
│   ├── AntProgress.tsx
│   ├── AntSalesTable.tsx
│   └── AntFooter.tsx
├── pages/
│   ├── HomeDashboard.tsx
│   └── AboutPage.tsx
├── App.tsx
├── AppLab.tsx
├── AppHome.tsx
└── main.tsx
```

---

## Fase 1 — Laboratorio

---

### `src/lab/LabAntButtons.tsx`

```tsx
// src/lab/LabAntButtons.tsx

import { Button, Space, Typography } from 'antd'

const { Title, Text } = Typography

export default function LabAntButtons() {
  return (
    <div style={{ maxWidth: 800, margin: '0 auto', padding: 24 }}>
      <Title level={4} style={{ marginBottom: 4 }}>LAB: Buttons</Title>
      <Text type="secondary" style={{ display: 'block', marginBottom: 20 }}>
        Tipos, tamaños y estados.
      </Text>

      <Text strong style={{ display: 'block', marginBottom: 8 }}>Tipos</Text>
      <Space wrap style={{ marginBottom: 20 }}>
        <Button type="primary">Primary</Button>
        <Button>Default</Button>
        <Button type="dashed">Dashed</Button>
        <Button type="link">Link</Button>
        <Button type="text">Text</Button>
        <Button danger>Danger</Button>
        <Button type="primary" danger>Primary Danger</Button>
      </Space>

      <Text strong style={{ display: 'block', marginBottom: 8 }}>Tamaños</Text>
      <Space wrap style={{ marginBottom: 20 }}>
        <Button type="primary" size="large">Grande</Button>
        <Button type="primary">Normal</Button>
        <Button type="primary" size="small">Pequeño</Button>
      </Space>

      <Text strong style={{ display: 'block', marginBottom: 8 }}>Estados</Text>
      <Space wrap>
        <Button type="primary" loading>Cargando</Button>
        <Button type="primary" disabled>Deshabilitado</Button>
      </Space>
    </div>
  )
}
```

### Prueba esto

- Cambia `type="primary"` a `type="dashed"` en uno de los botones — el botón cambia a un estilo con borde punteado y fondo transparente
- Añade `shape="round"` al botón Primary — las esquinas se vuelven completamente redondeadas; prueba también `shape="circle"` con un solo carácter de texto
- Cambia `size="large"` a `size="small"` en la sección de tamaños — los tres botones se ajustan; AntD tiene exactamente tres tamaños: `small`, `middle` (default) y `large`
- Añade `ghost` al `<Button type="primary">` — el fondo sólido desaparece y queda solo el borde y el texto en color primario; útil sobre fondos de color
- Quita `loading` del botón de estado y ponlo en `false` explícitamente — el spinner desaparece; `loading` es un booleano que AntD gestiona directamente en el componente
- Envuelve todos los botones con `<ConfigProvider theme={{ token: { colorPrimary: '#52c41a' } }}>` — todos los botones de tipo `primary` cambian a verde sin modificar ningún componente individual

---

### `src/lab/LabAntAlert.tsx`

```tsx
// src/lab/LabAntAlert.tsx

import { useState } from 'react'
import { Alert, Button, Space, Typography } from 'antd'

const { Title, Text } = Typography

export default function LabAntAlert() {
  const [show, setShow] = useState(true)

  return (
    <div style={{ maxWidth: 800, margin: '0 auto', padding: 24 }}>
      <Title level={4} style={{ marginBottom: 4 }}>LAB: Alert</Title>
      <Text type="secondary" style={{ display: 'block', marginBottom: 20 }}>
        Las cuatro variantes y control de visibilidad.
      </Text>

      <Space direction="vertical" style={{ width: '100%', marginBottom: 20 }}>
        <Alert message="Información de referencia." type="info"    showIcon />
        <Alert message="Operación exitosa."         type="success" showIcon />
        <Alert message="Advertencia importante."    type="warning" showIcon />
        <Alert message="Se produjo un error."       type="error"   showIcon />
      </Space>

      <Text strong style={{ display: 'block', marginBottom: 8 }}>Con closable</Text>
      <Space direction="vertical" style={{ width: '100%', marginBottom: 16 }}>
        {show && (
          <Alert
            message="Alerta con cierre"
            description="Haz clic en la X para cerrar esta alerta."
            type="success"
            showIcon
            closable
            onClose={() => setShow(false)}
          />
        )}
      </Space>
      <Button onClick={() => setShow(true)}>Mostrar alerta</Button>
    </div>
  )
}
```

### Prueba esto

- Cambia `type="info"` a `type="success"` en la primera alerta — el icono y el color de fondo cambian al verde de éxito; AntD tiene cuatro tipos: `info`, `success`, `warning` y `error`
- Quita `showIcon` de una alerta — el icono desaparece pero el color de fondo se mantiene; `showIcon` es opcional y añade el icono semántico de cada tipo
- Añade `description="Texto de descripción adicional."` a una alerta — aparece un texto secundario más pequeño bajo el mensaje principal; útil para detalles de error
- Añade `banner` a una alerta — elimina el borde redondeado y los márgenes, adecuado para mostrar una alerta que ocupa todo el ancho de la pantalla
- Cambia `onClose={() => setShow(false)}` por `afterClose={() => console.log('cerrada')}` — `afterClose` se ejecuta después de la animación de cierre; verifica en la consola del navegador
- Añade `action={<Button size="small" type="link">Acción</Button>}` a una alerta — aparece un botón de acción alineado a la derecha de la alerta

---

### `src/lab/LabAntCard.tsx`

```tsx
// src/lab/LabAntCard.tsx

import { Card, Tag, Space, Typography, Row, Col } from 'antd'

const { Title, Text } = Typography

interface ProductCardProps {
  name:     string
  category: string
  price:    number
  active:   boolean
}

function ProductCard({ name, category, price, active }: ProductCardProps) {
  return (
    <Card
      hoverable
      styles={{ body: { display: 'flex', flexDirection: 'column', gap: 8 } }}
    >
      <Text strong>{name}</Text>
      <Text type="secondary" style={{ fontSize: 13 }}>{category}</Text>
      <Space>
        <Tag color={active ? 'success' : 'default'}>
          {active ? 'Activo' : 'Inactivo'}
        </Tag>
        <Text strong>${price.toFixed(2)}</Text>
      </Space>
    </Card>
  )
}

export default function LabAntCard() {
  const products: ProductCardProps[] = [
    { name: 'Teclado mecánico',  category: 'Periféricos', price: 89.99,  active: true  },
    { name: 'Monitor 27"',       category: 'Pantallas',   price: 349.99, active: true  },
    { name: 'Mouse inalámbrico', category: 'Periféricos', price: 29.99,  active: false },
  ]

  return (
    <div style={{ maxWidth: 800, margin: '0 auto', padding: 24 }}>
      <Title level={4} style={{ marginBottom: 4 }}>LAB: Cards</Title>
      <Text type="secondary" style={{ display: 'block', marginBottom: 20 }}>
        Cards tipadas con grid de Ant Design.
      </Text>
      <Row gutter={[16, 16]}>
        {products.map(p => (
          <Col key={p.name} xs={24} sm={12} md={8}>
            <ProductCard {...p} />
          </Col>
        ))}
      </Row>
    </div>
  )
}
```

> En antd v6 `Card.body` se pasa mediante la prop `styles={{ body: {...} }}`
> en lugar del antiguo `bodyStyle`.

### Prueba esto

- Quita `hoverable` de la `<Card>` — el efecto de elevación al pasar el cursor desaparece; `hoverable` añade una sombra sutil en hover sin necesidad de CSS
- Añade `title="Producto destacado"` a la `<Card>` — aparece una cabecera con borde inferior; también puedes pasar JSX para personalizar el encabezado
- Cambia `color={active ? 'success' : 'default'}` a `color={active ? 'blue' : 'red'}` en el `<Tag>` — AntD acepta colores predefinidos y también colores hexadecimales directamente en `color`
- Añade `extra={<Button type="link" size="small">Ver más</Button>}` a la `<Card>` — aparece un enlace alineado a la derecha del título de la tarjeta
- Cambia `xs={24} sm={12} md={8}` a `xs={24} sm={24} md={12}` en el `<Col>` — las tarjetas ocupan la mitad del ancho en escritorio en lugar de un tercio
- Añade `bordered={false}` a la `<Card>` y un `style={{ background: '#f5f5f5' }}` — la tarjeta pierde el borde y se funde con un fondo gris suave

---

### `src/lab/LabAntForm.tsx`

```tsx
// src/lab/LabAntForm.tsx

import { useState } from 'react'
import { Form, Input, Select, Button, Alert, Typography } from 'antd'

const { Title, Text } = Typography
const { Option } = Select

interface FormValues {
  name:  string
  email: string
  role:  string
}

export default function LabAntForm() {
  const [success, setSuccess] = useState(false)
  const [form] = Form.useForm<FormValues>()  // instancia tipada del formulario

  function onFinish(_values: FormValues) {
    setSuccess(true)
    form.resetFields()
    setTimeout(() => setSuccess(false), 3000)
  }

  return (
    <div style={{ maxWidth: 800, margin: '0 auto', padding: 24 }}>
      <Title level={4} style={{ marginBottom: 4 }}>LAB: Form</Title>
      <Text type="secondary" style={{ display: 'block', marginBottom: 20 }}>
        Con Form.useForm, validación integrada y feedback.
      </Text>

      {success && (
        <Alert
          message="Registro enviado correctamente"
          type="success"
          showIcon
          style={{ marginBottom: 16 }}
        />
      )}

      <Form
        form={form}
        layout="vertical"
        onFinish={onFinish}
        style={{ maxWidth: 480 }}
      >
        <Form.Item
          label="Nombre completo"
          name="name"
          rules={[
            { required: true, message: 'El nombre es requerido' },
            { min: 2,         message: 'Mínimo 2 caracteres'    },
          ]}
        >
          <Input placeholder="Ana García" />
        </Form.Item>

        <Form.Item
          label="Correo electrónico"
          name="email"
          rules={[
            { required: true, message: 'El email es requerido'       },
            { type: 'email',  message: 'Formato de email inválido'   },
          ]}
        >
          <Input placeholder="ana@ejemplo.com" />
        </Form.Item>

        <Form.Item
          label="Rol"
          name="role"
          initialValue="viewer"
        >
          <Select>
            <Option value="viewer">Viewer</Option>
            <Option value="editor">Editor</Option>
            <Option value="admin">Admin</Option>
          </Select>
        </Form.Item>

        <Form.Item>
          <Button type="primary" htmlType="submit">Registrar</Button>
        </Form.Item>
      </Form>
    </div>
  )
}
```

> `Form.useForm<FormValues>()` tipa la instancia del form.
> `onFinish` recibe los valores ya validados y tipados — solo se llama
> cuando todas las `rules` pasan.

### Prueba esto

- Cambia `layout="vertical"` a `layout="horizontal"` en el `<Form>` — las etiquetas se posicionan a la izquierda de los inputs; prueba también `layout="inline"` para poner todos los campos en una sola fila
- Añade `{ max: 50, message: 'Máximo 50 caracteres' }` al array `rules` del campo nombre — AntD valida el límite automáticamente y muestra el mensaje sin código extra
- Cambia `htmlType="submit"` por `onClick={() => form.submit()}` en el botón — llama manualmente a la validación y envío; útil cuando el botón está fuera del `<Form>`
- Añade `validateTrigger="onBlur"` al `<Form>` — la validación ocurre al salir del campo en lugar de al escribir; reduce mensajes de error prematuros
- Usa `form.setFieldsValue({ name: 'Ana García', email: 'ana@test.com' })` en un botón separado — rellena el formulario programáticamente; útil para cargar datos de edición
- Añade `{ whitespace: true, message: 'No puede ser solo espacios' }` a las rules del nombre — AntD rechaza cadenas con solo espacios en blanco

---

### `src/lab/LabAntTable.tsx`

```tsx
// src/lab/LabAntTable.tsx

import { useState } from 'react'
import { Table, Tag, Input, Typography, Space } from 'antd'
import type { ColumnsType } from 'antd/es/table'

const { Title, Text } = Typography

interface Product {
  key:      number
  name:     string
  category: string
  price:    number
  active:   boolean
}

const PRODUCTS: Product[] = [
  { key: 1, name: 'Teclado mecánico',  category: 'Periféricos', price: 89.99,  active: true  },
  { key: 2, name: 'Monitor 27"',       category: 'Pantallas',   price: 349.99, active: true  },
  { key: 3, name: 'Mouse inalámbrico', category: 'Periféricos', price: 29.99,  active: false },
  { key: 4, name: 'Webcam HD',         category: 'Cámaras',     price: 59.99,  active: true  },
  { key: 5, name: 'Auriculares BT',    category: 'Audio',       price: 149.99, active: false },
]

// ColumnsType<T> — tipado completo de columnas con render y sorter
const COLUMNS: ColumnsType<Product> = [
  { title: '#', dataIndex: 'key', width: 50 },
  {
    title:     'Nombre',
    dataIndex: 'name',
    render:    (v: string) => <Text strong>{v}</Text>,
  },
  { title: 'Categoría', dataIndex: 'category' },
  {
    title:     'Precio',
    dataIndex: 'price',
    align:     'right',
    render:    (v: number) => <Text strong>${v.toFixed(2)}</Text>,
    sorter:    (a, b) => a.price - b.price,   // sorting integrado
  },
  {
    title:     'Estado',
    dataIndex: 'active',
    render:    (v: boolean) => (
      <Tag color={v ? 'success' : 'default'}>{v ? 'Activo' : 'Inactivo'}</Tag>
    ),
    filters: [
      { text: 'Activo',   value: true  },
      { text: 'Inactivo', value: false },
    ],
    onFilter: (value, record) => record.active === value,  // filtrado integrado
  },
]

export default function LabAntTable() {
  const [search, setSearch] = useState('')

  const filtered = PRODUCTS.filter(p =>
    p.name.toLowerCase().includes(search.toLowerCase()) ||
    p.category.toLowerCase().includes(search.toLowerCase())
  )

  return (
    <div style={{ maxWidth: 900, margin: '0 auto', padding: 24 }}>
      <Title level={4} style={{ marginBottom: 4 }}>LAB: Table</Title>
      <Text type="secondary" style={{ display: 'block', marginBottom: 16 }}>
        Con ColumnsType, sorting, filtering y búsqueda.
      </Text>
      <Space direction="vertical" style={{ width: '100%' }}>
        <Input.Search
          placeholder="Buscar producto o categoría..."
          allowClear
          onChange={e => setSearch(e.target.value)}
          style={{ maxWidth: 340 }}
        />
        <Table<Product>
          columns={COLUMNS}
          dataSource={filtered}
          pagination={{ pageSize: 4 }}
          size="middle"
        />
      </Space>
    </div>
  )
}
```

> `ColumnsType<T>` del import `antd/es/table` tipa el array de columnas completo.
> `sorter` y `filters`/`onFilter` son props de cada columna — no código externo.

### Prueba esto

- Haz clic en el encabezado "Precio" en la tabla renderizada — la columna ordena de menor a mayor y vuelve a hacer clic para invertir; el `sorter` está declarado solo en la columna, no en el componente
- Usa el menú de filtro en la columna "Estado" y selecciona "Activo" — la tabla filtra sin código extra; `filters` y `onFilter` en la columna lo gestionan solos
- Cambia `pagination={{ pageSize: 4 }}` a `pagination={{ pageSize: 2 }}` — la paginación se ajusta automáticamente al nuevo tamaño; AntD genera los controles de página
- Añade `defaultSortOrder: 'ascend'` a la columna de precio — la tabla carga ya ordenada por precio ascendente sin intervención del usuario
- Cambia `size="middle"` a `size="small"` en la `<Table>` — las filas se compactan; AntD tiene tres densidades: `small`, `middle` y `large`
- Añade `rowSelection={{ type: 'checkbox', onChange: (keys) => console.log(keys) }}` a la `<Table>` — aparecen checkboxes en cada fila y la selección se registra en consola

---

### `src/AppLab.tsx`

```tsx
// src/AppLab.tsx

import { useState } from 'react'
import { Layout, Menu } from 'antd'
import LabAntButtons from './lab/LabAntButtons'
import LabAntAlert   from './lab/LabAntAlert'
import LabAntCard    from './lab/LabAntCard'
import LabAntForm    from './lab/LabAntForm'
import LabAntTable   from './lab/LabAntTable'

const { Header, Content } = Layout

type LabKey = 'buttons' | 'alert' | 'card' | 'form' | 'table'

const ITEMS = [
  { key: 'buttons', label: 'Buttons' },
  { key: 'alert',   label: 'Alert'   },
  { key: 'card',    label: 'Cards'   },
  { key: 'form',    label: 'Form'    },
  { key: 'table',   label: 'Table'   },
]

export default function AppLab() {
  const [lab, setLab] = useState<LabKey>('buttons')

  return (
    <Layout style={{ minHeight: '100vh' }}>
      <Header style={{ display: 'flex', alignItems: 'center', gap: 16 }}>
        <span style={{ color: 'white', fontWeight: 700, whiteSpace: 'nowrap' }}>
          Ant Design v6 LAB
        </span>
        <Menu
          theme="dark"
          mode="horizontal"
          selectedKeys={[lab]}
          items={ITEMS}
          onClick={({ key }) => setLab(key as LabKey)}
          style={{ flex: 1 }}
        />
      </Header>
      <Content style={{ padding: '24px 0' }}>
        {lab === 'buttons' && <LabAntButtons />}
        {lab === 'alert'   && <LabAntAlert />}
        {lab === 'card'    && <LabAntCard />}
        {lab === 'form'    && <LabAntForm />}
        {lab === 'table'   && <LabAntTable />}
      </Content>
    </Layout>
  )
}
```

### Prueba esto

- Cambia `theme="dark"` a `theme="light"` en el `<Menu>` — el fondo del header pasa a blanco y el texto a oscuro; el color del `<Header>` es independiente del tema del `<Menu>`
- Cambia `mode="horizontal"` a `mode="inline"` en el `<Menu>` — el menú se convierte en una lista vertical; útil para probar cómo se vería en un sidebar
- Añade `<Sider width={200}>` en lugar del `<Header>` para crear un layout de sidebar — `Layout` de AntD soporta composición con `Sider`, `Header`, `Content` y `Footer`
- Cambia `padding: '24px 0'` en el `<Content>` a `padding: 24` — añade padding en todos los lados y el contenido de cada lab queda con más espacio en los laterales
- Añade un nuevo item al array `ITEMS`: `{ key: 'extra', label: 'Extra' }` — aparece en el menú; agrega el componente correspondiente en el bloque de renderizado
- Envuelve el `<Layout>` con `<ConfigProvider theme={{ token: { colorPrimary: '#eb2f96' } }}>` — el color primario de todo el lab cambia a rosa sin modificar ningún componente individual

---

## Fase 2 — Home: dashboard de ventas con Router

---

### `src/components/antd/AntNavbar.tsx`

```tsx
// src/components/antd/AntNavbar.tsx

import { NavLink }    from 'react-router-dom'
import { Layout, Menu } from 'antd'

const { Header } = Layout

export default function AntNavbar() {
  const items = [
    { key: 'home',  label: <NavLink to="/">Dashboard</NavLink>  },
    { key: 'about', label: <NavLink to="/about">Acerca de</NavLink> },
  ]

  return (
    <Header style={{ display: 'flex', alignItems: 'center', gap: 16 }}>
      <span style={{ color: 'white', fontWeight: 900, fontSize: 16, whiteSpace: 'nowrap' }}>
        SalesBoard
      </span>
      <Menu
        theme="dark"
        mode="horizontal"
        items={items}
        style={{ flex: 1 }}
        defaultSelectedKeys={['home']}
      />
    </Header>
  )
}
```

> En antd v6 `Menu` usa la prop `items` en lugar de elementos JSX anidados.
> Cada item tiene `key` y `label` — el `label` puede ser un `NavLink`.

### Prueba esto

- Añade `style={{ background: '#001529' }}` al `<Header>` y quita el fondo por defecto — el header adopta el color oscuro estándar de AntD; prueba otros colores como `'#141414'` para un aspecto más neutro
- Cambia `defaultSelectedKeys={['home']}` a `defaultSelectedKeys={['about']}` — el item "Acerca de" aparece resaltado al cargar la página; nota que no refleja la ruta actual de forma automática
- Añade `overflowedIndicator={<span>Más</span>}` al `<Menu>` — cuando el menú se estrecha y no caben todos los items, aparece un "Más" en lugar del icono por defecto
- Añade un tercer item al array: `{ key: 'reports', label: <NavLink to="/reports">Reportes</NavLink> }` — aparece en el menú; sin ruta configurada en el Router simplemente no renderiza contenido
- Cambia `fontWeight: 900` a `fontWeight: 400` en el título "SalesBoard" — el logo pierde el peso visual; ajusta también `fontSize: 20` para hacer el cambio más notorio
- Añade `style={{ minWidth: 140 }}` al `<Menu>` — fuerza un ancho mínimo para evitar que los items se compriman demasiado en pantallas estrechas

---

### `src/components/antd/AntKpis.tsx`

```tsx
// src/components/antd/AntKpis.tsx

import { Card, Col, Row, Statistic, Typography } from 'antd'
import {
  ArrowUpOutlined,
  ShoppingCartOutlined,
  TeamOutlined,
  PercentageOutlined,
} from '@ant-design/icons'

const { Title } = Typography

interface KpiData {
  title:   string
  value:   number
  suffix?: string
  prefix?: string
  trend:   number
  icon:    React.ReactNode
  color:   string
}

const KPIS: KpiData[] = [
  { title: 'Ventas',     value: 12840, prefix: '$', trend:  12.5, icon: <ArrowUpOutlined />,     color: '#1677ff' },
  { title: 'Órdenes',    value: 342,                trend:   8.2, icon: <ShoppingCartOutlined />, color: '#52c41a' },
  { title: 'Clientes',   value: 1280,               trend:   5.1, icon: <TeamOutlined />,         color: '#722ed1' },
  { title: 'Conversión', value: 3.4,  suffix: '%',  trend:  -0.8, icon: <PercentageOutlined />,   color: '#fa8c16' },
]

export default function AntKpis() {
  return (
    <div style={{ marginBottom: 24 }}>
      <Title level={5} style={{ marginBottom: 12 }}>Resumen del período</Title>
      <Row gutter={[16, 16]}>
        {KPIS.map(kpi => (
          <Col key={kpi.title} xs={24} sm={12} md={6}>
            <Card>
              <Statistic
                title={kpi.title}
                value={kpi.value}
                suffix={kpi.suffix}
                prefix={kpi.prefix}
                precision={kpi.suffix === '%' ? 1 : 0}
                valueStyle={{ color: kpi.color }}
              />
              <div style={{ marginTop: 8, fontSize: 12, color: kpi.trend > 0 ? '#52c41a' : '#ff4d4f' }}>
                {kpi.trend > 0 ? '↑' : '↓'} {Math.abs(kpi.trend)}% vs mes anterior
              </div>
            </Card>
          </Col>
        ))}
      </Row>
    </div>
  )
}
```

### Prueba esto

- Cambia `precision={kpi.suffix === '%' ? 1 : 0}` a `precision={2}` en el `<Statistic>` — todos los valores muestran dos decimales; `precision` controla los dígitos después del punto
- Añade `loading={true}` a la `<Card>` de uno de los KPIs — la tarjeta muestra un skeleton de carga; AntD incluye este estado sin necesidad de un componente extra
- Cambia `gutter={[16, 16]}` a `gutter={[24, 24]}` en el `<Row>` — el espacio entre columnas aumenta; el primer valor es el espacio horizontal y el segundo el vertical
- Modifica el `color` de uno de los KPIs a `'#fa541c'` (naranja intenso) — el `valueStyle` acepta cualquier color CSS; no está limitado a los colores del token de AntD
- Cambia `xs={24} sm={12} md={6}` a `xs={24} sm={24} md={12}` — los KPIs pasan a ocupar la mitad del ancho en escritorio y se muestran de dos en dos en lugar de cuatro
- Añade `bordered={false}` a cada `<Card>` — los bordes desaparecen y los KPIs se ven más limpios; combínalo con un `style={{ background: '#fafafa' }}` para diferenciarlos del fondo

---

### `src/components/antd/AntProgress.tsx`

```tsx
// src/components/antd/AntProgress.tsx

import { Card, Progress, Space, Typography } from 'antd'

const { Title, Text } = Typography

interface GoalProps {
  label:   string
  current: number
  target:  number
  color:   string
}

const GOALS: GoalProps[] = [
  { label: 'Meta de ventas mensual', current: 12840, target: 20000, color: '#1677ff' },
  { label: 'Nuevos clientes',        current: 48,    target: 80,    color: '#52c41a' },
  { label: 'Tickets resueltos',      current: 132,   target: 150,   color: '#722ed1' },
]

export default function AntGoals() {
  return (
    <Card style={{ marginBottom: 24 }}>
      <Title level={5} style={{ marginBottom: 16 }}>Objetivos del mes</Title>
      <Space direction="vertical" style={{ width: '100%' }}>
        {GOALS.map(g => {
          const percent = Math.round((g.current / g.target) * 100)
          return (
            <div key={g.label}>
              <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 4 }}>
                <Text style={{ fontSize: 13 }}>{g.label}</Text>
                <Text type="secondary" style={{ fontSize: 12 }}>
                  {g.current.toLocaleString()} / {g.target.toLocaleString()}
                </Text>
              </div>
              <Progress
                percent={percent}
                strokeColor={g.color}
                size="small"
                style={{ margin: 0 }}
              />
            </div>
          )
        })}
      </Space>
    </Card>
  )
}
```

### Prueba esto

- Quita `size="small"` del `<Progress>` — las barras se hacen más gruesas; prueba también `size="large"` para un aspecto más prominente
- Añade `showInfo={false}` al `<Progress>` — el porcentaje numérico a la derecha desaparece; útil cuando ya muestras los valores en el texto de arriba
- Cambia `strokeColor={g.color}` por `strokeColor={{ from: '#108ee9', to: '#87d068' }}` en uno de los progresos — la barra muestra un degradado de azul a verde
- Añade `status="exception"` al `<Progress>` del primer objetivo — la barra cambia a rojo e icono de error; los valores posibles son `normal`, `exception`, `active` y `success`
- Añade `type="circle"` al primer `<Progress>` — la barra lineal se convierte en un círculo; cambia también el `size` a un número como `80` para controlar el diámetro
- Añade un nuevo objetivo al array `GOALS` con `current` mayor que `target` — el porcentaje calculado superará 100 y AntD lo mostrará como completado; considera limitar con `Math.min(percent, 100)`

---

### `src/components/antd/AntSalesTable.tsx`

```tsx
// src/components/antd/AntSalesTable.tsx

import { Table, Tag, Typography, Progress } from 'antd'
import type { ColumnsType } from 'antd/es/table'

const { Title, Text } = Typography

interface SaleRow {
  key:      number
  customer: string
  product:  string
  total:    number
  progress: number
  status:   'paid' | 'pending' | 'failed'
}

const DATA: SaleRow[] = [
  { key: 1, customer: 'Empresa Alpha',    product: 'Plan Pro',    total: 1200, progress: 100, status: 'paid'    },
  { key: 2, customer: 'StartupBeta',      product: 'Plan Básico', total: 740,  progress: 65,  status: 'pending' },
  { key: 3, customer: 'Consultora Gamma', product: 'Plan Pro',    total: 1200, progress: 100, status: 'paid'    },
  { key: 4, customer: 'MiPyme Delta',     product: 'Plan Team',   total: 310,  progress: 0,   status: 'failed'  },
  { key: 5, customer: 'TechEpsilon',      product: 'Plan Pro',    total: 1200, progress: 40,  status: 'pending' },
]

const STATUS_COLOR: Record<SaleRow['status'], string> = {
  paid:    'success',
  pending: 'warning',
  failed:  'error',
}

const STATUS_LABEL: Record<SaleRow['status'], string> = {
  paid:    'Pagado',
  pending: 'Pendiente',
  failed:  'Fallido',
}

const COLUMNS: ColumnsType<SaleRow> = [
  {
    title:     'Cliente',
    dataIndex: 'customer',
    render:    (v: string) => <Text strong>{v}</Text>,
  },
  { title: 'Producto', dataIndex: 'product' },
  {
    title:     'Total',
    dataIndex: 'total',
    align:     'right',
    render:    (v: number) => <Text strong>${v.toLocaleString()}</Text>,
    sorter:    (a, b) => a.total - b.total,
  },
  {
    title:     'Avance',
    dataIndex: 'progress',
    render:    (v: number) => (
      <Progress percent={v} size="small" style={{ margin: 0 }} />
    ),
  },
  {
    title:     'Estado',
    dataIndex: 'status',
    render:    (v: SaleRow['status']) => (
      <Tag color={STATUS_COLOR[v]}>{STATUS_LABEL[v]}</Tag>
    ),
    filters: [
      { text: 'Pagado',    value: 'paid'    },
      { text: 'Pendiente', value: 'pending' },
      { text: 'Fallido',   value: 'failed'  },
    ],
    onFilter: (value, record) => record.status === value,
  },
]

export default function AntSalesTable() {
  return (
    <div style={{ marginBottom: 24 }}>
      <Title level={5} style={{ marginBottom: 12 }}>Últimas ventas</Title>
      <Table<SaleRow>
        columns={COLUMNS}
        dataSource={DATA}
        pagination={{ pageSize: 4 }}
        size="middle"
      />
    </div>
  )
}
```

### Prueba esto

- Haz clic en el encabezado "Total" en la tabla renderizada — las filas se ordenan de menor a mayor; el `sorter` de la columna gestiona el estado internamente sin código extra
- Abre el menú de filtro en la columna "Estado" y selecciona "Pendiente" — solo las filas pendientes se muestran; `filters` + `onFilter` en la columna lo resuelven sin estado externo
- Cambia `pagination={{ pageSize: 4 }}` a `pagination={false}` — la paginación desaparece y se muestran todas las filas; útil cuando el dataset es pequeño
- Añade `summary={() => <Table.Summary.Row><Table.Summary.Cell index={0} colSpan={2}>Total</Table.Summary.Cell><Table.Summary.Cell index={2} align="right">${DATA.reduce((s, r) => s + r.total, 0).toLocaleString()}</Table.Summary.Cell></Table.Summary.Row>}` — aparece una fila de totales al pie de la tabla
- Cambia `size="middle"` a `size="small"` — las filas se compactan; útil para dashboards con mucha información
- Añade `scroll={{ x: 600 }}` a la `<Table>` — la tabla muestra scroll horizontal en pantallas pequeñas en lugar de romper el layout

---

### `src/components/antd/AntFooter.tsx`

```tsx
// src/components/antd/AntFooter.tsx

import { Layout, Typography } from 'antd'

const { Footer } = Layout
const { Text }   = Typography

export default function AntFooter() {
  return (
    <Footer style={{ textAlign: 'center' }}>
      <Text type="secondary">
        SalesBoard © {new Date().getFullYear()} — Ant Design v6 + React 19
      </Text>
    </Footer>
  )
}
```

---

### `src/pages/HomeDashboard.tsx`

```tsx
// src/pages/HomeDashboard.tsx

import { Typography }  from 'antd'
import AntKpis         from '../components/antd/AntKpis'
import AntGoals        from '../components/antd/AntProgress'
import AntSalesTable   from '../components/antd/AntSalesTable'

const { Title, Text } = Typography

export default function HomeDashboard() {
  return (
    <div style={{ maxWidth: 960, margin: '0 auto', padding: '24px 16px' }}>
      <Title level={3} style={{ marginBottom: 4 }}>Dashboard de Ventas</Title>
      <Text type="secondary" style={{ display: 'block', marginBottom: 24 }}>
        KPIs, objetivos y últimas transacciones del período.
      </Text>
      <AntKpis />
      <AntGoals />
      <AntSalesTable />
    </div>
  )
}
```

---

### `src/pages/AboutPage.tsx`

```tsx
// src/pages/AboutPage.tsx

import { Card, Typography, List } from 'antd'

const { Title } = Typography

const STACK = [
  'React 19 + TypeScript',
  'Ant Design v6',
  'React Router v7',
  'Vite 8',
]

export default function AboutPage() {
  return (
    <div style={{ maxWidth: 600, margin: '0 auto', padding: '24px 16px' }}>
      <Title level={3} style={{ marginBottom: 20 }}>Acerca de este proyecto</Title>
      <Card>
        <List
          dataSource={STACK}
          renderItem={item => <List.Item>{item}</List.Item>}
        />
      </Card>
    </div>
  )
}
```

---

### `src/AppHome.tsx`

```tsx
// src/AppHome.tsx

import { BrowserRouter, Routes, Route } from 'react-router-dom'
import { Layout }        from 'antd'
import AntNavbar         from './components/antd/AntNavbar'
import AntFooter         from './components/antd/AntFooter'
import HomeDashboard     from './pages/HomeDashboard'
import AboutPage         from './pages/AboutPage'

const { Content } = Layout

export default function AppHome() {
  return (
    <BrowserRouter>
      <Layout style={{ minHeight: '100vh' }}>
        <AntNavbar />
        <Content>
          <Routes>
            <Route path="/"      element={<HomeDashboard />} />
            <Route path="/about" element={<AboutPage />} />
          </Routes>
        </Content>
        <AntFooter />
      </Layout>
    </BrowserRouter>
  )
}
```

---

### `src/App.tsx` — alterna entre fases

```tsx
// src/App.tsx

import AppLab from './AppLab'
// import AppHome from './AppHome'

export default function App() {
  return <AppLab />
  // Fase 2 — descomenta y comenta la línea anterior:
  // return <AppHome />
}
```

---

## Componentes más usados — referencia rápida

| Componente | Import | Uso típico |
|---|---|---|
| `Button` | `antd` | Acciones con `type`, `size`, `loading`, `danger` |
| `Alert` | `antd` | Mensajes con `type`, `showIcon`, `closable`, `description` |
| `Card` | `antd` | Contenedor. `styles={{ body: {...} }}` para el interior |
| `Form`, `Form.Item` | `antd` | Formularios con `rules` de validación integrada |
| `Form.useForm<T>()` | `antd` | Instancia tipada del formulario |
| `Input`, `Input.Search` | `antd` | Inputs con `allowClear`, `onChange` |
| `Select`, `Option` | `antd` | Selector con búsqueda integrada |
| `Table<T>` | `antd` | Tabla con sorting, filtering y paginación |
| `ColumnsType<T>` | `antd/es/table` | Tipado del array de columnas |
| `Tag` | `antd` | Etiquetas de estado con `color` semántico |
| `Statistic` | `antd` | KPIs numéricos con `prefix`, `suffix`, `valueStyle` |
| `Progress` | `antd` | Barra de progreso con `strokeColor` |
| `Space` | `antd` | Separación flexible con `direction` y `wrap` |
| `Row`, `Col` | `antd` | Grid responsive de 24 columnas |
| `Typography.Title` | `antd` | Encabezados `h1`–`h5` con `level` |
| `Typography.Text` | `antd` | Texto con `type`, `strong`, `style` |
| `Layout`, `Header`, `Content`, `Footer` | `antd` | Estructura de página completa |
| `Menu` | `antd` | Navegación con `items`, `theme`, `mode` |

---

## Resumen de la página 18

- `antd/dist/reset.css` se importa **una vez** en `main.tsx` — no en cada componente.
- `Form.useForm<FormValues>()` tipado proporciona `form.resetFields()`, `form.setFieldsValue()` y más. `onFinish` solo se ejecuta si todas las `rules` pasan.
- `ColumnsType<T>` del import `antd/es/table` tipa el array de columnas incluyendo `render`, `sorter` y `onFilter`.
- `sorter` y `filters`/`onFilter` son props de cada columna en `Table` — no hay que gestionar el estado externo.
- `Card` en v6 usa `styles={{ body: {...} }}` en lugar del antiguo `bodyStyle`.
- `Menu` en v6 usa la prop `items` en lugar de elementos JSX `<Menu.Item>` anidados.
- `Row`/`Col` de antd usa un grid de **24 columnas** — diferente al de 12 de Bootstrap.
- Los iconos de `@ant-design/icons` están incluidos en el paquete de antd v6 — no necesitas instalación extra.

---

> **Siguiente página →** Página 19: Material UI v7 — LAB + dashboard con API de productos.