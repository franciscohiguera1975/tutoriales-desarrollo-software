# Curso React.js + TypeScript — Página 19
## Frameworks de estilos · Material UI v7
### LAB: 5 componentes aislados — HOME: dashboard con API de productos

---

## ¿Qué es Material UI?

Material UI (MUI) es la librería de componentes React más popular basada en
Material Design de Google. A diferencia de Bootstrap o Ant Design, MUI
usa **Emotion** para los estilos — CSS-in-JS con la prop `sx` que acepta
objetos de estilo con acceso directo al tema.

```tsx
// Bootstrap / Tailwind     →  clases en className
<div className="p-4 bg-slate-950 rounded-xl">

// Ant Design               →  objetos en style o props del componente
<Card style={{ padding: 16 }}>

// Material UI              →  prop sx con acceso al tema
<Box sx={{ p: 2, bgcolor: 'background.paper', borderRadius: 2 }}>
```

---

## Instalación

```bash
npm install @mui/material @emotion/react @emotion/styled
npm install @mui/icons-material   # íconos (paquete separado)
npm install axios                  # para el dashboard con API
```

| Paquete | Versión actual |
|---|---|
| `@mui/material` | 7.3.9 |
| `@mui/icons-material` | 7.3.9 |
| `@emotion/react` | 11.14.0 |

---

## `src/main.tsx` — ThemeProvider obligatorio

```tsx
// src/main.tsx

import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { createTheme, ThemeProvider, CssBaseline } from '@mui/material'
import App from './App.tsx'

const theme = createTheme({
  palette: { mode: 'light', primary: { main: '#1976d2' } },
  shape:   { borderRadius: 10 },
})

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <ThemeProvider theme={theme}>
      <CssBaseline />   {/* normalize CSS global de MUI */}
      <App />
    </ThemeProvider>
  </StrictMode>,
)
```

> `ThemeProvider` + `CssBaseline` son obligatorios. Sin ellos los componentes
> funcionan pero sin el tema personalizado ni el reset de estilos.

---

## Cambio crítico de MUI v7: `Grid`

En MUI v7 el componente `Grid` ya **no acepta** las props `item`, `xs`, `md`.
En su lugar se usa la prop `size`:

```tsx
// ❌ MUI v6 y anterior — ya NO funciona en v7
<Grid size={{ xs: 12, md: 4 }}>  {/* en v7 item/xs/md se reemplaza por size */}

// ✅ MUI v7 — nueva API
<Grid size={{ xs: 12, md: 4 }}>

// ✅ También válido — shorthand para columna completa
<Grid size={12}>
```

---

## Estructura del proyecto

```
src/
├── lab/
│   ├── LabMuiButtons.tsx
│   ├── LabMuiAlert.tsx
│   ├── LabMuiCard.tsx
│   ├── LabMuiForm.tsx
│   └── LabMuiTable.tsx
├── api/
│   ├── http.ts
│   └── productsApi.ts
├── types/
│   └── product.ts
├── components/mui/
│   └── MuiShell.tsx
├── pages/
│   ├── DashboardPage.tsx
│   └── AboutPage.tsx
├── App.tsx
├── AppLab.tsx
├── AppHome.tsx
└── main.tsx
```

---

## Fase 1 — Laboratorio

---

### `src/lab/LabMuiButtons.tsx`

