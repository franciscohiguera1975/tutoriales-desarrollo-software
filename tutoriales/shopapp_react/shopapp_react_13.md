# ShopApp React Web — Módulo 13

## Admin Usuarios — Listado y gestión de permisos

**Duración estimada:** 2–3 horas
**Prerequisitos:** Módulos 1–12 completados. El backend expone `/api/users/` con soporte para `?search=` y requiere `is_staff=True` para listar todos los usuarios y modificar `is_staff` o `is_active`.

---

> **Objetivo**
> Implementar la gestión de usuarios desde el panel de administración: extender el API layer y el AdminStore con la slice de usuarios, construir la página `AdminUsersPage` con búsqueda debounced, tabla con controles inline (shadcn Switch) para cambiar el rol y el estado activo, y proteger al usuario actual de modificar sus propios permisos.
>
> **Checkpoint final**
> - La ruta `/admin/users` lista todos los usuarios del sistema con su rol y estado.
> - El buscador debounced filtra por nombre de usuario o email sin recargar la página completa.
> - Los switches de Rol y Activo se deshabilitan para el usuario autenticado en sesión.
> - Al intentar cambiar el rol a staff, se muestra un `AlertDialog` de confirmación.
> - `npm run build` compila sin errores de TypeScript.

---

## 13.1 Tipos de dominio para usuarios admin

**Archivo:** `src/domain/model/user.types.ts` (sección a agregar)

El tipo base `User` ya existe en la app (usado en el auth flow). Aquí se extiende con `AdminUser`, que incluye campos adicionales que el endpoint de administración de usuarios devuelve: `first_name`, `last_name`, `is_active` y el contador opcional `order_count` que algunos backends calculan como anotación.

```typescript
// src/domain/model/user.types.ts
// --- Agregar después de los tipos existentes ---

// Tipo base ya existente en el proyecto:
// export interface User {
//   user_id: number;
//   username: string;
//   email: string;
//   is_staff: boolean;
//   date_joined: string;
// }

// AdminUser extiende la información disponible desde el panel de administración.
// order_count es opcional porque no todos los backends lo anotan;
// si no está presente, se muestra "—" en la UI.
export interface AdminUser {
  user_id: number;
  username: string;
  email: string;
  first_name: string;
  last_name: string;
  is_staff: boolean;
  is_active: boolean;
  date_joined: string;
  order_count?: number;
}

// Helper para obtener el nombre completo o caer en el username si no está definido.
export function getDisplayName(user: AdminUser): string {
  const full = [user.first_name, user.last_name].filter(Boolean).join(" ");
  return full || user.username;
}
```

Separar `AdminUser` de `User` es intencional: el payload del token JWT o del endpoint `/auth/me/` generalmente no incluye `is_active` ni los campos de nombre, y combinar ambos en un solo tipo generaría campos opcionales artificiales que complican las guardias de tipo.

---

## 13.2 Funciones API de usuarios para admin

**Archivo:** `src/data/api/admin.api.ts` (sección a agregar)

Las tres funciones siguen el mismo patrón que las de órdenes: parámetros tipados, retorno de promesas con tipos del dominio y uso de `PATCH` para actualizaciones parciales. El backend debe tener un `UserViewSet` con permisos `IsAdminUser` en todos los métodos de escritura.

```typescript
// src/data/api/admin.api.ts
// --- Agregar después de las funciones de órdenes del Módulo 12 ---

import type { AdminUser } from "@/domain/model/user.types";
import type { PaginatedResponse } from "@/domain/model/catalog.types";
// apiClient ya está importado en el archivo

// Lista todos los usuarios del sistema paginados.
// El parámetro `search` filtra por username, email, first_name o last_name
// según la configuración del SearchFilter en el backend Django.
export async function getAdminUsers(
  page = 1,
  search?: string
): Promise<PaginatedResponse<AdminUser>> {
  const params: Record<string, string | number> = { page };
  if (search) params.search = search;

  const { data } = await apiClient.get<PaginatedResponse<AdminUser>>(
    "/users/",
    { params }
  );
  return data;
}

// Actualiza únicamente el campo `is_staff` del usuario indicado.
// Solo puede ser llamado por otro staff; el backend debe rechazar intentos
// de que un usuario se quite a sí mismo los permisos de staff.
// La protección en el frontend (deshabilitar el switch) es adicional,
// no reemplaza la validación del backend.
export async function updateUserStaffStatus(
  userId: number,
  is_staff: boolean
): Promise<AdminUser> {
  const { data } = await apiClient.patch<AdminUser>(`/users/${userId}/`, {
    is_staff,
  });
  return data;
}

// Actualiza únicamente el campo `is_active` del usuario indicado.
// Desactivar un usuario evita que pueda autenticarse sin eliminar su cuenta.
export async function updateUserActiveStatus(
  userId: number,
  is_active: boolean
): Promise<AdminUser> {
  const { data } = await apiClient.patch<AdminUser>(`/users/${userId}/`, {
    is_active,
  });
  return data;
}
```

