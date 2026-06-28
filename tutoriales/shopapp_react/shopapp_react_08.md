# ShopApp React Web — Módulo 8

## Perfil — Vista y edición del perfil de usuario

**Duración estimada:** 2–3 horas
**Prerequisitos:** Módulos 1–7 completados, backend con endpoint `GET /users/me/` y `PATCH /users/me/` funcionando.

---

> **Objetivo**
> Implementar el módulo de perfil de usuario: tipos, API, store Zustand, componente de avatar con
> fallback de iniciales, y una página de perfil con tabs que permita ver la información y editar los
> datos mediante un formulario validado con React Hook Form + Zod. Al finalizar, el avatar también
> se mostrará en el encabezado de la aplicación.
>
> **Checkpoint final**
> - La ruta `/profile` está protegida y carga los datos del usuario desde `GET /users/me/`.
> - El formulario de edición valida los campos y llama a `PATCH /users/me/` al guardar.
> - Un toast de shadcn confirma la actualización exitosa.
> - El componente `UserAvatar` muestra la imagen del perfil o las iniciales del usuario.
> - El avatar del header se actualiza desde `profileStore` después de iniciar sesión.

---

## 8.1 Tipos de usuario

**Archivo:** `src/domain/model/user.types.ts`

Este archivo define los contratos de datos del módulo de perfil. `UserProfile` representa la
respuesta completa del endpoint `/users/me/`, e incluye todos los campos que el backend devuelve.
`UpdateProfilePayload` describe los campos opcionales que se pueden enviar en una solicitud `PATCH`.

```typescript
// src/domain/model/user.types.ts

export interface UserProfile {
  user_id: number;
  username: string;
  email: string;
  first_name: string;
  last_name: string;
  avatar: string | null;
  date_joined: string;   // ISO 8601, ej. "2024-03-15T10:30:00Z"
  is_staff: boolean;
}

export interface UpdateProfilePayload {
  first_name?: string;
  last_name?: string;
  email?: string;
}
```

---

## 8.2 API de usuarios

**Archivo:** `src/data/api/users.api.ts`

Este módulo encapsula todas las llamadas HTTP relacionadas con el perfil de usuario. Utiliza el
`apiClient` (que ya incluye el interceptor JWT) para que las solicitudes se autentiquen
automáticamente. La separación de la lógica de red en esta capa permite reemplazar el transporte
sin tocar la presentación.

```typescript
// src/data/api/users.api.ts

import { apiClient } from './api.client';
import type { UserProfile, UpdateProfilePayload } from '../../domain/model/user.types';

/**
 * Obtiene el perfil completo del usuario autenticado.
 * Requiere token JWT válido en la cabecera Authorization.
 */
export async function getMyProfile(): Promise<UserProfile> {
  const response = await apiClient.get<UserProfile>('/users/me/');
  return response.data;
}

/**
 * Actualiza parcialmente el perfil del usuario autenticado.
 * Solo envía los campos incluidos en el payload.
 */
export async function updateMyProfile(
  payload: UpdateProfilePayload
): Promise<UserProfile> {
  const response = await apiClient.patch<UserProfile>('/users/me/', payload);
  return response.data;
}
```

---

## 8.3 ProfileStore

**Archivo:** `src/domain/store/profile.store.ts`

El store de perfil mantiene en memoria el `UserProfile` cargado, así como indicadores de carga
(`isLoading`) y guardado (`isSaving`). La acción `fetchProfile` obtiene los datos del backend y
los almacena; `updateProfile` envía el payload de edición y actualiza el estado local con la
respuesta del servidor, garantizando que la UI refleje siempre la verdad del backend.

