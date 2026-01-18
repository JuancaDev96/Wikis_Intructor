# Client Credentials Flow entre APIs con Keycloak

**Instructor:** Juan Carlos De La Cruz Ch.

---

**Resumen:**
Guía práctica que muestra, paso a paso, cómo implementar el **Client Credentials Flow** con Keycloak para comunicación **API → API** o entre microservicios. Incluye: levantar Keycloak con Docker, crear los clients (cliente y recurso), asignar roles a la cuenta de servicio, ejemplos `curl`, proyecto .NET 9 para la API protegida (resource) y ejemplo de cliente consumidor en .NET 9 que solicita el token y llama a la API.

---

## Índice

1. Concepto y caso de uso
2. Preparar entorno (Docker)
3. Configuración en Keycloak (realm, clients, roles, service account)
4. Obtener token: `client_credentials` (curl / Postman)
5. Validación del token: estructura y claims importantes
6. Implementación en .NET 9 — API protegida (resource)
7. Implementación en .NET 9 — API cliente (consumidor) que obtiene token y llama al resource
8. Pruebas y comprobaciones (curl + Postman)
9. Buenas prácticas y recomendaciones

---

## 1. Concepto y caso de uso

**Client Credentials Flow** es un flujo OAuth2/OIDC para escenarios máquina-a-máquina (M2M). No hay usuarios humanos: un servicio se identifica ante el Identity Provider (Keycloak) mediante `client_id` y `client_secret`, obtiene un `access_token` y lo usa para llamar a otro servicio protegido.

Casos típicos:

* Microservicios que se llaman entre sí.
* API Gateway que llama a internal APIs.
* Integraciones backend, jobs o scripts.

Ventajas:

* Simple y seguro para M2M.
* Los tokens representan al servicio (service account).

Limitaciones:

* No contiene identidad de usuario.
* Para acciones en nombre de un usuario, usar Authorization Code con user consent.

---

## 2. Preparar entorno (Docker)

Archivo `docker-compose.yml` (desarrollo):

```yaml
version: '3.8'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - 8080:8080
```

Levantar:

```bash
docker compose up -d
```

Accede a la consola: `http://localhost:8080` → `http://localhost:8080/admin` (user/password: `admin/admin`).

> **Nota:** `start-dev` es para desarrollo. En producción usar configuración de base de datos, HTTPS y HA.

---

## 3. Configuración en Keycloak

**Objetivo:** crear un realm `mi_realm`, un client que represente la API protegida `api_recurso`, y otro client que represente el servicio consumidor `api_cliente`. Asignar roles al recurso y dar esos roles a la cuenta de servicio de `api_cliente`.

### 3.1 Crear Realm

* Admin Console → Create realm → `mi_realm`.

### 3.2 Crear Client: API protegida (`api_recurso`)

1. Realm `mi_realm` → Clients → Create client.
2. Client ID: `api_recurso`.
3. Client type: **OpenID Connect** → Save.
4. Configuraciones recomendadas para resource:

   * Client authentication: **ON**
   * Service accounts enabled: **OFF** (no necesita)
   * Standard Flow Enabled: **OFF**
   * Direct Access Grants Enabled: **OFF**

Guarda.

> Este client simboliza la API que validará tokens.

### 3.3 Crear Client: API cliente (consumidor) (`api_cliente`)

1. Realm → Clients → Create client.
2. Client ID: `api_cliente`.
3. Client type: **OpenID Connect** → Save.
4. Configuraciones:

   * Client authentication: **ON**
   * Service accounts enabled: **ON** (¡importante!)
   * Standard Flow Enabled: **OFF**
   * Direct Access Grants Enabled: **OFF**

Guarda.

### 3.4 Crear roles en `api_recurso`

1. Realm → Clients → seleccione `api_recurso` → Roles → Add Role.
2. Crear `read` y `write`.