Nótese que `updateUserStaffStatus` y `updateUserActiveStatus` son funciones separadas aunque ambas usen `PATCH /users/{id}/`. Esta separación es intencional: permite que los tipos sean precisos (cada función sabe exactamente qué campo modifica) y facilita el manejo de errores diferenciado en el store.

---

## 13.3 AdminStore — estado de usuarios

**Archivo:** `src/domain/store/admin.store.ts` (sección a agregar)

La slice de usuarios sigue el mismo patrón que la de órdenes: estado local, carga paginada y actualización optimista tras éxito del API. La diferencia clave es que `toggleUserStaff` y `toggleUserActive` actualizan el objeto en el array local en lugar de volver a fetchar, para evitar que el usuario pierda el contexto de scroll o búsqueda.

```typescript
// src/domain/store/admin.store.ts
// --- Agregar la siguiente slice al store existente ---

import { create } from "zustand";
import type { AdminUser } from "@/domain/model/user.types";
import {
  getAdminUsers,
  updateUserStaffStatus,
  updateUserActiveStatus,
} from "@/data/api/admin.api";

interface AdminUsersSlice {
  adminUsers: AdminUser[];
  usersLoading: boolean;
  usersTotal: number;

  fetchAdminUsers: (page?: number, search?: string) => Promise<void>;
  toggleUserStaff: (userId: number, isStaff: boolean) => Promise<void>;
  toggleUserActive: (userId: number, isActive: boolean) => Promise<void>;
}

export const useAdminUsersStore = create<AdminUsersSlice>((set) => ({
  adminUsers: [],
  usersLoading: false,
  usersTotal: 0,

  // Carga la lista de usuarios con los parámetros recibidos.
  // La página y la búsqueda se pasan directamente desde la UI;
  // el store no almacena el filtro de búsqueda porque el componente
  // lo gestiona localmente con debounce antes de llamar a esta acción.
  fetchAdminUsers: async (page = 1, search?: string) => {
    set({ usersLoading: true });
    try {
      const data = await getAdminUsers(page, search);
      set({ adminUsers: data.results, usersTotal: data.count });
    } finally {
      set({ usersLoading: false });
    }
  },

  // Llama al backend para cambiar is_staff y actualiza el objeto local
  // en el array sin re-fetchear toda la lista.
  // Si el backend rechaza (ej. el usuario intenta quitarse a sí mismo el staff),
  // el error se propaga al componente que muestra el AlertDialog.
  toggleUserStaff: async (userId: number, isStaff: boolean) => {
    const updated = await updateUserStaffStatus(userId, isStaff);
    set((state) => ({
      adminUsers: state.adminUsers.map((u) =>
        u.user_id === userId ? { ...u, is_staff: updated.is_staff } : u
      ),
    }));
  },

  // Llama al backend para cambiar is_active y actualiza el objeto local.
  toggleUserActive: async (userId: number, isActive: boolean) => {
    const updated = await updateUserActiveStatus(userId, isActive);
    set((state) => ({
      adminUsers: state.adminUsers.map((u) =>
        u.user_id === userId ? { ...u, is_active: updated.is_active } : u
      ),
    }));
  },
}));
```

