# ShopApp Android — Módulo 2
## Retrofit + Interceptores JWT + DTOs + Repositorios + DataStore

> **Objetivo:** Capa de datos completa: cliente HTTP con renovación automática de tokens, DTOs mapeados a modelos de dominio y DataStore para persistir la sesión.
> **Checkpoint final:** la app llama a `GET /api/categories/` y muestra las categorías en la pantalla de verificación.

---

## 2.1 DataStore — `data/local/TokenDataStore.kt`

Persiste access token, refresh token y datos del usuario entre sesiones.

```kotlin
// data/local/TokenDataStore.kt
package com.shopapp.data.local

import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.*
import androidx.datastore.preferences.preferencesDataStore
import dagger.hilt.android.qualifiers.ApplicationContext
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.flow.map
import javax.inject.Inject
import javax.inject.Singleton

private val Context.dataStore: DataStore<Preferences>
    by preferencesDataStore(name = "shop_session")

@Singleton
class TokenDataStore @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    companion object {
        val KEY_ACCESS   = stringPreferencesKey("access_token")
        val KEY_REFRESH  = stringPreferencesKey("refresh_token")
        val KEY_USER_ID  = intPreferencesKey("user_id")
        val KEY_USERNAME = stringPreferencesKey("username")
        val KEY_EMAIL    = stringPreferencesKey("email")
        val KEY_IS_STAFF = booleanPreferencesKey("is_staff")
    }

    // ── Reads ─────────────────────────────────────────────────

    val accessToken: Flow<String?>  = context.dataStore.data.map { it[KEY_ACCESS]  }
    val refreshToken: Flow<String?> = context.dataStore.data.map { it[KEY_REFRESH] }

    suspend fun getAccessToken():  String? = accessToken.first()
    suspend fun getRefreshToken(): String? = refreshToken.first()

    val isLoggedIn: Flow<Boolean> = context.dataStore.data.map {
        !it[KEY_ACCESS].isNullOrBlank()
    }

    val isStaff: Flow<Boolean> = context.dataStore.data.map {
        it[KEY_IS_STAFF] == true
    }

    // ── Writes ────────────────────────────────────────────────

    suspend fun saveTokens(access: String, refresh: String) {
        context.dataStore.edit { prefs ->
            prefs[KEY_ACCESS]  = access
            prefs[KEY_REFRESH] = refresh
        }
    }

    suspend fun saveAccessToken(access: String) {
        context.dataStore.edit { it[KEY_ACCESS] = access }
    }

    suspend fun saveUser(id: Int, username: String, email: String, isStaff: Boolean) {
        context.dataStore.edit { prefs ->
            prefs[KEY_USER_ID]  = id
            prefs[KEY_USERNAME] = username
            prefs[KEY_EMAIL]    = email
            prefs[KEY_IS_STAFF] = isStaff
        }
    }

    suspend fun clearSession() {
        context.dataStore.edit { it.clear() }
    }

    // ── User snapshot ─────────────────────────────────────────

    data class UserSnapshot(
        val id:       Int,
        val username: String,
        val email:    String,
        val isStaff:  Boolean,
    )

    val userSnapshot: Flow<UserSnapshot?> = context.dataStore.data.map { prefs ->
        val id = prefs[KEY_USER_ID] ?: return@map null
        UserSnapshot(
            id       = id,
            username = prefs[KEY_USERNAME] ?: "",
            email    = prefs[KEY_EMAIL]    ?: "",
            isStaff  = prefs[KEY_IS_STAFF] == true,
        )
    }
}
```

---

## 2.2 DTOs — un archivo por dominio

Los DTOs son la representación exacta del JSON que devuelve Django.

### `data/remote/dto/AuthDto.kt`

