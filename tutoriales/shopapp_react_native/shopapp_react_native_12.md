# ShopApp React Native — Módulo 12

## Imágenes — Fotos de producto y avatar de usuario

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulos 1–11 completados. Backend `shopapi_07` desplegado (añade campo `image` a `Product` y `avatar` a `UserProfile`).

---

> **Objetivo**
> Integrar la visualización y carga de imágenes en la app: fotos de producto usando FastImage, avatar de usuario con fallback de iniciales, y la lógica de subida con `react-native-image-picker` y `multipart/form-data`.

---

> **Checkpoint final**
> Al terminar este módulo deberás poder:
> - Ver imágenes de productos en las tarjetas y pantalla de detalle
> - Como usuario staff, seleccionar y subir una foto de producto
> - Cambiar el avatar de perfil desde la pantalla de perfil
> - Ver un placeholder con icono mientras la imagen carga o si no hay imagen

---

## 12.1 Dependencias

Antes de continuar, verifica que las librerías estén instaladas. Estas fueron incluidas en el módulo 1, pero aquí se confirma su configuración completa.

### Verificar instalación

```bash
# En la raíz del proyecto
npx react-native info | grep -i image
```

Las librerías requeridas son:
- `react-native-image-picker` — para abrir galería/cámara
- `react-native-fast-image` — para mostrar imágenes con caché eficiente

Si alguna falta:

```bash
npm install react-native-image-picker react-native-fast-image
cd ios && pod install && cd ..
```

### Permisos iOS

Edita `ios/<NombreApp>/Info.plist` y agrega:

```xml
<key>NSPhotoLibraryUsageDescription</key>
<string>ShopApp necesita acceso a tu galería para seleccionar imágenes de productos y avatar.</string>
<key>NSCameraUsageDescription</key>
<string>ShopApp necesita acceso a la cámara para tomar fotos de productos.</string>
<key>NSPhotoLibraryAddUsageDescription</key>
<string>ShopApp necesita guardar imágenes en tu galería.</string>
```

### Permisos Android

Edita `android/app/src/main/AndroidManifest.xml`:

```xml
<!-- Dentro de <manifest> pero fuera de <application> -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.CAMERA" />
```

En Android 13+ (API 33) el permiso `READ_EXTERNAL_STORAGE` fue reemplazado por `READ_MEDIA_IMAGES`. El atributo `maxSdkVersion="32"` garantiza que en dispositivos modernos se use el permiso correcto.

---

## 12.2 Servicio de imágenes

**Archivo:** `src/data/api/image.service.ts`

Este servicio centraliza las llamadas de subida de imágenes. Construye el `FormData` con el archivo seleccionado por `react-native-image-picker` y lo envía con el header `multipart/form-data`. El token de autorización se obtiene de `EncryptedStorage`.

