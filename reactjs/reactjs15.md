# Curso React.js + TypeScript — Página 12
## Módulo 6 · Ecosistema
### Formularios y validación: controlados, validación manual y Zod v4

---

## Formularios en React: controlados vs no controlados

En React hay dos enfoques para manejar formularios:

| Enfoque | Quién controla el valor | Cuándo usarlo |
|---|---|---|
| **Controlado** | React (el estado) | La mayoría de los casos — formularios con validación, valores que dependen entre sí |
| **No controlado** | El DOM | Formularios muy simples, subida de archivos, integración con librerías externas |

El curso usa formularios **controlados** — el valor de cada campo vive
en el estado de React y se sincroniza con el DOM mediante `value` + `onChange`.

---

## Instalación de Zod

```bash
npm install zod
```

La versión actual es **Zod v4** — tiene cambios de API respecto a v3.
El curso usa v4 directamente.

---

## Formulario controlado básico

El patrón mínimo: un estado por campo, un `onChange` que lo actualiza,
un `onSubmit` que previene el comportamiento nativo y procesa los datos.

```tsx
// src/components/BasicForm.tsx

import { useState } from 'react'

export default function BasicForm() {
  const [name,  setName]  = useState('')
  const [email, setEmail] = useState('')
  const [error, setError] = useState<string | null>(null)

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()   // evita que la página se recargue

    if (!name.trim() || !email.includes('@')) {
      setError('Completa todos los campos correctamente')
      return
    }

    setError(null)
    console.log('Datos enviados:', { name, email })
    // aquí iría el fetch a la API
  }

  return (
    <form
      onSubmit={handleSubmit}
      style={{ display: 'flex', flexDirection: 'column', gap: 10, maxWidth: 320 }}
    >
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Nombre"
        style={inputStyle}
      />
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Correo electrónico"
        style={inputStyle}
      />

      {error && (
        <p style={{ margin: 0, fontSize: 13, color: '#ef4444' }}>{error}</p>
      )}

      <button type="submit" style={submitBtn}>
        Enviar
      </button>
    </form>
  )
}

const inputStyle = {
  padding: '8px 12px',
  border: '1px solid #d1d5db',
  borderRadius: 6, fontSize: 14,
}

const submitBtn = {
  padding: '10px',
  background: '#0070f3', color: '#fff',
  border: 'none', borderRadius: 6,
  cursor: 'pointer', fontWeight: 500,
}
```

### Prueba esto

- Haz clic en "Enviar" con los campos vacíos — observa que aparece el mensaje "Completa todos los campos correctamente" en rojo
- Escribe un nombre válido pero deja el email sin `@` (por ejemplo `usuarioemail.com`) y haz clic en "Enviar" — el mismo mensaje de error aparece porque la validación es conjunta
- Escribe un email con `@` pero sin dominio (por ejemplo `usuario@`) — el formulario se envía igualmente porque `includes('@')` solo verifica la presencia del símbolo
- Abre la consola del navegador (F12) y envía el formulario con datos válidos — observa que aparece `Datos enviados: { name: '...', email: '...' }` en la consola
- Cambia `!email.includes('@')` por `!/^[^@]+@[^@]+\.[^@]+$/.test(email)` en la validación — ahora `usuario@` ya no pasa la validación y el error aparece correctamente
- Añade un tercer campo `phone` con su propio `useState` y agrega la condición `!phone.trim()` a la validación — el formulario ahora requiere los tres campos

---

## Validación manual con TypeScript

Para formularios más complejos conviene separar la lógica de validación
en una función pura tipada y gestionar el estado de errores y campos
tocados (touched) por separado.

