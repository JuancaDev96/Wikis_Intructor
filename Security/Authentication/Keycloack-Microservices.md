# 📘 **POC Microservicios con .NET 9 + Ocelot + Keycloak + Docker**

### **Instructor: Juan Carlos De La Cruz**

---

# #️⃣ **Descripción General**

Este proyecto es una **Prueba de Concepto (POC)** que integra:

-   **Autenticación:** Keycloak
-   **API Gateway:** Ocelot
-   **Microservicios:** 3 Minimal APIs en .NET 9
-   **Orquestación:** Docker (opcional, pero incluido en el proyecto)
-   **Pruebas:** Postman (colecciones para generar token y consumir endpoints)

Incluye:

-   Paso a paso
-   Arquitectura
-   Explicación conceptual
-   Variables y scripts Postman
-   Buenas prácticas

---

# #️⃣ **Arquitectura General**

Postman (Client)  
|  
▼  
API Gateway (Ocelot)  
|  
├──> Microservicio 1 (.NET 9 Minimal API)  
├──> Microservicio 2 (.NET 9 Minimal API)  
└──> Microservicio 3 (.NET 9 Minimal API)

Keycloak (Auth Server)

---

# #️⃣ **1\. Configuración del API Gateway con Ocelot**

## 📄 Archivo: `ocelot.json`

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/service1/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        { "Host": "localhost", "Port": 7081 }
      ],
      "UpstreamPathTemplate": "/service1/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost:54841"
  }
}
```

📌 Conceptos clave:  
Upstream: Ruta pública expuesta por Ocelot.

Downstream: Ruta interna hacia el microservicio.

Host/Port: Deben coincidir con el microservicio local.

BaseUrl: URL pública del gateway.

#️⃣ 2. Microservicio Minimal API – Ejemplo completo  
📄 Program.cs

```json
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication()
    .AddJwtBearer("Keycloak", options =>
    {
        options.Authority = "http://localhost:8080/realms/mi_realm";
        options.Audience = "service1_client";
        options.RequireHttpsMetadata = false;
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/api/service1/public", () =>
{
    return Results.Json(new[] {
        new { Id = 1, Name = "public item 1" },
        new { Id = 2, Name = "public item 2" }
    });
});

app.MapGet("/api/service1/private",
[Authorize] (HttpContext http) =>
{
    var user = http.User?.Identity?.Name ?? "anonymous";
    return Results.Json(new { Message = $"Hello {user}, protected endpoint!" });
});

app.MapGet("/api/service1/claims",
[Authorize] (ClaimsPrincipal user) =>
{
    return Results.Json(user.Claims.Select(c => new { c.Type, c.Value }));
});

app.Run();
```

#️⃣ 3. Configuración de Keycloak  
✔️ 1. Crear un Realm  
Nombre: mi\_realm

✔️ 2. Crear un Client  
Nombre: service1\_client

Configurar:

Client Authentication: ON

Access Type: confidential

Service Accounts Enabled: ON

✔️ 3. Obtener el Secret del Client  
Este secreto es necesario para Postman.

#️⃣ 4. Colección de Token en Postman  
📌 URL del Token Endpoint

```json
POST http://localhost:8080/realms/mi_realm/protocol/openid-connect/token
📌 Body (x-www-form-urlencoded)
makefile
Copiar código
client_id: {{client_id}}
client_secret: {{client_secret}}
grant_type: client_credentials
```

📌 Recomendación  
Como usarás Postman (sin interfaz gráfica de login), el mejor flujo es:

✅ Client Credentials  
Por eso tu client en Keycloak debe ser CONFIDENTIAL.

#️⃣ 5. Endpoints del API Gateway para probar  
📌 Público

```json
GET {{base_url}}/service1/public
```

📌 Requiere Token

```json
GET {{base_url}}/service1/private
Authorization: Bearer {{token}}
```

📌 Claims

```json
GET {{base_url}}/service1/claims
Authorization: Bearer {{token}}
```

#️⃣ 6. Variables de Entorno en Postman  
En tu Environment de Postman crea:

Variable Ejemplo  
base\_url [https://localhost:54841](https://localhost:54841)  
realm mi\_realm  
client\_id service1\_client  
client\_secret XXXXXXXXXXXXX  
token vacío (se llenará automáticamente)

#️⃣ 7. Script Postman para guardar automáticamente el token  
Agrega esto en Tests del request del token:

```json
let body = pm.response.text(); 
try {
    let json = pm.response.json(); 

    if (json.access_token) {
        pm.environment.set("token", json.access_token);
        console.log("TOKEN SET:", json.access_token);
    } else {
        console.log("No se encontró access_token en la respuesta JSON");
    }

} catch (e) {
    console.log("La respuesta no es JSON, contenido recibido:");
    console.log(body);
}

```

#️⃣ 8. Usar el Token en Requests Posteriores  
En los endpoints protegidos:

Headers:

```json
Authorization: Bearer {{token}}
```

#️⃣ 9. Docker (Opcional)  
Si luego deseas contenerizar todo:

Keycloak

API Gateway

Microservicios

Puedes usar un docker-compose.yml como:

```json
services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    command: start-dev
    ports:
      - "8080:8080"
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin

  gateway:
    build: ./Gateway
    ports:
      - "54841:8080"

  service1:
    build: ./Service1
    ports:
      - "7081:8080"
```

🎓 Instructor: Juan Carlos De La Cruz