```kotlin
// data/remote/dto/AuthDto.kt
package com.shopapp.data.remote.dto

import com.google.gson.annotations.SerializedName

data class LoginRequest(
    val username: String,
    val password: String,
)

data class RegisterRequest(
    val username:  String,
    val email:     String,
    val password:  String,
    @SerializedName("password2") val password2: String,
)

data class TokenRefreshRequest(
    val refresh: String,
)

data class LogoutRequest(
    val refresh: String,
)

data class AuthResponseDto(
    val access:   String,
    val refresh:  String,
    @SerializedName("user_id")  val userId:  Int,
    val username: String,
    val email:    String,
    @SerializedName("is_staff") val isStaff: Boolean,
)

data class TokenRefreshResponseDto(
    val access:  String,
    val refresh: String?,   // con ROTATE_REFRESH_TOKENS=True también devuelve nuevo refresh
)
```

### `data/remote/dto/CategoryDto.kt`

```kotlin
// data/remote/dto/CategoryDto.kt
package com.shopapp.data.remote.dto

import com.google.gson.annotations.SerializedName
import com.shopapp.domain.model.Category
import com.shopapp.domain.model.CategoryPayload

data class CategoryDto(
    val id:          Int,
    val name:        String,
    val slug:        String,
    val description: String,
    @SerializedName("is_active")      val isActive:      Boolean,
    @SerializedName("total_products") val totalProducts: Int,
    @SerializedName("created_at")     val createdAt:     String,
)

data class CategoryRequestDto(
    val name:        String,
    val slug:        String,
    val description: String,
    @SerializedName("is_active") val isActive: Boolean,
)

data class CategoryStatsDto(
    val total:    Int,
    val active:   Int,
    val inactive: Int,
)

// ── Mappers ───────────────────────────────────────────────────

fun CategoryDto.toDomain() = Category(
    id            = id,
    name          = name,
    slug          = slug,
    description   = description,
    isActive      = isActive,
    totalProducts = totalProducts,
    createdAt     = createdAt,
)

fun CategoryPayload.toRequest() = CategoryRequestDto(
    name        = name,
    slug        = slug,
    description = description,
    isActive    = isActive,
)
```

### `data/remote/dto/ProductDto.kt`

```kotlin
// data/remote/dto/ProductDto.kt
package com.shopapp.data.remote.dto

import com.google.gson.annotations.SerializedName
import com.shopapp.domain.model.Product
import com.shopapp.domain.model.ProductPayload

data class CategorySummaryDto(
    val id:   Int,
    val name: String,
)

data class ProductDto(
    val id:          Int,
    val name:        String,
    val description: String,
    val price:       String,          // Django devuelve Decimal como String
    @SerializedName("price_with_tax") val priceWithTax: Double,
    val stock:       Int,
    @SerializedName("in_stock")  val inStock:  Boolean,
    @SerializedName("is_active") val isActive: Boolean,
    val image:       String?,
    @SerializedName("image_url") val imageUrl: String?,
    val category:    CategorySummaryDto?,
    @SerializedName("created_at") val createdAt: String,
    @SerializedName("updated_at") val updatedAt: String,
)

data class ProductRequestDto(
    val name:        String,
    val description: String,
    val price:       Double,
    val stock:       Int,
    @SerializedName("is_active")   val isActive:   Boolean,
    @SerializedName("category_id") val categoryId: Int,
)

data class ProductStatsDto(
    @SerializedName("total_active")   val totalActive:   Int,
    @SerializedName("total_inactive") val totalInactive: Int,
    @SerializedName("avg_price")      val avgPrice:      Double?,
    @SerializedName("max_price")      val maxPrice:      Double?,
    @SerializedName("min_price")      val minPrice:      Double?,
    @SerializedName("total_stock")    val totalStock:    Int?,
    @SerializedName("out_of_stock")   val outOfStock:    Int,
)

data class RestockResponseDto(
    val id:          Int,
    val name:        String,
    @SerializedName("new_stock") val newStock: Int,
)

data class RestockRequestDto(
    val quantity: Int,
)

// ── Mappers ───────────────────────────────────────────────────

fun ProductDto.toDomain() = Product(
    id           = id,
    name         = name,
    description  = description,
    price        = price.toDoubleOrNull() ?: 0.0,
    priceWithTax = priceWithTax,
    stock        = stock,
    inStock      = inStock,
    isActive     = isActive,
    imageUrl     = imageUrl,
    categoryId   = category?.id,
    categoryName = category?.name,
    createdAt    = createdAt,
    updatedAt    = updatedAt,
)

fun ProductPayload.toRequest() = ProductRequestDto(
    name        = name,
    description = description,
    price       = price,
    stock       = stock,
    isActive    = isActive,
    categoryId  = categoryId,
)
```