```tsx
// src/components/ManualValidationForm.tsx

import { useState } from 'react'

interface FormValues {
  name:     string
  email:    string
  password: string
}

// Partial — cada campo de error es opcional
type FormErrors = Partial<Record<keyof FormValues, string>>
type TouchedFields = Partial<Record<keyof FormValues, boolean>>

// Función pura — recibe valores, retorna errores
function validateForm(values: FormValues): FormErrors {
  const errors: FormErrors = {}

  if (!values.name.trim())
    errors.name = 'El nombre es requerido'
  else if (values.name.trim().length < 2)
    errors.name = 'Mínimo 2 caracteres'

  if (!values.email.includes('@'))
    errors.email = 'Introduce un email válido'

  if (values.password.length < 8)
    errors.password = 'La contraseña debe tener al menos 8 caracteres'

  return errors
}

export default function ManualValidationForm() {
  const [values, setValues] = useState<FormValues>({
    name: '', email: '', password: '',
  })
  const [errors,  setErrors]  = useState<FormErrors>({})
  const [touched, setTouched] = useState<TouchedFields>({})
  const [success, setSuccess] = useState(false)

  function handleChange(field: keyof FormValues, value: string) {
    const newValues = { ...values, [field]: value }
    setValues(newValues)

    // Valida en tiempo real solo si el campo ya fue tocado (blur)
    if (touched[field]) {
      const newErrors = validateForm(newValues)
      setErrors((prev) => ({ ...prev, [field]: newErrors[field] }))
    }
  }

  function handleBlur(field: keyof FormValues) {
    setTouched((prev) => ({ ...prev, [field]: true }))
    const fieldErrors = validateForm(values)
    setErrors((prev) => ({ ...prev, [field]: fieldErrors[field] }))
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()

    // Valida todos los campos al enviar
    const allErrors = validateForm(values)

    if (Object.keys(allErrors).length > 0) {
      setErrors(allErrors)
      setTouched({ name: true, email: true, password: true })
      return
    }

    setSuccess(true)
    console.log('Datos válidos:', values)
  }

  const hasErrors = Object.keys(errors).some(
    (k) => errors[k as keyof FormErrors]
  )

  return (
    <form
      onSubmit={handleSubmit}
      style={{ display: 'flex', flexDirection: 'column', gap: 12, maxWidth: 320 }}
    >
      {success && (
        <div style={{ padding: 12, background: '#dcfce7', borderRadius: 6, color: '#166534' }}>
          ✅ Formulario enviado correctamente
        </div>
      )}

      <Field
        label="Nombre"
        value={values.name}
        error={errors.name}
        placeholder="Tu nombre completo"
        onChange={(v) => handleChange('name', v)}
        onBlur={() => handleBlur('name')}
      />

      <Field
        label="Email"
        type="email"
        value={values.email}
        error={errors.email}
        placeholder="tu@email.com"
        onChange={(v) => handleChange('email', v)}
        onBlur={() => handleBlur('email')}
      />

      <Field
        label="Contraseña"
        type="password"
        value={values.password}
        error={errors.password}
        placeholder="Mínimo 8 caracteres"
        onChange={(v) => handleChange('password', v)}
        onBlur={() => handleBlur('password')}
      />

      <button
        type="submit"
        disabled={success}
        style={{
          padding: '10px', background: '#0070f3', color: '#fff',
          border: 'none', borderRadius: 6, cursor: 'pointer', fontWeight: 500,
          opacity: success ? 0.5 : 1,
        }}
      >
        Registrar
      </button>
    </form>
  )
}

// Componente auxiliar para evitar repetición
interface FieldProps {
  label:       string
  value:       string
  error?:      string
  placeholder?: string
  type?:       string
  onChange:    (value: string) => void
  onBlur:      () => void
}

function Field({ label, value, error, placeholder, type = 'text', onChange, onBlur }: FieldProps) {
  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 4 }}>
      <label style={{ fontSize: 13, fontWeight: 500, color: '#374151' }}>{label}</label>
      <input
        type={type}
        value={value}
        onChange={(e) => onChange(e.target.value)}
        onBlur={onBlur}
        placeholder={placeholder}
        style={{
          padding: '8px 12px', fontSize: 14,
          border: `1px solid ${error ? '#ef4444' : '#d1d5db'}`,
          borderRadius: 6,
          outline: error ? '2px solid #fecaca' : 'none',
        }}
      />
      {error && (
        <p style={{ margin: 0, fontSize: 12, color: '#ef4444' }}>{error}</p>
      )}
    </div>
  )
}
```

### Prueba esto

