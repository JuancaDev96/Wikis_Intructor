# Repositories Layer (Capa de Repositorios)
*Instructor: Juan Carlos De La Cruz Ch*
## 📌 Propósito de la capa Repositories

La **capa de Repositories** es responsable de **abstraer el acceso a datos** y encapsular toda la lógica relacionada con la persistencia. Actúa como un puente entre la **capa de dominio/aplicación** y la **capa de acceso a datos (DataAccess)**, evitando que el resto del sistema dependa directamente de **Entity Framework Core** o de detalles de infraestructura.

En esta arquitectura, los repositorios:
- Centralizan operaciones CRUD
- Exponen contratos claros mediante interfaces
- Facilitan pruebas unitarias y mantenimiento
- Promueven bajo acoplamiento y alta cohesión

---

## 📂 Ubicación en la arquitectura

```
Galaxy.Pedidos
│
├── Entities
├── DataAccess
│   └── Contexts
│       └── BdPedidosContext
├── Repositories
│   ├── Interfaces
│   │   ├── IBaseRepository.cs
│   │   └── IClienteRepository.cs
│   └── Implementations
│       ├── BaseRepository.cs
│       └── ClienteRepository.cs
```

---

## 🧱 Repositorio Genérico – BaseRepository

El **BaseRepository<TEntity>** implementa las operaciones comunes para cualquier entidad del sistema, evitando duplicación de código y promoviendo reutilización.

### ✔ Restricción de tipo

```csharp
where TEntity : BaseEntity
```

Esto garantiza que todas las entidades tengan propiedades base como:
- `Id`
- `Estado` (Soft Delete)

---

## 🔑 Operaciones soportadas

### 📄 Listado simple con filtro

- Usa expresiones lambda
- Aplica `AsNoTracking()` para mejorar performance

### 📄 Listado paginado y proyectado

- Soporta:
  - Filtro (`predicate`)
  - Proyección (`selector`)
  - Ordenamiento (`orderBy`)
  - Paginación
- Retorna:
  - Colección de resultados
  - Total de filas

### 🔍 Obtener por Id

- Aplica filtro de `Estado = true`
- Evita retornar registros lógicamente eliminados

### ➕ Inserción

- Agrega la entidad
- Persiste cambios inmediatamente

### ✏ Actualización

- Persiste los cambios ya trackeados por el contexto

### ❌ Eliminación lógica (Soft Delete)

- Usa `ExecuteUpdateAsync`
- No elimina el registro físicamente
- Cambia el estado a `false`

---

## 🧠 Principios SOLID aplicados

### ✅ Single Responsibility Principle (SRP)

Cada repositorio tiene **una única responsabilidad**: manejar el acceso a datos de una entidad.

---

### ✅ Open/Closed Principle (OCP)

El `BaseRepository`:
- Está **abierto para extensión** (repositorios específicos)
- Está **cerrado para modificación**

Ejemplo: `ClienteRepository` hereda sin modificar la lógica base.

---

### ✅ Liskov Substitution Principle (LSP)

Cualquier repositorio específico puede sustituir al repositorio base sin romper el comportamiento esperado.

---

### ✅ Interface Segregation Principle (ISP)

Las interfaces son específicas:
- `IBaseRepository<TEntity>` define operaciones comunes
- `IClienteRepository` puede extender solo lo necesario

---

### ✅ Dependency Inversion Principle (DIP)

- El sistema depende de **interfaces**, no de implementaciones concretas
- La inyección de dependencias se realiza vía constructor

---

## 🧩 Patrones de diseño implementados

### 🧩 Repository Pattern

- Encapsula la lógica de acceso a datos
- Simula una colección en memoria
- Oculta EF Core al resto del sistema

---

### 🧩 Generic Repository Pattern

- Permite reutilizar lógica común
- Reduce duplicación de código
- Centraliza reglas de acceso

---

### 🧩 Unit of Work (implícito)

- `DbContext` actúa como unidad de trabajo
- `SaveChangesAsync()` confirma la transacción

---

## 🧼 Buenas prácticas aplicadas

✔ Uso de `AsNoTracking()` para consultas de solo lectura  
✔ Soft Delete para integridad histórica  
✔ Expresiones lambda para consultas flexibles  
✔ Paginación para evitar sobrecarga de memoria  
✔ Separación clara de responsabilidades  
✔ Repositorios específicos solo cuando es necesario

---

## 🚀 Uso del repositorio genérico

### 📌 Implementación de un repositorio específico

```csharp
public class ClienteRepository
    : BaseRepository<Cliente>, IClienteRepository
{
    public ClienteRepository(BdPedidosContext context)
        : base(context)
    {
    }
}
```

---

### 📌 Inyección de dependencias

```csharp
services.AddScoped<IClienteRepository, ClienteRepository>();
```

---

### 📌 Uso desde la capa Application / Services

```csharp
var clientes = await _clienteRepository.ListAsync(c => c.Estado);
```

---

## 📎 Recomendaciones

- Evitar lógica de negocio en repositorios
- Usar repositorios específicos solo cuando haya queries complejas
- Mantener el repositorio genérico lo más limpio posible
- No exponer `DbContext` fuera de la capa Repositories

---

## 📘 Conclusión

La capa de **Repositories**, combinada con un **Repositorio Genérico**, proporciona:

- Código limpio y reutilizable
- Bajo acoplamiento
- Alta mantenibilidad
- Escalabilidad para proyectos empresariales

Esta implementación es ideal para soluciones en **.NET + EF Core** bajo una **arquitectura en capas** bien definida.

---

✍️ *Documentación alineada a prácticas profesionales y arquitectura limpia.*