### `data/remote/dto/OrderDto.kt`

```kotlin
// data/remote/dto/OrderDto.kt
package com.shopapp.data.remote.dto

import com.google.gson.annotations.SerializedName
import com.shopapp.domain.model.Order
import com.shopapp.domain.model.OrderItem
import com.shopapp.domain.model.OrderStatus

data class ProductInItemDto(
    val id:    Int,
    val name:  String,
    val price: String,
    val stock: Int,
    @SerializedName("is_active") val isActive: Boolean,
)

data class OrderItemDto(
    val id:       Int,
    val product:  ProductInItemDto,
    val quantity: Int,
    @SerializedName("unit_price") val unitPrice: String,
    val subtotal: Double,
)

data class OrderDto(
    val id:       Int,
    val username: String,
    val status:   String,
    val total:    String,
    @SerializedName("num_items")  val numItems:  Int,
    val items:    List<OrderItemDto>,
    @SerializedName("created_at") val createdAt: String,
    @SerializedName("updated_at") val updatedAt: String,
)

data class AddItemRequestDto(
    @SerializedName("product_id") val productId: Int,
    val quantity: Int,
)

data class UpdateStatusRequestDto(
    val status: String,
)

data class OrderStatsDto(
    @SerializedName("total_orders")   val totalOrders:   Int,
    @SerializedName("total_revenue")  val totalRevenue:  Double,
    @SerializedName("by_status")      val byStatus:      Map<String, Int>,
)

// ── Mappers ───────────────────────────────────────────────────

fun OrderItemDto.toDomain() = OrderItem(
    id          = id,
    productId   = product.id,
    productName = product.name,
    quantity    = quantity,
    unitPrice   = unitPrice.toDoubleOrNull() ?: 0.0,
    subtotal    = subtotal,
)

fun OrderDto.toDomain() = Order(
    id        = id,
    username  = username,
    status    = OrderStatus.fromValue(status),
    total     = total.toDoubleOrNull() ?: 0.0,
    numItems  = numItems,
    items     = items.map { it.toDomain() },
    createdAt = createdAt,
    updatedAt = updatedAt,
)
```

### `data/remote/dto/UserDto.kt`

```kotlin
// data/remote/dto/UserDto.kt
package com.shopapp.data.remote.dto

import com.google.gson.annotations.SerializedName
import com.shopapp.domain.model.User
import com.shopapp.domain.model.UserPayload

data class UserDto(
    val id:         Int,
    val username:   String,
    val email:      String,
    @SerializedName("first_name")  val firstName:  String,
    @SerializedName("last_name")   val lastName:   String,
    @SerializedName("is_staff")    val isStaff:    Boolean,
    @SerializedName("is_active")   val isActive:   Boolean,
    @SerializedName("date_joined") val dateJoined: String,
    @SerializedName("num_orders")  val numOrders:  Int,
)

data class UserRequestDto(
    val username:   String,
    val email:      String,
    @SerializedName("first_name") val firstName: String,
    @SerializedName("last_name")  val lastName:  String,
    @SerializedName("is_staff")   val isStaff:   Boolean,
    @SerializedName("is_active")  val isActive:  Boolean,
    val password:   String? = null,
)

data class ToggleActiveResponseDto(
    val message:   String,
    @SerializedName("is_active") val isActive: Boolean,
)

data class UserStatsDto(
    val total:    Int,
    val active:   Int,
    val inactive: Int,
    val staff:    Int,
)

// ── Mappers ───────────────────────────────────────────────────

fun UserDto.toDomain() = User(
    id         = id,
    username   = username,
    email      = email,
    firstName  = firstName,
    lastName   = lastName,
    isStaff    = isStaff,
    isActive   = isActive,
    dateJoined = dateJoined,
    numOrders  = numOrders,
)

fun UserPayload.toRequest() = UserRequestDto(
    username  = username,
    email     = email,
    firstName = firstName,
    lastName  = lastName,
    isStaff   = isStaff,
    isActive  = isActive,
    password  = password,
)
```