```typescript
// src/domain/store/profile.store.ts

import { create } from 'zustand';
import type { UserProfile, UpdateProfilePayload } from '../model/user.types';
import { getMyProfile, updateMyProfile } from '../../data/api/users.api';

interface ProfileState {
  profile: UserProfile | null;
  isLoading: boolean;
  isSaving: boolean;
  error: string | null;

  fetchProfile: () => Promise<void>;
  updateProfile: (payload: UpdateProfilePayload) => Promise<void>;
  clearProfile: () => void;
}

export const useProfileStore = create<ProfileState>((set) => ({
  profile: null,
  isLoading: false,
  isSaving: false,
  error: null,

  fetchProfile: async () => {
    set({ isLoading: true, error: null });
    try {
      const profile = await getMyProfile();
      set({ profile, isLoading: false });
    } catch (err) {
      const message =
        err instanceof Error ? err.message : 'No se pudo cargar el perfil.';
      set({ error: message, isLoading: false });
    }
  },

  updateProfile: async (payload: UpdateProfilePayload) => {
    set({ isSaving: true, error: null });
    try {
      const updated = await updateMyProfile(payload);
      set({ profile: updated, isSaving: false });
    } catch (err) {
      const message =
        err instanceof Error ? err.message : 'No se pudo actualizar el perfil.';
      set({ error: message, isSaving: false });
      // Re-lanzamos para que el formulario pueda capturarlo
      throw new Error(message);
    }
  },

  clearProfile: () => set({ profile: null, error: null }),
}));
```

---

## 8.4 Componente UserAvatar

**Archivo:** `src/presentation/components/UserAvatar.tsx`

`UserAvatar` es un componente reutilizable que muestra la imagen de perfil del usuario usando el
componente `Avatar` de shadcn/ui. Cuando no hay imagen disponible, calcula y muestra las iniciales
del usuario: toma la primera letra del nombre y del apellido, o la primera letra del nombre de
usuario como fallback final. El prop `size` permite usarlo tanto en el header (pequeño) como en la
página de perfil (grande).

Antes de continuar, asegúrate de tener instalado el componente Avatar de shadcn:

```bash
npx shadcn@latest add avatar
```

```tsx
// src/presentation/components/UserAvatar.tsx

import { Avatar, AvatarImage, AvatarFallback } from '@/components/ui/avatar';
import { cn } from '@/lib/utils';
import type { UserProfile } from '../../domain/model/user.types';

interface UserAvatarProps {
  user: UserProfile | null;
  size?: 'sm' | 'md' | 'lg';
}

const sizeClasses: Record<NonNullable<UserAvatarProps['size']>, string> = {
  sm: 'h-8 w-8 text-xs',
  md: 'h-10 w-10 text-sm',
  lg: 'h-20 w-20 text-2xl',
};

function getInitials(user: UserProfile): string {
  const first = user.first_name?.trim();
  const last = user.last_name?.trim();

  if (first && last) {
    return `${first[0]}${last[0]}`.toUpperCase();
  }
  if (first) {
    return first[0].toUpperCase();
  }
  return user.username[0].toUpperCase();
}

export function UserAvatar({ user, size = 'md' }: UserAvatarProps) {
  const sizeClass = sizeClasses[size];

  return (
    <Avatar className={cn(sizeClass)}>
      {user?.avatar && (
        <AvatarImage
          src={user.avatar}
          alt={`Avatar de ${user.username}`}
        />
      )}
      <AvatarFallback className="bg-primary text-primary-foreground font-semibold">
        {user ? getInitials(user) : '?'}
      </AvatarFallback>
    </Avatar>
  );
}
```

---

## 8.5 ProfilePage

**Archivo:** `src/presentation/pages/profile/ProfilePage.tsx`

La página de perfil es una ruta protegida que se organiza en dos pestañas mediante `Tabs` de
shadcn/ui. La pestaña "Información" muestra el avatar grande, el nombre de usuario, el email, la
fecha de ingreso formateada con `formatDate` del módulo de formatters, y un badge de
administrador si corresponde. La pestaña "Editar perfil" contiene el formulario de React Hook
Form con validación Zod; al guardar llama a `profileStore.updateProfile()` y muestra un toast de
confirmación o el mensaje de error inline. La pestaña "Órdenes" redirige al historial de pedidos.

Primero instala los componentes necesarios de shadcn si aún no los tienes:

```bash
npx shadcn@latest add card form input label button separator tabs badge toast
```

