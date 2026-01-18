# Configurando KeyCloak

**Instructor:** Juan Carlos De La Cruz Ch.

---

> **Resumen:** Este manual detalla la instalación, configuración y uso de Keycloak (v24+), la creación de realms, clients, users, client scopes, mapeo de audiencias (aud), obtención de tokens (client_credentials y password), y la integración paso a paso con una API en .NET 9. Está pensado como guía práctica y referencia para clases técnicas.

---

## Índice

1. Introducción a Keycloak
2. Arquitectura y conceptos clave
3. Preparar el entorno: levantar Keycloak con Docker
4. Crear un Realm
5. Crear Clients (APIs y/o aplicaciones)
6. Usuarios y asignación de roles
7. Client Scopes y mapeo de Audience (aud)
8. Obtener tokens (client_credentials y password)
9. Cuándo usar cada flujo
10. Integración: asegurar una API en .NET 9 (código y configuración)
11. Pruebas con Postman y Swagger
12. Buenas prácticas y recomendaciones
13. Anexos (comandos y fragmentos útiles)

---

## 1. Introducción a Keycloak

Keycloak es un Identity Provider (IdP) Open Source que implementa varios estándares de federación y autorización como OAuth2, OpenID Connect (OIDC) y SAML. Facilita la autenticación, autorización, federación de identidades, gestión de usuarios y emisión de tokens (JWT, SAML assertions).

Keycloak es ampliamente usado para: autenticación de usuarios en aplicaciones web/mobile, protección de APIs mediante JWT y federación con proveedores externos (Google, Azure AD, etc.).

Ventajas principales:

* Interfaz de administración para gestionar realms, clients, roles y usuarios.
* Integración con múltiples proveedores de identidad.
* Token introspection, revocación, refresh tokens y sesiones.
* Soporta flujos estándar de OAuth2/OIDC.

---

## 2. Arquitectura y conceptos clave

* **Realm:** Contenedor lógico de seguridad. Un realm agrupa usuarios, aplicaciones (clients), roles y políticas. Normalmente se usa un realm por entorno o por dominio de seguridad.
* **Client:** Representa una aplicación o recurso que usa Keycloak. Puede ser una aplicación frontend, una API (resource server) o un servicio que solicita tokens. En Keycloak v24+ se selecciona *OpenID Connect* o *SAML* y luego se habilita la opción de "Client authentication" para convertirlo en confidential.
* **User:** Identidad humana (o sistema) que se autentica en el realm.
* **Role:** Permisos o grupos lógicos asignados a usuarios o a clientes (service accounts).
* **Client Scope:** Conjunto de mappers y settings que determinan qué claims entran al token.
* **Audience (aud):** Claim del JWT que indica el destinatario del token (qué recurso o client está autorizado a consumirlo).
* **Service Account:** Cuenta asociada a un client (útil para `client_credentials`).

---

## 3. Preparar el entorno: levantar Keycloak con Docker

Archivo `docker-compose.yml` mínimo para desarrollo:

```yaml
version: '3.8'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    command: start-dev
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
    ports:
      - 8080:8080
```

Comandos:

```bash
docker compose up -d
# Ver logs si lo necesitas
docker compose logs -f keycloak
```

Accede al admin: `http://localhost:8080` y entra a `http://localhost:8080/admin`.

> **Nota de seguridad:** `start-dev` es para desarrollo. En producción usa el modo productivo con configuración de base de datos persistente, HTTPS, backups y alta disponibilidad.

---

## 4. Crear un Realm

1. Login al admin con `admin/admin` (según `docker-compose`).
2. En la esquina superior izquierda, click en **Master** (o en el realm actual) y luego **Create realm**.
3. Nombre: `mi_realm` (o el que prefieras).
4. Guardar.

Un realm es tu contenedor de seguridad: separa configuraciones y usuarios entre proyectos o ambientes.

---

## 5. Crear Clients (APIs y/o aplicaciones)

