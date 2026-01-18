# 🧩 Guía Profesional de Integración de CAPTCHA con CAP Services y .NET 9

**Instructor:** Juan Carlos De La Cruz Chinga  
**Versión:** Octubre 2025  

---

## 📘 1. Introducción

La seguridad en formularios web es un componente esencial en cualquier aplicación moderna.  
Los **CAPTCHA** (Completely Automated Public Turing test to tell Computers and Humans Apart) ayudan a diferenciar usuarios humanos de bots automatizados.

En esta guía implementaremos un sistema **CAPTCHA autoalojado** con **CAP Services** (open source y sin dependencias de terceros como Google reCAPTCHA), integrado con una aplicación **ASP.NET 9 + Blazor**, bajo una **arquitectura limpia** y escalable.

---

## 🧱 2. Arquitectura General del Sistema

El flujo completo es el siguiente:

```
+---------------------+
|     Usuario (UI)    |
|  Formulario Blazor  |
+---------+-----------+
          |
          | Token generado por CAP widget
          v
+---------------------+
|   Backend .NET 9    |
|  (CaptchaController)|
+---------+-----------+
          |
          | Solicitud de validación
          v
+---------------------+
|   CAP Services API  |
|  (Docker Container) |
+---------------------+
          |
          | Respuesta de validación (true/false)
          v
+---------------------+
|     Usuario (UI)    |
| Resultado mostrado  |
+---------------------+
```

---

## 🐳 3. Preparación del Entorno con Docker

### 3.1 Definición del contenedor CAP Services

```yaml
services:
  cap:
    image: tiago2/cap:latest
    container_name: cap-standalone
    ports:
      - "3000:3000"
    environment:
      ADMIN_KEY: ddfgfdgdfgdfg343dsfdgdffdg432r24323r3232c
      ENABLE_ASSETS_SERVER: "true"
      WIDGET_VERSION: "latest"
      WASM_VERSION: "latest"
      DATA_PATH: "./.data"
    restart: unless-stopped
    volumes:
      - cap-data:/usr/src/app/.data

volumes:
  cap-data:
```

### 3.2 Explicación
- **ADMIN_KEY**: clave maestra para acceder al panel administrativo.  
- **ENABLE_ASSETS_SERVER**: activa el servidor de recursos estáticos para los widgets.  
- **DATA_PATH**: define el lugar donde se guardan los tokens y sitios registrados.  
- **WIDGET_VERSION / WASM_VERSION**: controlan las versiones del widget cliente.  

Ejecuta el contenedor:

```bash
docker compose up -d
```

Verifica que el servicio esté activo:

```bash
docker ps
```

El panel administrativo estará disponible en:

```
http://localhost:3000
```

---

## 🔑 4. Configuración del Sitio y API Key

1. Accede al panel admin usando la `ADMIN_KEY` definida en tu `docker-compose.yml`.  
2. Crea un nuevo **Site** (por ejemplo, “default”).  
3. El sistema generará:  
   - **Site ID:** `fbbf4d0823`  
   - **Bot Token:** `Bot <BOT_TOKEN>`  
   - **Secret Key:** `<SECRET_KEY>`  
4. Estos valores se usarán en el backend para crear, redimir y verificar los tokens.

---

## 🌐 5. Integración del Frontend (Blazor + JavaScript)

### 5.1 Archivo `wwwroot/js/cap.js`

```javascript
window.CAP = {
    init: function (options) {
        const el = document.querySelector(options.element);
        if (!el) {
            console.error("No se encontró el contenedor:", options.element);
            return;
        }

        const widget = document.createElement('cap-widget');
        widget.setAttribute('site', 'default');
        widget.setAttribute('data-cap-api-endpoint', 'http://localhost:5117/captcha/');

        widget.addEventListener('solve', (e) => {
            const token = e.detail.token;
            console.log("✅ Token generado:", token);
            DotNet.invokeMethodAsync('Captcha.Security.UI', 'SetCaptchaToken', token);
        });

        el.innerHTML = "";
        el.appendChild(widget);
    }
};
```

### 5.2 Explicación

- **`cap-widget`**: componente del frontend que renderiza el desafío CAPTCHA.  
- **`data-cap-api-endpoint`**: apunta al endpoint del backend que gestionará las solicitudes de verificación.  
- **`solve`**: evento que se dispara cuando el usuario resuelve el desafío.  
- **`DotNet.invokeMethodAsync`**: permite enviar el token a Blazor (interop JS → .NET).

---

## 🧩 6. Arquitectura Limpia del Backend

### 6.1 Estructura de carpetas

```
/Application
  /Interfaces
  /Services
/Infrastructure
/WebAPI
  /Controllers
/BlazorUI
```

Cada capa cumple un rol:
- **Application:** lógica de negocio y reglas.  
- **Infrastructure:** implementación de dependencias externas.  
- **WebAPI:** capa de presentación para endpoints REST.  
- **BlazorUI:** interfaz de usuario y eventos JS.  

---

## ⚙️ 7. Implementación del Servicio Captcha (`Application/Services/CaptchaService.cs`)