```tsx
// src/presentation/pages/profile/ProfilePage.tsx

import { useEffect } from 'react';
import { Link } from 'react-router-dom';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Loader2, ShieldCheck, ExternalLink } from 'lucide-react';

import { useProfileStore } from '../../../domain/store/profile.store';
import { UserAvatar } from '../../components/UserAvatar';
import { formatDate } from '../../../core/utils/formatters';

import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from '@/components/ui/card';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import { Separator } from '@/components/ui/separator';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Badge } from '@/components/ui/badge';
import { useToast } from '@/components/ui/use-toast';

// ── Esquema Zod ─────────────────────────────────────────────────────────────

const profileSchema = z.object({
  first_name: z
    .string()
    .max(150, 'Máximo 150 caracteres')
    .optional()
    .or(z.literal('')),
  last_name: z
    .string()
    .max(150, 'Máximo 150 caracteres')
    .optional()
    .or(z.literal('')),
  email: z
    .string()
    .email('Introduce un correo electrónico válido')
    .min(1, 'El correo es requerido'),
});

type ProfileFormValues = z.infer<typeof profileSchema>;

// ── Componente principal ─────────────────────────────────────────────────────

export default function ProfilePage() {
  const { profile, isLoading, isSaving, error, fetchProfile, updateProfile } =
    useProfileStore();
  const { toast } = useToast();

  // Carga el perfil al montar la página
  useEffect(() => {
    fetchProfile();
  }, [fetchProfile]);

  const form = useForm<ProfileFormValues>({
    resolver: zodResolver(profileSchema),
    values: {
      first_name: profile?.first_name ?? '',
      last_name: profile?.last_name ?? '',
      email: profile?.email ?? '',
    },
  });

  // Sincroniza los valores del formulario cuando el perfil se carga
  useEffect(() => {
    if (profile) {
      form.reset({
        first_name: profile.first_name,
        last_name: profile.last_name,
        email: profile.email,
      });
    }
  }, [profile, form]);

  const onSubmit = async (data: ProfileFormValues) => {
    try {
      await updateProfile({
        first_name: data.first_name || undefined,
        last_name: data.last_name || undefined,
        email: data.email,
      });
      toast({
        title: 'Perfil actualizado',
        description: 'Tus datos se guardaron correctamente.',
      });
    } catch {
      // El error ya está en el store; también lo mostramos inline debajo del form
    }
  };

  // ── Estados de carga ──────────────────────────────────────────────────────

  if (isLoading) {
    return (
      <div className="flex items-center justify-center min-h-[60vh]">
        <Loader2 className="h-8 w-8 animate-spin text-primary" />
      </div>
    );
  }

  // ── Render ────────────────────────────────────────────────────────────────

  return (
    <div className="container max-w-2xl py-8 space-y-6">
      <h1 className="text-3xl font-bold tracking-tight">Mi perfil</h1>

      <Tabs defaultValue="info">
        <TabsList className="grid w-full grid-cols-3">
          <TabsTrigger value="info">Información</TabsTrigger>
          <TabsTrigger value="edit">Editar perfil</TabsTrigger>
          <TabsTrigger value="orders">Órdenes</TabsTrigger>
        </TabsList>

        {/* ── Tab: Información ─────────────────────────────────────────── */}
        <TabsContent value="info" className="mt-6">
          <Card>
            <CardHeader>
              <CardTitle>Información de cuenta</CardTitle>
              <CardDescription>
                Tus datos personales registrados en ShopApp.
              </CardDescription>
            </CardHeader>
            <CardContent className="space-y-6">
              {/* Avatar + nombre */}
              <div className="flex items-center gap-5">
                <UserAvatar user={profile} size="lg" />
                <div className="space-y-1">
                  <div className="flex items-center gap-2">
                    <p className="text-xl font-semibold">
                      {profile?.first_name || profile?.last_name
                        ? `${profile?.first_name} ${profile?.last_name}`.trim()
                        : profile?.username}
                    </p>
                    {profile?.is_staff && (
                      <Badge
                        variant="secondary"
                        className="flex items-center gap-1"
                      >
                        <ShieldCheck className="h-3 w-3" />
                        Staff
                      </Badge>
                    )}
                  </div>
                  <p className="text-sm text-muted-foreground">
                    @{profile?.username}
                  </p>
                </div>
              </div>

              <Separator />

              {/* Campos de detalle */}
              <dl className="grid grid-cols-1 gap-4 sm:grid-cols-2 text-sm">
                <div className="space-y-1">
                  <dt className="text-muted-foreground font-medium">
                    Correo electrónico
                  </dt>
                  <dd>{profile?.email ?? '—'}</dd>
                </div>
                <div className="space-y-1">
                  <dt className="text-muted-foreground font-medium">
                    Miembro desde
                  </dt>
                  <dd>
                    {profile?.date_joined
                      ? formatDate(profile.date_joined)
                      : '—'}
                  </dd>
                </div>
                <div className="space-y-1">
                  <dt className="text-muted-foreground font-medium">
                    Nombre
                  </dt>
                  <dd>{profile?.first_name || '—'}</dd>
                </div>
                <div className="space-y-1">
                  <dt className="text-muted-foreground font-medium">
                    Apellido
                  </dt>
                  <dd>{profile?.last_name || '—'}</dd>
                </div>
              </dl>
            </CardContent>
          </Card>
        </TabsContent>

        {/* ── Tab: Editar perfil ───────────────────────────────────────── */}
        <TabsContent value="edit" className="mt-6">
          <Card>
            <CardHeader>
              <CardTitle>Editar perfil</CardTitle>
              <CardDescription>
                Actualiza tus datos personales. El nombre de usuario no se
                puede cambiar.
              </CardDescription>
            </CardHeader>
            <CardContent>
              <Form {...form}>
                <form
                  onSubmit={form.handleSubmit(onSubmit)}
                  className="space-y-5"
                >
                  {/* Nombre */}
                  <FormField
                    control={form.control}
                    name="first_name"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel>Nombre</FormLabel>
                        <FormControl>
                          <Input placeholder="Tu nombre" {...field} />
                        </FormControl>
                        <FormMessage />
                      </FormItem>
                    )}
                  />

                  {/* Apellido */}
                  <FormField
                    control={form.control}
                    name="last_name"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel>Apellido</FormLabel>
                        <FormControl>
                          <Input placeholder="Tu apellido" {...field} />
                        </FormControl>
                        <FormMessage />
                      </FormItem>
                    )}
                  />

                  {/* Email */}
                  <FormField
                    control={form.control}
                    name="email"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel>Correo electrónico</FormLabel>
                        <FormControl>
                          <Input
                            type="email"
                            placeholder="tu@email.com"
                            {...field}
                          />
                        </FormControl>
                        <FormMessage />
                      </FormItem>
                    )}
                  />

                  {/* Error global del store */}
                  {error && (
                    <p className="text-sm text-destructive font-medium">
                      {error}
                    </p>
                  )}

                  {/* Botón de guardar */}
                  <Button
                    type="submit"
                    disabled={isSaving}
                    className="w-full sm:w-auto"
                  >
                    {isSaving && (
                      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                    )}
                    {isSaving ? 'Guardando…' : 'Guardar cambios'}
                  </Button>
                </form>
              </Form>
            </CardContent>
          </Card>
        </TabsContent>

        {/* ── Tab: Órdenes ─────────────────────────────────────────────── */}
        <TabsContent value="orders" className="mt-6">
          <Card>
            <CardHeader>
              <CardTitle>Historial de órdenes</CardTitle>
              <CardDescription>
                Consulta el estado y detalle de todos tus pedidos.
              </CardDescription>
            </CardHeader>
            <CardContent>
              <p className="text-sm text-muted-foreground mb-4">
                Tu historial completo de compras está disponible en la sección
                de órdenes.
              </p>
              <Button variant="outline" asChild>
                <Link to="/orders" className="flex items-center gap-2">
                  Ver mis órdenes
                  <ExternalLink className="h-4 w-4" />
                </Link>
              </Button>
            </CardContent>
          </Card>
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

---

## 8.6 Actualizar AppShell

**Archivo:** `src/presentation/components/AppShell.tsx` (modificación)

En este paso integramos `useProfileStore` en el encabezado de la aplicación para que el avatar
del usuario refleje la imagen o iniciales reales del perfil. También añadimos un efecto en
`AppShell` que llama a `fetchProfile` una única vez cuando hay una sesión activa, evitando
llamadas duplicadas en cada navegación.

La sección relevante a modificar o reemplazar en tu `AppShell.tsx` existente es:

```tsx
// src/presentation/components/AppShell.tsx
// ── Añadir al inicio del archivo ────────────────────────────────────────────