### 5.1 Crear un client para una API (ej. `orders-api`)

1. En el menú lateral → **Clients** → **Create client**.
2. Selecciona **Client type**: `OpenID Connect`.
3. Client ID: `orders-api`.
4. Click **Next** / **Save**.

### 5.2 Configuraciones importantes tras crear el client

* **Client authentication**: ON → hace que el client sea *confidential* (puede tener client secret y usar `client_credentials`).
* **Service accounts enabled**: ON → habilita la cuenta de servicio (útil para `client_credentials`).
* **Authorization**: ON → si usarás el Authorization Services (policies, permissions).
* **Standard Flow Enabled**: ON → para `authorization_code` (usualmente usado por Apps con usuario).
* **Direct Access Grants Enabled**: ON → permite `password` grant (Resource Owner Password Credentials, solo usar en casos controlados y legacy).

Guarda los cambios.

### 5.3 Obtener el Client Secret

* En la pestaña **Credentials** del cliente encontrarás el `Client secret`. Copia ese valor: lo necesitarás para `client_credentials`.

---

## 6. Usuarios y asignación de roles

### 6.1 Crear roles

* En el menú lateral → **Roles** → **Add Role**.
* Ejemplos: `admin`, `user`, `api_consumer`.

Los roles pueden ser *realm roles*. También existen *client roles* (roles definidos por cliente).

### 6.2 Crear un usuario

1. Menú → **Users** → **Add user**.
2. Username: `juan`.
3. Guarda.
4. En la pestaña **Credentials** agrega una contraseña y desactiva `Temporary`.
5. En la pestaña **Role Mappings** asigna roles (ej. `user`).

### 6.3 Asignar roles a la service account (client)

Si tu client usa `client_credentials` y necesitas que el token tenga roles, asigna roles al `Service Account` del client:

1. En el cliente → pestaña **Service Account Roles**.
2. Asignar roles (ej. `api_consumer`) a la cuenta de servicio.

---

## 7. Client Scopes y mapeo de Audience (aud)

### 7.1 ¿Por qué mapear la audiencia?

El claim `aud` indica para qué recurso fue emitido el token. Si tu API valida `aud`, debes asegurar que Keycloak incluya el identificador correcto del recurso (client) en el token. Esto evita que tokens emitidos para otro recurso sean válidos en tu API.

### 7.2 Crear un Client Scope para audiencia personalizada

1. Menú → **Client Scopes** → **Create client scope**.
2. Name: `api_audience`.
3. Type: `Default` o `Optional` según prefieras (Default si quieres que se añada a todos los tokens por defecto a ese client).
4. Guardar.

### 7.3 Crear un Mapper de tipo Audience

Dentro del `api_audience`:

1. Pestaña **Mappers** → **Create**.
2. Name: `aud-mapper`.
3. Mapper Type: `Audience`
4. In "Included Client Audience", selecciona el client protegido (ej. `orders-api`).
5. Guarda.

### 7.4 Asignar el Client Scope al Client

1. Ve al client (`orders-api`).
2. Pestaña **Client Scopes** → **Assigned Default Client Scopes** → **Add client scope**.
3. Selecciona `api_audience`.

Ahora los tokens emitidos para ese client incluirán la audiencia con el nombre que configuraste.

---

## 8. Obtener tokens (client_credentials y password)

### 8.1 Token con `client_credentials` (máquinas/servicios)

**Uso:** comunicación entre servicios o llamadas server-to-server donde no hay usuario.

**Request:**

```
POST http://localhost:8080/realms/mi_realm/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=orders-api
client_secret=<CLIENT_SECRET>
grant_type=client_credentials
```

**Respuesta esperada:**

```json
{
  "access_token": "eyJ...",
  "expires_in": 300,
  "token_type": "Bearer"
}
```

**Asegúrate** de que el client tenga `Service accounts enabled` y roles asignados si quieres claims/roles en el token.

### 8.2 Token con `password` (resource owner password credentials)

