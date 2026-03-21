# 📘 Entity Framework Core - Enfoque Code First

## 👨‍🏫 Instructor: Juan Carlos De La Cruz

------------------------------------------------------------------------

# 🚀 1. El Paradigma Code First

En el enfoque **Code First**, el código es la única fuente de verdad. La base de datos no es más que una representación del modelo de clases definido en C#.

👉 **Flujo de Trabajo Profesional:**
1.  **Modelo**: Creas clases POCO (Plain Old CLR Objects).
2.  **Contexto**: Defines el `DbContext` y las reglas de mapeo.
3.  **Migración**: Generas un archivo de "snapshot" con los cambios.
4.  **Despliegue**: Aplicas la migración para actualizar la base de datos.

------------------------------------------------------------------------

# 🧠 2. Mapeo por Convenciones (Conventions)

EF Core aplica reglas automáticas si no defines nada explícitamente:
-   **PK**: Busca propiedades llamadas `Id` o `[NombreClase]Id`.
-   **Tablas**: Usa el nombre del `DbSet<T>` en el contexto.
-   **Nulidad**: Tipos por referencia (`string`) son nulables; tipos por valor (`int`, `DateTime`) no lo son, a menos que uses `?`.
-   **FK**: Detecta automáticamente si existe una propiedad `Navegación + Id`.

------------------------------------------------------------------------

# 🏗️ 3. Modelado Avanzado: Fluent API

Aunque existen las **Data Annotations** (ej: `[Required]`), el estándar profesional es **Fluent API** en el método `OnModelCreating`.

## 🔹 Conversión de Valores (Value Converters)
Útil para guardar Enums como strings en la base de datos.
```csharp
modelBuilder.Entity<Pedido>()
    .Property(p => p.Estado)
    .HasConversion<string>();
```

## 🔹 Tipos Complejos y Propiedades de Sombra
Permite manejar lógica de auditoría sin "ensuciar" tu entidad.
```csharp
modelBuilder.Entity<Cliente>()
    .Property<DateTime>("Version") // Shadow Property
    .IsRowVersion(); // Concurrencia
```

------------------------------------------------------------------------

# 🧬 4. Estrategias de Herencia

EF Core soporta varias formas de mapear jerarquías de clases a tablas:

1.  **TPH (Table Per Hierarchy)**: Todas las clases en una sola tabla con una columna `Discriminator` (Default).
2.  **TPT (Table Per Type)**: Una tabla base y tablas adicionales para cada subclase (Más normalizado, menos performance).
3.  **TPC (Table Per Concrete Type)**: Tablas completamente separadas para cada clase concreta (Sin tabla base).

```csharp
// Configuración TPT
modelBuilder.Entity<Persona>().ToTable("Personas");
modelBuilder.Entity<Empleado>().ToTable("Empleados");
```

------------------------------------------------------------------------

# 🔄 5. Migraciones en Detalle

Las migraciones son esenciales en Code First. Permiten que la base de datos evolucione junto al código.

## 🔹 Comandos Esenciales (CLI)
-   `dotnet ef migrations add Initial` - Genera el código de migración.
-   `dotnet ef database update` - Aplica los cambios a la DB.
-   `dotnet ef migrations script` - Genera el SQL puro (ideal para producción).

## 🔹 Manejo de Datos Base (Seed Data)
Excelente para insertar catálogos iniciales (ej: Roles, Provincias).
```csharp
modelBuilder.Entity<Rol>().HasData(
    new Rol { Id = 1, Nombre = "Admin" },
    new Rol { Id = 2, Nombre = "Editor" }
);
```

------------------------------------------------------------------------

# ⏱️ 6. Tablas Temporales (System-Versioned)

EF Core permite habilitar temporalidad directamente desde Code First (disponible en SQL Server). Esto guarda un historial automático de cambios.

```csharp
modelBuilder.Entity<Cliente>()
    .ToTable("Clientes", t => t.IsTemporal());
```

------------------------------------------------------------------------

# ⚡ 7. Optimización desde el Modelo

No esperes a tener problemas de rendimiento; configúralos en tu Code First:

-   **Índices Filtrados**: Índices que solo cubren ciertos registros.
-   **Included Columns**: Columnas extra en el índice para evitar "Key Lookups".
-   **Configuración de Esquemas**: Organiza tus tablas por `Schema` (`dbo`, `audit`, `security`).

```csharp
modelBuilder.Entity<Pedido>()
    .HasIndex(p => p.Fecha)
    .HasFilter("[Estado] = 'Pendiente'");
```

------------------------------------------------------------------------

# 🏛️ 8. Arquitectura Limpia y Code First

En proyectos grandes, el mapeo debe estar separado de las entidades:

1.  **Domain**: Clases puras.
2.  **Infrastructure**: Clases `IEntityTypeConfiguration<T>`.

```csharp
// ClienteConfiguration.cs
public class ClienteConfiguration : IEntityTypeConfiguration<Cliente>
{
    public void Configure(EntityTypeBuilder<Cliente> builder)
    {
        builder.ToTable("Clientes", "Sales");
        builder.HasKey(c => c.Id);
        // ... otras reglas
    }
}
```

------------------------------------------------------------------------

# ✅ 9. Reglas de Oro para Code First

✔ **Nunca toques la base de datos manualmente**: Perderás la sincronía con tus migraciones.\
✔ **Revisa la migración generada**: Los archivos en la carpeta `Migrations` son código; léelos antes de aplicarlos.\
✔ **Snapshots**: EF guarda un archivo `DbContextModelSnapshot.cs`. No lo borres, es vital para calcular cambios.\
✔ **Idempotencia**: Usa `-Idempotent` al generar scripts SQL para asegurar ejecución segura.

------------------------------------------------------------------------

# 🎯 Conclusión

Code First no es solo "dejar que EF cree las tablas", es una disciplina de diseño que prioriza el dominio del negocio. Dominar sus configuraciones avanzadas es lo que distingue a un arquitecto .NET de un desarrollador promedio.

**Instructor: Juan Carlos De La Cruz**
*Especialista en Desarrollo de Software .NET*