```typescript
import { ImagePickerResponse, Asset } from 'react-native-image-picker';
import EncryptedStorage from 'react-native-encrypted-storage';
import { API_BASE_URL } from '../config/api.config';

// El tipo Asset viene de react-native-image-picker
export interface UploadResult {
  image_url: string;  // URL pública retornada por el backend
}

/**
 * Construye el FormData con la imagen seleccionada por el usuario.
 * El campo 'image' debe coincidir con el nombre del campo en el
 * serializer de Django (Product.image o UserProfile.avatar).
 */
const buildImageFormData = (
  asset: Asset,
  fieldName: 'image' | 'avatar',
): FormData => {
  const formData = new FormData();

  formData.append(fieldName, {
    uri: asset.uri!,
    type: asset.type || 'image/jpeg',
    name: asset.fileName || `upload_${Date.now()}.jpg`,
  } as any); // 'as any' necesario porque RN extiende el tipo File nativo

  return formData;
};

/**
 * Obtiene el token JWT almacenado en EncryptedStorage.
 * Lanza un error si no hay sesión activa.
 */
const getAuthToken = async (): Promise<string> => {
  const token = await EncryptedStorage.getItem('access_token');
  if (!token) throw new Error('No hay sesión activa');
  return token;
};

/**
 * Sube la imagen de un producto.
 * Endpoint: PATCH /api/products/{productId}/
 * Solo usuarios staff pueden ejecutar esta acción.
 */
export const uploadProductImage = async (
  productId: number,
  asset: Asset,
): Promise<UploadResult> => {
  const token = await getAuthToken();
  const formData = buildImageFormData(asset, 'image');

  const response = await fetch(`${API_BASE_URL}/api/products/${productId}/`, {
    method: 'PATCH',
    headers: {
      Authorization: `Bearer ${token}`,
      // NO incluir Content-Type manualmente; fetch lo genera automáticamente
      // con el boundary correcto para multipart/form-data
    },
    body: formData,
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({}));
    throw new Error(error?.detail || `Error ${response.status} al subir imagen`);
  }

  const data = await response.json();
  // El serializer de Django retorna la URL del archivo subido
  return { image_url: data.image_url || data.image || '' };
};

/**
 * Sube el avatar del usuario autenticado.
 * Endpoint: PATCH /api/profile/avatar/
 */
export const uploadAvatar = async (asset: Asset): Promise<UploadResult> => {
  const token = await getAuthToken();
  const formData = buildImageFormData(asset, 'avatar');

  const response = await fetch(`${API_BASE_URL}/api/profile/avatar/`, {
    method: 'PATCH',
    headers: {
      Authorization: `Bearer ${token}`,
    },
    body: formData,
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({}));
    throw new Error(error?.detail || `Error ${response.status} al subir avatar`);
  }

  const data = await response.json();
  return { image_url: data.avatar_url || data.avatar || '' };
};
```

**Por qué `fetch` en lugar de `axios`:** Axios tiene problemas conocidos con `FormData` en React Native cuando se trata de archivos binarios; el boundary del multipart puede corromperse. Usar `fetch` nativo garantiza compatibilidad.

---

## 12.3 Actualizar CatalogStore

El tipo `Product` debe incluir `image_url`. Actualiza `src/domain/model/catalog.types.ts`:

```typescript
export interface Product {
  id: number;
  name: string;
  slug: string;
  description: string;
  price: string;
  stock: number;
  category: number;
  category_name: string;
  sku: string;
  is_active: boolean;
  image_url: string | null;  // NUEVO: puede ser null si el producto no tiene imagen
  created_at: string;
}
```

Agrega la acción `updateProductImage` al CatalogStore en `src/domain/store/catalog.store.ts`:

```typescript
// Dentro de las acciones del store
updateProductImage: (productId: number, imageUrl: string) => {
  set(state => ({
    products: state.products.map(p =>
      p.id === productId ? { ...p, image_url: imageUrl } : p,
    ),
    selectedProduct:
      state.selectedProduct?.id === productId
        ? { ...state.selectedProduct, image_url: imageUrl }
        : state.selectedProduct,
  }));
},
```

---

## 12.4 ProductImage

**Archivo:** `src/presentation/screens/catalog/components/ProductImage.tsx`

Componente reutilizable que muestra la imagen de un producto con FastImage. Si `image_url` es null o la carga falla, muestra un placeholder gris con un ícono de cámara.

