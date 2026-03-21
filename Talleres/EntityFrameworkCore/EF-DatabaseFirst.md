# 📘 Entity Framework Core - Enfoque Database First

## 👨‍🏫 Instructor: Juan Carlos De La Cruz

------------------------------------------------------------------------

# 🚀 1. El Escenario Database First

En el enfoque **Database First**, la base de datos es la soberana. Este escenario es el estándar en entornos corporativos, sistemas legacy o cuando el diseño de la DB está a cargo de un DBA (Database Administrator).

👉 **Flujo de Trabajo Profesional:**
1.  **DB Design**: Se crea o modifica la DB directamente en el motor (SQL Server, PostgreSQL, etc.).
2.  **Scaffolding**: Se usa la CLI de EF Core para realizar ingeniería inversa (Reverse Engineering).
3.  **Refinement**: Se ajustan las clases resultantes mediante clases parciales y configuraciones adicionales.

------------------------------------------------------------------------

# 🏗️ 2. Ingeniería Inversa Pro (Scaffolding)

El comando `scaffold` es potente pero requiere precisión para no generar código "basura".

## 🔹 Comando Avanzado
```bash
dotnet ef dbcontext scaffold "tu_connection_string" \
    Microsoft.EntityFrameworkCore.SqlServer \
    --output-dir Models/Generated \
    --context-dir Data \
    --context MyDbContext \
    --namespace MyProject.Infrastructure.Data \
    --table Sales.Orders,Sales.Customers \
    --use-database-names \
    --force
```

## 🔍 Parámetros Clave:
-   **--use-database-names**: Mantiene los nombres reales de la DB (evita que EF intente "singularizar" a su criterio).
-   **--data-annotations**: Genera atributos en lugar de Fluent API (útil para validaciones rápidas).
-   **--force**: Sobrescribe el modelo actual. **¡Cuidado!** Cualquier cambio manual en los archivos generados se perderá.

------------------------------------------------------------------------

# 🔍 3. Mapeo de Vistas y Procedimientos

Database First brilla cuando necesitamos consumir objetos que no son tablas simples.

## 🔹 Mapeo de Vistas (Keyless Entities)
Las vistas no suelen tener Primary Key. EF Core requiere configurarlas como `HasNoKey()`.
```csharp
modelBuilder.Entity<VwReporteVentas>()
    .HasNoKey()
    .ToView("Vw_Reporte_Ventas");
```

## 🔹 Stored Procedures (SP)
EF Core permite ejecutar SPs y mapear el resultado a entidades o tipos simples.
```csharp
var result = await context.Clientes
    .FromSqlRaw("EXEC GetClientesByRegion @region={0}", "LATAM")
    .ToListAsync();
```

------------------------------------------------------------------------

# 🧩 4. Personalización Segura: Partial Classes

Dado que el `scaffold` sobrescribe los archivos cada vez que lo ejecutas, **nunca** debes escribir lógica dentro de las clases generadas. Para eso usamos el modificador `partial`.

```csharp
// Archivo Generado: Cliente.cs
public partial class Cliente { ... }

// Archivo Creado por Ti: Cliente.Partials.cs
public partial class Cliente {
    public string NombreCompleto => $"{Nombre} {Apellido}";
    
    [NotMapped]
    public bool EsVip => TotalCompras > 1000;
}
```

------------------------------------------------------------------------

# 🔄 5. Sincronización y Mantenibilidad

El mayor reto es mantener el código en sintonía con la DB sin perder horas en el proceso.

## 🔹 Estrategia de "Generación Limpia"
1.  Mantén el `DbContext` y las entidades generadas en una carpeta aislada (`/Models/Generated`).
2.  Si necesitas añadir interceptores o filtros globales, extiende el `DbContext` en otro archivo parcial.

```csharp
public partial class MyDbContext : DbContext {
    protected override void OnModelCreating(ModelBuilder modelBuilder) {
        base.OnModelCreating(modelBuilder);
        // Tus configuraciones manuales aquí (ej: Soft Delete)
    }
}
```

------------------------------------------------------------------------

# ⚡ 6. Rendimiento en Sistemas Legacy

Las bases de datos antiguas suelen tener problemas de diseño. Puedes mitigarlos desde EF Core:
-   **Split Queries**: Para tablas con muchas relaciones distribuidas, usa `.AsSplitQuery()` para evitar el "Cartesian Explosion".
-   **Buffered vs Streaming**: Usa `AsEnumerable()` o `AsAsyncEnumerable()` para procesar grandes volúmenes de datos legacy.

------------------------------------------------------------------------

# 🏛️ 7. Arquitectura Sugerida

Para proyectos profesionales, no uses las entidades generadas en toda la aplicación:
1.  **Infrastructure/Data**: Donde ocurre el `scaffold`.
2.  **Domain/Application**: Usa Mappers (como AutoMapper) para convertir entidades de DB a **DTOs** o modelos de dominio. Esto protege tu lógica de cambios estructurales repentinos en la base de datos por parte del DBA.

------------------------------------------------------------------------

# ✅ 8. Reglas de Oro del Instructor

✔ **Usa Git rigurosamente**: Haz commit antes de cada `scaffold --force` para poder comparar los cambios generados.\
✔ **Prefiere Fluent API**: Es mucho más flexible para mapear nombres de columnas complejos o esquemas no estándar.\
✔ **Cuidado con las Relaciones N:N**: EF Core no siempre las mapea perfecto en DB First si la tabla intermedia tiene columnas extra.\
✔ **Stored Procedures para Escritura**: Si la DB tiene triggers o reglas complejas, prefiere llamar a un SP para insertar/actualizar.

------------------------------------------------------------------------

# 🎯 Conclusión

Database First es una habilidad vital para arquitectos que enfrentan desafíos reales en la industria. Dominar el **Scaffolding**, las **Partial Classes** y la **Integración de SPs** te permitirá dominar cualquier base de datos legacy con la elegancia y potencia de C#.

**Instructor: Juan Carlos De La Cruz**
*Especialista en Desarrollo de Software .NET*