```csharp
using System.Net.Http.Headers;
using System.Text;
using System.Text.Json;

namespace Application.Services;

public class CaptchaService
{
    private readonly HttpClient _http;
    public CaptchaService(HttpClient http) => _http = http;

    public async Task<string> CreateChallengeAsync()
    {
        const string url = "http://localhost:3000/fbbf4d0823/challenge";
        _http.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", "Bot <BOT_TOKEN>");
        var response = await _http.PostAsync(url, null);
        return await response.Content.ReadAsStringAsync();
    }

    public async Task<string> RedeemAsync(string token, List<long> solutions)
    {
        const string url = "http://localhost:3000/fbbf4d0823/redeem";
        _http.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", "Bot <BOT_TOKEN>");
        var body = new { token, solutions };
        var content = new StringContent(JsonSerializer.Serialize(body), Encoding.UTF8, "application/json");
        var response = await _http.PostAsync(url, content);
        return await response.Content.ReadAsStringAsync();
    }

    public async Task<bool> VerifyAsync(string token)
    {
        const string url = "http://localhost:3000/fbbf4d0823/siteverify";
        _http.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", "Bot <BOT_TOKEN>");
        var payload = new
        {
            secret = "<SECRET_KEY>",
            response = token
        };
        var response = await _http.PostAsJsonAsync(url, payload);
        var raw = await response.Content.ReadAsStringAsync();
        var json = JsonDocument.Parse(raw);
        return json.RootElement.TryGetProperty("success", out var success) && success.GetBoolean();
    }
}
```

### 7.2 Buenas prácticas aplicadas
- **Inyección de dependencias:** `HttpClient` se inyecta mediante `IHttpClientFactory`.  
- **Asincronía:** todas las llamadas son `async/await` para no bloquear el hilo principal.  
- **Serialización segura:** uso de `System.Text.Json`.  

Registrar en `Program.cs`:

```csharp
builder.Services.AddHttpClient<CaptchaService>();
```

---

## 🧩 8. Controlador WebAPI (`WebAPI/Controllers/CaptchaController.cs`)

```csharp
using Application.Services;
using Microsoft.AspNetCore.Mvc;

namespace WebAPI.Controllers;

[ApiController]
[Route("captcha")]
public class CaptchaController : ControllerBase
{
    private readonly CaptchaService _captchaService;
    public CaptchaController(CaptchaService captchaService) => _captchaService = captchaService;

    [HttpPost("challenge")]
    public async Task<IActionResult> Challenge() =>
        Ok(await _captchaService.CreateChallengeAsync());

    [HttpPost("redeem")]
    public async Task<IActionResult> Redeem([FromBody] dynamic body) =>
        Ok(await _captchaService.RedeemAsync(body.token.ToString(), body.solutions.ToObject<List<long>>()));

    [HttpPost("verify")]
    public async Task<IActionResult> Verify([FromBody] dynamic body)
    {
        var valid = await _captchaService.VerifyAsync(body.Token.ToString());
        return valid ? Ok(new { valid = true }) : BadRequest(new { valid = false });
    }
}
```

---

## 💻 9. Componente Blazor (`BlazorUI/Pages/CaptchaForm.razor`)

```razor
@page "/captcha-form"
@inject HttpClient Http
@inject IJSRuntime JS

<h3>Formulario con CAPTCHA</h3>

<div class="card p-3">
    <EditForm Model="@formModel" OnValidSubmit="@OnSubmit">
        <InputText @bind-Value="@formModel.Nombre" placeholder="Tu nombre" class="form-control mb-2" />
        <div id="cap-container"></div>
        <button class="btn btn-primary" type="submit">Enviar</button>
    </EditForm>
</div>

@if (resultado != null)
{
    <div class="alert @(resultado == true ? "alert-success" : "alert-danger") mt-2">
        @mensaje
    </div>
}

@code {
    private FormModel formModel = new();
    private bool? resultado;
    private string? mensaje;
    private static string? _lastToken;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
            await JS.InvokeVoidAsync("CAP.init", new { element = "#cap-container" });
    }

    [JSInvokable("SetCaptchaToken")]
    public static void SetCaptchaToken(string token) => _lastToken = token;

    private async Task OnSubmit()
    {
        if (string.IsNullOrEmpty(_lastToken))
        {
            mensaje = "Por favor, resuelve el CAPTCHA antes de enviar.";
            resultado = false;
            return;
        }

        var response = await Http.PostAsJsonAsync("http://localhost:5117/captcha/verify", new { Token = _lastToken });

        if (response.IsSuccessStatusCode)
        {
            resultado = true;
            mensaje = "Verificación exitosa ✅";
        }
        else
        {
            resultado = false;
            mensaje = "Verificación fallida ❌";
        }
    }

    public class FormModel
    {
        public string Nombre { get; set; } = string.Empty;
    }
}
```

### 9.1 Flujo de validación
1. El usuario llena el formulario y resuelve el CAPTCHA.  
2. El JS genera un **token temporal** y lo envía a Blazor.  
3. El formulario envía ese token al **endpoint `/captcha/verify`** del backend.  
4. El backend valida con **CAP Services** si el token es legítimo.  
5. Si la validación es exitosa, el formulario muestra un mensaje de éxito.  

---

## 🔐 10. Seguridad y Buenas Prácticas

- Nunca expongas tus `Bot Token` ni `Secret Key` en el frontend.  
- Define variables de entorno o usa **User Secrets** en desarrollo.  
- Implementa **timeouts** y manejo de excepciones para prevenir ataques DoS.  
- Mantén el contenedor CAP actualizado (`image: tiago2/cap:latest`).  
- Usa HTTPS para todas las comunicaciones entre backend y frontend.  

---

## 🧠 11. Conclusiones

Esta implementación demuestra cómo:
- Integrar **CAP Services** con **.NET 9 y Blazor** sin depender de servicios externos.  
- Aplicar **arquitectura limpia** separando responsabilidades.  
- Utilizar **interoperabilidad JS/.NET** de manera eficiente.  
- Mejorar la seguridad del backend al validar cada solicitud mediante tokens temporales.  

---

© 2025 - **Instructor: Juan Carlos De La Cruz Chinga**  
_Desarrollo Seguro y Escalable con .NET 9 + CAP Services_
