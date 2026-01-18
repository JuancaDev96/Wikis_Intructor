# Business Layer (Capa de Negocio)

## 📌 Propósito de la Capa de Negocio

La **capa de negocio (Business Layer)** es el **corazón de la aplicación**. Aquí se orquestan los casos de uso, se aplican reglas de negocio y se controla el flujo entre la capa de presentación (API / UI) y la capa de persistencia (Repositories).

Esta capa **NO** conoce detalles de infraestructura (EF Core, DbContext, SQL) ni de presentación (HTTP, Controllers, Blazor, etc.). Su responsabilidad es **decidir qué se hace y cómo se hace**, no **cómo se guarda** ni **cómo se muestra**.

---

## 📂 Ubicación en la arquitectura

```
Galaxy.Pedidos
│
├── API / UI
│   └── Controllers / Pages
├── Business
│   ├── Interfaces
│   │   └── IClienteService.cs
│   └── Implementations
│       └── ClienteService.cs
├── Repositories
├── DataAccess
└── Entities
```

---

## 🧠 Caso de estudio: ClienteService

La clase `ClienteService` implementa los **casos de uso del dominio Cliente**, utilizando repositorios, DTOs, mapeadores y mecanismos de logging.

### Dependencias

- `IClienteRepository` → acceso a datos
- `ILogger<ClienteService>` → trazabilidad
- `Mapster` → mapeo DTO ↔ Entity

Todas las dependencias se inyectan por **constructor**, cumpliendo inversión de dependencias.

---

## 🎯 Responsabilidades de la Capa Business

✔ Orquestar casos de uso (Add, Update, List, Delete, GetById)  
✔ Aplicar validaciones y reglas de negocio  
✔ Controlar el flujo transaccional  
✔ Traducir entidades a DTOs y viceversa  
✔ Manejar errores y mensajes de negocio  
✔ Retornar respuestas estandarizadas

🚫 **NO debe**:
- Acceder directamente a DbContext
- Ejecutar SQL o LINQ complejos de persistencia
- Exponer entidades a capas externas
- Contener lógica de presentación

---

## 🧱 Principios SOLID aplicados

### ✅ Single Responsibility Principle (SRP)

`ClienteService` tiene **una sola razón de cambio**: modificaciones en las reglas de negocio del cliente.

No:
- Persiste datos
- Renderiza vistas
- Gestiona infraestructura

---

### ✅ Open / Closed Principle (OCP)

La clase está **abierta a extensión y cerrada a modificación**:

- Se pueden agregar nuevos métodos de negocio
- Se pueden decorar servicios (logging, caching)
- No se modifica la lógica existente

---

### ✅ Liskov Substitution Principle (LSP)

`ClienteService` puede ser sustituido por cualquier implementación que respete `IClienteService` sin romper el sistema.

---

### ✅ Interface Segregation Principle (ISP)

Las interfaces son:
- Específicas
- Orientadas a casos de uso

Evita interfaces gigantes y acopladas.

---

### ✅ Dependency Inversion Principle (DIP)

La capa de negocio depende de **abstracciones**, no de implementaciones:

- `IClienteRepository`
- `ILogger<T>`

Esto permite:
- Testing
- Mocking
- Sustitución de infraestructura

---

## 🧩 Patrones de diseño utilizados

### 🧩 Service Layer Pattern

`ClienteService` actúa como una **capa de servicios de aplicación**, centralizando la lógica de los casos de uso.

---

### 🧩 Repository Pattern

Delegación completa del acceso a datos a repositorios.

---

### 🧩 DTO Pattern

Uso de **DTOs** para:
- Entrada (Request)
- Salida (Response)

Evita exponer entidades del dominio.

---

### 🧩 Mapper Pattern

Uso de **Mapster** para:
- Convertir DTO → Entity
- Convertir Entity → DTO

Reduce código repetitivo y errores.

---

## 🔁 Ciclo de vida de los DTOs

### 📥 DTOs de Request

- **Nacen** en la capa de presentación
- **Viven** en Business
- **Mueren** al convertirse en Entidades

Ejemplo:
```
AddClienteRequest → Cliente
```

---

### 📤 DTOs de Response

- **Nacen** desde entidades
- **Viven** en Business
- **Mueren** al ser serializados por la API/UI

Ejemplo:
```
Cliente → ClienteResponse
```

🚫 **Regla de oro**:
> Las entidades **NUNCA** deben salir de la capa Business.

---

## 🔄 Conversión y transferencia de datos

### DTO → Entity

- Se realiza mediante Mapster
- Se aplica solo en la capa Business
- Permite validar antes de persistir

```csharp
request.Adapt<Cliente>();
```

---

### Entity → DTO

- Se usa para exponer datos al exterior
- Se controla qué campos salen

```csharp
cliente.Adapt<ClienteResponse>();
```

---

## 🧼 Buenas prácticas aplicadas

✔ Uso de DTOs para aislamiento del dominio  
✔ Respuestas estandarizadas (`BaseResponse`)  
✔ Manejo centralizado de errores  
✔ Logging estructurado  
✔ Paginación controlada  
✔ Reglas de negocio explícitas  
✔ Dependencias inyectadas  
✔ Código testeable

---

## ⚠️ Límites y reglas claras

### ✔ Permitido
- Validaciones de negocio
- Orquestación de repositorios
- Conversión DTO ↔ Entity
- Decisiones de flujo

### ❌ Prohibido
- SQL / LINQ de persistencia compleja
- Acceso directo a DbContext
- Uso de ViewModels
- Lógica de UI o HTTP

---

## 📎 Recomendaciones arquitectónicas

- Mantener servicios delgados y expresivos
- Evitar lógica duplicada entre servicios
- Usar excepciones solo para flujos excepcionales
- Centralizar validaciones complejas
- Versionar DTOs si el contrato cambia

---

## 📘 Conclusión

La **capa de negocio**:

- Define el comportamiento del sistema
- Protege el dominio
- Coordina reglas y flujos
- Garantiza mantenibilidad y escalabilidad

Una capa Business bien diseñada es la diferencia entre un sistema **frágil** y uno **evolutivo**.

---

✍️ *Documentación alineada a arquitectura en capas, SOLID y Clean Architecture.*
