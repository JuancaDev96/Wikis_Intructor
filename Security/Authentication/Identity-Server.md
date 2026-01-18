# Identity Server + ASPNet Core Identity

> **Documento:** Guía detallada y documentación de configuración y comportamiento de IdentityServer + ASP.NET Identity + API cliente (JwtBearer).

*Instructor: Juan Carlos De La Cruz*

---

## Índice

1. Introducción
2. Requisitos previos
3. Explicación línea a línea (Program.cs - IdentityServer / Identity)

   * Configurar EF Core
   * Configurar Identity
   * Integrar IdentityServer con ASP.NET Identity
   * app.UseIdentityServer()
4. `ProfileService` — explicación completa

   * Constructor
   * `GetProfileDataAsync`
   * `IsActiveAsync`
   * Notas sobre `JwtClaimTypes`, `context.Subject` y `context.IssuedClaims`
5. `Config` — IdentityResources, ApiScopes y Clients

   * IdentityResources explicados (openid, profile, email, roles)
   * ApiScope: qué es y cómo usarlo
   * Client explicado (Resource Owner Password Grant)
6. Código cliente (API que valida tokens): explicación línea a línea

   * AddAuthentication / AddJwtBearer
   * TokenValidationParameters.ValidateAudience = false
   * AddAuthorization
   * Middleware pipeline: UseAuthentication, UseAuthorization, MapOpenApi, UseHttpsRedirection
   * Endpoint protegido con RequireAuthorization
7. Pasos operativos y comandos

   * Migraciones EF
   * Ejecutar y probar (Postman / curl)
8. Buenas prácticas y consideraciones de seguridad

   * Entorno producción: signing credentials, HTTPS, secret handling
   * Tokens: lifetime, refresh tokens, revocation
   * Roles y claims mínimos en el token
9. Ejemplos rápidos (appsettings, requests)
10. FAQ / Problemas comunes y soluciones
11. Referencias y siguientes pasos

---

## 1. Introducción

Este documento detalla **cada línea**, **cada servicio** y **cada método** de la configuración mínima para integrar **EF Core (PostgreSQL)** + **ASP.NET Identity** + **Duende IdentityServer** y una **API cliente** que valida tokens via JwtBearer. Está pensado para que quede claro qué hace cada llamada, cuándo se ejecuta y por qué es importante.

---

## 2. Requisitos previos

* .NET 8 (o versión compatible)
* Paquetes NuGet instalados (ejemplos):

  * Microsoft.EntityFrameworkCore
  * Npgsql.EntityFrameworkCore.PostgreSQL
  * Microsoft.AspNetCore.Identity.EntityFrameworkCore
  * Duende.IdentityServer
  * Duende.IdentityServer.AspNetIdentity
  * Microsoft.AspNetCore.Authentication.JwtBearer
* Base de datos PostgreSQL disponible (puede ser Docker)
* Migrations habilitadas (`dotnet ef` tools)

---

## 3. Explicación línea a línea (Program.cs - IdentityServer / Identity)

A continuación verás el fragmento de código original y la explicación detallada de cada elemento.

```csharp
// Configurar EF Core
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));
```

### `AddDbContext<ApplicationDbContext>(...)`

* **Qué hace:** registra el `ApplicationDbContext` en el contenedor de dependencias (DI container) con *scoped lifetime* por defecto.
* **Por qué:** EF Core necesita un `DbContext` para realizar operaciones con la base de datos (migraciones, queries, inserts, updates).
* **`ApplicationDbContext`:** debe heredar de `IdentityDbContext<TUser>` (cuando usamos ASP.NET Identity). Contiene las tablas de Identity (Users, Roles, Claims, UserRoles, etc.).
* **Opciones:** `UseNpgsql(...)` indica que EF Core utilizará el proveedor Npgsql (PostgreSQL). Puedes sustituir por `UseSqlServer(...)` u otro proveedor.
* **Connection string:** `builder.Configuration.GetConnectionString("DefaultConnection")` lee la cadena desde `appsettings.json` o variables de entorno.

---

```csharp
// Configurar Identity
builder.Services.AddIdentity<ApplicationUser, IdentityRole>()
    .AddEntityFrameworkStores<ApplicationDbContext>()
    .AddDefaultTokenProviders();
```