import { useEffect } from 'react';
import { UserAvatar } from './UserAvatar';
import { useProfileStore } from '../../domain/store/profile.store';
import { useAuthStore } from '../../domain/store/auth.store';

// ── Dentro del componente AppShell ──────────────────────────────────────────
// (Sustituye o combina con el contenido existente)

export function AppShell({ children }: { children: React.ReactNode }) {
  const { user } = useAuthStore();
  const { profile, fetchProfile } = useProfileStore();

  // Carga el perfil una sola vez cuando hay sesión activa
  useEffect(() => {
    if (user && !profile) {
      fetchProfile();
    }
  }, [user, profile, fetchProfile]);

  return (
    <div className="min-h-screen flex flex-col bg-background">
      {/* ── Header ─────────────────────────────────────────────────────── */}
      <header className="border-b sticky top-0 z-40 bg-background/95 backdrop-blur supports-[backdrop-filter]:bg-background/60">
        <div className="container flex h-14 items-center justify-between">
          {/* Logo / marca */}
          <Link to="/" className="flex items-center gap-2 font-bold text-lg">
            <ShoppingBag className="h-5 w-5 text-primary" />
            ShopApp
          </Link>

          {/* Nav principal */}
          <nav className="hidden md:flex items-center gap-6 text-sm">
            <Link
              to="/catalog"
              className="text-muted-foreground hover:text-foreground transition-colors"
            >
              Catálogo
            </Link>
            <Link
              to="/cart"
              className="text-muted-foreground hover:text-foreground transition-colors"
            >
              Carrito
            </Link>
          </nav>

          {/* Acciones de usuario */}
          <div className="flex items-center gap-3">
            {user ? (
              <DropdownMenu>
                <DropdownMenuTrigger asChild>
                  <Button
                    variant="ghost"
                    className="relative h-9 w-9 rounded-full p-0"
                  >
                    {/* Avatar real desde profileStore */}
                    <UserAvatar user={profile} size="sm" />
                  </Button>
                </DropdownMenuTrigger>
                <DropdownMenuContent align="end" className="w-48">
                  <DropdownMenuLabel className="font-normal">
                    <div className="flex flex-col space-y-1">
                      <p className="text-sm font-medium leading-none">
                        {user.username}
                      </p>
                      <p className="text-xs leading-none text-muted-foreground">
                        {user.email}
                      </p>
                    </div>
                  </DropdownMenuLabel>
                  <DropdownMenuSeparator />
                  <DropdownMenuItem asChild>
                    <Link to="/profile">Mi perfil</Link>
                  </DropdownMenuItem>
                  <DropdownMenuItem asChild>
                    <Link to="/orders">Mis órdenes</Link>
                  </DropdownMenuItem>
                  {user.is_staff && (
                    <>
                      <DropdownMenuSeparator />
                      <DropdownMenuItem asChild>
                        <Link to="/admin">Panel de administración</Link>
                      </DropdownMenuItem>
                    </>
                  )}
                  <DropdownMenuSeparator />
                  <DropdownMenuItem
                    onClick={() => {
                      useAuthStore.getState().logout();
                      useProfileStore.getState().clearProfile();
                    }}
                    className="text-destructive focus:text-destructive"
                  >
                    Cerrar sesión
                  </DropdownMenuItem>
                </DropdownMenuContent>
              </DropdownMenu>
            ) : (
              <Button asChild size="sm">
                <Link to="/auth/login">Iniciar sesión</Link>
              </Button>
            )}
          </div>
        </div>
      </header>

      {/* ── Contenido principal ────────────────────────────────────────── */}
      <main className="flex-1">{children}</main>

      {/* ── Footer ─────────────────────────────────────────────────────── */}
      <footer className="border-t py-6 text-center text-xs text-muted-foreground">
        © {new Date().getFullYear()} ShopApp — Todos los derechos reservados.
      </footer>
    </div>
  );
}
```

> **Nota sobre el `clearProfile`:** cuando el usuario cierra sesión, es importante llamar a
> `clearProfile()` del `profileStore` para limpiar los datos del usuario anterior. De lo contrario,
> si otro usuario inicia sesión en la misma sesión del navegador, vería el perfil anterior hasta
> que se recargue la página.

---

## 8.7 Registrar la ruta en el router

**Archivo:** `src/presentation/router/AppRouter.tsx` (modificación)

Agrega la ruta `/profile` dentro del bloque de rutas protegidas. Si ya tienes un componente
`ProtectedRoute` de módulos anteriores, simplemente añade la entrada:

```tsx
// src/presentation/router/AppRouter.tsx
// ── Añadir el import ─────────────────────────────────────────────────────────