```tsx
// src/lab/LabMuiButtons.tsx

import { Box, Button, Stack, Typography, IconButton, Tooltip } from '@mui/material'
import DeleteIcon from '@mui/icons-material/Delete'
import SendIcon   from '@mui/icons-material/Send'
import AddIcon    from '@mui/icons-material/Add'

export default function LabMuiButtons() {
  return (
    <Box sx={{ maxWidth: 800, mx: 'auto', p: 3 }}>
      <Typography variant="h6" fontWeight={700} gutterBottom>LAB: Buttons</Typography>
      <Typography variant="body2" color="text.secondary" sx={{ mb: 3 }}>
        Variantes, colores, tamaños, íconos y estados.
      </Typography>

      <Typography variant="subtitle2" sx={{ mb: 1 }}>Variantes</Typography>
      <Stack direction="row" spacing={2} flexWrap="wrap" sx={{ mb: 3 }}>
        <Button variant="contained">Contained</Button>
        <Button variant="outlined">Outlined</Button>
        <Button variant="text">Text</Button>
      </Stack>

      <Typography variant="subtitle2" sx={{ mb: 1 }}>Colores</Typography>
      <Stack direction="row" spacing={2} flexWrap="wrap" sx={{ mb: 3 }}>
        <Button variant="contained" color="primary">Primary</Button>
        <Button variant="contained" color="secondary">Secondary</Button>
        <Button variant="contained" color="success">Success</Button>
        <Button variant="contained" color="error">Error</Button>
        <Button variant="contained" color="warning">Warning</Button>
      </Stack>

      <Typography variant="subtitle2" sx={{ mb: 1 }}>Con íconos</Typography>
      <Stack direction="row" spacing={2} flexWrap="wrap" sx={{ mb: 3 }}>
        <Button variant="contained" startIcon={<SendIcon />}>Enviar</Button>
        <Button variant="outlined"  startIcon={<AddIcon />}>Agregar</Button>
        <Button variant="outlined"  endIcon={<DeleteIcon />} color="error">Eliminar</Button>
        <Tooltip title="Eliminar elemento">
          <IconButton color="error"><DeleteIcon /></IconButton>
        </Tooltip>
      </Stack>

      <Typography variant="subtitle2" sx={{ mb: 1 }}>Tamaños y estados</Typography>
      <Stack direction="row" spacing={2} flexWrap="wrap" alignItems="center">
        <Button variant="contained" size="large">Grande</Button>
        <Button variant="contained">Normal</Button>
        <Button variant="contained" size="small">Pequeño</Button>
        <Button variant="contained" disabled>Deshabilitado</Button>
        <Button variant="contained" loading>Cargando</Button>
      </Stack>
    </Box>
  )
}
```

> `loading` en `Button` es nativo de MUI v7 — muestra un spinner sin
> necesidad de gestionar estado ni importar `CircularProgress`.

---

### `src/lab/LabMuiAlert.tsx`

```tsx
// src/lab/LabMuiAlert.tsx

import { useState } from 'react'
import { Box, Alert, AlertTitle, Button, Collapse, Stack, Typography } from '@mui/material'

export default function LabMuiAlert() {
  const [show, setShow] = useState(true)

  return (
    <Box sx={{ maxWidth: 800, mx: 'auto', p: 3 }}>
      <Typography variant="h6" fontWeight={700} gutterBottom>LAB: Alert</Typography>
      <Typography variant="body2" color="text.secondary" sx={{ mb: 3 }}>
        Severidades, títulos y control de visibilidad con Collapse.
      </Typography>

      <Stack spacing={2} sx={{ mb: 3 }}>
        <Alert severity="info">    Información de referencia.</Alert>
        <Alert severity="success"> Operación completada correctamente.</Alert>
        <Alert severity="warning"> Advertencia: acción irreversible.</Alert>
        <Alert severity="error">   Se produjo un error inesperado.</Alert>
      </Stack>

      <Typography variant="subtitle2" sx={{ mb: 1 }}>Con título (AlertTitle)</Typography>
      <Stack spacing={2} sx={{ mb: 3 }}>
        <Alert severity="success">
          <AlertTitle>Éxito</AlertTitle>
          El formulario fue enviado correctamente (demo).
        </Alert>
        <Alert severity="error">
          <AlertTitle>Error</AlertTitle>
          No se pudo conectar al servidor. Intenta de nuevo.
        </Alert>
      </Stack>

      <Typography variant="subtitle2" sx={{ mb: 1 }}>Con Collapse (animación suave)</Typography>
      <Collapse in={show}>
        <Alert severity="info" onClose={() => setShow(false)} sx={{ mb: 1 }}>
          Esta alerta se puede cerrar con la X o con el botón de abajo.
        </Alert>
      </Collapse>
      {!show && (
        <Button size="small" onClick={() => setShow(true)}>Mostrar alerta</Button>
      )}
    </Box>
  )
}
```

> `Collapse` envuelve el `Alert` para que la animación de entrada/salida
> sea suave. `in={show}` controla si está desplegado.

---

### `src/lab/LabMuiCard.tsx`