---

## 2.3 Modelo de respuesta paginada — `data/remote/dto/PaginatedDto.kt`

```kotlin
// data/remote/dto/PaginatedDto.kt
package com.shopapp.data.remote.dto

data class PaginatedDto<T>(
    val count:    Int,
    val next:     String?,
    val previous: String?,
    val results:  List<T>,
)
```

---

## 2.4 APIs de Retrofit — `data/remote/api/`

### `data/remote/api/AuthApi.kt`

```kotlin
// data/remote/api/AuthApi.kt
package com.shopapp.data.remote.api

import com.shopapp.data.remote.dto.*
import retrofit2.Response
import retrofit2.http.Body
import retrofit2.http.POST

interface AuthApi {
    @POST("auth/login/")
    suspend fun login(@Body body: LoginRequest): Response<AuthResponseDto>

    @POST("auth/register/")
    suspend fun register(@Body body: RegisterRequest): Response<AuthResponseDto>

    @POST("auth/token/refresh/")
    suspend fun refreshToken(@Body body: TokenRefreshRequest): Response<TokenRefreshResponseDto>

    @POST("auth/logout/")
    suspend fun logout(@Body body: LogoutRequest): Response<Unit>
}
```

### `data/remote/api/CategoryApi.kt`

```kotlin
// data/remote/api/CategoryApi.kt
package com.shopapp.data.remote.api

import com.shopapp.data.remote.dto.*
import retrofit2.Response
import retrofit2.http.*

interface CategoryApi {
    @GET("categories/")
    suspend fun getCategories(): Response<PaginatedDto<CategoryDto>>

    @GET("categories/{id}/")
    suspend fun getCategory(@Path("id") id: Int): Response<CategoryDto>

    @POST("categories/")
    suspend fun createCategory(@Body body: CategoryRequestDto): Response<CategoryDto>

    @PATCH("categories/{id}/")
    suspend fun updateCategory(
        @Path("id") id: Int,
        @Body body: CategoryRequestDto,
    ): Response<CategoryDto>

    @DELETE("categories/{id}/")
    suspend fun deleteCategory(@Path("id") id: Int): Response<Unit>

    @GET("categories/stats/")
    suspend fun getStats(): Response<CategoryStatsDto>
}
```

### `data/remote/api/ProductApi.kt`

```kotlin
// data/remote/api/ProductApi.kt
package com.shopapp.data.remote.api

import com.shopapp.data.remote.dto.*
import retrofit2.Response
import retrofit2.http.*

interface ProductApi {
    @GET("products/")
    suspend fun getProducts(
        @QueryMap filters: Map<String, String>,
    ): Response<PaginatedDto<ProductDto>>

    @GET("products/{id}/")
    suspend fun getProduct(@Path("id") id: Int): Response<ProductDto>

    @GET("products/available/")
    suspend fun getAvailable(): Response<PaginatedDto<ProductDto>>

    @POST("products/")
    suspend fun createProduct(@Body body: ProductRequestDto): Response<ProductDto>

    @PATCH("products/{id}/")
    suspend fun updateProduct(
        @Path("id") id: Int,
        @Body body: ProductRequestDto,
    ): Response<ProductDto>

    @DELETE("products/{id}/")
    suspend fun deleteProduct(@Path("id") id: Int): Response<Unit>

    @POST("products/{id}/restock/")
    suspend fun restock(
        @Path("id") id: Int,
        @Body body: RestockRequestDto,
    ): Response<RestockResponseDto>

    @GET("products/stats/")
    suspend fun getStats(): Response<ProductStatsDto>
}
```

### `data/remote/api/OrderApi.kt`