### `AddIdentity<TUser, TRole>()`

* **Qué hace:** configura los servicios de ASP.NET Identity (UserManager, RoleManager, SignInManager, PasswordHasher, PasswordValidators, etc.).
* **`ApplicationUser`:** clase que extiende `IdentityUser` donde puedes añadir propiedades personalizadas (NombreCompleto, TelefonoAlternativo, etc.).
* **`IdentityRole`:** tipo por defecto para roles. Puedes crear una clase personalizada si necesitas más campos.

### `.AddEntityFrameworkStores<ApplicationDbContext>()`

* **Qué hace:** registra la implementación que utiliza EF Core para persistir usuarios/roles/claims en las tablas que crea `IdentityDbContext`.
* **Por qué:** sin esto, Identity no sabrá cómo almacenar usuarios en la DB.

### `.AddDefaultTokenProviders()`

* **Qué hace:** añade providers para tokens que Identity usa para acciones como confirmación de email, recupero de contraseña, cambio de email, autenticación de dos factores (2FA).
* **Consecuencia:** permite utilizar métodos como `GenerateEmailConfirmationTokenAsync` o `GeneratePasswordResetTokenAsync`.

---

```csharp
// Integrar IdentityServer con ASP.NET Identity
builder.Services.AddIdentityServer()
    .AddAspNetIdentity<ApplicationUser>()
    .AddInMemoryIdentityResources(Config.IdentityResources)
    .AddInMemoryApiScopes(Config.ApiScopes)
    .AddInMemoryClients(Config.Clients)
    .AddProfileService<ProfileService>()
    .AddDeveloperSigningCredential();
```

### `AddIdentityServer()`

* **Qué hace:** registra el pipeline y servicios centrales de IdentityServer en DI. Proporciona endpoints OIDC/OAuth2 (ej. `/connect/token`, `/connect/authorize`, `/.well-known/openid-configuration`).

### `.AddAspNetIdentity<ApplicationUser>()`

* **Qué hace:** integra ASP.NET Identity como la tienda de usuarios para IdentityServer. Esto permite que los flujos interactivos utilicen `UserManager` y SignInManager` para autenticación y gestión de cuentas.
* **Importancia:** habilita la emisión de tokens basada en usuarios reales almacenados por Identity.

### `.AddInMemoryIdentityResources(Config.IdentityResources)`

* **Qué hace:** registra en memoria los *IdentityResources* (scopes relacionados con la identidad del usuario, como `openid`, `profile`, `email`, etc.).
* **Cuándo usar:** bueno para desarrollo y pruebas. En producción podrías mantenerlos en DB, pero la mayoría usa configuración in-memory o desde config files.

### `.AddInMemoryApiScopes(Config.ApiScopes)`

* **Qué hace:** registra en memoria los *ApiScopes* (los recursos protegidos a los que los clientes pueden pedir acceso con `scope`).
* **Ej:** `api1`.

### `.AddInMemoryClients(Config.Clients)`

* **Qué hace:** registra los `Client` (aplicaciones que pedirán tokens) en memoria.
* **Nota:** en producción suele usarse una store persistente, pero in-memory es más simple para PoC y pruebas.

### `.AddProfileService<ProfileService>()`

* **Qué hace:** inyecta un servicio personalizado que implementa `IProfileService`. Este servicio será llamado por IdentityServer cada vez que necesite construir los claims que se incluirán en los tokens (id token y access token cuando se emitan claims).
* **Por qué personalizar:** para incluir claims desde tu base de datos (NombreCompleto, roles, permisos, etc.), y controlar qué claims se emiten en función del `client` o del `scope` solicitado.

### `.AddDeveloperSigningCredential()`

* **Qué hace:** crea una clave temporal para firmar tokens (X509 o clave RSA generada localmente) — **solo para desarrollo**.
* **Importante:** en producción debes usar un certificado real y `AddSigningCredential(...)` con un certificado seguro.

---

```csharp
var app = builder.Build();

