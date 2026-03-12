# Networking & API

## Stack

| Component | Library | Location |
|-----------|---------|----------|
| HTTP Client | Retrofit 3 + OkHttp | `core/network/HttpClient.kt` |
| Serialization | Gson | `GsonConverterFactory` |
| Auth | TokenManager + AuthTokenInterceptor | `core/network/` |
| Interceptors | Auth, BaseUrl, Network, Response | `core/network/interceptors/` |

## API Interface Pattern

All API interfaces live in `data/api/`:

```kotlin
interface IFooAPI {

    @GET("foo/{id}")
    suspend fun getFoo(@Path("id") id: String): FooResponse

    @GET("foo/")
    suspend fun getFoos(@Query("page") page: Int): ListResponse<FooResponse>

    @POST("foo/")
    suspend fun createFoo(@Body request: CreateFooRequest): FooResponse

    @PUT("foo/{id}")
    suspend fun updateFoo(@Path("id") id: String, @Body request: UpdateFooRequest): FooResponse

    @DELETE("foo/{id}")
    suspend fun deleteFoo(@Path("id") id: String): Unit

    companion object {
        const val FOO_ENDPOINT = "foo/"
    }
}
```

Rules:
- **All methods are `suspend`** — no blocking, no `Call<T>` wrappers
- Return the response model directly (Retrofit handles deserialization)
- Use `@Path`, `@Query`, `@Body`, `@Header` annotations
- Endpoint constants in `companion object`
- KDoc on each method describing what it does
- **Never call APIs directly from ViewModels** — always through a repository or service

## API Registration

In `core/di/APIModule.kt`:

```kotlin
@Provides @Singleton
fun provideFooAPI(httpClient: HttpClient): IFooAPI =
    httpClient.createService(IFooAPI::class.java)
```

## Request/Response Models

API models live in `domain/model/api/<feature>/`:

```kotlin
// Request
data class CreateFooRequest(
    val name: String,
    val weight: Double,
    val type: String,
)

// Response
data class FooResponse(
    val id: String,
    val name: String,
    val weight: Double,
    val createdAt: String,
)
```

- Use `data class` for all API models
- Field names match JSON keys (Gson handles serialization)
- Use `@SerializedName("json_key")` only when Kotlin name differs from JSON

## Repository Pattern (API → Domain)

Repositories bridge API calls to domain models:

```kotlin
class FooRepository @Inject constructor(
    private val fooAPI: IFooAPI,
    private val fooDao: FooDao,
) : IFooRepository {

    override suspend fun getFoo(id: String): Foo {
        val response = fooAPI.getFoo(id)
        return response.toDomain()  // Map API model → domain model
    }

    override suspend fun createFoo(foo: Foo): Foo {
        val request = foo.toRequest()  // Map domain model → API model
        val response = fooAPI.createFoo(request)
        fooDao.insert(response.toEntity())  // Cache in Room
        return response.toDomain()
    }
}
```

Rules:
- Repositories handle API call + caching + model mapping
- **No business logic in repositories** — that belongs in services
- Map between API models, domain models, and storage entities

## Interceptor Chain

Requests pass through (in order):

1. **BaseUrlInterceptor** — dynamic base URL switching
2. **AuthTokenInterceptor** — adds `Authorization: Bearer <token>`, proactive token refresh (5-min buffer before expiry)
3. **NetworkInterceptor** — network diagnostics
4. **ResponseInterceptor** — response handling

Token refresh uses mutex lock for thread safety and retry logic (2 retries with exponential backoff on 500+ errors).

**Thread safety:** Interceptors run on OkHttp's concurrent thread pool. Use `DateTimeFormatter` (thread-safe), never `SimpleDateFormat`.

## Token Storage

Tokens are stored in **`SecureTokenStore`** using `EncryptedSharedPreferences` (Android Keystore-backed encryption). Not plain DataStore.

- `SecureTokenStore` — `core/network/SecureTokenStore.kt`
- `TokenMigrationHelper` — handles migration from legacy DataStore → SecureTokenStore
- `TokenManager` uses `SecureTokenStore` for all token read/write operations
- Multi-account tokens stored in `ConcurrentHashMap<String, Token>` in memory, persisted to SecureTokenStore

## Multi-Account Headers

For multi-account API calls:

```kotlin
@GET("account/entries")
suspend fun getEntries(
    @Header("X-Account-ID") accountId: String,
): List<EntryResponse>
```

The `X-Account-ID` header tells `AuthTokenInterceptor` which account's token to use.

## Build Types & Base URLs

| Build Type | Base URL | Analytics |
|------------|----------|-----------|
| `debug` | `http://ec2-54-161-28-150.compute-1.amazonaws.com:3005/` (staging) | Disabled |
| `release` | `https://api.weightgurus.com/v3/` (production) | Enabled |

Debug builds allow cleartext HTTP to the staging server only.

## Error Handling

```kotlin
// In service/repository
override suspend fun saveFoo(foo: Foo): Result<Foo> {
    return try {
        val response = fooAPI.createFoo(foo.toRequest())
        Result.success(response.toDomain())
    } catch (e: HttpException) {
        AppLog.e(TAG, "API error: ${e.code()}", e)
        Result.failure(e)
    } catch (e: IOException) {
        AppLog.e(TAG, "Network error", e)
        Result.failure(e)
    }
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Non-suspend API method | All Retrofit methods must be `suspend` |
| Calling API from ViewModel directly | Route through repository → service |
| Business logic in repository | Repositories only fetch/cache — logic goes in services |
| Forgetting to register API in APIModule | Add `@Provides` in `core/di/APIModule.kt` |
| Hardcoding base URL | Use `BuildConfig.BASE_URL` — it switches per build type |
| `SimpleDateFormat` in interceptor | Use `DateTimeFormatter` — thread-safe for OkHttp's thread pool |
| Storing tokens in plain DataStore | Use `SecureTokenStore` (EncryptedSharedPreferences) |