El patrón de actualización local (`map` sobre el array con spread del objeto actualizado) es equivalente a una actualización optimista diferida: se espera la respuesta del servidor antes de actualizar la UI, garantizando consistencia con el backend sin mostrar un estado incorrecto.

---

## 13.4 Página AdminUsersPage

**Archivo:** `src/presentation/pages/admin/AdminUsersPage.tsx`

Esta página implementa todos los requisitos: buscador con debounce, tabla con switches inline, confirmación antes de cambiar el rol de staff, paginación y protección del usuario en sesión. El buscador usa `useRef` + `useEffect` para implementar el debounce sin dependencias externas.

```tsx
// src/presentation/pages/admin/AdminUsersPage.tsx

import { useEffect, useRef, useState, useCallback } from "react";
import { Search, ShieldCheck, User as UserIcon } from "lucide-react";
import { AdminShell } from "@/presentation/components/AdminShell";
import { useAdminUsersStore } from "@/domain/store/admin.store";
import { useAuthStore } from "@/domain/store/auth.store";
import { getDisplayName } from "@/domain/model/user.types";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Skeleton } from "@/components/ui/skeleton";
import { Switch } from "@/components/ui/switch";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from "@/components/ui/tooltip";
import type { AdminUser } from "@/domain/model/user.types";

const PAGE_SIZE = 15;
const DEBOUNCE_MS = 400;

// Pending confirmation almacena los datos del cambio de staff pendiente
// de confirmar en el AlertDialog.
interface PendingStaffChange {
  userId: number;
  username: string;
  newValue: boolean;
}

export function AdminUsersPage() {
  const { adminUsers, usersLoading, usersTotal, fetchAdminUsers, toggleUserStaff, toggleUserActive } =
    useAdminUsersStore();

  // El usuario en sesión se obtiene del auth store para la protección de autoedición.
  const currentUserId = useAuthStore((s) => s.user?.user_id);

  const [search, setSearch] = useState("");
  const [page, setPage] = useState(1);
  const [pendingStaff, setPendingStaff] = useState<PendingStaffChange | null>(null);
  const [togglingActive, setTogglingActive] = useState<number | null>(null);

  // Ref para el timeout del debounce; se limpia en cada keystroke.
  const debounceRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  const totalPages = Math.ceil(usersTotal / PAGE_SIZE);

  // fetchUsers es la función que realmente dispara la carga.
  // Se usa useCallback para estabilizarla y poder incluirla como dependencia.
  const fetchUsers = useCallback(
    (p: number, s: string) => {
      fetchAdminUsers(p, s || undefined);
    },
    [fetchAdminUsers]
  );

  // Al montar el componente, cargar la primera página sin filtro.
  useEffect(() => {
    fetchUsers(1, "");
  }, [fetchUsers]);

  // Handler del buscador: aplica debounce y resetea la página a 1.
  const handleSearchChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setSearch(value);
    setPage(1);

    if (debounceRef.current) clearTimeout(debounceRef.current);
    debounceRef.current = setTimeout(() => {
      fetchUsers(1, value);
    }, DEBOUNCE_MS);
  };

  const handlePageChange = (newPage: number) => {
    setPage(newPage);
    fetchUsers(newPage, search);
  };

  // El switch de staff abre el AlertDialog en lugar de actuar directamente.
  // Solo se guarda la intención; la acción ocurre en handleConfirmStaff.
  const handleStaffToggleIntent = (user: AdminUser, newValue: boolean) => {
    setPendingStaff({
      userId: user.user_id,
      username: user.username,
      newValue,
    });
  };

  const handleConfirmStaff = async () => {
    if (!pendingStaff) return;
    try {
      await toggleUserStaff(pendingStaff.userId, pendingStaff.newValue);
    } catch {
      // El error se puede manejar con un toast; aquí simplemente se cierra.
    } finally {
      setPendingStaff(null);
    }
  };

  // El switch de activo no requiere confirmación: el cambio es reversible
  // y tiene menos impacto de seguridad que cambiar el rol de staff.
  const handleActiveToggle = async (user: AdminUser, newValue: boolean) => {
    setTogglingActive(user.user_id);
    try {
      await toggleUserActive(user.user_id, newValue);
    } finally {
      setTogglingActive(null);
    }
  };

  // Un usuario es el "usuario actual" si su user_id coincide con el del token.
  // En ese caso, los controles de staff y activo se deshabilitan.
  const isSelf = (userId: number) => userId === currentUserId;

  const skeletonRows = Array.from({ length: 8 });

  return (
    <AdminShell>
      <TooltipProvider>
        <div className="space-y-6">
          {/* Cabecera */}
          <div className="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
            <div>
              <h1 className="text-2xl font-semibold tracking-tight">Usuarios</h1>
              <p className="text-sm text-muted-foreground">
                {usersTotal} {usersTotal === 1 ? "usuario registrado" : "usuarios registrados"}
              </p>
            </div>
            {/* Buscador debounced */}
            <div className="relative w-full sm:w-72">
              <Search className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
              <Input
                placeholder="Buscar por usuario o email…"
                value={search}
                onChange={handleSearchChange}
                className="pl-9"
              />
            </div>
          </div>

          {/* Tabla */}
          <div className="rounded-md border">
            <Table>
              <TableHeader>
                <TableRow>
                  <TableHead>Usuario</TableHead>
                  <TableHead>Email</TableHead>
                  <TableHead>Nombre completo</TableHead>
                  <TableHead className="text-center w-28">Rol</TableHead>
                  <TableHead className="text-center w-28">Estado</TableHead>
                  <TableHead className="w-36">Miembro desde</TableHead>
                </TableRow>
              </TableHeader>
              <TableBody>
                {usersLoading
                  ? skeletonRows.map((_, i) => (
                      <TableRow key={i}>
                        {Array.from({ length: 6 }).map((__, j) => (
                          <TableCell key={j}>
                            <Skeleton className="h-5 w-full" />
                          </TableCell>
                        ))}
                      </TableRow>
                    ))
                  : adminUsers.length === 0
                  ? (
                      <TableRow>
                        <TableCell
                          colSpan={6}
                          className="h-32 text-center text-muted-foreground"
                        >
                          No se encontraron usuarios con ese criterio de búsqueda.
                        </TableCell>
                      </TableRow>
                    )
                  : adminUsers.map((user) => {
                      const self = isSelf(user.user_id);
                      const activeToggling = togglingActive === user.user_id;

                      return (
                        <TableRow key={user.user_id}>
                          {/* Usuario: ícono + username */}
                          <TableCell>
                            <div className="flex items-center gap-2">
                              <div className="flex h-8 w-8 items-center justify-center rounded-full bg-muted text-muted-foreground">
                                <UserIcon className="h-4 w-4" />
                              </div>
                              <span className="font-medium text-sm">
                                {user.username}
                              </span>
                              {self && (
                                <Badge
                                  variant="outline"
                                  className="text-xs h-5 px-1"
                                >
                                  Tú
                                </Badge>
                              )}
                            </div>
                          </TableCell>

                          <TableCell className="text-sm text-muted-foreground">
                            {user.email}
                          </TableCell>

                          <TableCell className="text-sm">
                            {getDisplayName(user)}
                          </TableCell>

                          {/* Columna Rol: badge + switch de staff */}
                          <TableCell>
                            <div className="flex flex-col items-center gap-1.5">
                              {user.is_staff ? (
                                <Badge className="gap-1 bg-purple-100 text-purple-700 hover:bg-purple-100 text-xs">
                                  <ShieldCheck className="h-3 w-3" />
                                  Staff
                                </Badge>
                              ) : (
                                <Badge variant="secondary" className="text-xs">
                                  Cliente
                                </Badge>
                              )}
                              {/* Switch de staff con confirmación */}
                              <Tooltip>
                                <TooltipTrigger asChild>
                                  <span>
                                    <Switch
                                      checked={user.is_staff}
                                      disabled={self}
                                      onCheckedChange={(val) =>
                                        handleStaffToggleIntent(user, val)
                                      }
                                      aria-label={`Cambiar rol de ${user.username}`}
                                      className="data-[state=checked]:bg-purple-600"
                                    />
                                  </span>
                                </TooltipTrigger>
                                {self && (
                                  <TooltipContent>
                                    No puedes cambiar tu propio rol
                                  </TooltipContent>
                                )}
                              </Tooltip>
                            </div>
                          </TableCell>

                          {/* Columna Estado: badge + switch de activo */}
                          <TableCell>
                            <div className="flex flex-col items-center gap-1.5">
                              {user.is_active ? (
                                <Badge
                                  variant="outline"
                                  className="text-green-700 border-green-300 text-xs"
                                >
                                  Activo
                                </Badge>
                              ) : (
                                <Badge
                                  variant="outline"
                                  className="text-red-600 border-red-300 text-xs"
                                >
                                  Inactivo
                                </Badge>
                              )}
                              <Tooltip>
                                <TooltipTrigger asChild>
                                  <span>
                                    <Switch
                                      checked={user.is_active}
                                      disabled={self || activeToggling}
                                      onCheckedChange={(val) =>
                                        handleActiveToggle(user, val)
                                      }
                                      aria-label={`${user.is_active ? "Desactivar" : "Activar"} cuenta de ${user.username}`}
                                      className="data-[state=checked]:bg-green-600"
                                    />
                                  </span>
                                </TooltipTrigger>
                                {self && (
                                  <TooltipContent>
                                    No puedes desactivar tu propia cuenta
                                  </TooltipContent>
                                )}
                              </Tooltip>
                            </div>
                          </TableCell>

                          <TableCell className="text-sm text-muted-foreground">
                            {new Date(user.date_joined).toLocaleDateString(
                              "es-EC",
                              { year: "numeric", month: "short", day: "numeric" }
                            )}
                          </TableCell>
                        </TableRow>
                      );
                    })}
              </TableBody>
            </Table>
          </div>

          {/* Paginación */}
          {totalPages > 1 && (
            <div className="flex items-center justify-between text-sm text-muted-foreground">
              <span>
                Página {page} de {totalPages}
              </span>
              <div className="flex gap-2">
                <Button
                  variant="outline"
                  size="sm"
                  disabled={page <= 1}
                  onClick={() => handlePageChange(page - 1)}
                >
                  Anterior
                </Button>
                <Button
                  variant="outline"
                  size="sm"
                  disabled={page >= totalPages}
                  onClick={() => handlePageChange(page + 1)}
                >
                  Siguiente
                </Button>
              </div>
            </div>
          )}
        </div>

        {/* AlertDialog de confirmación para cambio de rol de staff */}
        <AlertDialog
          open={pendingStaff !== null}
          onOpenChange={(open) => !open && setPendingStaff(null)}
        >
          <AlertDialogContent>
            <AlertDialogHeader>
              <AlertDialogTitle>Confirmar cambio de rol</AlertDialogTitle>
              <AlertDialogDescription>
                {pendingStaff?.newValue
                  ? `¿Deseas otorgar permisos de administrador (staff) a "${pendingStaff?.username}"? Este usuario podrá acceder al panel de administración y gestionar órdenes, productos y otros usuarios.`
                  : `¿Deseas remover los permisos de administrador (staff) de "${pendingStaff?.username}"? Este usuario ya no podrá acceder al panel de administración.`}
              </AlertDialogDescription>
            </AlertDialogHeader>
            <AlertDialogFooter>
              <AlertDialogCancel>Cancelar</AlertDialogCancel>
              <AlertDialogAction
                onClick={handleConfirmStaff}
                className={
                  pendingStaff?.newValue
                    ? "bg-purple-600 hover:bg-purple-700"
                    : "bg-destructive hover:bg-destructive/90"
                }
              >
                {pendingStaff?.newValue
                  ? "Otorgar permisos"
                  : "Remover permisos"}
              </AlertDialogAction>
            </AlertDialogFooter>
          </AlertDialogContent>
        </AlertDialog>
      </TooltipProvider>
    </AdminShell>
  );
}
```