app.UseIdentityServer();
```

### `app.UseIdentityServer()`

* **Qué hace:** registra en el pipeline de middleware las rutas y el middleware de IdentityServer. Esto habilita los endpoints OIDC/OAuth2 y la lógica de emisión/validación de tokens.
* **Orden en pipeline:** normalmente `UseIdentityServer()` va antes de `UseAuthentication()`/`UseAuthorization()` en apps que exponen UI y APIs junto con IdentityServer.

---

## 4. `ProfileService` — explicación completa

A continuación se documenta la clase `ProfileService` que implementa `Duende.IdentityServer.Services.IProfileService`.

```csharp
public class ProfileService : Duende.IdentityServer.Services.IProfileService
{
    private readonly UserManager<ApplicationUser> _userManager;

    public ProfileService(UserManager<ApplicationUser> userManager)
    {
        _userManager = userManager;
    }

    // Se ejecuta cuando se construye el token
    public async Task GetProfileDataAsync(ProfileDataRequestContext context)
    {
        var user = await _userManager.GetUserAsync(context.Subject);
        if (user == null) return;

        var userClaims = new List<Claim>
    {
        new Claim(JwtClaimTypes.Subject, user.Id),
        new Claim(JwtClaimTypes.Email, user.Email ?? ""),
        new Claim(JwtClaimTypes.Name, user.NombreCompleto ?? user.UserName ?? ""),
    };

        // Agregar roles del usuario
        var roles = await _userManager.GetRolesAsync(user);
        foreach (var role in roles)
        {
            userClaims.Add(new Claim(JwtClaimTypes.Role, role));
        }

        // Agregar los claims personalizados al contexto
        context.IssuedClaims.AddRange(userClaims);
    }

    // Se ejecuta para verificar si el usuario sigue activo
    public async Task IsActiveAsync(IsActiveContext context)
    {
        var user = await _userManager.GetUserAsync(context.Subject);
        context.IsActive = user != null;
    }
}
```

### ¿Qué es `IProfileService`?

* Es una interfaz que IdentityServer usa para obtener información del usuario cuando construye tokens y para determinar si el usuario está activo.
* Se compone de dos métodos obligatorios: `GetProfileDataAsync` y `IsActiveAsync`.

---

### Constructor

```csharp
private readonly UserManager<ApplicationUser> _userManager;

public ProfileService(UserManager<ApplicationUser> userManager)
{
    _userManager = userManager;
}
```

* **UserManager<ApplicationUser>:** servicio de ASP.NET Identity que permite acceder a usuarios, claims, roles, cambiar contraseña, etc.
* **Por qué inyectarlo:** necesitamos acceder a la data del usuario almacenada en la DB.

---

### `GetProfileDataAsync(ProfileDataRequestContext context)`

* **Cuándo se ejecuta:** cada vez que IdentityServer necesita construir los claims que serán incluidos en un token (ID token o access token con claims).
* **`context.Subject`:** representa al usuario autenticado (un `ClaimsPrincipal` o `Sub` que identifica al usuario). `UserManager.GetUserAsync(context.Subject)` extrae el objeto `ApplicationUser` relacionado.
* **Construcción de claims:** en el ejemplo se crean claims estándar (`sub`, `email`, `name`) y se agregan roles.
* **`JwtClaimTypes`:** proviene de `IdentityModel` y define constantes como `Subject` (`sub`), `Email`, `Name`, `Role`.
* **`context.IssuedClaims`:** colección donde debes agregar los claims que quieres que aparezcan en el token emitido. IdentityServer filtrará los claims según el scope solicitado y las políticas.

#### Consideraciones prácticas:

* **Filtrar claims por `context.RequestedClaimTypes`** cuando quieras emitir solo los claims solicitados por el client/scope.
* **Evitar sobrecargar el token** con claims innecesarios (por tamaño y privacidad).
* **Evitar incluir información sensible** (contraseñas, tokens, info PII innecesaria).

---

### `IsActiveAsync(IsActiveContext context)`

* **Cuándo se ejecuta:** IdentityServer consulta si el usuario está activo (por ejemplo, si fue borrado, desactivado o bloqueado) antes de emitir un token o al validar sesiones.
* **Implementación mínima:** comprobar existencia del usuario. Puedes ampliar para comprobar `LockoutEnd`, `EmailConfirmed`, `Disabled` flag, etc.
* **`context.IsActive`** se debe setear a `true` o `false` según la lógica de negocio.

---

## 5. `Config` — IdentityResources, ApiScopes y Clients

Documento el `Config` que defines:

```csharp
public static class Config
{
    public static IEnumerable<IdentityResource> IdentityResources =>
        new IdentityResource[]
        {
            new IdentityResources.OpenId(),
            new IdentityResources.Profile(),
            new IdentityResources.Email(),
            new IdentityResource("roles", new [] { "role" })
        };