```tsx
// src/lab/LabMuiCard.tsx

import {
  Box, Card, CardContent, CardActions, CardMedia,
  Button, Typography, Chip, Grid,
} from '@mui/material'

interface ProductCardProps {
  name:     string
  category: string
  price:    number
  active:   boolean
  imageUrl: string
}

function ProductCard({ name, category, price, active, imageUrl }: ProductCardProps) {
  return (
    <Card sx={{ height: '100%', display: 'flex', flexDirection: 'column' }}>
      <CardMedia
        component="img"
        height={160}
        image={imageUrl}
        alt={name}
        onError={(e) => {
          ;(e.currentTarget as HTMLImageElement).src =
            'https://placehold.co/400x160?text=Sin+imagen'
        }}
      />
      <CardContent sx={{ flex: 1 }}>
        <Typography variant="subtitle1" fontWeight={700} gutterBottom>
          {name}
        </Typography>
        <Typography variant="body2" color="text.secondary" gutterBottom>
          {category}
        </Typography>
        <Box sx={{ display: 'flex', alignItems: 'center', gap: 1, mt: 1 }}>
          <Chip
            label={active ? 'Activo' : 'Inactivo'}
            color={active ? 'success' : 'default'}
            size="small"
          />
          <Typography variant="h6" fontWeight={700}>${price.toFixed(2)}</Typography>
        </Box>
      </CardContent>
      <CardActions>
        <Button size="small" variant="outlined">Ver detalle</Button>
      </CardActions>
    </Card>
  )
}

export default function LabMuiCard() {
  const products: ProductCardProps[] = [
    { name: 'Teclado mecánico',  category: 'Periféricos', price: 89.99,  active: true,  imageUrl: 'https://picsum.photos/seed/kb/400/160'  },
    { name: 'Monitor 27"',       category: 'Pantallas',   price: 349.99, active: true,  imageUrl: 'https://picsum.photos/seed/mon/400/160' },
    { name: 'Mouse inalámbrico', category: 'Periféricos', price: 29.99,  active: false, imageUrl: 'https://picsum.photos/seed/ms/400/160'  },
  ]

  return (
    <Box sx={{ maxWidth: 900, mx: 'auto', p: 3 }}>
      <Typography variant="h6" fontWeight={700} gutterBottom>LAB: Cards</Typography>
      <Typography variant="body2" color="text.secondary" sx={{ mb: 3 }}>
        Con CardMedia, CardContent, CardActions, Chip y Grid v7.
      </Typography>
      <Grid container spacing={2}>
        {products.map(p => (
          // MUI v7 — size en lugar de xs/md
          <Grid key={p.name} size={{ xs: 12, sm: 6, md: 4 }}>
            <ProductCard {...p} />
          </Grid>
        ))}
      </Grid>
    </Box>
  )
}
```

---

### `src/lab/LabMuiForm.tsx`

```tsx
// src/lab/LabMuiForm.tsx

import { useState } from 'react'
import {
  Box, Button, TextField, MenuItem,
  Alert, Typography, Stack,
} from '@mui/material'

interface FormValues {
  name:  string
  email: string
  role:  string
}

type FormErrors = Partial<Record<keyof FormValues, string>>

const ROLES = [
  { value: 'viewer', label: 'Viewer' },
  { value: 'editor', label: 'Editor' },
  { value: 'admin',  label: 'Admin'  },
]

export default function LabMuiForm() {
  const [values,  setValues]  = useState<FormValues>({ name: '', email: '', role: 'viewer' })
  const [errors,  setErrors]  = useState<FormErrors>({})
  const [success, setSuccess] = useState(false)

  function validate(): boolean {
    const e: FormErrors = {}
    if (!values.name.trim())         e.name  = 'El nombre es requerido'
    if (!values.email.includes('@')) e.email = 'Email inválido'
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
    <Box sx={{ maxWidth: 800, mx: 'auto', p: 3 }}>
      <Typography variant="h6" fontWeight={700} gutterBottom>LAB: Form</Typography>
      <Typography variant="body2" color="text.secondary" sx={{ mb: 3 }}>
        TextField con error/helperText, Select con MenuItem y validación manual.
      </Typography>

      {success && (
        <Alert severity="success" sx={{ mb: 2 }}>Registro enviado correctamente</Alert>
      )}

      <Box component="form" onSubmit={handleSubmit} sx={{ maxWidth: 480 }}>
        <Stack spacing={2}>
          <TextField
            label="Nombre completo"
            value={values.name}
            onChange={e => {
              setValues(v => ({ ...v, name: e.target.value }))
              setErrors(v => ({ ...v, name: undefined }))
            }}
            error={!!errors.name}
            helperText={errors.name}
            placeholder="Ana García"
            fullWidth
          />
          <TextField
            label="Correo electrónico"
            type="email"
            value={values.email}
            onChange={e => {
              setValues(v => ({ ...v, email: e.target.value }))
              setErrors(v => ({ ...v, email: undefined }))
            }}
            error={!!errors.email}
            helperText={errors.email}
            placeholder="ana@ejemplo.com"
            fullWidth
          />
          <TextField
            label="Rol"
            select
            value={values.role}
            onChange={e => setValues(v => ({ ...v, role: e.target.value }))}
            fullWidth
          >
            {ROLES.map(r => (
              <MenuItem key={r.value} value={r.value}>{r.label}</MenuItem>
            ))}
          </TextField>
          <Button type="submit" variant="contained" sx={{ alignSelf: 'flex-start' }}>
            Registrar
          </Button>
        </Stack>
      </Box>
    </Box>
  )
}
```