```typescript
import React, { useState } from 'react';
import {
  View,
  StyleSheet,
  Text,
  ActivityIndicator,
  ViewStyle,
} from 'react-native';
import FastImage, { ImageStyle } from 'react-native-fast-image';
import { Colors } from '../../../../theme/colors';

interface Props {
  imageUrl: string | null;
  width?: number | string;
  height?: number | string;
  borderRadius?: number;
  style?: ViewStyle;
}

export const ProductImage: React.FC<Props> = ({
  imageUrl,
  width = '100%',
  height = 200,
  borderRadius = 12,
  style,
}) => {
  const [isLoading, setIsLoading] = useState(true);
  const [hasError, setHasError] = useState(false);

  const showPlaceholder = !imageUrl || hasError;

  if (showPlaceholder) {
    return (
      <View
        style={[
          styles.placeholder,
          { width: width as number, height: height as number, borderRadius },
          style,
        ]}>
        <Text style={styles.placeholderIcon}>📷</Text>
        <Text style={styles.placeholderText}>Sin imagen</Text>
      </View>
    );
  }

  return (
    <View
      style={[
        styles.container,
        { width: width as number, height: height as number, borderRadius },
        style,
      ]}>
      <FastImage
        style={[
          styles.image,
          { borderRadius } as ImageStyle,
        ]}
        source={{
          uri: imageUrl,
          priority: FastImage.priority.normal,
          cache: FastImage.cacheControl.immutable,
        }}
        resizeMode={FastImage.resizeMode.cover}
        onLoadStart={() => setIsLoading(true)}
        onLoad={() => setIsLoading(false)}
        onError={() => {
          setIsLoading(false);
          setHasError(true);
        }}
      />
      {isLoading && (
        <View style={[styles.loadingOverlay, { borderRadius }]}>
          <ActivityIndicator size="small" color={Colors.accent} />
        </View>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    overflow: 'hidden',
    backgroundColor: Colors.surface2,
  },
  image: {
    width: '100%',
    height: '100%',
  },
  loadingOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: Colors.surface2,
    alignItems: 'center',
    justifyContent: 'center',
  },
  placeholder: {
    backgroundColor: Colors.surface2,
    alignItems: 'center',
    justifyContent: 'center',
    gap: 8,
    borderWidth: 1,
    borderColor: Colors.border,
    borderStyle: 'dashed',
  },
  placeholderIcon: {
    fontSize: 32,
    opacity: 0.4,
  },
  placeholderText: {
    color: Colors.textSecondary,
    fontSize: 12,
  },
});
```

Integra `ProductImage` en `ProductCard`:

```typescript
// En src/presentation/screens/catalog/components/ProductCard.tsx
import { ProductImage } from './ProductImage';

// Dentro del JSX de la tarjeta, antes del nombre del producto:
<ProductImage
  imageUrl={product.image_url}
  height={140}
  borderRadius={10}
/>
```

---

## 12.5 Upload de imagen de producto (staff)

Agrega el botón de subida de imagen en `ProductDetailScreen`. El botón solo es visible para usuarios staff.

```typescript
// src/presentation/screens/catalog/ProductDetailScreen.tsx
// Agregar al archivo existente:

import { launchImageLibrary } from 'react-native-image-picker';
import { uploadProductImage } from '../../../data/api/image.service';
import { useAuthStore } from '../../../domain/store/auth.store';
import { useCatalogStore } from '../../../domain/store/catalog.store';

// Dentro del componente ProductDetailScreen:
const { user } = useAuthStore();
const { selectedProduct, updateProductImage } = useCatalogStore();

const [isUploadingImage, setIsUploadingImage] = useState(false);

const handleUploadImage = async () => {
  // Abrir el selector de imágenes de la galería
  const result = await launchImageLibrary({
    mediaType: 'photo',
    quality: 0.85,
    maxWidth: 1200,
    maxHeight: 1200,
  });

  if (result.didCancel || !result.assets || result.assets.length === 0) {
    return; // El usuario canceló la selección
  }

  const asset = result.assets[0];
  setIsUploadingImage(true);

  try {
    const uploadResult = await uploadProductImage(selectedProduct!.id, asset);
    updateProductImage(selectedProduct!.id, uploadResult.image_url);
    Alert.alert('Imagen actualizada', 'La foto del producto fue subida exitosamente.');
  } catch (error: any) {
    Alert.alert('Error', error.message || 'No se pudo subir la imagen');
  } finally {
    setIsUploadingImage(false);
  }
};

// En el JSX, después de ProductImage y solo si el usuario es staff:
{user?.is_staff && (
  <TouchableOpacity
    style={styles.uploadImageButton}
    onPress={handleUploadImage}
    disabled={isUploadingImage}>
    {isUploadingImage ? (
      <ActivityIndicator size="small" color={Colors.accent} />
    ) : (
      <Text style={styles.uploadImageText}>Subir imagen</Text>
    )}
  </TouchableOpacity>
)}

// Estilos adicionales:
// uploadImageButton: { borderWidth: 1, borderColor: Colors.accent, borderRadius: 8,
//   paddingVertical: 8, paddingHorizontal: 16, alignSelf: 'center', marginTop: 8 }
// uploadImageText: { color: Colors.accent, fontSize: 14, fontWeight: '600' }
```