import ProfilePage from '../pages/profile/ProfilePage';

// ── Dentro del árbol de rutas, junto a las rutas protegidas existentes ───────

<Route
  path="/profile"
  element={
    <ProtectedRoute>
      <ProfilePage />
    </ProtectedRoute>
  }
/>
```

Asegúrate también de que el `Toaster` de shadcn esté montado en el `App.tsx` o en el `AppShell`
para que los toasts se muestren correctamente:

```tsx
// src/App.tsx (o donde montes tu árbol de rutas)
import { Toaster } from '@/components/ui/toaster';

// Al final del JSX raíz:
<Toaster />
```

---

## 8.8 Checkpoint y pruebas manuales

Antes de continuar al Módulo 9, verifica que todo funciona correctamente:

1. **Carga del perfil**
   - Inicia sesión y navega a `/profile`.
   - Abre las DevTools (pestaña Network) y confirma que se realizó `GET /users/me/`.
   - Los datos (nombre, email, fecha) deben aparecer en la pestaña "Información".

2. **Avatar con iniciales**
   - Si tu usuario no tiene avatar (`avatar: null`), verifica que aparezcan las iniciales.
   - Prueba con un usuario cuyo `first_name` y `last_name` estén vacíos: debe aparecer la
     primera letra del `username`.

3. **Formulario de edición**
   - Ve a la pestaña "Editar perfil" y modifica el nombre.
   - Guarda: debe aparecer el spinner en el botón y luego el toast "Perfil actualizado".
   - Recarga la página y verifica que el cambio persiste (viene del backend).

4. **Validación Zod**
   - Introduce un email inválido (por ejemplo `noesvalido`) y confirma que aparece el mensaje
     de error sin enviar la solicitud al servidor.

5. **Avatar en el header**
   - Confirma que el dropdown del header ahora usa `UserAvatar` en lugar de un icono genérico.

6. **Logout**
   - Cierra sesión y vuelve a iniciar sesión con el mismo usuario.
   - Verifica que el avatar se muestra correctamente al recargar.

---

## Resumen del módulo

| Archivo | Capa | Descripción |
|---|---|---|
| `src/domain/model/user.types.ts` | Domain / Model | Tipos `UserProfile` y `UpdateProfilePayload` |
| `src/data/api/users.api.ts` | Data / API | `getMyProfile()` y `updateMyProfile()` con axios |
| `src/domain/store/profile.store.ts` | Domain / Store | Store Zustand con `fetchProfile` y `updateProfile` |
| `src/presentation/components/UserAvatar.tsx` | Presentation / Components | Avatar con imagen o iniciales, tres tamaños |
| `src/presentation/pages/profile/ProfilePage.tsx` | Presentation / Pages | Página con tabs: info, edición y enlace a órdenes |
| `src/presentation/components/AppShell.tsx` | Presentation / Components | Header actualizado con `UserAvatar` desde el store |
| `src/presentation/router/AppRouter.tsx` | Presentation / Router | Ruta `/profile` protegida añadida |

### Dependencias de shadcn usadas en este módulo

| Componente | Comando de instalación |
|---|---|
| `Avatar` | `npx shadcn@latest add avatar` |
| `Card` | `npx shadcn@latest add card` |
| `Form` | `npx shadcn@latest add form` |
| `Input` | `npx shadcn@latest add input` |
| `Label` | `npx shadcn@latest add label` |
| `Button` | `npx shadcn@latest add button` |
| `Separator` | `npx shadcn@latest add separator` |
| `Tabs` | `npx shadcn@latest add tabs` |
| `Badge` | `npx shadcn@latest add badge` |
| `Toast` | `npx shadcn@latest add toast` |
