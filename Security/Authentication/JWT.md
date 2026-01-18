# Autenticación en .NET 9 — JWT, Refresh Tokens y Redis (detallado)

**Instructor:** Juan Carlos De La Cruz Chinga  
**Tecnologías:** .NET 9, ASP.NET Core, JWT, Redis, Identity, StackExchange.Redis

---


## 1. Resumen y objetivos

Este documento detalla un flujo de autenticación moderno en .NET 9 usando **JWT** (access token), **Refresh Tokens** y **Redis** como almacén de tokens. El objetivo es ofrecer teoría, explicación de cada método presente en los snippets que proporcionaste y propuestas prácticas y seguras para su implementación en APIs y aplicaciones web.

Se asume que se dispone de ASP.NET Core 7/8/9, Identity configurada (UserManager, RoleManager), y que la app usará cookies `HttpOnly` para almacenar tokens (patrón que evita exposición en `localStorage`, a cambio hay que mitigar CSRF).

---

## 2. Paquetes NuGet recomendados

- `Microsoft.AspNetCore.Authentication.JwtBearer`  
- `Microsoft.AspNetCore.Identity.EntityFrameworkCore` (o la variante que uses)  
- `System.IdentityModel.Tokens.Jwt`  
- `StackExchange.Redis`  
- `Mapster` (si usas `.Adapt<T>()` como en tus snippets)  
- `Microsoft.Extensions.Configuration` / `Microsoft.Extensions.DependencyInjection`

---

## 3. appsettings.json (sugerencia)

```json
{
  "Jwt": {
    "Key": "tu_clave_secreta_muy_larga_y_segura_aqui",
    "Issuer": "mi-api",
    "Audience": "mi-cliente",
    "AccessTokenExpirationMinutes": "15",
    "RefreshTokenExpirationDays": "7"
  },
  "ConnectionStrings": {
    "Redis": "localhost:6379" 
  }
}
```

> **IMPORTANTE:** la `Key` debe ser un secreto largo y protegido (Key Vault, Secrets Manager en producción). Para entornos distribuidos considera usar certificados (RS256) y rotación de claves.


---

## 4. Docker Compose — Redis (usado en tu ejemplo)

```yaml
redis:
  image: redis:7.4-alpine
  container_name: redis_auth
  restart: always
  ports:
    - "6379:6379"
  volumes:
    - ./redis_data:/data
  command: ["redis-server", "--appendonly", "yes"]
```

**Notas:**  
- `appendonly yes` activa el AOF para persistencia. Útil para que Redis recupere datos tras reinicios.  
- Para producción: considera alta disponibilidad (Sentinel o Cluster) y backups. No exponer Redis sin autenticación y red segura.  

---

## 5. `AddJwtWithCookies` — explicación detallada

Este método configura la autenticación JWT y permite leer el token desde una cookie `access_token` cuando las solicitudes llegan al middleware de JWT bearer.

```csharp
public static IServiceCollection AddJwtWithCookies(this IServiceCollection services, IConfiguration configuration)
{
    var jwtSection = configuration.GetSection("Jwt");
    var key = Encoding.UTF8.GetBytes(jwtSection["Key"]!);

    services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
    })
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = jwtSection["Issuer"],
            ValidAudience = jwtSection["Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(key)
        };

        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                if (context.Request.Cookies.TryGetValue("access_token", out var token))
                {
                    context.Token = token;
                }
                return Task.CompletedTask;
            }
        };
    });

    services.AddAuthorization();

    return services;
}
```

### Explicación por partes

- `jwtSection = configuration.GetSection("Jwt")`: lee la configuración JWT (clave, issuer, audiencia, expiraciones, etc.).  
- `key = Encoding.UTF8.GetBytes(...)`: obtiene bytes de la clave simétrica. En producción es recomendable usar claves de al menos 256 bits (o usar RS256 con certificados).  
- `AddAuthentication(...)`: registra el esquema de autenticación por defecto (JwtBearer).  
- `AddJwtBearer(...)` y `TokenValidationParameters`: parámetros que el middleware usa para validar: issuer, audience, firma y tiempo de expiración (`exp`). `ValidateLifetime = true` hará que tokens expirados sean rechazados automáticamente por el middleware.  
- `options.Events.OnMessageReceived`: **llave práctica** para soportar tokens almacenados en cookies: cuando llega la petición, el evento intenta leer `access_token` desde las cookies y lo pone en `context.Token` para que el resto del middleware lo trate como si viniera en la cabecera `Authorization`.  
  - **Riesgo/Tradeoff:** si guardas el access token en cookie `HttpOnly`, reduces riesgo de XSS. Sin embargo, usar cookies implica riesgos de CSRF — compénsalo con `SameSite` (Lax/Strict) y/o tokens CSRF/anti-forgery.  