**Uso:** flujos legacy o pruebas internas; NO recomendado para aplicaciones públicas ni móviles. Mejor usar `authorization_code` con PKCE.

**Request:**

```
POST http://localhost:8080/realms/mi_realm/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=mi_app
client_secret=<CLIENT_SECRET>   # si el client es confidential
username=juan
password=secret123
grant_type=password
```

**Respuesta:** similar a la de `client_credentials` pero el token contendrá claims del usuario (sub, preferred_username, roles, etc.).

### 8.3 Token con `authorization_code` (recomendado para apps con usuario)

Para flujos con usuario en aplicaciones web o móviles se recomienda `authorization_code` con PKCE (especialmente para aplicaciones públicas). No se detalla paso a paso aquí, pero el flujo implica:

1. Redirigir al usuario a Keycloak `/protocol/openid-connect/auth`.
2. Usuario se autentica y Keycloak redirige de vuelta con un `code`.
3. The app exchanges the `code` for tokens at `/protocol/openid-connect/token`.

---

## 9. ¿En qué caso usar cada flujo?

* **client_credentials**: para comunicación máquina a máquina (services, cron jobs, backends que llaman otras APIs). No hay usuario. Token incluye roles asignados a la cuenta de servicio.
* **password**: legacy; solo en entornos controlados (por ejemplo scripts internos). Evitar en producción para aplicaciones nuevas.
* **authorization_code (+ PKCE)**: recomendado para aplicaciones con usuario (SPA, móviles, web apps).
* **implicit**: obsoleto; evitar.

---

## 10. Integración: asegurar una API en .NET 9

A continuación se describe un ejemplo completo para proteger una API en .NET 9 que valide issuer, audience y use roles.

### 10.1 Crear el proyecto

```bash
dotnet new webapi -n KeycloakProtectedApi
cd KeycloakProtectedApi
```

### 10.2 Agregar paquete

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

### 10.3 `Program.cs` (configuración completa)

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// Configuración JWT/Bearer con Keycloak
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.Authority = "http://localhost:8080/realms/mi_realm"; // Issuer URL
    options.RequireHttpsMetadata = false; // SOLO para desarrollo

    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = "http://localhost:8080/realms/mi_realm",

        ValidateAudience = true,
        ValidAudience = "orders-api", // Ajusta según tu client_id o audiencia mapeada

        ValidateLifetime = true,
    };

    // Opcional: mapear claim roles si vienen en realm_access.roles o resource_access
    options.Events = new JwtBearerEvents
    {
        OnTokenValidated = ctx =>
        {
            // Aquí puedes transformar claims si lo necesitas
            return Task.CompletedTask;
        }
    };
});