```kotlin
// data/remote/api/OrderApi.kt
package com.shopapp.data.remote.api

import com.shopapp.data.remote.dto.*
import retrofit2.Response
import retrofit2.http.*

interface OrderApi {
    @GET("orders/")
    suspend fun getOrders(
        @Query("page")   page:   Int?    = null,
        @Query("status") status: String? = null,
    ): Response<PaginatedDto<OrderDto>>

    @GET("orders/{id}/")
    suspend fun getOrder(@Path("id") id: Int): Response<OrderDto>

    @POST("orders/")
    suspend fun createOrder(): Response<OrderDto>

    @POST("orders/{id}/add-item/")
    suspend fun addItem(
        @Path("id") id: Int,
        @Body body: AddItemRequestDto,
    ): Response<OrderDto>

    @POST("orders/{id}/confirm/")
    suspend fun confirmOrder(@Path("id") id: Int): Response<OrderDto>

    @POST("orders/{id}/update-status/")
    suspend fun updateStatus(
        @Path("id") id: Int,
        @Body body: UpdateStatusRequestDto,
    ): Response<OrderDto>

    @GET("orders/stats/")
    suspend fun getStats(): Response<OrderStatsDto>
}
```

### `data/remote/api/UserApi.kt`

```kotlin
// data/remote/api/UserApi.kt
package com.shopapp.data.remote.api

import com.shopapp.data.remote.dto.*
import retrofit2.Response
import retrofit2.http.*

interface UserApi {
    @GET("users/")
    suspend fun getUsers(
        @Query("search")    search:   String?  = null,
        @Query("is_staff")  isStaff:  Boolean? = null,
        @Query("is_active") isActive: Boolean? = null,
        @Query("page")      page:     Int?     = null,
    ): Response<PaginatedDto<UserDto>>

    @GET("users/{id}/")
    suspend fun getUser(@Path("id") id: Int): Response<UserDto>

    @POST("users/")
    suspend fun createUser(@Body body: UserRequestDto): Response<UserDto>

    @PATCH("users/{id}/")
    suspend fun updateUser(
        @Path("id") id: Int,
        @Body body: UserRequestDto,
    ): Response<UserDto>

    @DELETE("users/{id}/")
    suspend fun deleteUser(@Path("id") id: Int): Response<Unit>

    @POST("users/{id}/toggle-active/")
    suspend fun toggleActive(@Path("id") id: Int): Response<ToggleActiveResponseDto>

    @GET("users/profile/")
    suspend fun getProfile(): Response<UserDto>

    @GET("users/stats/")
    suspend fun getStats(): Response<UserStatsDto>
}
```

---

## 2.5 Interceptor JWT — `data/remote/interceptor/AuthInterceptor.kt`

```kotlin
// data/remote/interceptor/AuthInterceptor.kt
package com.shopapp.data.remote.interceptor

import com.shopapp.BuildConfig
import com.shopapp.data.local.TokenDataStore
import com.shopapp.data.remote.dto.TokenRefreshRequest
import com.google.gson.Gson
import kotlinx.coroutines.runBlocking
import okhttp3.*
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.RequestBody.Companion.toRequestBody
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class AuthInterceptor @Inject constructor(
    private val tokenDataStore: TokenDataStore,
) : Authenticator {

    override fun authenticate(route: Route?, response: Response): Request? {
        // Solo reintentar una vez
        if (response.request.header("X-Retry") != null) return null

        val refresh = runBlocking { tokenDataStore.getRefreshToken() }
            ?: return null

        // Llamada síncrona al endpoint de refresh
        val client   = OkHttpClient()
        val gson     = Gson()
        val body     = gson.toJson(TokenRefreshRequest(refresh))
        val request  = Request.Builder()
            .url("${BuildConfig.API_BASE_URL}auth/token/refresh/")
            .post(body.toRequestBody("application/json".toMediaType()))
            .build()

        val refreshResponse = try {
            client.newCall(request).execute()
        } catch (e: Exception) {
            return null
        }

        if (!refreshResponse.isSuccessful) {
            runBlocking { tokenDataStore.clearSession() }
            return null
        }

        val responseBody = refreshResponse.body?.string() ?: return null
        val newAccess    = try {
            gson.fromJson(responseBody, Map::class.java)["access"] as? String
        } catch (e: Exception) { null } ?: return null

        runBlocking { tokenDataStore.saveAccessToken(newAccess) }

        return response.request.newBuilder()
            .header("Authorization", "Bearer $newAccess")
            .header("X-Retry", "true")
            .build()
    }
}

// Interceptor que añade el Bearer token a cada petición
class BearerTokenInterceptor(
    private val tokenDataStore: TokenDataStore,
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val access = runBlocking { tokenDataStore.getAccessToken() }
        val request = if (access != null) {
            chain.request().newBuilder()
                .header("Authorization", "Bearer $access")
                .build()
        } else {
            chain.request()
        }
        return chain.proceed(request)
    }
}
```

