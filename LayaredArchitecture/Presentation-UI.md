# Presentation Layer (Capa de Presentación)
*Instructor: Juan Carlos De La Cruz Ch*
## 📌 Propósito de la Capa de Presentación

La **capa de presentación** es la responsable de **interactuar directamente con el usuario**. En una arquitectura en capas, esta capa:

- Captura acciones del usuario
- Muestra información visual
- Orquesta eventos de UI
- Consume servicios de la capa Business

En tu caso, esta capa está implementada con **Blazor Server**, lo que implica un modelo **stateful**, con renderizado en servidor y comunicación en tiempo real mediante **SignalR**.

🚫 Esta capa **NO contiene lógica de negocio** ni reglas de persistencia.

---

## 📂 Ubicación en la arquitectura

```
Galaxy.Pedidos
│
├── UI (Blazor Server)
│   ├── Pages
│   ├── Components
│   ├── Layouts
│   ├── Services (UI Services)
│   └── Program.cs
├── Business
├── Repositories
├── DataAccess
└── Entities
```

---

## 🎯 Responsabilidades de la Capa de Presentación

✔ Renderizar componentes y páginas  
✔ Manejar eventos de usuario (click, submit, change)  
✔ Validar datos de entrada (DataAnnotations)  
✔ Gestionar estado de UI  
✔ Consumir servicios de negocio  
✔ Manejar autenticación y autorización visual

🚫 **NO debe**:
- Acceder a repositorios
- Usar DbContext
- Contener reglas de negocio
- Manipular entidades del dominio

---

## 🧱 Blazor Server como tecnología de presentación

### Características clave

- Renderizado en servidor
- Comunicación vía SignalR
- Estado de componentes mantenido en memoria
- Ideal para aplicaciones empresariales

✔ Reduce complejidad en el cliente  
✔ Facilita validaciones y seguridad

---

## 🧩 Componentes Blazor

Los **componentes** son la unidad fundamental de Blazor.

### Ejemplo: Formulario de Cliente

- `EditForm` gestiona el ciclo de vida del formulario
- `Model` enlaza el DTO
- `OnValidSubmit` ejecuta lógica solo si el modelo es válido

### Responsabilidades del componente

✔ Capturar datos del usuario  
✔ Ejecutar validaciones  
✔ Emitir eventos al componente padre

---

## 🔁 Data Binding

Blazor utiliza **Two-Way Data Binding** mediante `@bind`.

Ejemplo conceptual:
- El input modifica el modelo
- El modelo actualiza la UI

✔ Sincronización automática  
✔ Menos código imperativo

---

## ✅ Validaciones con DataAnnotations

### DataAnnotationsValidator

Permite ejecutar validaciones definidas en los DTOs:

- `[Required]`
- `[StringLength]`
- `[EmailAddress]`

### ValidationMessage

Muestra errores por propiedad de forma declarativa.

📌 **Buena práctica**:
> Las validaciones básicas viven en los DTOs, no en la UI.

---

## 🧩 Comunicación entre componentes

### EventCallback

Se utiliza para comunicar eventos **hijo → padre**.

✔ Desacopla componentes  
✔ Facilita reutilización

Ejemplo conceptual:
- Componente formulario emite `OnSaveEvent`
- Página contenedora decide qué hacer

---

## 🧱 Componentes reutilizables

### RenderFragment

Permite crear componentes **composición-friendly**.

Ejemplo: Card reutilizable

✔ Encapsula layout  
✔ Permite inyectar contenido dinámico  
✔ Reduce duplicación

### EditorRequired

Fuerza que el consumidor del componente pase parámetros obligatorios.

---

## 🧭 Rutas y navegación

- Las rutas se definen a nivel de páginas
- Se recomienda centralizar rutas (`RoutesMenu`)

✔ Evita strings mágicos  
✔ Facilita mantenimiento

---

## 🔐 Autenticación y Autorización

### AuthenticationStateProvider

Blazor Server maneja autenticación a nivel de UI:

- Estado del usuario
- Claims
- Roles

El `AutenticacionService` actúa como puente entre sesión y UI.

---

## 🧠 Program.cs – Composición de la aplicación

### Responsabilidades

✔ Registrar servicios  
✔ Configurar middleware  
✔ Definir seguridad  
✔ Conectar capas

### Inyección de dependencias

- UI consume **Business Services**
- Business consume **Repositories**

Nunca:
> UI → Repository directamente ❌

---

## 📦 Servicios de UI

Uso de librerías especializadas:

- Blazored.SessionStorage → estado de sesión
- Blazored.Toast → notificaciones
- SweetAlert2 → feedback visual

📌 Estas librerías **solo pertenecen a la capa de presentación**.

---

## 🧼 Buenas prácticas en la Capa de Presentación

✔ Componentes pequeños y reutilizables  
✔ Separar Pages de Components  
✔ DTOs como modelos de UI  
✔ Eventos desacoplados  
✔ Validaciones declarativas  
✔ Nada de lógica de negocio  
✔ Nada de acceso a datos

---

## 🚧 Límites claros de la capa

### ✔ Permitido
- DTOs
- Servicios de negocio
- Validaciones
- Renderizado

### ❌ Prohibido
- Entities
- Repositories
- DbContext
- SQL / LINQ

---

## 📘 Conclusión

La **capa de presentación en Blazor Server**:

- Orquesta la experiencia del usuario
- Mantiene el sistema desacoplado
- Protege el dominio
- Facilita evolución visual sin romper negocio

Cuando esta capa está bien diseñada, el sistema es:

✔ Mantenible  
✔ Escalable  
✔ Seguro  
✔ Profesional

---

✍️ *Documentación alineada a arquitectura en capas, Blazor Server y buenas prácticas enterprise.*