> `TextField select` con `MenuItem` es la forma MUI de hacer un select
> con estilos consistentes con el resto del formulario.
> `error` + `helperText` muestran el estado de error sin CSS adicional.

---

### `src/lab/LabMuiTable.tsx`

```tsx
// src/lab/LabMuiTable.tsx

import { useState } from 'react'
import {
  Box, Table, TableBody, TableCell, TableContainer,
  TableHead, TableRow, Paper, Chip, Typography,
  TableSortLabel, InputAdornment, TextField,
} from '@mui/material'
import SearchIcon from '@mui/icons-material/Search'

interface Product {
  id:       number
  name:     string
  category: string
  price:    number
  active:   boolean
}

const PRODUCTS: Product[] = [
  { id: 1, name: 'Teclado mecánico',  category: 'Periféricos', price: 89.99,  active: true  },
  { id: 2, name: 'Monitor 27"',       category: 'Pantallas',   price: 349.99, active: true  },
  { id: 3, name: 'Mouse inalámbrico', category: 'Periféricos', price: 29.99,  active: false },
  { id: 4, name: 'Webcam HD',         category: 'Cámaras',     price: 59.99,  active: true  },
  { id: 5, name: 'Auriculares BT',    category: 'Audio',       price: 149.99, active: false },
]

type SortDir = 'asc' | 'desc'

export default function LabMuiTable() {
  const [search,  setSearch]  = useState('')
  const [sortDir, setSortDir] = useState<SortDir>('asc')

  const filtered = PRODUCTS
    .filter(p =>
      p.name.toLowerCase().includes(search.toLowerCase()) ||
      p.category.toLowerCase().includes(search.toLowerCase())
    )
    .sort((a, b) => sortDir === 'asc' ? a.price - b.price : b.price - a.price)

  return (
    <Box sx={{ maxWidth: 900, mx: 'auto', p: 3 }}>
      <Typography variant="h6" fontWeight={700} gutterBottom>LAB: Table</Typography>
      <Typography variant="body2" color="text.secondary" sx={{ mb: 2 }}>
        Con TableSortLabel, búsqueda con InputAdornment y Chip de estado.
      </Typography>

      <TextField
        size="small"
        placeholder="Buscar producto o categoría..."
        value={search}
        onChange={e => setSearch(e.target.value)}
        sx={{ mb: 2, maxWidth: 340 }}
        slotProps={{
          input: {
            startAdornment: (
              <InputAdornment position="start">
                <SearchIcon fontSize="small" />
              </InputAdornment>
            ),
          },
        }}
      />

      <TableContainer component={Paper} variant="outlined">
        <Table size="small">
          <TableHead>
            <TableRow sx={{ bgcolor: 'grey.50' }}>
              <TableCell>#</TableCell>
              <TableCell>Nombre</TableCell>
              <TableCell>Categoría</TableCell>
              <TableCell align="right">
                <TableSortLabel
                  active
                  direction={sortDir}
                  onClick={() => setSortDir(d => d === 'asc' ? 'desc' : 'asc')}
                >
                  Precio
                </TableSortLabel>
              </TableCell>
              <TableCell>Estado</TableCell>
            </TableRow>
          </TableHead>
          <TableBody>
            {filtered.map(p => (
              <TableRow key={p.id} hover>
                <TableCell>{p.id}</TableCell>
                <TableCell>
                  <Typography variant="body2" fontWeight={600}>{p.name}</Typography>
                </TableCell>
                <TableCell>{p.category}</TableCell>
                <TableCell align="right">
                  <Typography variant="body2" fontWeight={700}>${p.price.toFixed(2)}</Typography>
                </TableCell>
                <TableCell>
                  <Chip
                    label={p.active ? 'Activo' : 'Inactivo'}
                    color={p.active ? 'success' : 'default'}
                    size="small"
                  />
                </TableCell>
              </TableRow>
            ))}
            {filtered.length === 0 && (
              <TableRow>
                <TableCell colSpan={5} align="center">
                  <Typography variant="body2" color="text.secondary">
                    Sin resultados.
                  </Typography>
                </TableCell>
              </TableRow>
            )}
          </TableBody>
        </Table>
      </TableContainer>
    </Box>
  )
}
```

> `slotProps.input` es la forma v7 de pasar props al input interno de
> `TextField`. Reemplaza el antiguo `InputProps`.

---

### `src/AppLab.tsx`

