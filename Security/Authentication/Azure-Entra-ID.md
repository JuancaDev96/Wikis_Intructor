# Instructivo Completo de Integración con Azure Entra ID

**Instructor:** Juan Carlos De La Cruz

------------------------------------------------------------------------

## 1. Introducción

Este documento explica detalladamente cómo configurar:
- Una **API segura** en Azure Entra ID
- Un **cliente (app client)** capaz de solicitar tokens usando *Client
Credentials Flow*
- El consumo de tokens mediante **cURL** y **.NET**
- Conceptos fundamentales de OAuth2/OpenID Connect y Azure Entra ID

------------------------------------------------------------------------

## 2. Conceptos Clave

### **Azure Entra ID (AAD)**

Es el servicio de identidad de Microsoft que permite:
- Autenticación (identidad del cliente o usuario)
- Autorización (a qué recursos puede acceder)
- Administración de aplicaciones cliente y API

### **OAuth2 Client Credentials Flow**

Es un flujo diseñado para comunicación **API a API** sin usuario.
Un *cliente* obtiene un token enviando:
- `client_id`
- `client_secret`
- `scope` del recurso

### **Aplicación tipo API (App Registration -- Resource API)**

Define:
- Identificador del recurso (`api://<app_id>`)
- Permisos (*scopes*)
- Registra la API como recurso protegido

### **Aplicación Cliente (App Registration -- Client App)**

Una app que **consume** la API.
Debe tener configurado:
- `client_secret`
- Permisos asignados
- Grant "Admin consent"

------------------------------------------------------------------------

## 3. Configuración de la API Segura

### **Paso 1: Crear App Registration para la API**

1.  Ir a **Azure Portal → Azure Entra ID → App registrations → New
    Registration**
2.  Nombre: `api_security`
3.  Tipo de cuenta: *Accounts in this directory only*
4.  Registrar.

### **Paso 2: Configurar la API**

1.  Entrar a la app creada.

2.  Ir a **Expose an API**

3.  Pulsar *Set* en Application ID URI → quedará así:

        api://<APP_ID>

4.  Crear un scope:

    -   Name: `access_as_api`
    -   Admin consent required: Yes
    -   Save.

El scope queda como:

    api://<APP_ID>/access_as_api

------------------------------------------------------------------------

## 4. Configuración del Cliente (App Cliente)

### **Paso 1: Crear App Registration del Cliente**

1.  Ir a **Azure Entra ID → App registrations → New Registration**\
2.  Nombre: `api_client_net9`
3.  Registrar.

### **Paso 2: Crear un Client Secret**

Ir a **Certificates & secrets → New client secret**\
- Copiar **el valor**, no el ID.
*(El error más común es usar el ID del secreto)*

### **Paso 3: Asignar permisos del recurso**

1.  Ir a **API permissions**
2.  Click → *Add permission*
3.  → *My APIs*
4.  Seleccionar `api_security`
5.  Elegir:
    -   *Application permission*: `access_as_api`
6.  Dar **Grant admin consent**

### **Paso 4: Validar la entrada**

Tu configuración final debe mostrar:
- `client_id = ID de api_client_net9`
- `secret = valor del secret`
- `scope = api://<APP_ID_API>/.default`

------------------------------------------------------------------------

## 5. Solicitar Token con cURL

### **Plantilla general**

``` bash
curl --location 'https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token' --header 'Content-Type: application/x-www-form-urlencoded' --data-urlencode 'client_id=<CLIENT_ID_DEL_CLIENTE>' --data-urlencode 'client_secret=<CLIENT_SECRET>' --data-urlencode 'scope=api://<APP_ID_API>/.default' --data-urlencode 'grant_type=client_credentials'
```

### **Ejemplo real**

``` bash
curl --location 'https://login.microsoftonline.com/0c001ca9-cf39-4977-9649-4c31cccd3ac9/oauth2/v2.0/token' --header 'Content-Type: application/x-www-form-urlencoded' --data-urlencode 'client_id=00821893-3d61-4770-9bc5-47719744c2ac' --data-urlencode 'client_secret=TqF8Q~I~i-4vOgTYDDyBRU0sWZU_NkXnDBl8bcPC' --data-urlencode 'scope=api://00821893-3d61-4770-9bc5-47719744c2ac/.default' --data-urlencode 'grant_type=client_credentials'
```

------------------------------------------------------------------------

## 6. Integración en .NET 9

### **Program.cs**

``` csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://login.microsoftonline.com/<TENANT_ID>/v2.0";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidAudience = "api://<APP_ID_API>"
        };
    });

builder.Services.AddAuthorization();
```

### **Controller**

``` csharp
[Authorize]
[ApiController]
[Route("secure-demo")]
public class SecureDemoController : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok("Access granted!");
}
```

------------------------------------------------------------------------

## 7. Prueba de la API

Usa el token obtenido y llama a la API:

``` bash
curl --location 'https://localhost:7240/secure-demo' --header 'Authorization: Bearer <TOKEN>'
```

------------------------------------------------------------------------

## 8. Conclusión

Con esto tienes:
- El API registrada y protegida
- El cliente configurado correctamente
- El flujo Client Credentials funcionando
- cURL y .NET listos para pruebas

------------------------------------------------------------------------

**Instructor:**
### **Juan Carlos De La Cruz**