- Haz clic en "Registrar" sin tocar ningún campo — observa que los tres errores aparecen al mismo tiempo porque `handleSubmit` valida todos los campos a la vez
- Haz clic en el campo "Nombre", escribe una letra y luego sal del campo con Tab — observa que el error "Mínimo 2 caracteres" aparece inmediatamente al perder el foco (`onBlur`)
- Corrige el nombre hasta que sea válido, luego borra una letra — observa que el error reaparece en tiempo real porque el campo ya fue tocado (`touched.name === true`)
- Cambia `values.password.length < 8` por `values.password.length < 12` en `validateForm` — ahora se requieren mínimo 12 caracteres y el mensaje de error se actualiza
- Envía el formulario con todos los campos válidos — aparece el banner verde "Formulario enviado correctamente" y el botón se deshabilita (`disabled={success}`)
- Añade un campo `confirmPassword` al tipo `FormValues` y valida en `validateForm` que coincida con `password` — el error "Las contraseñas no coinciden" aparece si son diferentes
- Cambia el condicional `if (touched[field])` en `handleChange` por `if (true)` — la validación ocurre en tiempo real desde el primer carácter sin necesidad de salir del campo

---

## Validación con Zod v4

Zod valida datos en runtime y genera los tipos TypeScript automáticamente.
En lugar de escribir tanto la `interface` como la validación por separado,
el schema es la única fuente de verdad.

```bash
npm install zod   # instala Zod v4
```

