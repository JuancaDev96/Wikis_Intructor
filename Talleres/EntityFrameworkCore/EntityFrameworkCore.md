# 📘 Entity Framework Core con .NET 10

## 👨‍🏫 Instructor: Juan Carlos De La Cruz

------------------------------------------------------------------------

# 🚀 1. ¿Qué es Entity Framework Core?

Entity Framework Core (EF Core) es un **ORM (Object Relational Mapper)** que permite trabajar con bases de datos utilizando objetos de .NET en lugar de escribir SQL manualmente.

👉 **En pocas palabras:**
- Trabajas con clases (C#) -> EF Core traduce a SQL.

------------------------------------------------------------------------

# 🧠 2. Arquitectura y Change Tracker

## 🔹 El DbContext
Es el corazón de EF Core. Sus responsabilidades incluyen:
- **Gestión de Conexiones**: Manejo eficiente de la base de datos.
- **Mapeo de Datos**: Traducción de SQL a objetos C#.
- **Change Tracker**: Rastrea el estado de las entidades.

## 🔹 Estados de la Entidad
El Change Tracker detecta cambios para generar `INSERT`, `UPDATE` o `DELETE`.

| Estado | Descripción | Acción en SaveChanges |
| :--- | :--- | :--- |
| **Added** | Nueva entidad. | `INSERT` |
| **Unchanged** | Sin cambios detectados. | Ninguna |
| **Modified** | Propiedades cambiadas. | `UPDATE` |
| **Deleted** | Marcada para eliminar. | `DELETE` |
| **Detached** | No rastreada (ej: `AsNoTracking`). | Ninguna |

------------------------------------------------------------------------

# 🧱 3. Enfoque Code First

Primero defines tus clases de C# y EF Core genera la estructura de la base de datos.

```csharp
public class Cliente {
    public int Id { get; set; }
    public string Nombre { get; set; }
}
```

------------------------------------------------------------------------

# 🛠️ 4. Configuración en .NET 10

## 🔹 Paquetes NuGet
```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

## 🔹 Registro del Servicio
```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer("tu_connection_string"));
```

------------------------------------------------------------------------

# 🧩 5. Migraciones

Son el historial de cambios de tu base de datos.
- **Crear**: `dotnet ef migrations add NombreMigracion`
- **Aplicar**: `dotnet ef database update`

------------------------------------------------------------------------

# 🏗️ 6. Modelado con Fluent API

Es el enfoque recomendado para proyectos robustos, permitiendo configuraciones que los atributos (`Data Annotations`) no pueden manejar.

## 🔹 Tipos de Propiedad (Owned Types)
Permite agrupar campos (ej: Dirección) en un objeto sin crear otra tabla.
```csharp
modelBuilder.Entity<Cliente>().OwnsOne(c => c.Direccion);
```

## 🔹 Propiedades de Sombra (Shadow Properties)
Columnas en la DB que no existen en tu clase C# (útil para auditoría).
```csharp
modelBuilder.Entity<Cliente>().Property<DateTime>("LastUpdated");
```

------------------------------------------------------------------------

# 🔗 7. Relaciones y Navegación

1. **Uno a Muchos (1:N)**: La convención `ClienteId` crea la FK automáticamente.
2. **Muchos a Muchos (N:N)**: .NET 10 gestiona la tabla intermedia de forma transparente.

------------------------------------------------------------------------

# 🔍 8. Operaciones CRUD Pro

## 🔹 Inserción Eficiente
```csharp
context.Clientes.Add(nuevoCliente);
await context.SaveChangesAsync();
```

## 🔹 Proyecciones (Mejora Rendimiento)
No traigas toda la fila si solo necesitas dos columnas.
```csharp
var data = await context.Clientes
    .Select(c => new { c.Id, c.Nombre }) // SQL optimizado
    .ToListAsync();
```

## 🔹 Actualizaciones Masivas
```csharp
await context.Clientes
    .Where(c => c.Inactivo)
    .ExecuteDeleteAsync(); // Borrado directo en DB sin cargar en memoria
```

------------------------------------------------------------------------

# ⚡ 9. Estrategias de Carga (Loading)

- **Eager Loading**: `.Include()` trae los datos relacionados en el mismo query.
- **Explicit Loading**: Carga relaciones solo cuando las pides explícitamente.
- **Lazy Loading**: Carga automática al acceder a la propiedad (Cuidado con el rendimiento).

------------------------------------------------------------------------

# 🛡️ 10. Persistencia Avanzada: Interceptores

Permite ejecutar código automáticamente antes de `SaveChanges`.

```csharp
public class AuditInterceptor : SaveChangesInterceptor {
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(...) {
        // Lógica para llenar CreatedAt y UpdatedAt automáticamente
    }
}
```

------------------------------------------------------------------------

# 🚀 11. Optimización y Rendimiento

✔ **AsNoTracking**: Úsalo en consultas de solo lectura para mayor velocidad.\
✔ **Compiled Models**: Acelera el arranque de la app con `dotnet ef dbcontext optimize`.\
✔ **Global Query Filters**: Implementa **Soft Delete** fácilmente.

------------------------------------------------------------------------

# 🏛️ 12. Clean Architecture

- **Domain**: Entidades puras (POCOs).
- **Infrastructure**: DbContext, Migraciones y Fluent API.
- **Application**: Interfaces que abstraen el acceso a datos.

------------------------------------------------------------------------

# ✅ 13. Buenas Prácticas del Instructor

1. Siempre usa **CancellationToken** en métodos asíncronos.
2. Evita lógica de negocio pesada dentro del `DbContext`.
3. Revisa el SQL generado en la consola para evitar el problema de **N+1**.
4. Usa **Dapper** para consultas masivas de lectura si necesitas el máximo rendimiento.

------------------------------------------------------------------------

# 🎯 Conclusión

Dominar Entity Framework Core te permite construir aplicaciones escalables y mantenibles. Esta guía proporciona los cimientos para pasar de un uso básico a un nivel profesional.

**Instructor: Juan Carlos De La Cruz**
*Especialista en Desarrollo de Software .NET*