```tsx
// src/AppLab.tsx

import { useState } from 'react'
import { Box, AppBar, Toolbar, Typography, Tabs, Tab, Container } from '@mui/material'
import LabMuiButtons from './lab/LabMuiButtons'
import LabMuiAlert   from './lab/LabMuiAlert'
import LabMuiCard    from './lab/LabMuiCard'
import LabMuiForm    from './lab/LabMuiForm'
import LabMuiTable   from './lab/LabMuiTable'

type LabKey = 0 | 1 | 2 | 3 | 4

const LAB_LABELS = ['Buttons', 'Alert', 'Cards', 'Form', 'Table']

export default function AppLab() {
  const [tab, setTab] = useState<LabKey>(0)

  return (
    <Box sx={{ minHeight: '100vh', bgcolor: 'background.default' }}>
      <AppBar position="static" color="primary">
        <Toolbar sx={{ gap: 2 }}>
          <Typography variant="h6" fontWeight={700} sx={{ whiteSpace: 'nowrap' }}>
            Material UI v7 LAB
          </Typography>
          <Tabs
            value={tab}
            onChange={(_, v: LabKey) => setTab(v)}
            textColor="inherit"
            indicatorColor="secondary"
          >
            {LAB_LABELS.map((label, i) => (
              <Tab key={label} label={label} value={i as LabKey} />
            ))}
          </Tabs>
        </Toolbar>
      </AppBar>
      <Container sx={{ py: 3 }}>
        {tab === 0 && <LabMuiButtons />}
        {tab === 1 && <LabMuiAlert />}
        {tab === 2 && <LabMuiCard />}
        {tab === 3 && <LabMuiForm />}
        {tab === 4 && <LabMuiTable />}
      </Container>
    </Box>
  )
}
```

---

## Fase 2 — Dashboard con API de productos

### `src/types/product.ts`

```ts
// src/types/product.ts

export interface Product {
  id:            number
  name:          string
  slug:          string
  price:         string     // viene como string desde la API
  stock:         number
  is_active:     boolean
  url_image:     string
  category_name: string
  created_at:    string
  updated_at:    string
}

export interface PaginatedResponse<T> {
  count:    number
  next:     string | null
  previous: string | null
  results:  T[]
}
```

---

### `src/api/http.ts`

```ts
// src/api/http.ts

import axios from 'axios'

export const http = axios.create({
  baseURL: 'https://higuera-billing-api.desarrollo-software.xyz/api',
  timeout: 15000,
})
```

---

### `src/api/productsApi.ts`

```ts
// src/api/productsApi.ts

import { http } from './http'
import type { PaginatedResponse, Product } from '../types/product'

interface GetProductsParams {
  page:     number
  pageSize: number
  search?:  string
}

export async function getProducts(params: GetProductsParams) {
  const { page, pageSize, search } = params
  const res = await http.get<PaginatedResponse<Product>>('/products/', {
    params: {
      page,
      page_size: pageSize,
      ...(search ? { search } : {}),
    },
  })
  return res.data
}
```

---

### `src/components/mui/MuiShell.tsx`

Layout con AppBar fijo + Drawer permanente en escritorio / temporal en móvil.
`NavLink` de React Router se usa directamente dentro de los `ListItemButton`.