```tsx
// src/components/ZodRegistrationForm.tsx

import { useState } from 'react'
import { z }        from 'zod'

// 1. Definir el schema — es la única fuente de verdad
const RegisterSchema = z.object({
  fullName:  z.string().min(2, 'Mínimo 2 caracteres'),
  email:     z.string().email('Introduce un email válido'),
  password:  z.string()
    .min(8, 'Mínimo 8 caracteres')
    .regex(/[A-Z]/, 'Debe contener al menos una mayúscula')
    .regex(/[0-9]/, 'Debe contener al menos un número'),
  confirm:   z.string(),
  role:      z.enum(['admin', 'editor', 'viewer']),
  birthYear: z.number({ error: 'Debe ser un número' })
    .int('Debe ser un año completo')
    .min(1900, 'Año inválido')
    .max(new Date().getFullYear() - 18, 'Debes ser mayor de edad'),
}).refine(
  (data) => data.password === data.confirm,
  { message: 'Las contraseñas no coinciden', path: ['confirm'] }
)

// 2. Inferir el tipo desde el schema — sin duplicar la interface
type RegisterFormData = z.infer<typeof RegisterSchema>

// Errores: un string opcional por campo
type FormErrors = Partial<Record<keyof RegisterFormData, string>>

const INITIAL_VALUES: RegisterFormData = {
  fullName:  '',
  email:     '',
  password:  '',
  confirm:   '',
  role:      'viewer',
  birthYear: 2000,
}

export default function ZodRegistrationForm() {
  const [values, setValues] = useState<RegisterFormData>(INITIAL_VALUES)
  const [errors, setErrors] = useState<FormErrors>({})
  const [success, setSuccess] = useState(false)

  // Tipado genérico — field es una clave válida, value tiene el tipo correcto
  function handleChange<K extends keyof RegisterFormData>(
    field: K,
    value: RegisterFormData[K]
  ) {
    setValues((prev) => ({ ...prev, [field]: value }))
    // Limpia el error del campo al escribir
    setErrors((prev) => ({ ...prev, [field]: undefined }))
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()

    // safeParse — nunca lanza, retorna { success, data } o { success, error }
    const result = RegisterSchema.safeParse(values)

    if (!result.success) {
      // Convertir los issues de Zod a nuestro mapa de errores
      const zodErrors: FormErrors = {}
      for (const issue of result.error.issues) {
        const field = issue.path[0] as keyof RegisterFormData
        // Guarda solo el primer error por campo
        if (field && !zodErrors[field]) {
          zodErrors[field] = issue.message
        }
      }
      setErrors(zodErrors)
      return
    }

    // result.data está completamente tipado — TypeScript lo sabe
    console.log('Datos validados:', result.data)
    setSuccess(true)
  }

  return (
    <form
      onSubmit={handleSubmit}
      style={{ display: 'flex', flexDirection: 'column', gap: 12, maxWidth: 360 }}
    >
      {success && (
        <div style={{ padding: 12, background: '#dcfce7', borderRadius: 6, color: '#166534' }}>
          ✅ Registro completado
        </div>
      )}

      {/* Nombre */}
      <FormField
        label="Nombre completo"
        value={values.fullName}
        error={errors.fullName}
        placeholder="Ana García"
        onChange={(v) => handleChange('fullName', v)}
      />

      {/* Email */}
      <FormField
        label="Correo electrónico"
        type="email"
        value={values.email}
        error={errors.email}
        placeholder="tu@email.com"
        onChange={(v) => handleChange('email', v)}
      />

      {/* Contraseña */}
      <FormField
        label="Contraseña"
        type="password"
        value={values.password}
        error={errors.password}
        placeholder="Mín. 8 caracteres, 1 mayúscula, 1 número"
        onChange={(v) => handleChange('password', v)}
      />

      {/* Confirmar contraseña */}
      <FormField
        label="Confirmar contraseña"
        type="password"
        value={values.confirm}
        error={errors.confirm}
        placeholder="Repite la contraseña"
        onChange={(v) => handleChange('confirm', v)}
      />

      {/* Rol */}
      <div style={{ display: 'flex', flexDirection: 'column', gap: 4 }}>
        <label style={{ fontSize: 13, fontWeight: 500, color: '#374151' }}>Rol</label>
        <select
          value={values.role}
          onChange={(e) =>
            handleChange('role', e.target.value as RegisterFormData['role'])
          }
          style={{ padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6 }}
        >
          <option value="viewer">Viewer</option>
          <option value="editor">Editor</option>
          <option value="admin">Admin</option>
        </select>
        {errors.role && <p style={errorStyle}>{errors.role}</p>}
      </div>

      {/* Año de nacimiento */}
      <div style={{ display: 'flex', flexDirection: 'column', gap: 4 }}>
        <label style={{ fontSize: 13, fontWeight: 500, color: '#374151' }}>
          Año de nacimiento
        </label>
        <input
          type="number"
          value={values.birthYear}
          onChange={(e) => handleChange('birthYear', Number(e.target.value))}
          style={{ padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6 }}
        />
        {errors.birthYear && <p style={errorStyle}>{errors.birthYear}</p>}
      </div>

      <button
        type="submit"
        style={{
          padding: '10px', background: '#0070f3', color: '#fff',
          border: 'none', borderRadius: 6, cursor: 'pointer', fontWeight: 500,
        }}
      >
        Registrar
      </button>
    </form>
  )
}

interface FormFieldProps {
  label:        string
  value:        string
  error?:       string
  placeholder?: string
  type?:        string
  onChange:     (value: string) => void
}

function FormField({ label, value, error, placeholder, type = 'text', onChange }: FormFieldProps) {
  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 4 }}>
      <label style={{ fontSize: 13, fontWeight: 500, color: '#374151' }}>{label}</label>
      <input
        type={type}
        value={value}
        onChange={(e) => onChange(e.target.value)}
        placeholder={placeholder}
        style={{
          padding: '8px 12px', fontSize: 14,
          border: `1px solid ${error ? '#ef4444' : '#d1d5db'}`,
          borderRadius: 6,
        }}
      />
      {error && <p style={errorStyle}>{error}</p>}
    </div>
  )
}

const errorStyle = { margin: 0, fontSize: 12, color: '#ef4444' }
```

### Prueba esto