- `services.AddAuthorization()`: añade el componente de autorización (atributos `[Authorize]` etc.).


---

## 6. `AuthService` — explicación detallada (método a método)

A continuación se explica la clase `AuthService` que gestionará la generación de tokens y la renovación con refresh.

```csharp
public class AuthService : IAuthService
{
    private readonly IConfiguration _configuration;
    private readonly UserManager<UserApplication> _userManager;
    private readonly IRefreshTokenStore _refreshTokenStore;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public AuthService(IConfiguration configuration, UserManager<UserApplication> userManager, IRefreshTokenStore refreshTokenStore, IHttpContextAccessor httpContextAccessor)
    {
        _configuration = configuration;
        _userManager = userManager;
        _refreshTokenStore = refreshTokenStore;
        _httpContextAccessor = httpContextAccessor;
    }
    ...
}
```

### Constructor
- Inyecta configuraciones, `UserManager` (Identity), el `IRefreshTokenStore` (abstracción para persistencia en Redis) y `IHttpContextAccessor` para leer/escribir cookies en la respuesta y petición.


### `GenerateTokensAsync(User userApp)`

Función responsable de crear **access token** (JWT) y un **refresh token** y almacenarlo. Explicación paso a paso:

```csharp
var jwtSettings = _configuration.GetSection("Jwt");
var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtSettings["Key"]!));
var user = userApp.Adapt<UserApplication>();
var roles = await _userManager.GetRolesAsync(user);
```

- Obtiene las settings JWT y construye `SymmetricSecurityKey` con la `Key` (en bytes).  
- Convierte/ajusta el DTO `User` a `UserApplication` (usando Mapster `.Adapt`).  
- Recupera roles del usuario desde Identity.


```csharp
var claims = new List<Claim>
{
    new Claim(JwtRegisteredClaimNames.Sub, user.Id),
    new Claim(JwtRegisteredClaimNames.Email, user.Email ?? string.Empty),
    new Claim(ClaimTypes.Name, user.UserName ?? string.Empty),
    new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
};

foreach (var role in roles)
    claims.Add(new Claim(ClaimTypes.Role, role));
```

- Se crean claims fundamentales: `sub` (subject / identificador del usuario), `email`, `name` y `jti` (identificador único del token).  
- Se añaden `ClaimTypes.Role` por cada rol para que la autorización basada en roles funcione.


```csharp
var tokenDescriptor = new JwtSecurityToken(
    issuer: jwtSettings["Issuer"],
    audience: jwtSettings["Audience"],
    claims: claims,
    expires: DateTime.UtcNow.AddMinutes(double.Parse(jwtSettings["AccessTokenExpirationMinutes"]!)),
    signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256)
);
var accessToken = new JwtSecurityTokenHandler().WriteToken(tokenDescriptor);
```

- Se configura el token con issuer, audience, claims y expiración. **La expiración aquí será la vida corta del JWT**.  
- `WriteToken` serializa el JWT firmado a string.


```csharp
var refreshToken = GenerateSecureToken();
var expiration = TimeSpan.FromDays(double.Parse(jwtSettings["RefreshTokenExpirationDays"]!));

await _refreshTokenStore.SaveTokenAsync(user.Id, refreshToken, expiration);
// Guardar en cookies seguras
SetAuthCookies(accessToken, refreshToken);
```

- Se genera un refresh token largo y aleatorio mediante `GenerateSecureToken` (más abajo).  
- Se calcula expiración basada en configuración y se guarda en el `IRefreshTokenStore` (p. ej. Redis).  
- Se escriben cookies `access_token` y `refresh_token` en la respuesta.

**Retorna** `(accessToken, refreshToken)` al caller (por si el front necesita otra forma de almacenarlo).


### `GenerateSecureToken()`

```csharp
private string GenerateSecureToken()
{
    var randomBytes = new byte[64];
    using var rng = RandomNumberGenerator.Create();
    rng.GetBytes(randomBytes);
    return Convert.ToBase64String(randomBytes);
}
```

- Usa `RandomNumberGenerator` (cryptographically secure) para crear 64 bytes aleatorios y codificarlos a Base64 como token.  
- **Recomendación:** en vez de almacenar ese Base64 en texto plano en Redis, almacena su hash (SHA256) para que si Redis se filtra no tengamos tokens utilizables. Para verificar, hashea el token enviado por el cliente y comparas.

