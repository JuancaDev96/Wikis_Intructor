# Guía: Integración Blazor WebAssembly + API .NET con Keycloak (v25)
**Instructor:** Juan Carlos De La Cruz Ch.
> Documento paso a paso y explicación conceptual de cada pieza implementada.

---

## Resumen

Este documento describe cómo configurar Keycloak v25 (realm, clientes, scopes, usuarios, roles) y cómo integrar una aplicación **Blazor WebAssembly** (cliente SPA) con una **API .NET** protegida por JWT. Incluye:

- Conceptos básicos y decisiones de diseño.
- Configuración detallada en Keycloak para `blazor_client` y para la API.
- Código y explicación de las piezas clave en Blazor (Program.cs, App.razor, `ApiAuthorizationMessageHandler`, index.html).
- Código y explicación para la API (.NET) (CORS, autenticación JWT, autorización, endpoints mínimos).
- Cómo probar, errores comunes y solución de problemas.

> Suponemos las siguientes URLs usadas en tu ejemplo:
>
> - Keycloak: `http://localhost:8080`
> - Realm: `galaxy_realm`
> - Blazor (cliente SPA): `https://localhost:7228`
> - API .NET (resource server): `https://localhost:7053`

---

## Conceptos clave (breve teoría)

- **OpenID Connect (OIDC)**: Protocolo encima de OAuth2 que permite autenticación (sign-in) y obtención de información del usuario.  
- **OAuth2 (Autorización)**: Delegación de permisos.  
- **Access Token (JWT)**: Token que la API valida.  
- **Client (Keycloak)**: Aplicación registrada.  
- **Client Scopes**: Claims que puede solicitar un cliente.  
- **Roles y Mappers**: Control de autorización.  
- **CORS**: Permite que el navegador llame a la API.  
- **PKCE**: Seguridad adicional para SPAs.

---

## 1) Preparar Keycloak (v25)

### 1.1 Instalar/Levantar Keycloak

Docker:

```
docker run --rm -p 8080:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:25.0.0 start-dev
```

### 1.2 Crear Realm

- Nombre: `galaxy_realm`.

### 1.3 Crear Client: `blazor_client` (SPA)

- Client ID: `blazor_client`
- Access type: **public**
- Standard Flow: ON
- PKCE: ON
- Redirect URIs: `https://localhost:7228/*`
- Web Origins: `https://localhost:7228`

### 1.4 Crear Client opcional para API

Si quieres un client `aspnetcore_api`:

- Access type: **bearer-only** o **confidential**.

### 1.5 Client Scopes & Mappers

Crear un mapper:

- Mapper type: Realm role
- Token Claim Name: `roles`
- Add to access token: ON

### 1.6 Usuarios y roles

Asignar roles a usuarios creados.

---

## 2) Configuración del cliente Blazor

### 2.1 Program.cs

```csharp
builder.Services.AddOidcAuthentication(options =>
{
    options.ProviderOptions.Authority = "http://localhost:8080/realms/galaxy_realm";
    options.ProviderOptions.ClientId = "blazor_client";
    options.ProviderOptions.ResponseType = "code";

    options.ProviderOptions.DefaultScopes.Add("openid");
    options.ProviderOptions.DefaultScopes.Add("profile");
});

builder.Services.AddScoped<ApiAuthorizationMessageHandler>();

builder.Services.AddScoped(sp =>
{
    var handler = sp.GetRequiredService<ApiAuthorizationMessageHandler>();
    handler.InnerHandler = new HttpClientHandler();

    return new HttpClient(handler)
    {
        BaseAddress = new Uri("https://localhost:7053")
    };
});
```

### 2.2 ApiAuthorizationMessageHandler

```csharp
public class ApiAuthorizationMessageHandler : AuthorizationMessageHandler
{
    public ApiAuthorizationMessageHandler(
        IAccessTokenProvider provider,
        NavigationManager navigationManager)
        : base(provider, navigationManager)
    {
        ConfigureHandler(
            authorizedUrls: new[] { "https://localhost:7053" },
            scopes: new[] { "openid", "profile" });
    }
}
```

### 2.3 App.razor

```razor
<CascadingAuthenticationState>
    <Router AppAssembly="@typeof(App).Assembly">
        <Found Context="routeData">
            <AuthorizeRouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)">
            </AuthorizeRouteView>
        </Found>
        <NotFound>
            <LayoutView Layout="@typeof(MainLayout)">
                <p>Sorry, there's nothing at this address.</p>
            </LayoutView>
        </NotFound>
    </Router>
</CascadingAuthenticationState>
```

---

## 3) Configuración API .NET

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowBlazorClient", policy =>
    {
        policy.WithOrigins("https://localhost:7228")
        .AllowAnyHeader()
        .AllowAnyMethod()
        .AllowCredentials();
    });
});

builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "http://localhost:8080/realms/galaxy_realm";
        options.RequireHttpsMetadata = false;
        options.TokenValidationParameters = new()
        {
            ValidateAudience = false
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseCors("AllowBlazorClient");

app.MapGet("/Secure", () => {
    var mensaje = "Acceso autorizado";
    return Results.Ok(mensaje);
}).RequireAuthorization();
```

---

## 4) Flujo completo

1. Blazor redirige a Keycloak.  
2. Usuario inicia sesión.  
3. Keycloak devuelve un `code`.  
4. Blazor intercambia por tokens (PKCE).  
5. Blazor llama API con Bearer Token.  
6. API valida token.  

---

## 5) Debugging

- Revisar `Redirect URIs`.  
- Revisar CORS.  
- Ver claim `aud` si usas validación estricta.  
- Revisar el mapeo de roles si usas autorización basada en roles.  

---

## 6) Seguridad recomendada para producción

- HTTPS obligatorio.  
- Validar audience.  
- Tokens con expiración corta.  
- Revisar scopes asignados a clientes.  

---

## 7) Checklist

- Realm creado  
- Cliente blazor configurado  
- Mapper de roles  
- Usuarios con roles  
- API con JWT configurado  
- CORS habilitado  

---