---

## 12.6 AvatarImage

**Archivo:** `src/presentation/screens/profile/components/AvatarImage.tsx`

Avatar circular que muestra la foto de perfil del usuario. Si no hay avatar, muestra las iniciales del usuario. Al presionar, abre el selector de imagen y sube el nuevo avatar.

```typescript
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
  Alert,
} from 'react-native';
import FastImage from 'react-native-fast-image';
import { launchImageLibrary } from 'react-native-image-picker';
import { uploadAvatar } from '../../../../data/api/image.service';
import { Colors } from '../../../../theme/colors';

interface Props {
  avatarUrl: string | null;
  firstName: string;
  lastName: string;
  size?: number;
  editable?: boolean;
  onAvatarUpdated?: (newUrl: string) => void;
}

export const AvatarImage: React.FC<Props> = ({
  avatarUrl,
  firstName,
  lastName,
  size = 80,
  editable = false,
  onAvatarUpdated,
}) => {
  const [currentUrl, setCurrentUrl] = useState<string | null>(avatarUrl);
  const [isUploading, setIsUploading] = useState(false);
  const [imageError, setImageError] = useState(false);

  const initials =
    `${firstName.charAt(0)}${lastName.charAt(0)}`.toUpperCase() || '?';

  const showInitials = !currentUrl || imageError;

  const handlePress = async () => {
    if (!editable) return;

    const result = await launchImageLibrary({
      mediaType: 'photo',
      quality: 0.9,
      maxWidth: 400,
      maxHeight: 400,
    });

    if (result.didCancel || !result.assets?.[0]) return;

    const asset = result.assets[0];
    setIsUploading(true);

    try {
      const uploadResult = await uploadAvatar(asset);
      setCurrentUrl(uploadResult.image_url);
      setImageError(false);
      onAvatarUpdated?.(uploadResult.image_url);
    } catch (error: any) {
      Alert.alert('Error', error.message || 'No se pudo subir el avatar');
    } finally {
      setIsUploading(false);
    }
  };

  return (
    <TouchableOpacity
      onPress={handlePress}
      disabled={!editable || isUploading}
      activeOpacity={editable ? 0.7 : 1}>
      <View
        style={[
          styles.container,
          { width: size, height: size, borderRadius: size / 2 },
        ]}>
        {showInitials ? (
          <View
            style={[
              styles.initialsContainer,
              { width: size, height: size, borderRadius: size / 2 },
            ]}>
            <Text style={[styles.initials, { fontSize: size * 0.36 }]}>
              {initials}
            </Text>
          </View>
        ) : (
          <FastImage
            style={{ width: size, height: size, borderRadius: size / 2 }}
            source={{
              uri: currentUrl!,
              cache: FastImage.cacheControl.web,
            }}
            resizeMode={FastImage.resizeMode.cover}
            onError={() => setImageError(true)}
          />
        )}

        {/* Overlay de carga */}
        {isUploading && (
          <View
            style={[
              styles.uploadOverlay,
              { borderRadius: size / 2 },
            ]}>
            <ActivityIndicator size="small" color={Colors.textPrimary} />
          </View>
        )}

        {/* Ícono de edición */}
        {editable && !isUploading && (
          <View style={styles.editBadge}>
            <Text style={styles.editBadgeText}>✎</Text>
          </View>
        )}
      </View>
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  container: {
    position: 'relative',
  },
  initialsContainer: {
    backgroundColor: Colors.accent + '33',
    borderWidth: 2,
    borderColor: Colors.accent + '55',
    alignItems: 'center',
    justifyContent: 'center',
  },
  initials: {
    color: Colors.accent,
    fontWeight: '700',
  },
  uploadOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(0,0,0,0.5)',
    alignItems: 'center',
    justifyContent: 'center',
  },
  editBadge: {
    position: 'absolute',
    bottom: 0,
    right: 0,
    width: 24,
    height: 24,
    borderRadius: 12,
    backgroundColor: Colors.accent,
    alignItems: 'center',
    justifyContent: 'center',
    borderWidth: 2,
    borderColor: Colors.background,
  },
  editBadgeText: {
    color: Colors.textPrimary,
    fontSize: 12,
  },
});
```