### 3.5 Asignar roles de `api_recurso` a la Service Account de `api_cliente`

1. Realm → Clients → seleccione `api_cliente` → Service Account Roles.
2. En el selector "Client Roles", elige `api_recurso`.
3. Mueve `read` (y `write` si aplica) a la lista asignada.

### 3.6 Obtener el client secret de `api_cliente`

1. Clients → `api_cliente` → Credentials.
2. Copia `Client secret` (lo usarás en los requests).

---

## 4. Obtener token: `client_credentials` (curl / Postman)

### 4.1 `curl` ejemplo

```bash
curl -X POST "http://localhost:8080/realms/mi_realm/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=api_cliente" \
  -d "client_secret=<CLIENT_SECRET_DE_API_CLIENTE>" \
  -d "grant_type=client_credentials"
```

Respuesta:

```json
{
  "access_token": "eyJ...",
  "expires_in": 300,
  "token_type": "Bearer",
  "scope": ""
}
```

### 4.2 Postman

* POST a `http://localhost:8080/realms/mi_realm/protocol/openid-connect/token`.
* Body `x-www-form-urlencoded`: `client_id`, `client_secret`, `grant_type=client_credentials`.

---

## 5. Validación del token: estructura y claims importantes

Decodifica el `access_token` (jwt.io) y revisa:

* `iss` (issuer): `http://localhost:8080/realms/mi_realm`
* `sub`: `service-account-api_cliente`
* `azp`: `api_cliente` (authorized party)
* `resource_access`: contiene los roles asignados a `api_recurso` bajo la propiedad `api_recurso.roles`.

Ejemplo (parcial):

```json
{
  "sub": "service-account-api_cliente",
  "azp": "api_cliente",
  "resource_access": {
    "api_recurso": {
      "roles": ["read"]
    }
  }
}
```

---

## 6. Implementación en .NET 9 — API protegida (resource)

### 6.1 Crear el proyecto

```bash
dotnet new webapi -n ApiRecurso
cd ApiRecurso
```

### 6.2 Agregar paquete

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