**Registro de la ruta en el router:**

```tsx
// src/presentation/router/AppRouter.tsx
// --- Agregar junto a las rutas de admin existentes ---

import { AdminUsersPage } from "@/presentation/pages/admin/AdminUsersPage";

// Dentro del bloque de rutas protegidas de staff:
<Route path="/admin/users" element={<AdminUsersPage />} />
```

---

## 13.5 Lógica de auto-protección

La protección contra que el usuario actual modifique sus propios permisos se implementa en tres capas complementarias:

**Capa 1 — UI (frontend):** El switch se deshabilita cuando `user.user_id === currentUserId`. Esto es la primera barrera visible: el administrador no puede ni intentar el cambio.

```tsx
// Comparación directa: el ID del usuario de la fila contra el ID del token en sesión.
// currentUserId proviene de useAuthStore((s) => s.user?.user_id)
const isSelf = (userId: number) => userId === currentUserId;

<Switch
  checked={user.is_staff}
  disabled={self}           // ← deshabilita el control
  onCheckedChange={...}
/>
```

**Capa 2 — Tooltip informativo:** Cuando el switch está deshabilitado por ser el propio usuario, un `Tooltip` explica el motivo. Esto cumple con el principio de usabilidad de no dejar controles deshabilitados sin explicación.