    public static IEnumerable<ApiScope> ApiScopes =>
        new ApiScope[]
        {
            new ApiScope("api1", "My API #1")
        };

    public static IEnumerable<Client> Clients =>
        new Client[]
        {
            new Client
            {
                ClientId = "ro.client",
                AllowedGrantTypes = GrantTypes.ResourceOwnerPassword,
                ClientSecrets = { new Secret("secret".Sha256()) },
                AllowedScopes = { "api1", "openid", "profile", "email", "roles" }
            }
        };
}
```

### `IdentityResources` (identity scopes)

* **`OpenId()`**: *obligatorio* cuando se usa OIDC. Expone el `sub` claim (identificador único del usuario). Sin `openid` no es OIDC.
* **`Profile()`**: incluye claims comunes como `name`, `family_name`, `given_name`, `preferred_username`, `picture`, etc.
* **`Email()`**: incluye `email` y `email_verified`.
* **`new IdentityResource("roles", new [] { "role" })`**: recurso personalizado para exponer roles. Se define así para que los clientes puedan solicitar el scope `roles` y obtener la lista de roles como claim `role`.

#### Notas:

* IdentityResources controlan qué *claims de identidad* pueden solicitarse vía scopes. El `ProfileService` decide qué claims realmente emitir.

---

### `ApiScopes`

* **Qué es:** representa un permiso a consumir un recurso protegido.
* **Ejemplo:** `api1` representa tu API backend. Un client solicita `scope=api1` para obtener un token usable en la API.
* **Diferencia con API Resource (legacy):** `ApiScope` es la unidad de autorización usada en OAuth2. Puedes mapear scopes a recursos o a permisos más finos.

---

### `Clients` (Resource Owner Password example)

* **`ClientId`**: identificador de la aplicación cliente.
* **`AllowedGrantTypes = GrantTypes.ResourceOwnerPassword`**: permite el flujo del propietario del recurso (password) en el que la app envía `username` y `password` al token endpoint. *Desaconsejado* para aplicaciones públicas y móviles; aceptable en trusted first-party apps.
* **`ClientSecrets`**: secreto del cliente (hashed con `Sha256()`). Importante para flujos confidenciales.
* **`AllowedScopes`**: scopes que el cliente puede solicitar. Debe incluir `openid` si necesita identidad.

---

## 6. Código cliente (API que valida tokens): explicación línea a línea

Fragmento original:

```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://localhost:7117"; // tu IdentityServer
        options.TokenValidationParameters = new()
        {
            ValidateAudience = false
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

app.UseAuthentication();

app.UseAuthorization();

app.MapGet("/secure", () => "Acceso autorizado").RequireAuthorization();
```

### `AddAuthentication("Bearer")`

* **Qué hace:** configura la autenticación por defecto para la app. `"Bearer"` es el esquema por defecto, útil cuando usamos tokens JWT en `Authorization: Bearer <token>`.

### `.AddJwtBearer("Bearer", options => { ... })`

* **Qué hace:** añade soporte para validar JWTs. El nombre `"Bearer"` es el esquema utilizado.
* **`options.Authority`**: URL base del IdentityServer (issuer). La librería usará esta URL para descubrir la metadata OIDC (ej. `/.well-known/openid-configuration`) y obtener la clave pública para validar la firma del token.
* **`TokenValidationParameters.ValidateAudience = false`**: indica que la validación de la audiencia (`aud`) será desactivada. Esto es útil cuando no se usa audiencia estricta o cuando múltiples APIs comparten un issuer. Sin embargo, **para mayor seguridad** conviene validar `aud` y/o configurar `ValidAudience`.

### `AddAuthorization()`

* **Qué hace:** agrega servicios de autorización (policies, roles). Necesario para usar `[Authorize]`, `RequireAuthorization()` y políticas por rol/claim.

### Middleware

* **`app.UseHttpsRedirection()`**: redirige HTTP→HTTPS (recomendado en producción).
* **`app.UseAuthentication()`**: añade el middleware que extrae y valida credenciales (ej. el token Bearer) y crea la identidad (`ClaimsPrincipal`) en `HttpContext.User`.
* **`app.UseAuthorization()`**: aplica políticas de autorización basadas en la identidad creada por el middleware de autenticación.
* **Orden importante:** `UseAuthentication()` debe ir antes de `UseAuthorization()`.

### `MapGet("/secure", ...).RequireAuthorization()`

* **Qué hace:** define un endpoint protegido; la petición debe incluir un token válido. Si la validación falla, el cliente recibe 401/403.

---

## 7. Pasos operativos y comandos

### EF Migrations (crear DB y tablas Identity)

1. Agrega las herramientas EF: `dotnet tool install --global dotnet-ef` (si no lo tienes)
2. Añade paquete `Microsoft.EntityFrameworkCore.Design` si falta.
3. Crear migración:

   ```bash
   dotnet ef migrations add InitialIdentity
   dotnet ef database update
   ```

Esto creará tablas como `AspNetUsers`, `AspNetRoles`, `AspNetUserRoles`, `AspNetUserClaims`, etc.

### Levantar Docker (Postgres)

```bash
docker compose up -d
```

### Probar token (Resource Owner Password)

```bash
POST https://localhost:5001/connect/token
Content-Type: application/x-www-form-urlencoded

client_id=ro.client&client_secret=secret&grant_type=password&username=admin@demo.com&password=Admin123$&scope=api1 openid profile email roles
```

Respuesta: JSON con `access_token` y `id_token` (si se solicitó `openid`).

### Llamar API protegida

```bash
GET https://localhost:5003/secure
Authorization: Bearer <access_token>
```

---

## 8. Buenas prácticas y consideraciones de seguridad

1. **Signing credentials en producción:** usa certificados X509 o Azure Key Vault / AWS KMS para firmar tokens. No uses `AddDeveloperSigningCredential()` en producción.
2. **HTTPS obligatorio:** siempre exponer el IdentityServer sobre HTTPS.
3. **Client secrets:** guarda secrets en un secreto manager o vault, no en appsettings en texto plano.
4. **Grant types seguros:** evita `ResourceOwnerPassword` en apps públicas; prefer `Authorization Code + PKCE` para SPAs y móviles.
5. **Scopes y claims mínimos:** emite solo los claims que la app necesita. Minimiza la PII en el token.
6. **Refresh tokens:** controla su uso y revocación. Implementa revocation endpoints y blacklists si necesario.
7. **Control de sesiones y logout:** configura `front-channel`/`back-channel` logout si tu sistema lo requiere.
8. **Rate limiting / brute force:** aplica límites de intentos de login y bloqueo por IP/usuario.

---

## 9. Ejemplos rápidos

### appsettings.json (fragmento)

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=identitydb;Username=postgres;Password=postgres123"
  },
  "IdentityServer": {
    "Authority": "https://localhost:7117"
  }
}
```

### cURL para token

```bash
curl -X POST https://localhost:5001/connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=ro.client&client_secret=secret&grant_type=password&username=admin@demo.com&password=Admin123$&scope=api1 openid profile email roles"
```

---

## 10. FAQ / Problemas comunes y soluciones

* **Token inválido / firma no válida:** revisa `Authority`, la discovery URL (`/.well-known/openid-configuration`) y que el cliente use el issuer correcto.
* **Claims no aparecen en el token:** confirma que `ProfileService` añade los claims y que el cliente solicitó los scopes apropiados (`openid profile email roles`). Revisa `context.RequestedClaimTypes` si filtras.
* **Roles no enviados:** asegúrate de haber creado roles en la DB y de que `GetRolesAsync` devuelva valores.
* **DB migrations no aplican:** revisa la cadena de conexión y privilegios del usuario DB.

---

## 11. Referencias y siguientes pasos

* Duende IdentityServer docs
* ASP.NET Core Identity docs
* Npgsql & EF Core provider docs