---

## 12.7 Actualizar ProfileScreen

Integra `AvatarImage` en la pantalla de perfil del usuario. El avatar queda en la parte superior y es editable.

```typescript
// src/presentation/screens/profile/ProfileScreen.tsx
// Modifica la sección del encabezado del perfil:

import { AvatarImage } from './components/AvatarImage';
import { useAuthStore } from '../../../domain/store/auth.store';

// Dentro del componente:
const { user, updateUserProfile } = useAuthStore();

const handleAvatarUpdated = (newUrl: string) => {
  // Actualiza la URL del avatar en el authStore para reflejar el cambio
  // en toda la app sin necesidad de recargar
  updateUserProfile({ avatar_url: newUrl });
};

// JSX del encabezado de perfil:
<View style={styles.profileHeader}>
  <AvatarImage
    avatarUrl={user?.avatar_url ?? null}
    firstName={user?.first_name || ''}
    lastName={user?.last_name || ''}
    size={90}
    editable
    onAvatarUpdated={handleAvatarUpdated}
  />
  <View style={styles.profileInfo}>
    <Text style={styles.profileName}>
      {user?.first_name} {user?.last_name}
    </Text>
    <Text style={styles.profileUsername}>@{user?.username}</Text>
    <Text style={styles.profileEmail}>{user?.email}</Text>
  </View>
</View>
```

Agrega los estilos del encabezado si no existen:

```typescript
profileHeader: {
  flexDirection: 'row',
  alignItems: 'center',
  gap: 16,
  backgroundColor: Colors.surface,
  padding: 20,
  borderRadius: 16,
  marginBottom: 16,
  borderWidth: 1,
  borderColor: Colors.border,
},
profileInfo: {
  flex: 1,
  gap: 2,
},
profileName: {
  color: Colors.textPrimary,
  fontSize: 18,
  fontWeight: '700',
},
profileUsername: {
  color: Colors.accent,
  fontSize: 14,
},
profileEmail: {
  color: Colors.textSecondary,
  fontSize: 13,
},
```

### Actualizar el tipo User en authStore

Si el tipo `AuthUser` no incluye `avatar_url`, agrégalo en `src/domain/model/auth.types.ts`:

```typescript
export interface AuthUser {
  id: number;
  username: string;
  email: string;
  first_name: string;
  last_name: string;
  is_staff: boolean;
  avatar_url: string | null;  // NUEVO
}
```

Y agrega la acción `updateUserProfile` al authStore:

```typescript
// En el authStore (src/domain/store/auth.store.ts):
updateUserProfile: (updates: Partial<AuthUser>) => {
  set(state => ({
    user: state.user ? { ...state.user, ...updates } : null,
  }));
},
```

---

> **Checkpoint — Módulo 12**
>
> Verifica que puedes:
> - [ ] Ver imágenes de productos en `ProductCard` y `ProductDetailScreen` (o el placeholder si no hay imagen)
> - [ ] Como usuario staff: tocar "Subir imagen" en el detalle de producto, seleccionar una foto de la galería, y ver la imagen actualizada sin recargar la pantalla
> - [ ] En la pantalla de perfil: ver el avatar (o iniciales si no hay foto), tocar el avatar, seleccionar una nueva foto, y ver el cambio reflejado inmediatamente
> - [ ] El indicador de carga (spinner) aparece mientras se sube la imagen

---

**Siguiente módulo:** Módulo 13 — Testing: Unit tests con Jest + React Native Testing Library