- Haz clic en "Registrar" con todos los campos vacíos — observa los mensajes de error de Zod en rojo bajo cada campo: "Mínimo 2 caracteres", "Introduce un email válido", etc.
- Escribe una contraseña sin mayúsculas como `contraseña1` — el error "Debe contener al menos una mayúscula" aparece al intentar enviar el formulario
- Escribe una contraseña válida en "Contraseña" pero distinta en "Confirmar contraseña" — el error "Las contraseñas no coinciden" aparece solo bajo el campo de confirmación gracias a `path: ['confirm']` en `refine`
- Cambia el año de nacimiento a `2010` y haz clic en "Registrar" — aparece el error "Debes ser mayor de edad" porque `2010` es menor que `añoActual - 18`
- Modifica la regla `.min(8, ...)` de la contraseña a `.min(12, 'Mínimo 12 caracteres')` en el schema — el mensaje de error se actualiza automáticamente sin cambiar nada más en el componente
- Escribe en el campo "Nombre completo" y luego bórralo — el error desaparece al escribir porque `handleChange` ejecuta `setErrors((prev) => ({ ...prev, [field]: undefined }))` en cada cambio
- Envía el formulario con todos los datos válidos y abre la consola — observa que `result.data` está completamente tipado: TypeScript infiere los tipos desde el schema de Zod

---

## Formulario de contacto con `useReducer`

Para formularios que necesitan gestionar envío async con múltiples estados,
`useReducer` es más ordenado que varios `useState` coordinados.

```tsx
// src/components/ContactForm.tsx

import { useReducer } from 'react'

interface ContactState {
  name:    string
  email:   string
  subject: string
  message: string
  errors:  Partial<Record<'name' | 'email' | 'subject' | 'message', string>>
  status:  'idle' | 'sending' | 'sent' | 'error'
}

type ContactAction =
  | { type: 'SET_FIELD'; field: keyof Pick<ContactState, 'name' | 'email' | 'subject' | 'message'>; value: string }
  | { type: 'SET_ERRORS'; errors: ContactState['errors'] }
  | { type: 'SENDING' }
  | { type: 'SENT' }
  | { type: 'ERROR' }
  | { type: 'RESET' }

const INITIAL: ContactState = {
  name: '', email: '', subject: '', message: '',
  errors: {}, status: 'idle',
}

function contactReducer(state: ContactState, action: ContactAction): ContactState {
  switch (action.type) {
    case 'SET_FIELD':
      return {
        ...state,
        [action.field]: action.value,
        errors: { ...state.errors, [action.field]: undefined },
      }
    case 'SET_ERRORS': return { ...state, errors: action.errors }
    case 'SENDING':    return { ...state, status: 'sending' }
    case 'SENT':       return { ...INITIAL, status: 'sent' }
    case 'ERROR':      return { ...state, status: 'error' }
    case 'RESET':      return INITIAL
  }
}

export default function ContactForm() {
  const [state, dispatch] = useReducer(contactReducer, INITIAL)

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()

    const errors: ContactState['errors'] = {}
    if (!state.name.trim())         errors.name    = 'Requerido'
    if (!state.email.includes('@')) errors.email   = 'Email inválido'
    if (!state.subject.trim())      errors.subject = 'Requerido'
    if (state.message.length < 10)  errors.message = 'Mínimo 10 caracteres'

    if (Object.keys(errors).length > 0) {
      dispatch({ type: 'SET_ERRORS', errors })
      return
    }

    dispatch({ type: 'SENDING' })
    try {
      await new Promise((r) => setTimeout(r, 1200)) // simula fetch
      dispatch({ type: 'SENT' })
    } catch {
      dispatch({ type: 'ERROR' })
    }
  }

  const isSending = state.status === 'sending'

  return (
    <form
      onSubmit={handleSubmit}
      style={{ display: 'flex', flexDirection: 'column', gap: 12, maxWidth: 360 }}
    >
      {state.status === 'sent' && (
        <div style={{ padding: 12, background: '#dcfce7', borderRadius: 6, color: '#166534' }}>
          ✅ Mensaje enviado. Te responderemos pronto.
        </div>
      )}
      {state.status === 'error' && (
        <div style={{ padding: 12, background: '#fef2f2', borderRadius: 6, color: '#dc2626' }}>
          ❌ Error al enviar. Inténtalo de nuevo.
        </div>
      )}

      {(['name', 'email', 'subject'] as const).map((field) => (
        <div key={field} style={{ display: 'flex', flexDirection: 'column', gap: 4 }}>
          <label style={{ fontSize: 13, fontWeight: 500, color: '#374151', textTransform: 'capitalize' }}>
            {field === 'name' ? 'Nombre' : field === 'email' ? 'Email' : 'Asunto'}
          </label>
          <input
            type={field === 'email' ? 'email' : 'text'}
            value={state[field]}
            onChange={(e) => dispatch({ type: 'SET_FIELD', field, value: e.target.value })}
            disabled={isSending}
            style={{
              padding: '8px 12px', fontSize: 14,
              border: `1px solid ${state.errors[field] ? '#ef4444' : '#d1d5db'}`,
              borderRadius: 6,
            }}
          />
          {state.errors[field] && (
            <p style={{ margin: 0, fontSize: 12, color: '#ef4444' }}>
              {state.errors[field]}
            </p>
          )}
        </div>
      ))}

      <div style={{ display: 'flex', flexDirection: 'column', gap: 4 }}>
        <label style={{ fontSize: 13, fontWeight: 500, color: '#374151' }}>Mensaje</label>
        <textarea
          value={state.message}
          onChange={(e) => dispatch({ type: 'SET_FIELD', field: 'message', value: e.target.value })}
          disabled={isSending}
          rows={4}
          placeholder="Escribe tu mensaje (mínimo 10 caracteres)"
          style={{
            padding: '8px 12px', fontSize: 14, resize: 'vertical',
            border: `1px solid ${state.errors.message ? '#ef4444' : '#d1d5db'}`,
            borderRadius: 6, fontFamily: 'inherit',
          }}
        />
        {state.errors.message && (
          <p style={{ margin: 0, fontSize: 12, color: '#ef4444' }}>{state.errors.message}</p>
        )}
      </div>

      <div style={{ display: 'flex', gap: 8 }}>
        <button
          type="submit"
          disabled={isSending}
          style={{
            flex: 1, padding: '10px',
            background: isSending ? '#93c5fd' : '#0070f3',
            color: '#fff', border: 'none', borderRadius: 6,
            cursor: isSending ? 'not-allowed' : 'pointer', fontWeight: 500,
          }}
        >
          {isSending ? 'Enviando...' : 'Enviar mensaje'}
        </button>
        <button
          type="button"
          onClick={() => dispatch({ type: 'RESET' })}
          disabled={isSending}
          style={{
            padding: '10px 16px', background: '#f3f4f6', color: '#6b7280',
            border: 'none', borderRadius: 6, cursor: 'pointer',
          }}
        >
          Limpiar
        </button>
      </div>
    </form>
  )
}
```

