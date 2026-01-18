# Common & DTO Layer (Capa Common y Data Transfer Objects)
*Instructor: Juan Carlos De La Cruz Ch*
## 📌 Propósito de la Capa Common

La **capa Common** es una capa **transversal** dentro de la arquitectura. Su objetivo es **centralizar elementos reutilizables y agnósticos al dominio**, que pueden ser consumidos por **todas las capas** sin romper principios de diseño.

Esta capa **NO contiene lógica de negocio**, **NO accede a datos** y **NO depende de infraestructura**.

---

## 📌 Propósito de la Capa DTO

La capa **DTO (Data Transfer Objects)** define los **contratos de entrada y salida** del sistema. Su función principal es **transportar datos entre capas**, protegiendo el dominio y desacoplando la UI y la persistencia.

> 📌 DTO ≠ Entity

---

## 📂 Ubicación en la arquitectura

```
Galaxy.Pedidos
│
├── Common
│   ├── Helper.cs
│   └── Constants
│       └── ErrorMessages.cs
├── DTO
│   ├── Request
│   │   └── Producto
│   │       └── AddProductoRequest.cs
│   └── Response
│       ├── BaseResponse.cs
│       └── PagedResponse.cs
├── Business
├── Repositories
├── DataAccess
└── Entities
```

---

## 🎯 Responsabilidades de la Capa Common

✔ Centralizar helpers reutilizables  
✔ Definir constantes globales  
✔ Proveer utilidades puras (stateless)  
✔ Evitar duplicación de lógica transversal

🚫 **NO debe**:
- Conocer entidades
- Conocer DbContext
- Tener dependencias de EF / UI
- Contener reglas de negocio

---

## 🧠 Ejemplo: Helper

### Método GetTotalPages

```csharp
public static int GetTotalPages(int totalRows, int rowsPerPage)
```

### Características

✔ Método puro (sin efectos colaterales)  
✔ Reutilizable en múltiples capas  
✔ Independiente de dominio

📌 Uso típico:
- Business → cálculo de paginación
- Presentation → mostrar totales

---

## 🎯 Responsabilidades de la Capa DTO

✔ Definir contratos de entrada (Request)  
✔ Definir contratos de salida (Response)  
✔ Validar datos de entrada (DataAnnotations)  
✔ Controlar qué datos viajan entre capas

🚫 **NO debe**:
- Contener lógica de negocio
- Contener reglas de persistencia
- Tener comportamiento complejo

---

## 📥 DTOs de Request

### Ejemplo: AddProductoRequest

Este DTO representa **datos que ingresan al sistema**.

### Buenas prácticas aplicadas

✔ Uso de `DataAnnotations` para validación básica  
✔ Mensajes de error centralizados (`ErrorMessages`)  
✔ Valores por defecto controlados  
✔ Nombres expresivos

### Validaciones declarativas

- `[Required]` → obligatoriedad
- `[DeniedValues]` → valores inválidos
- `[Display]` → nombres amigables

📌 Estas validaciones:
- Se ejecutan en la UI
- Se respetan en Business

---

## 📤 DTOs de Response

### BaseResponse

Define un **contrato estándar de respuesta**.

✔ Estado de operación  
✔ Mensaje de negocio  
✔ Código de error opcional

---

### BaseResponse<T>

Permite retornar resultados tipados sin romper consistencia.

---

### PagedResponse<T>

Especialización para listados paginados:

✔ Colección de resultados  
✔ Total de filas  
✔ Total de páginas

📌 Ideal para grids, tablas y listados.

---

## 🔁 Ciclo de vida de los DTOs

### Request DTOs

- **Nacen** en Presentation
- **Viven** en Business
- **Mueren** al convertirse en Entities

---

### Response DTOs

- **Nacen** desde Entities
- **Viven** en Business
- **Mueren** al serializarse en UI / API

🚫 **Regla estricta**:
> Las entidades nunca cruzan los límites del Business.

---

## 🧱 Patrones de diseño aplicados

### 🧩 DTO Pattern

- Aísla el dominio
- Evita acoplamiento
- Permite versionado de contratos

---

### 🧩 Result Pattern

- Uso de `BaseResponse`
- Manejo consistente de errores y estados

---

### 🧩 Helper / Utility Pattern

- Métodos estáticos
- Funciones puras
- Reutilización transversal

---

## 🧼 Buenas prácticas

✔ DTOs simples y planos  
✔ Sin lógica de negocio  
✔ Validaciones declarativas  
✔ Constantes centralizadas  
✔ Helpers pequeños y específicos  
✔ Reutilización sin acoplamiento

---

## 🚧 Límites claros

### ✔ Permitido
- DataAnnotations
- Constantes
- Métodos utilitarios
- Clases genéricas de respuesta

### ❌ Prohibido
- Acceso a datos
- Entidades
- Servicios
- Lógica de negocio

---

## 📎 Recomendaciones arquitectónicas

- Versionar DTOs si el contrato cambia
- Evitar lógica condicional en DTOs
- No inflar la capa Common
- Mantener Helpers cohesionados

---

## 📘 Conclusión

La capa **Common + DTO**:

- Protege el dominio
- Estabiliza contratos
- Reduce duplicación
- Facilita mantenimiento

Bien diseñada, esta capa es la **base silenciosa** que mantiene el sistema ordenado y escalable.

---

✍️ *Documentación alineada a arquitectura en capas, SOLID y buenas prácticas enterprise.*