```tsx
// src/components/mui/MuiShell.tsx

import { useState } from 'react'
import { NavLink } from 'react-router-dom'
import {
  AppBar, Box, CssBaseline, Divider, Drawer,
  IconButton, List, ListItemButton, ListItemIcon,
  ListItemText, Toolbar, Typography,
} from '@mui/material'
import MenuIcon      from '@mui/icons-material/Menu'
import DashboardIcon from '@mui/icons-material/Dashboard'
import InfoIcon      from '@mui/icons-material/Info'

const DRAWER_WIDTH = 240

const NAV = [
  { to: '/',      label: 'Dashboard', icon: <DashboardIcon /> },
  { to: '/about', label: 'Acerca de', icon: <InfoIcon /> },
]

// NavLink acepta style como función — { isActive } para resaltar el link activo
const linkSx = ({ isActive }: { isActive: boolean }) => ({
  display:        'block',
  textDecoration: 'none',
  color:          'inherit',
  borderRadius:   1,
  background:     isActive ? 'rgba(25,118,210,.10)' : 'transparent',
})

export default function MuiShell({ children }: { children: React.ReactNode }) {
  const [mobileOpen, setMobileOpen] = useState(false)

  const drawer = (
    <Box sx={{ p: 2 }}>
      <Typography variant="subtitle1" fontWeight={900} sx={{ mb: 0.5 }}>
        Billing Admin
      </Typography>
      <Typography variant="caption" color="text.secondary">
        MUI v7 + API Products
      </Typography>
      <Divider sx={{ my: 1.5 }} />
      <List disablePadding sx={{ display: 'flex', flexDirection: 'column', gap: 0.5 }}>
        {NAV.map(({ to, label, icon }) => (
          <NavLink key={to} to={to} end={to === '/'} style={linkSx}>
            <ListItemButton dense>
              <ListItemIcon sx={{ minWidth: 36 }}>{icon}</ListItemIcon>
              <ListItemText
                primary={label}
                primaryTypographyProps={{ fontSize: 14 }}
              />
            </ListItemButton>
          </NavLink>
        ))}
      </List>
    </Box>
  )

  return (
    <Box sx={{ display: 'flex' }}>
      <CssBaseline />

      <AppBar position="fixed" sx={{ zIndex: t => t.zIndex.drawer + 1 }}>
        <Toolbar>
          <IconButton
            color="inherit"
            edge="start"
            onClick={() => setMobileOpen(v => !v)}
            sx={{ mr: 2, display: { md: 'none' } }}
          >
            <MenuIcon />
          </IconButton>
          <Typography variant="h6" fontWeight={700} noWrap>
            Billing Admin
          </Typography>
        </Toolbar>
      </AppBar>

      <Box component="nav" sx={{ width: { md: DRAWER_WIDTH }, flexShrink: { md: 0 } }}>
        {/* Drawer temporal — móvil */}
        <Drawer
          variant="temporary"
          open={mobileOpen}
          onClose={() => setMobileOpen(false)}
          ModalProps={{ keepMounted: true }}
          sx={{
            display: { xs: 'block', md: 'none' },
            '& .MuiDrawer-paper': { width: DRAWER_WIDTH },
          }}
        >
          {drawer}
        </Drawer>

        {/* Drawer permanente — escritorio */}
        <Drawer
          variant="permanent"
          sx={{
            display: { xs: 'none', md: 'block' },
            '& .MuiDrawer-paper': { width: DRAWER_WIDTH, boxSizing: 'border-box' },
          }}
          open
        >
          {drawer}
        </Drawer>
      </Box>

      <Box
        component="main"
        sx={{
          flexGrow: 1,
          p: 3,
          width: { md: `calc(100% - ${DRAWER_WIDTH}px)` },
          minHeight: '100vh',
          bgcolor: 'background.default',
        }}
      >
        <Toolbar />   {/* spacer para que el contenido no quede bajo el AppBar */}
        {children}
      </Box>
    </Box>
  )
}
```

---

### `src/pages/DashboardPage.tsx`