### `RefreshTokensAsync()` — renovación / rotación

```csharp
public async Task<(bool Success, string? Message, string AccessToken, string RefreshToken, User? User)> RefreshTokensAsync()
{
    var context = _httpContextAccessor.HttpContext!;

    // 🔹 1. Leer refresh token desde cookies
    var refreshToken = context.Request.Cookies["refresh_token"];

    if (string.IsNullOrEmpty(refreshToken))
        return (false, "Refresh token no encontrado", string.Empty, string.Empty, null);

    // 🔹 2. Obtener el UserId asociado al refresh token en Redis
    var userId = await _refreshTokenStore.GetUserIdFromTokenAsync(refreshToken);
    if (string.IsNullOrEmpty(userId))
        return (false, "Refresh token inválido o expirado", string.Empty, string.Empty, null);

    // 🔹 3. Obtener usuario
    var user = await _userManager.FindByIdAsync(userId);
    if (user is null)
        return (false, "Usuario no encontrado", string.Empty, string.Empty, null);

    var usuario = user.Adapt<User>();

    // 🔹 4. Invalida el refresh token anterior
    await _refreshTokenStore.InvalidateTokenAsync(user.Id);

    // 🔹 5. Genera nuevos tokens y actualiza cookies
    var tokens = await GenerateTokensAsync(usuario);

    return (true, "Tokens renovados correctamente", tokens.AccessToken, tokens.RefreshToken, usuario);
}
```

Explicación paso a paso:

1. Lee `refresh_token` desde cookies (HttpOnly cookie).  
2. Consulta el `IRefreshTokenStore` para obtener el `userId` asociado al refresh token pasado. Esto valida que el token esté activo, no expirado y que efectivamente pertenezca a un usuario.  
3. Recupera el `User` desde `UserManager`.  
4. **Invalida** el refresh token anterior (rotación). Esta llamada debe eliminar cualquier rastro del token anterior. Si implementas rotación, asegúrate de invalidar la entrada adecuada en Redis (ver más abajo).  
5. Llama a `GenerateTokensAsync` para crear nuevos tokens, guardar el nuevo refresh token y actualizar cookies.

**Comentarios/Mejoras:**  
- Mejor implementar **rotación estricta**: el refresh token usado para pedir uno nuevo se invalida y se reemplaza por uno nuevo; si un atacante usa un refresh token antiguo, debe detectarse y forzar revocación de sesiones.  
- Registra eventos sospechosos (token replay).  
- Considera almacenar metadata: `deviceId`, `ip`, `userAgent` para auditar y permitir revocar tokens por dispositivo.


### `SetAuthCookies(...)`

```csharp
private void SetAuthCookies(string accessToken, string refreshToken)
{
    var context = _httpContextAccessor.HttpContext!;
    var accessCookieOptions = new CookieOptions
    {
        HttpOnly = true,
        Secure = true,
        SameSite = SameSiteMode.Strict,
        Expires = DateTime.UtcNow.AddMinutes(15)
    };

    var refreshCookieOptions = new CookieOptions
    {
        HttpOnly = true,
        Secure = true,
        SameSite = SameSiteMode.Strict,
        Expires = DateTime.UtcNow.AddDays(7)
    };

    context.Response.Cookies.Append("access_token", accessToken, accessCookieOptions);
    context.Response.Cookies.Append("refresh_token", refreshToken, refreshCookieOptions);
}
```

**Explicación:**  
- `HttpOnly = true`: impide acceso desde JS (protege contra XSS).  
- `Secure = true`: cookie enviada solo sobre HTTPS.  
- `SameSite = Strict`: ayuda a prevenir CSRF (puede romper algunos flujos cross-site; Lax es menos restrictivo).  
- `Expires`: define la caducidad de la cookie (debe alinearse con las expiraciones configuradas para los tokens).

**Recomendación:** Si usas cookies HttpOnly para refresh token, protege endpoints que usan cookies con mitigaciones CSRF (antiforgery tokens, SameSite + double submit cookie pattern).


---

## 7. Modelo `RefreshToken` e interfaz `IRefreshTokenStore`

### Modelo sugerido (separado)

```csharp
public class RefreshToken
{
    public string UserId { get; set; } = string.Empty;
    public string TokenHash { get; set; } = string.Empty; // almacenar hash (recomendado)
    public DateTime CreatedAt { get; set; }
    public DateTime ExpiresAt { get; set; }
    public string? DeviceId { get; set; }
    public string? UserAgent { get; set; }
    public string? IpAddress { get; set; }
}
```