---

## 2.6 Módulo Hilt de red — `di/NetworkModule.kt`

```kotlin
// di/NetworkModule.kt
package com.shopapp.di

import com.shopapp.BuildConfig
import com.shopapp.data.local.TokenDataStore
import com.shopapp.data.remote.api.*
import com.shopapp.data.remote.interceptor.AuthInterceptor
import com.shopapp.data.remote.interceptor.BearerTokenInterceptor
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.util.concurrent.TimeUnit
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides @Singleton
    fun provideLoggingInterceptor() = HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
    }

    @Provides @Singleton
    fun provideOkHttpClient(
        tokenDataStore: TokenDataStore,
        authInterceptor: AuthInterceptor,
        logging: HttpLoggingInterceptor,
    ): OkHttpClient = OkHttpClient.Builder()
        .authenticator(authInterceptor)                          // renueva el token en 401
        .addInterceptor(BearerTokenInterceptor(tokenDataStore))  // añade Bearer a cada request
        .addInterceptor(logging)
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .build()

    @Provides @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit = Retrofit.Builder()
        .baseUrl(BuildConfig.API_BASE_URL)
        .client(client)
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    @Provides @Singleton
    fun provideAuthApi(retrofit: Retrofit): AuthApi =
        retrofit.create(AuthApi::class.java)

    @Provides @Singleton
    fun provideCategoryApi(retrofit: Retrofit): CategoryApi =
        retrofit.create(CategoryApi::class.java)

    @Provides @Singleton
    fun provideProductApi(retrofit: Retrofit): ProductApi =
        retrofit.create(ProductApi::class.java)

    @Provides @Singleton
    fun provideOrderApi(retrofit: Retrofit): OrderApi =
        retrofit.create(OrderApi::class.java)

    @Provides @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi =
        retrofit.create(UserApi::class.java)
}
```

---

## 2.7 Interfaces de repositorio — `domain/repository/`

```kotlin
// domain/repository/CategoryRepository.kt
package com.shopapp.domain.repository

import com.shopapp.domain.model.Category
import com.shopapp.domain.model.CategoryPayload

interface CategoryRepository {
    suspend fun getCategories(): Result<List<Category>>
    suspend fun getCategory(id: Int): Result<Category>
    suspend fun createCategory(payload: CategoryPayload): Result<Category>
    suspend fun updateCategory(id: Int, payload: CategoryPayload): Result<Category>
    suspend fun deleteCategory(id: Int): Result<Unit>
}
```

```kotlin
// domain/repository/ProductRepository.kt
package com.shopapp.domain.repository

import com.shopapp.data.remote.dto.PaginatedDto
import com.shopapp.data.remote.dto.ProductStatsDto
import com.shopapp.data.remote.dto.RestockResponseDto
import com.shopapp.domain.model.Product
import com.shopapp.domain.model.ProductFilters
import com.shopapp.domain.model.ProductPayload

interface ProductRepository {
    suspend fun getProducts(filters: ProductFilters): Result<Pair<List<Product>, Int>>
    suspend fun getProduct(id: Int): Result<Product>
    suspend fun createProduct(payload: ProductPayload): Result<Product>
    suspend fun updateProduct(id: Int, payload: ProductPayload): Result<Product>
    suspend fun deleteProduct(id: Int): Result<Unit>
    suspend fun restock(id: Int, quantity: Int): Result<Int>
    suspend fun getStats(): Result<Map<String, Any>>
}
```