### Prueba esto

- Haz clic en "Enviar mensaje" con todos los campos vacíos — observa que los cuatro errores aparecen a la vez: "Requerido", "Email inválido", "Requerido" y "Mínimo 10 caracteres"
- Escribe algo en el campo "Nombre" — observa que el error de ese campo desaparece de inmediato porque `SET_FIELD` incluye `errors: { ...state.errors, [field]: undefined }` en el reducer
- Haz clic en "Enviar mensaje" con datos válidos — el botón cambia a "Enviando..." y se deshabilita durante 1,2 segundos simulando una llamada a la API
- Tras el envío exitoso, observa que el formulario se limpia automáticamente y aparece el banner verde — esto ocurre porque `SENT` retorna `{ ...INITIAL, status: 'sent' }`, no el estado actual
- Haz clic en "Limpiar" mientras el formulario tiene texto — todos los campos vuelven a estar vacíos porque `RESET` retorna `INITIAL` directamente
- Añade un nuevo caso al reducer: `case 'RESET_ERRORS': return { ...state, errors: {} }` y agrega un botón que lo dispare — los errores desaparecen sin resetear los valores
- Cambia el `setTimeout` de 1200 a 50 y lanza un `throw new Error()` dentro del `try` — el banner rojo "Error al enviar" aparece y el formulario mantiene los datos para reintentarlo

---

## `src/App.tsx`