### 6.3 Program.cs completo

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "http://localhost:8080/realms/mi_realm";
        options.RequireHttpsMetadata = false; // SOLO desarrollo

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "http://localhost:8080/realms/mi_realm",

            ValidateAudience = true,
            ValidAudience = "api_recurso", // el client_id de la API

            ValidateLifetime = true
        };

        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = ctx =>
            {
                // Mapear roles desde resource_access para que .NET los reconozca
                var identity = ctx.Principal?.Identity as System.Security.Claims.ClaimsIdentity;
                if (identity != null)
                {
                    var resourceAccess = ctx.Principal?.FindFirst("resource_access")?.Value;
                    if (!string.IsNullOrEmpty(resourceAccess))
                    {
                        try
                        {
                            using var doc = System.Text.Json.JsonDocument.Parse(resourceAccess);
                            if (doc.RootElement.TryGetProperty("api_recurso", out var apiRes))
                            {
                                if (apiRes.TryGetProperty("roles", out var rolesArray))
                                {
                                    foreach (var r in rolesArray.EnumerateArray())
                                    {
                                        var role = r.GetString();
                                        if (!string.IsNullOrEmpty(role))
                                        {
                                            identity.AddClaim(new System.Security.Claims.Claim(System.Security.Claims.ClaimTypes.Role, role));
                                        }
                                    }
                                }
                            }
                        }
                        catch { /* ignore parse errors */ }
                    }
                }

                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireReadRole", policy => policy.RequireRole("read"));
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/secure", () => Results.Ok("Acceso concedido ✅")).RequireAuthorization("RequireReadRole");

app.Run();
```

### 6.4 Ejecutar la API (puedes correr en Kestrel)

```bash
dotnet run
# Por defecto en https://localhost:5001 (puede variar)
```

---

## 7. Implementación en .NET 9 — API cliente (consumidor)

Este ejemplo muestra un pequeño client console que solicita el token y luego llama a la API protegida.

### 7.1 Crear proyecto cliente

```bash
dotnet new console -n ApiCliente
cd ApiCliente
```

### 7.2 Código `Program.cs`

```csharp
using System.Net.Http.Headers;
using System.Text.Json;

var http = new HttpClient();

// 1) Solicitar token a Keycloak
var tokenRequest = new FormUrlEncodedContent(new Dictionary<string, string>
{
    {"client_id", "api_cliente"},
    {"client_secret", "<CLIENT_SECRET_DE_API_CLIENTE>"},
    {"grant_type", "client_credentials"}
});

var tokenResponse = await http.PostAsync("http://localhost:8080/realms/mi_realm/protocol/openid-connect/token", tokenRequest);
var tokenBody = await tokenResponse.Content.ReadAsStringAsync();
var tokenJson = JsonDocument.Parse(tokenBody);
var accessToken = tokenJson.RootElement.GetProperty("access_token").GetString();

Console.WriteLine("Access token (primeros 80 chars): " + accessToken?.Substring(0, Math.Min(80, accessToken.Length)));

// 2) Consumir la API protegida
http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
var apiResp = await http.GetAsync("https://localhost:5001/secure");
var apiBody = await apiResp.Content.ReadAsStringAsync();
Console.WriteLine("API responde: " + apiResp.StatusCode + " - " + apiBody);
```

> **Nota:** Si la API corre en HTTPS con certificado de desarrollo y tu cliente falla por certificado, puedes ejecutar con `DOTNET_SYSTEM_NET_HTTP_USESOCKETSHTTPHANDLER=0` o configurar para ignorar errores solo en desarrollo.

---

## 8. Pruebas y comprobaciones (curl + Postman)

### 8.1 Obtener token (curl)

```bash
curl -X POST "http://localhost:8080/realms/mi_realm/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=api_cliente" \
  -d "client_secret=<CLIENT_SECRET>" \
  -d "grant_type=client_credentials"
```

### 8.2 Llamar a la API protegida (curl)

```bash
curl -X GET "https://localhost:5001/secure" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" -k
```

(`-k` permite ignorar certificado SSL en desarrollo)


# 🔐 Acceso entre múltiples APIs con Keycloak (Client Credentials Flow Avanzado)

## 🧩 Escenario

Tienes la siguiente arquitectura de microservicios:

* `ApiCliente` → Servicio que consume otros servicios.
* `ApiRecurso1` → API protegida (por ejemplo, Pedidos).
* `ApiRecurso2` → API protegida (por ejemplo, Facturación).
* `Keycloak` → Servidor de identidad que emite y valida tokens JWT.

Objetivo: permitir que `ApiCliente` se comunique de forma segura con **ambas APIs protegidas** usando el **Client Credentials Flow**.

---

## 🔐 Principios de diseño

Cada API protegida debe tener su **propio cliente en Keycloak**. Esto permite:

| Motivo                     | Beneficio                                                              |
| -------------------------- | ---------------------------------------------------------------------- |
| 🔹 Aislamiento de permisos | Cada API tiene su propio conjunto de roles (`read`, `write`, `admin`). |
| 🔹 Audiencias separadas    | Cada token define claramente para qué API es válido.                   |
| 🔹 Rotación independiente  | Cada API mantiene su propio `client_secret` y configuración.           |
| 🔹 Control granular        | Puedes decidir qué clientes acceden a qué APIs.                        |

✅ **Regla general:** *Cada API = un Client en Keycloak.*

---

## ⚙️ Configuración paso a paso

### 1️⃣ Crear el nuevo cliente para la API protegida

Ejemplo: `api_recurso2`

* **Client ID:** `api_recurso2`
* **Access Type:** `bearer-only`
* **Roles:** `read`, `write`

### 2️⃣ Dar permisos al cliente que consumirá

Ve a:

```
Clients → api_cliente → Service Account Roles → Client Roles → api_recurso2
```

Asigna los roles que necesite (por ejemplo, `read`, `write`).

Resultado: el token emitido incluirá acceso a múltiples recursos.

```json
"aud": ["api_recurso1", "api_recurso2", "account"],
"resource_access": {
  "api_recurso1": { "roles": ["read"] },
  "api_recurso2": { "roles": ["read", "write"] }
}
```

---

## 🧠 Validación por cada API

Cada API valida el token **de forma independiente**, asegurando que:

* El `aud` contenga su identificador (`api_recurso1` o `api_recurso2`).
* Los roles dentro de `resource_access` sean correctos.

Ejemplo (fragmento en `Program.cs`):

```csharp
if (doc.RootElement.TryGetProperty("api_recurso2", out var apiRes))
{
    if (apiRes.TryGetProperty("roles", out var roles))
    {
        foreach (var role in roles.EnumerateArray())
        {
            identity.AddClaim(new Claim(ClaimTypes.Role, role.GetString()));
        }
    }
}
```

Cada API interpreta **solo su sección** del `resource_access`.

---

## 🧾 Roles compartidos vs. roles reutilizados

* Puedes **reutilizar nombres** (`read`, `write`) por consistencia.
* Pero cada API administra **sus propios roles**, incluso si se llaman igual.

Esto permite que:

* `read` en `api_recurso1` = “ver pedidos”.
* `read` en `api_recurso2` = “ver facturas”.

Son conceptos distintos aunque tengan el mismo nombre.

---

## ✅ Resumen práctico

| Caso                               | Acción recomendada                                                  |
| ---------------------------------- | ------------------------------------------------------------------- |
| Nueva API protegida                | Crear un nuevo cliente en Keycloak.                                 |
| Permitir acceso desde `ApiCliente` | Asignar roles de esa nueva API al service account de `api_cliente`. |
| Reutilizar nombres de roles        | Sí, pero gestionados por cada API.                                  |
| Token resultante                   | Incluirá múltiples `aud` y `resource_access` para todas las APIs.   |

---

## 💡 Beneficios arquitectónicos

* Seguridad de microservicios sin acoplamiento.
* Permisos granulares gestionados centralmente en Keycloak.
* Tokens JWT reutilizables para múltiples recursos.
* Escalabilidad en mallas de servicios con autorización distribuida.




### 8.3 Comprobaciones

* Token válido → `200 OK` y cuerpo `Acceso concedido ✅`.
* Token inválido / sin roles → `401 Unauthorized` (o `403 Forbidden` si falla la autorización por roles).
* Revisa `resource_access` en el token para confirmar que contiene `api_recurso.roles`.

---

## 9. Buenas prácticas y recomendaciones

* **Usar HTTPS en producción.** Nunca `RequireHttpsMetadata = false` en producción.
* **Rotación de secretos:** usa un vault para secretos y rota periódicamente.
* **Principio de menor privilegio:** asigna solo los roles estrictamente necesarios (ej. `read` en vez de `write` cuando corresponda).
* **Auditoría y monitoreo:** registra tokens emitidos y accesos para inspección.
* **Limitar la vida del token:** configura `accessTokenLifespan` en Keycloak para tokens cortos y usa refresh si aplica (aunque client_credentials no usa refresh tokens por defecto).
* **Seguridad de la red:** restringe que solo hosts autorizados puedan contactar Keycloak en producción.

---

## Anexos: Comandos útiles

* Levantar Keycloak:

```bash
docker compose up -d
```

* Obtener token via curl (client_credentials):

```bash
curl -X POST 'http://localhost:8080/realms/mi_realm/protocol/openid-connect/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'client_id=api_cliente' \
  -d 'client_secret=<secret>' \
  -d 'grant_type=client_credentials'
```

---


Instructor: **Juan Carlos De La Cruz Ch.**