> **Nota:** Guardar `TokenHash` en lugar del token completo es una buena práctica. Para verificar: hashea el token proporcionado por el cliente y comparas contra `TokenHash` almacenado.

### Interfaz `IRefreshTokenStore`

```csharp
public interface IRefreshTokenStore
{
    Task SaveTokenAsync(string userId, string refreshToken, TimeSpan expiration, string? deviceId = null);
    Task<RefreshToken?> GetTokenByUserIdAsync(string userId);
    Task<bool> ValidateTokenAsync(string userId, string refreshToken);
    Task InvalidateTokenAsyncByUserIdAsync(string userId);
    Task InvalidateTokenByTokenAsync(string refreshToken);
    Task<string?> GetUserIdFromTokenAsync(string refreshToken);
}
```

- Al exponer métodos por userId **y** por token, permites operaciones eficientes: invalidar todos los tokens de un usuario, validar por token, buscar userId a partir del token, etc.


---

## 8. `RefreshTokenStore` (Redis) — explicación y versión robusta

La implementación que compartiste tenía **inconsistencias en las claves** usadas (a veces `refresh_token:{refreshToken}`, otras `refresh_token:{userId}`). Abajo propongo una versión corregida y robusta que mantiene dos índices:

- `refresh_token:{token}` -> JSON con información completa (UserId, tokenHash, expiresAt...)  
- `user_refresh:{userId}` -> token (o lista de tokens por usuario) para poder invalidar fácilmente por usuario

### Implementación sugerida (simplificada) — ejemplo:

```csharp
public class RefreshTokenStore : IRefreshTokenStore
{
    private readonly IDatabase _db;

    public RefreshTokenStore(IConnectionMultiplexer redis)
    {
        _db = redis.GetDatabase();
    }

    public async Task SaveTokenAsync(string userId, string refreshToken, TimeSpan expiration, string? deviceId = null)
    {
        // Guardar el token **en dos claves** para consultas eficientes
        var tokenHash = ComputeSha256(refreshToken); // almacenar hash en la estructura para seguridad

        var data = new RefreshToken
        {
            UserId = userId,
            TokenHash = tokenHash,
            CreatedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.Add(expiration),
            DeviceId = deviceId
        };

        var keyToken = $"refresh_token:{tokenHash}";
        var keyUser = $"user_refresh:{userId}";

        var json = JsonSerializer.Serialize(data);
        // Guardamos por tokenHash y también por user (simple: almacenar el tokenHash como valor del user)
        await _db.StringSetAsync(keyToken, json, expiration);
        // Para simplificar: mantener una sola entrada por usuario (si quieres soportar múltiples dispositivos, usa un Set)
        await _db.StringSetAsync(keyUser, tokenHash, expiration);
    }

    public async Task<string?> GetUserIdFromTokenAsync(string refreshToken)
    {
        var tokenHash = ComputeSha256(refreshToken);
        var keyToken = $"refresh_token:{tokenHash}";
        var json = await _db.StringGetAsync(keyToken);
        if (json.IsNullOrEmpty) return null;
        var obj = JsonSerializer.Deserialize<RefreshToken>(json!);
        if (obj == null || obj.ExpiresAt <= DateTime.UtcNow) return null;
        return obj.UserId;
    }

    public async Task<RefreshToken?> GetTokenByUserIdAsync(string userId)
    {
        var keyUser = $"user_refresh:{userId}";
        var tokenHash = await _db.StringGetAsync(keyUser);
        if (tokenHash.IsNullOrEmpty) return null;
        var keyToken = $"refresh_token:{tokenHash}";
        var json = await _db.StringGetAsync(keyToken);
        return json.IsNullOrEmpty ? null : JsonSerializer.Deserialize<RefreshToken>(json!);
    }

    public async Task<bool> ValidateTokenAsync(string userId, string refreshToken)
    {
        var tokenHash = ComputeSha256(refreshToken);
        var stored = await GetTokenByUserIdAsync(userId);
        if (stored == null) return false;
        return stored.TokenHash == tokenHash && stored.ExpiresAt > DateTime.UtcNow;
    }

    public async Task InvalidateTokenAsyncByUserIdAsync(string userId)
    {
        var keyUser = $"user_refresh:{userId}";
        var tokenHash = await _db.StringGetAsync(keyUser);
        if (!tokenHash.IsNullOrEmpty)
        {
            var keyToken = $"refresh_token:{tokenHash}";
            await _db.KeyDeleteAsync(keyToken);
            await _db.KeyDeleteAsync(keyUser);
        }
    }

    public async Task InvalidateTokenByTokenAsync(string refreshToken)
    {
        var tokenHash = ComputeSha256(refreshToken);
        var keyToken = $"refresh_token:{tokenHash}";
        var json = await _db.StringGetAsync(keyToken);
        if (json.IsNullOrEmpty) return;
        var obj = JsonSerializer.Deserialize<RefreshToken>(json!);
        if (obj != null)
        {
            var keyUser = $"user_refresh:{obj.UserId}";
            await _db.KeyDeleteAsync(keyToken);
            await _db.KeyDeleteAsync(keyUser);
        }
    }

    private static string ComputeSha256(string input)
    {
        using var sha = System.Security.Cryptography.SHA256.Create();
        var bytes = sha.ComputeHash(System.Text.Encoding.UTF8.GetBytes(input));
        return Convert.ToBase64String(bytes);
    }
}
```

