# 🚀 Guía paso a paso: Integración de HashiCorp Vault con .NET 9 + Docker

**Instructor:** Juan Carlos De La Cruz Ch.

---

## 🔹 1. Iniciar Vault en Docker con un token root (solo para desarrollo)
En tu `docker-compose.yml`:

```yaml
vault:
  image: hashicorp/vault:1.17
  ports:
    - "8200:8200"
  environment:
    VAULT_DEV_ROOT_TOKEN_ID: "myroot"        # solo para dev
    VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
  command: "server -dev -dev-listen-address=0.0.0.0:8200"
```

Con esto puedes entrar a la UI con el token `myroot`.

---

## 🔹 2. Crear un secreto en Vault (UI)
1. Ingresa a la UI de Vault en tu navegador:  
   [http://localhost:8200/ui](http://localhost:8200/ui)  
   e inicia sesión con el token configurado en tu `docker-compose.yml` (ej. `myroot`).  

2. En el menú lateral, haz clic en **Secrets Engines**.  

3. Selecciona el engine **secret/** (que viene por defecto).  

4. Haz clic en **Create secret**.  

5. Completa los campos:  
   - **Path for this secret**: `GalaxySecurity`  
   - **Secret Json**:  
     
```hcl
{
  "APIEmailPath": "https://localhost:7299/api/",
  "DbSecurity": "Host=localhost;Port=1500;Database=security_db;Username=admin;Password=Password2025",
  "JwtSecretKey": "d85e6517ce040ce3708f895cd17b4f5e269a5d64eae32698fba7710f",
  "RedisConnection": "localhost:6379"
}
```

6. Haz clic en **Save**.  

El secreto quedará disponible en la ruta:  
[http://localhost:8200/ui/vault/secrets/secret/kv/connectionsString](http://localhost:8200/ui/vault/secrets/secret/kv/connectionsString)  

👉 En tu código, lo accederás como:  
```csharp
var cs = secrets["DbSecurity"];
```

---

## 🔹 3. Crear una política en Vault
Ejemplo de política llamada `policy-galaxysecurity-read` que da permisos de **lectura** solo al path `secret/data/GalaxySecurity`:

```hcl
path "secret/data/GalaxySecurity" {
  capabilities = ["read"]
}
```

Puedes crearla desde la UI (sección **Policies**) o con CLI.

---

## 🔹 4. Crear un token asociado a la política
### 4.1. Mediante la API REST (ejemplo en Swagger o Postman)
Endpoint:
```
POST http://localhost:8200/v1/auth/token/create
```

Body:
```json
{
  "display_name": "token-api-permanent",
  "ttl": "0",
  "explicit_max_ttl": "0",
  "policies": ["policy-galaxysecurity-read"],
  "renewable": true
}

```

### 4.2. Mediante cURL
```bash
curl --location 'http://localhost:8200/v1/auth/token/create' \
--header 'X-Vault-Token: Ucyi1ziPEHQO4b' \
--header 'Content-Type: application/json' \
--data '{
  "display_name": "token-api-permanent",
  "ttl": "0",
  "explicit_max_ttl": "0",
  "policies": ["policy-galaxysecurity-read"],
  "renewable": true
}
'
```

La respuesta te devolverá un `client_token` (ejemplo: `hvs.CAESIDxxxx...`), que es el token que tu API usará.

---

## 🔹 5. Configuración en appsettings.json
```json
"Vault": {
  "Address": "http://localhost:8200",
  "Token": "hvs.CAESIDxxxx...",  // el token que generaste
  "SecretsPath": "connectionsString",
  "MountPoint": "secret"
}
```

---

## 🔹 6. Servicio para consumir secretos desde Vault
```csharp
public async Task<Dictionary<string, string>> GetSecretsAsync()
{
    var secret = await _vaultClient.V1.Secrets.KeyValue.V2.ReadSecretAsync(
        path: _secretsPath,
        mountPoint: _mountPoint
    );

    var data = secret.Data.Data;
    var dict = new Dictionary<string, string>();

    foreach (var kv in data)
    {
        dict[kv.Key] = kv.Value?.ToString() ?? "";
    }

    return dict;
}

public async Task<string?> GetSecretAsync(string key)
{
    var all = await GetSecretsAsync();
    all.TryGetValue(key, out var val);
    return val;
}
```

---

## 🔹 7. Registro del servicio en Dependency Injection
```csharp
public static IServiceCollection AddVaultSecrets(this IServiceCollection services, IConfiguration configuration)
{
    var vaultConfig = configuration.GetSection("Vault");
    var vaultAddress = vaultConfig["Address"];
    var vaultToken = vaultConfig["Token"];
    var secretsPath = vaultConfig["SecretsPath"];
    var mountPoint = vaultConfig["MountPoint"];

    // crear cliente de Vault
    var authMethod = new TokenAuthMethodInfo(vaultToken);
    var vaultClientSettings = new VaultClientSettings(vaultAddress, authMethod);
    IVaultClient vaultClient = new VaultClient(vaultClientSettings);

    // puedes ponerlo como singleton
    services.AddSingleton(vaultClient);

    // Registrar un servicio que obtenga secretos cuando se necesite
    services.AddScoped<IVaultSecretProvider>(sp =>
        new VaultSecretProvider(vaultClient, secretsPath, mountPoint));

    return services;
}
```

---

## 🔹 8. Uso del servicio en Startup/Program.cs
```csharp
// Integrar Vault
services.AddVaultSecrets(configuration);

// Inyección del contexto de base de datos
services.AddDbContext<IdentityDbContext>((sp, options) =>
{
    // acá no haces await, solo registras
    var secretProvider = sp.GetRequiredService<IVaultSecretProvider>();
    var secrets = secretProvider.GetSecretsAsync().GetAwaiter().GetResult(); // sincronizar
    var cs = secrets["DbSecurity"];
    options.UseNpgsql(cs);
});
```

---

✅ Con estos pasos tienes:  
- Vault en Docker con un root token (`myroot`).  
- Un secreto creado en la UI (`connectionsString`).  
- Una política segura para tu conexión (`policy-connectionstring-read`).  
- Un token asociado a esa policy (`token-api`).  
- Tu API .NET obteniendo secretos de Vault en tiempo de ejecución.  

---

👨‍🏫 **Instructor:** Juan Carlos De La Cruz Ch.