builder.Services.AddAuthorization(options =>
{
    // Opcional: políticas basadas en roles
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("admin"));
});

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.AddSecurityDefinition("Bearer", new Microsoft.OpenApi.Models.OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme. Example: 'Bearer {token}'",
        Name = "Authorization",
        In = Microsoft.OpenApi.Models.ParameterLocation.Header,
        Type = Microsoft.OpenApi.Models.SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    c.AddSecurityRequirement(new Microsoft.OpenApi.Models.OpenApiSecurityRequirement
    {
        {
            new Microsoft.OpenApi.Models.OpenApiSecurityScheme
            {
                Reference = new Microsoft.OpenApi.Models.OpenApiReference
                {
                    Type = Microsoft.OpenApi.Models.ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            new string[] {}
        }
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

### 10.4 Controlador de ejemplo `Controllers/SecureController.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace KeycloakProtectedApi.Controllers;

[ApiController]
[Route("api/[controller]")]
public class SecureController : ControllerBase
{
    [HttpGet("public")]
    public IActionResult PublicEndpoint() => Ok("Endpoint público - no requiere token");

    [Authorize]
    [HttpGet("protected")]
    public IActionResult ProtectedEndpoint()
    {
        var user = User.Identity?.Name ?? "Anonymous";
        var roles = User.Claims.Where(c => c.Type.EndsWith("role") || c.Type == "role" || c.Type == "realm_access").Select(c => c.Value).ToList();
        return Ok(new { message = "Acceso autorizado", user, roles });
    }

    [Authorize(Roles = "admin")]
    [HttpGet("admin")]
    public IActionResult AdminEndpoint() => Ok("Solo admins");
}
```

> **Nota sobre roles/claims:** Keycloak normalmente emite roles bajo `realm_access.roles` y `resource_access.{client}.roles`. Si necesitas que `.NET` reconozca roles directamente en `User.IsInRole(...)`, puede que necesites un `ClaimMapper` o transformar las claims en `OnTokenValidated`.

---

## 11. Pruebas con Postman y Swagger

### 11.1 Obtener token (client_credentials)

```
POST http://localhost:8080/realms/mi_realm/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=orders-api
client_secret=<CLIENT_SECRET>
grant_type=client_credentials
```

Copia `access_token` y en Postman agrega header:

```
Authorization: Bearer <access_token>
```

Llama a `GET https://localhost:7041/api/secure/protected`.

### 11.2 Obtener token (password)

```
POST http://localhost:8080/realms/mi_realm/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=mi_app
client_secret=<CLIENT_SECRET>
username=juan
password=secret123
grant_type=password
```

### 11.3 Verificar token en jwt.io

Pega el `access_token` y revisa los claims: `iss`, `aud`, `exp`, `realm_access`, `resource_access`.

---

## 12. Buenas prácticas y recomendaciones

* **HTTPS en producción**: nunca uses `RequireHttpsMetadata = false` en producción.
* **Separar clients por recursos**: un client por API para controlar audiencias y permisos.
* **Usar Authorization Code + PKCE para apps públicas**: SPA y móviles.
* **Evitar ROPC (`password`)**: solo en entornos controlados o legados.
* **Rotación de secrets**: maneja secrets y credenciales con vaults (HashiCorp Vault, AWS Secrets Manager, etc.).
* **Auditing y sesiones**: monitoriza sesiones, tokens emitidos y permite revocación si es necesario.
* **Minimizar scopes/claims**: no incluyas datos sensibles o innecesarios en el token.
* **Refresh tokens y revocación**: controla la durabilidad de los refresh tokens y políticas de revocación.

---

## 13. Anexos

### 13.1 Comandos útiles

* Levantar Keycloak:

```bash
docker compose up -d
```

* Obtener token vía curl (client_credentials):

```bash
curl -X POST 'http://localhost:8080/realms/mi_realm/protocol/openid-connect/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'client_id=orders-api' \
  -d 'client_secret=<secret>' \
  -d 'grant_type=client_credentials'
```

### 13.2 Fragmento para mapear roles a claims legibles por .NET

Si Keycloak coloca roles en `realm_access.roles`, .NET no los reconoce directamente como roles simples. Puedes mapearlos en `OnTokenValidated`:

```csharp
options.Events = new JwtBearerEvents
{
    OnTokenValidated = ctx =>
    {
        var claimsIdentity = ctx.Principal?.Identity as ClaimsIdentity;
        if (claimsIdentity == null) return Task.CompletedTask;

        // Ejemplo: extraer realm_access.roles
        var realmAccess = ctx.Principal?.FindFirst("realm_access")?.Value;
        if (!string.IsNullOrEmpty(realmAccess))
        {
            // realmAccess es JSON, parsearlo para extraer roles
            var json = System.Text.Json.JsonDocument.Parse(realmAccess);
            if (json.RootElement.TryGetProperty("roles", out var rolesElement))
            {
                foreach (var role in rolesElement.EnumerateArray())
                {
                    claimsIdentity.AddClaim(new Claim(ClaimTypes.Role, role.GetString()));
                }
            }
        }

        return Task.CompletedTask;
    }
};
```

---

### Créditos
Instructor: **Juan Carlos De La Cruz Ch.**