### Por qué esta versión es mejor

- **Seguridad:** almacenamos hash del token (`TokenHash`) en Redis en vez del token en texto plano. Si Redis se filtra, el atacante no podrá usar directamente el hash como token (porque se compara el hash del token enviado por el cliente).  
- **Consistencia:** las claves siguen un esquema claro (`refresh_token:{tokenHash}` y `user_refresh:{userId}`).  
- **Flexibilidad:** soporta invalidación por token o por usuario. Para múltiples dispositivos, cambia `user_refresh:{userId}` de string a Redis Set o Hash para almacenar múltiples tokenHash por usuario.


---

## 9. Inyección de Redis en `Program.cs`

```csharp
// Redis connection
services.AddSingleton<IConnectionMultiplexer>(sp =>
{
    var configuration = sp.GetRequiredService<IConfiguration>();
    var redisConnection = configuration.GetConnectionString("Redis");
    return ConnectionMultiplexer.Connect(redisConnection);
});
```

**Notas:**  
- `ConnectionMultiplexer` debe ser singleton por proceso. Evita crear muchas conexiones.  
- Maneja excepciones de conexión al iniciar, y considera usar retry/backoff.  
- Para entornos productivos usa la URL con autenticación y opciones TLS si corresponde.

---

## 10. Buenas prácticas de seguridad y operativas

- **Almacena hash en Redis:** nunca guardes refresh tokens en texto plano. Usa SHA256/HMAC y compara hashes.  
- **Rotación de refresh tokens:** cada vez que calles `/refresh`, emite un nuevo refresh token y anula el anterior.  
- **Short-lived access token:** 10–30 minutos para el access token.  
- **Refresh token expirations razonables:** 7–30 días típicamente; invalidar tokens inactivos.  
- **Protege cookies frente a CSRF:** usar `SameSite=Strict/Lax`, implementar token anti-forgery para endpoints que acepten cookies. Alternativamente usa `Authorization: Bearer` para el access token almacenado en memoria y un refresh token HttpOnly cookie para renovación.  
- **Detección de replay:** mantén `jti` y audita reutilización de refresh tokens (intentos de usar un token ya rotado).  
- **Múltiples dispositivos:** guarda metadata por token (`deviceId`, `userAgent`, `ip`) y permite revocar por dispositivo.  
- **Key rotation:** si rotas la clave de firma (Key), diseña estrategia para aceptar tokens firmados con claves antiguas por un breve periodo o forzar re-login. Considera RS256 con JWKS para rotación más segura.  
- **Logging / auditoría:** registra eventos de login, refresh, invalidación.  
- **Alta disponibilidad Redis:** no confiar en un único nodo en producción; usar Sentinel/Cluster y backups.  
- **Rate limiting:** protege endpoints `/login` y `/refresh` para evitar abuso.  

---

## 11. Resumen y cierre

Con el patrón JWT + Refresh Tokens + Redis puedes construir una autenticación escalable, segura y adecuada para arquitecturas distribuidas. Las claves del éxito son:

- Mantener el access token de corta duración.  
- Implementar rotación y hashing de refresh tokens.  
- Usar Redis correctamente (índices por token y por usuario) para permitir validaciones y revocaciones rápidas.  
- Aplicar mitigaciones de seguridad adicionales (CSRF, XSS, rate-limiting, logging).

---

**Instructor:** Juan Carlos De La Cruz Chinga  
**Documento:** Autenticación en .NET 9 — JWT, Refresh Tokens y Redis (detallado)