```tsx
<Tooltip>
  <TooltipTrigger asChild>
    <span>  {/* span necesario porque Switch disabled no propaga eventos de Tooltip */}
      <Switch disabled={self} ... />
    </span>
  </TooltipTrigger>
  {self && (
    <TooltipContent>No puedes cambiar tu propio rol</TooltipContent>
  )}
</Tooltip>
```

El `<span>` envolviendo al `Switch` disabled es necesario porque los elementos deshabilitados no propagan eventos del mouse en algunos navegadores, lo que impediría que el Tooltip se muestre.

**Capa 3 — Backend (Django):** La validación definitiva ocurre en el servidor. El ViewSet debe verificar que `request.user.id != kwargs['pk']` antes de permitir cambios en `is_staff`. El frontend solo es una capa de conveniencia; nunca reemplaza la validación del backend.

```python
# Ejemplo de validación en el backend Django (referencia)
# En el serializer o el ViewSet:
def validate(self, attrs):
    if self.context['request'].user.id == self.instance.id:
        if 'is_staff' in attrs and attrs['is_staff'] != self.instance.is_staff:
            raise serializers.ValidationError(
                "No puedes modificar tus propios permisos de staff."
            )
    return attrs
```

La combinación de las tres capas garantiza que la restricción sea robusta: la UI es conveniente para el usuario, el tooltip es informativo, y el backend es la autoridad final.