```tsx
// src/pages/DashboardPage.tsx

import { useState, useEffect, useCallback } from 'react'
import {
  Box, Grid, Card, CardContent, CardMedia, Chip,
  Typography, CircularProgress, Alert, TextField,
  Table, TableBody, TableCell, TableContainer,
  TableHead, TableRow, Paper, Tabs, Tab,
  Pagination, MenuItem, Stack, Avatar,
} from '@mui/material'
import { getProducts }  from '../api/productsApi'
import type { Product } from '../types/product'

type ViewMode = 'cards' | 'table'

export default function DashboardPage() {
  const [products, setProducts] = useState<Product[]>([])
  const [count,    setCount]    = useState(0)
  const [page,     setPage]     = useState(1)
  const [pageSize, setPageSize] = useState(5)
  const [search,   setSearch]   = useState('')
  const [view,     setView]     = useState<ViewMode>('cards')
  const [loading,  setLoading]  = useState(false)
  const [error,    setError]    = useState<string | null>(null)

  const load = useCallback(async () => {
    setLoading(true)
    setError(null)
    try {
      const data = await getProducts({ page, pageSize, search: search.trim() || undefined })
      setProducts(data.results)
      setCount(data.count)
    } catch (err) {
      setError('No se pudieron cargar los productos. Verifica la conexión.')
      console.error(err)
    } finally {
      setLoading(false)
    }
  }, [page, pageSize, search])

  useEffect(() => { load() }, [load])

  const totalPages = Math.max(1, Math.ceil(count / pageSize))

  return (
    <Box>
      <Typography variant="h5" fontWeight={700} gutterBottom>Productos</Typography>
      <Typography variant="body2" color="text.secondary" sx={{ mb: 3 }}>
        API: /products/?page={page}&page_size={pageSize}
        {search ? ` &search=${search}` : ''}
      </Typography>

      {/* Controles */}
      <Stack direction={{ xs: 'column', sm: 'row' }} spacing={2} sx={{ mb: 2 }}>
        <TextField
          label="Buscar"
          size="small"
          value={search}
          onChange={e => { setSearch(e.target.value); setPage(1) }}
          sx={{ maxWidth: 280 }}
        />
        <TextField
          label="Por página"
          select
          size="small"
          value={pageSize}
          onChange={e => { setPageSize(Number(e.target.value)); setPage(1) }}
          sx={{ width: 120 }}
        >
          {[5, 10, 20].map(n => (
            <MenuItem key={n} value={n}>{n}</MenuItem>
          ))}
        </TextField>
        <Tabs
          value={view}
          onChange={(_, v: ViewMode) => setView(v)}
          sx={{ ml: { sm: 'auto' } }}
        >
          <Tab value="cards" label="Cards" />
          <Tab value="table" label="Tabla" />
        </Tabs>
      </Stack>

      {error && <Alert severity="error" sx={{ mb: 2 }}>{error}</Alert>}

      {loading ? (
        <Box sx={{ display: 'flex', justifyContent: 'center', py: 6 }}>
          <CircularProgress />
        </Box>
      ) : products.length === 0 ? (
        <Alert severity="info">No hay productos para mostrar.</Alert>
      ) : view === 'cards' ? (

        // ─── Vista cards ───────────────────────────────────────────────
        <Grid container spacing={2} sx={{ mb: 3 }}>
          {products.map(p => (
            <Grid key={p.id} size={{ xs: 12, sm: 6, md: 4 }}>
              <Card sx={{ height: '100%', display: 'flex', flexDirection: 'column' }}>
                <CardMedia
                  component="img"
                  height={140}
                  image={p.url_image}
                  alt={p.name}
                  onError={(e) => {
                    ;(e.currentTarget as HTMLImageElement).src =
                      'https://placehold.co/400x140?text=Sin+imagen'
                  }}
                />
                <CardContent sx={{ flex: 1 }}>
                  <Typography variant="subtitle2" fontWeight={700} gutterBottom>
                    {p.name}
                  </Typography>
                  <Typography variant="caption" color="text.secondary">
                    {p.category_name} · Stock: {p.stock}
                  </Typography>
                  <Box sx={{ display: 'flex', alignItems: 'center', gap: 1, mt: 1 }}>
                    <Chip
                      label={p.is_active ? 'Activo' : 'Inactivo'}
                      color={p.is_active ? 'success' : 'default'}
                      size="small"
                    />
                    <Typography variant="subtitle2" fontWeight={700}>
                      ${p.price}
                    </Typography>
                  </Box>
                </CardContent>
              </Card>
            </Grid>
          ))}
        </Grid>

      ) : (

        // ─── Vista tabla ────────────────────────────────────────────────
        <TableContainer component={Paper} variant="outlined" sx={{ mb: 3 }}>
          <Table size="small">
            <TableHead>
              <TableRow sx={{ bgcolor: 'grey.50' }}>
                <TableCell>Producto</TableCell>
                <TableCell>Categoría</TableCell>
                <TableCell align="right">Precio</TableCell>
                <TableCell align="right">Stock</TableCell>
                <TableCell>Estado</TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {products.map(p => (
                <TableRow key={p.id} hover>
                  <TableCell>
                    <Box sx={{ display: 'flex', alignItems: 'center', gap: 1.5 }}>
                      <Avatar
                        src={p.url_image}
                        variant="rounded"
                        sx={{ width: 36, height: 36 }}
                        imgProps={{
                          onError: (e) => {
                            ;(e.currentTarget as HTMLImageElement).src =
                              'https://placehold.co/80?text=?'
                          },
                        }}
                      />
                      <Box>
                        <Typography variant="body2" fontWeight={600}>{p.name}</Typography>
                        <Typography variant="caption" color="text.secondary">
                          {p.slug}
                        </Typography>
                      </Box>
                    </Box>
                  </TableCell>
                  <TableCell>{p.category_name}</TableCell>
                  <TableCell align="right">
                    <Typography variant="body2" fontWeight={700}>${p.price}</Typography>
                  </TableCell>
                  <TableCell align="right">{p.stock}</TableCell>
                  <TableCell>
                    <Chip
                      label={p.is_active ? 'Activo' : 'Inactivo'}
                      color={p.is_active ? 'success' : 'default'}
                      size="small"
                    />
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </TableContainer>
      )}

      {/* Paginación MUI — mucho más limpia que botones manuales */}
      {!loading && products.length > 0 && (
        <Box sx={{ display: 'flex', justifyContent: 'center' }}>
          <Pagination
            count={totalPages}
            page={page}
            onChange={(_, p) => setPage(p)}
            color="primary"
            showFirstButton
            showLastButton
          />
        </Box>
      )}
    </Box>
  )
}
```