---

## 2.8 Implementación de repositorio — `data/repository/CategoryRepositoryImpl.kt`

```kotlin
// data/repository/CategoryRepositoryImpl.kt
package com.shopapp.data.repository

import com.shopapp.data.remote.api.CategoryApi
import com.shopapp.data.remote.dto.toDomain
import com.shopapp.data.remote.dto.toRequest
import com.shopapp.domain.model.Category
import com.shopapp.domain.model.CategoryPayload
import com.shopapp.domain.repository.CategoryRepository
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class CategoryRepositoryImpl @Inject constructor(
    private val api: CategoryApi,
) : CategoryRepository {

    override suspend fun getCategories(): Result<List<Category>> = runCatching {
        val response = api.getCategories()
        if (response.isSuccessful) {
            response.body()!!.results.map { it.toDomain() }
        } else {
            error("Error ${response.code()}: ${response.errorBody()?.string()}")
        }
    }

    override suspend fun getCategory(id: Int): Result<Category> = runCatching {
        val response = api.getCategory(id)
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}")
    }

    override suspend fun createCategory(payload: CategoryPayload): Result<Category> = runCatching {
        val response = api.createCategory(payload.toRequest())
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}: ${response.errorBody()?.string()}")
    }

    override suspend fun updateCategory(id: Int, payload: CategoryPayload): Result<Category> =
        runCatching {
            val response = api.updateCategory(id, payload.toRequest())
            if (response.isSuccessful) response.body()!!.toDomain()
            else error("Error ${response.code()}: ${response.errorBody()?.string()}")
        }

    override suspend fun deleteCategory(id: Int): Result<Unit> = runCatching {
        val response = api.deleteCategory(id)
        if (!response.isSuccessful) error("Error ${response.code()}")
    }
}
```

---

## 2.9 Módulo Hilt de repositorios — `di/RepositoryModule.kt`

```kotlin
// di/RepositoryModule.kt
package com.shopapp.di

import com.shopapp.data.repository.CategoryRepositoryImpl
import com.shopapp.domain.repository.CategoryRepository
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds @Singleton
    abstract fun bindCategoryRepository(
        impl: CategoryRepositoryImpl,
    ): CategoryRepository

    // Los demás repositorios se añaden en los módulos siguientes
}
```

---

## 2.10 Verificación — actualizar `MainActivity.kt`

```kotlin
// MainActivity.kt — temporalmente inyectar el repositorio para probar
package com.shopapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import com.shopapp.domain.model.Category
import com.shopapp.domain.repository.CategoryRepository
import com.shopapp.theme.ShopAppTheme
import dagger.hilt.android.AndroidEntryPoint
import javax.inject.Inject

@AndroidEntryPoint
class MainActivity : ComponentActivity() {

    @Inject lateinit var categoryRepository: CategoryRepository

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            ShopAppTheme {
                Surface(modifier = Modifier.fillMaxSize()) {
                    var categories by remember { mutableStateOf<List<Category>>(emptyList()) }
                    var status     by remember { mutableStateOf("Conectando...") }

                    LaunchedEffect(Unit) {
                        categoryRepository.getCategories()
                            .onSuccess {
                                categories = it
                                status     = "✅ ${it.size} categorías del backend"
                            }
                            .onFailure {
                                status = "❌ ${it.message}"
                            }
                    }

                    VerificationScreen(
                        connectionStatus = status,
                        categories       = categories,
                    )
                }
            }
        }
    }
}
```

Actualizar `VerificationScreen` para aceptar los nuevos parámetros:

```kotlin
package com.shopapp

import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.HorizontalDivider
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.shopapp.domain.model.Category
import com.shopapp.theme.*

@Composable
fun VerificationScreen(
    connectionStatus: String = "Sin conectar",
    categories: List<Category> = emptyList(),
) {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(Background),
    ) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .verticalScroll(rememberScrollState())
                .padding(top = 56.dp, start = 24.dp, end = 24.dp, bottom = 24.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
        ) {
            Text(
                text = "ShopApp",
                fontSize = 40.sp,
                fontWeight = FontWeight.Bold,
                color = Accent,
            )

            Text(
                text = "Módulo 2 · Conexión con backend",
                style = MaterialTheme.typography.bodyMedium,
                color = TextSecondary,
                modifier = Modifier.padding(top = 8.dp, bottom = 32.dp),
            )

            EnvCard(
                items = listOf(
                    "Kotlin" to "2.0.21",
                    "Compose BOM" to "2024.10.01",
                    "Material 3" to "✓",
                    "Hilt" to "2.52",
                    "Retrofit" to "2.11.0",
                    "API URL" to BuildConfig.API_BASE_URL,
                ),
            )

            Spacer(modifier = Modifier.height(24.dp))

            Text(
                text = connectionStatus,
                style = MaterialTheme.typography.bodyMedium,
                color = if (connectionStatus.startsWith("✅")) Success else Error,
            )

            Spacer(modifier = Modifier.height(16.dp))

            categories.take(3).forEach { cat ->
                Text(
                    text = "• ${cat.name} (${cat.totalProducts} productos)",
                    style = MaterialTheme.typography.bodySmall,
                    color = TextSecondary,
                    modifier = Modifier.padding(vertical = 4.dp),
                )
            }
        }
    }
}

@Composable
private fun EnvCard(items: List<Pair<String, String>>) {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .background(Surface, RoundedCornerShape(16.dp))
            .border(1.dp, Border, RoundedCornerShape(16.dp))
            .padding(16.dp),
    ) {
        Text(
            text = "Estado del entorno",
            style = MaterialTheme.typography.labelSmall,
            color = TextSecondary,
            letterSpacing = 1.sp,
            modifier = Modifier.padding(bottom = 12.dp),
        )

        items.forEachIndexed { index, item ->
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(vertical = 8.dp),
                horizontalArrangement = Arrangement.SpaceBetween,
            ) {
                Text(
                    text = item.first,
                    style = MaterialTheme.typography.bodySmall,
                    color = TextSecondary,
                )
                Text(
                    text = item.second,
                    style = MaterialTheme.typography.bodySmall,
                    color = TextPrimary,
                    fontWeight = FontWeight.SemiBold,
                    maxLines = 1,
                )
            }

            if (index < items.lastIndex) {
                HorizontalDivider(color = BorderLight, thickness = 0.5.dp)
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun VerificationScreenPreview() {
    ShopAppTheme {
        VerificationScreen()
    }
}
```

---

## ✅ Checkpoint Módulo 2

### Django debe estar corriendo

```bash
uv run python manage.py runserver 0.0.0.0:8000
```

### En el emulador Android

| Verificación | Resultado esperado |
|---|---|
| App compila sin errores | 0 errores en Build |
| Status "✅ N categorías del backend" | Los datos reales de Django aparecen |
| Log de OkHttp | `D/OkHttp: --> GET http://10.0.2.2:8000/api/categories/` |
| Status "❌ ..." con Django apagado | Mensaje de error visible |

### Ver logs de red en Logcat

```
Logcat → filtrar por "OkHttp" → ver peticiones y respuestas HTTP
```

---

## Resumen

| Elemento | Estado |
|---|---|
| `TokenDataStore` con DataStore Preferences | ✅ |
| DTOs con `@SerializedName` y mappers `toDomain()` | ✅ |
| `PaginatedDto<T>` genérico | ✅ |
| 5 interfaces de API con Retrofit | ✅ |
| `AuthInterceptor` (Authenticator) — renueva token en 401 | ✅ |
| `BearerTokenInterceptor` — añade Bearer a cada request | ✅ |
| `NetworkModule` con Hilt | ✅ |
| `CategoryRepositoryImpl` con `runCatching` | ✅ |
| `RepositoryModule` con Hilt | ✅ |
| Conexión real verificada en el emulador | ✅ |

**Siguiente módulo →** M3: Auth — Login, Registro, AuthViewModel y persistencia de sesión