---

## 13.6 Checkpoint y tabla resumen

Antes de dar por completado el módulo, verifica cada punto en el navegador:

1. Navega a `/admin/users` — la tabla debe cargar con todos los usuarios.
2. Escribe en el buscador — la tabla debe actualizarse aproximadamente 400ms después de dejar de escribir.
3. Localiza tu propio usuario en la tabla — los switches de Rol y Estado deben aparecer deshabilitados y con el badge "Tú".
4. Pasa el mouse sobre un switch deshabilitado — el tooltip "No puedes cambiar tu propio rol" debe aparecer.
5. Cambia el switch de staff de otro usuario — el `AlertDialog` de confirmación debe aparecer con el mensaje correcto según si se está otorgando o removiendo el rol.
6. Cancela el dialog — el switch debe volver al valor original sin llamar al backend.
7. Confirma el cambio — el badge de rol debe actualizarse en la tabla sin recargar la página.
8. Ejecuta `npm run build` y verifica que no haya errores de TypeScript.

### Archivos creados o modificados en este módulo

| Archivo | Tipo | Descripción |
|---|---|---|
| `src/domain/model/user.types.ts` | Modificado | Tipo `AdminUser` con campos completos y helper `getDisplayName` |
| `src/data/api/admin.api.ts` | Modificado | Funciones `getAdminUsers`, `updateUserStaffStatus`, `updateUserActiveStatus` |
| `src/domain/store/admin.store.ts` | Modificado | Slice de usuarios: `fetchAdminUsers`, `toggleUserStaff`, `toggleUserActive` |
| `src/presentation/pages/admin/AdminUsersPage.tsx` | Nuevo | Tabla de usuarios con búsqueda debounced, switches inline, AlertDialog y auto-protección |
| `src/presentation/router/AppRouter.tsx` | Modificado | Ruta `/admin/users` registrada en el router |