---

### `src/AppHome.tsx`

```tsx
// src/AppHome.tsx

import { BrowserRouter, Routes, Route } from 'react-router-dom'
import MuiShell      from './components/mui/MuiShell'
import DashboardPage from './pages/DashboardPage'
import AboutPage     from './pages/AboutPage'

export default function AppHome() {
  return (
    <BrowserRouter>
      <MuiShell>
        <Routes>
          <Route path="/"      element={<DashboardPage />} />
          <Route path="/about" element={<AboutPage />} />
        </Routes>
      </MuiShell>
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

## Prop `sx` — referencia rápida

La prop `sx` es la forma principal de aplicar estilos en MUI.
Accede directamente a los valores del tema:

```tsx
<Box sx={{
  p:      2,           // padding: theme.spacing(2) = 16px
  px:     3,           // paddingLeft + paddingRight
  py:     1,           // paddingTop + paddingBottom
  m:      'auto',      // margin: auto
  mt:     2,           // marginTop
  mb:     3,           // marginBottom

  bgcolor: 'background.paper',   // color del tema
  color:   'text.secondary',
  border:  '1px solid',
  borderColor: 'divider',

  borderRadius: 2,          // theme.shape.borderRadius * 2
  boxShadow:    1,           // theme.shadows[1]

  display:        'flex',
  alignItems:     'center',
  justifyContent: 'space-between',
  gap:            2,

  width:   '100%',
  maxWidth: 480,

  // Responsive — valores por breakpoint
  p:       { xs: 1, sm: 2, md: 3 },
  display: { xs: 'none', md: 'flex' },

  // Pseudo-clases y selectores
  '&:hover': { bgcolor: 'action.hover' },
  '& .MuiDrawer-paper': { width: 240 },
}}
```

---

## Componentes más usados — referencia rápida

| Componente | Uso típico |
|---|---|
| `Box` | Contenedor genérico con prop `sx` |
| `Stack` | Flex con `direction`, `spacing`, `flexWrap` |
| `Grid` | Grid responsive — `size={{ xs:12, md:4 }}` en v7 |
| `Container` | Centra el contenido con max-width |
| `Typography` | Textos con `variant`, `fontWeight`, `color` |
| `Button` | `variant`, `color`, `size`, `startIcon`, `loading` |
| `IconButton` + `Tooltip` | Botones de ícono con tooltip |
| `TextField` | Input, email, password, `select` con `MenuItem` |
| `Alert` + `AlertTitle` + `Collapse` | Mensajes con severidad y animación |
| `Card`, `CardContent`, `CardMedia`, `CardActions` | Tarjetas con imagen y acciones |
| `Chip` | Etiquetas con `color` semántico |
| `Table` + `TableSortLabel` | Tabla con sorting manual |
| `Pagination` | Paginación automática con `count` y `page` |
| `Tabs` + `Tab` | Navegación por pestañas |
| `AppBar` + `Toolbar` | Barra de navegación fija |
| `Drawer` | Menú lateral temporal y permanente |
| `CircularProgress` | Spinner de carga |
| `Avatar` | Imagen redonda o rectangular con fallback |

---

## Resumen de la página 19

- `ThemeProvider` + `CssBaseline` son obligatorios en `main.tsx` — sin ellos los componentes funcionan pero sin tema ni reset CSS.
- MUI v7 elimina las props `item`, `xs`, `md` de `Grid` — se reemplaza con `size={{ xs: 12, md: 4 }}`.
- La prop `sx` accede a valores del tema por nombre — `bgcolor: 'background.paper'`, `color: 'text.secondary'`, `borderRadius: 2`.
- `TextField select` + `MenuItem` es la forma MUI de construir selects tipados y estilizados consistentemente.
- `slotProps.input` reemplaza al antiguo `InputProps` en MUI v7.
- `Button loading` es nativo en MUI v7 — muestra el spinner sin código extra.
- `Collapse` + `Alert` hace aparecer y desaparecer alertas con animación suave.
- `Pagination` con `count={totalPages}` gestiona la paginación automáticamente — no necesitas botones manuales.
- `NavLink` de React Router funciona directamente con `style` como función — `style={({ isActive }) => ({...})}`.
- Los íconos de `@mui/icons-material` requieren instalación por separado: `npm install @mui/icons-material`.

---

> Con esta página concluyen los 4 frameworks de estilos del curso:
> React-Bootstrap v2 (pág. 16), Tailwind CSS v4 (pág. 17),
> Ant Design v6 (pág. 18) y Material UI v7 (pág. 19).