```tsx
// src/App.tsx

import BasicForm            from './components/BasicForm'
import ManualValidationForm from './components/ManualValidationForm'
import ZodRegistrationForm  from './components/ZodRegistrationForm'
import ContactForm          from './components/ContactForm'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  BasicForm            — formulario controlado básico             │
// │  2  ManualValidationForm — validación manual con touched            │
// │  3  ZodRegistrationForm  — validación con Zod v4                   │
// │  4  ContactForm          — envío async con useReducer               │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content =
    PASO === 1 ? <BasicForm /> :
    PASO === 2 ? <ManualValidationForm /> :
    PASO === 3 ? <ZodRegistrationForm /> :
    PASO === 4 ? <ContactForm /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 420, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

---

## Zod v4 — validators más usados

```ts
import { z } from 'zod'

z.string()                           // string cualquiera
z.string().min(2)                    // mínimo 2 caracteres
z.string().max(100)                  // máximo 100 caracteres
z.string().email()                   // formato email
z.string().url()                     // formato URL
z.string().regex(/[A-Z]/)           // expresión regular
z.string().trim()                    // elimina espacios al validar
z.string().optional()               // string | undefined
z.string().nullable()               // string | null
z.string().default('valor')          // valor por defecto si undefined

z.number()                           // número
z.number().int()                     // entero
z.number().min(0)                    // mínimo
z.number().max(100)                  // máximo
z.number({ error: 'Mensaje' })       // mensaje de error personalizado en v4

z.boolean()
z.date()
z.enum(['a', 'b', 'c'])             // valores literales
z.array(z.string())                  // array de strings
z.object({ ... })                    // objeto con schema

// Refinamiento — validación cruzada entre campos
schema.refine(
  (data) => data.password === data.confirm,
  { message: 'No coinciden', path: ['confirm'] }
)

// Transformación
z.string().transform((v) => v.trim().toLowerCase())

// Inferir tipo desde schema
type MyType = z.infer<typeof MySchema>
```

---

## Qué estrategia usar según el caso

| Caso | Estrategia recomendada |
|---|---|
| Formulario simple, 1–2 campos | `useState` + validación inline |
| Formulario mediano con varios campos | `useState` + función `validate()` separada + campo `touched` |
| Formulario con validaciones complejas o reutilizables | `useState` + **Zod** |
| Formulario con estados de envío async | `useReducer` + Zod o validación manual |
| Muchos formularios en la app | Considera **React Hook Form** + Zod |

---

## Ejercicios propuestos

1. **`LoginForm` con Zod** — crea un formulario de login con email y contraseña.
   Valida con Zod: email debe ser válido, contraseña mínimo 6 caracteres.
   Al enviar, simula una llamada async y muestra el estado de carga.

2. **`ProfileForm` con imagen** — formulario con nombre, bio (máx 200 chars)
   y un input `type="file"` para avatar. El campo de archivo es no controlado
   (usa `useRef<HTMLInputElement>`). Valida nombre y bio con Zod.

3. **`SearchForm` con URL** — formulario con un input de búsqueda que sincroniza
   con `useSearchParams` de React Router. Al enviar, actualiza `?q=valor` en la URL.

---

## Resumen de la página 12

- Los formularios controlados vinculan cada campo a un estado de React con `value` + `onChange`.
- `e.preventDefault()` en `onSubmit` es obligatorio — evita que el navegador recargue la página.
- El patrón `touched` (campo tocado) permite mostrar errores solo después de que el usuario interactuó con el campo — mejor UX que validar al escribir desde el inicio.
- Zod v4: `z.object().safeParse()` nunca lanza — retorna `{ success, data }` o `{ success, error }`.
- `z.infer<typeof Schema>` genera el tipo TypeScript automáticamente — el schema es la única fuente de verdad.
- En Zod v4 el mensaje de error para tipos base (`z.number()`) va en `{ error: 'mensaje' }`, no en `message`.
- `useReducer` es ideal para formularios con estados de envío async — centraliza todos los estados en un solo lugar.
- Para apps con muchos formularios complejos, considera **React Hook Form** + Zod como capa superior.

---

> **Siguiente página →** Testing con Vitest y React Testing Library:
> testear componentes, hooks y comportamiento del usuario.