# Capa de Acceso a Datos - Arquitectura en Capas

**Instructor: Juan Carlos De La Cruz**

------------------------------------------------------------------------

## 1. Arquitectura en Capas (Layered Architecture)

La arquitectura en capas es un estilo arquitectónico clásico donde el sistema se organiza en niveles claramente definidos, cada uno con una responsabilidad específica.\
El objetivo principal es **separar responsabilidades**, mejorar el **mantenimiento**, la **testabilidad** y la **escalabilidad**.

### Capas comunes

1.  **Capa de Presentación**
    -   UI, APIs, Blazor, MVC, Minimal APIs.
    -   No contiene lógica de negocio.
2.  **Capa de Aplicación**
    -   Casos de uso.
    -   Orquesta la lógica del dominio.
3.  **Capa de Dominio**
    -   Entidades, Value Objects, reglas de negocio.
4.  **Capa de Acceso a Datos (Data Access Layer)**
    -   Persistencia.
    -   Comunicación con bases de datos.
    -   Implementa repositorios.

------------------------------------------------------------------------

## 2. Capa de Acceso a Datos (DAL)

La **DAL** abstrae el mecanismo de almacenamiento y recuperación de
datos.\
No debe conocer detalles de la UI ni de la infraestructura externa.

### Responsabilidades

-   Acceso a base de datos.
-   Mapeo de entidades.
-   Implementación de repositorios.
-   Uso de ORM (Entity Framework).

------------------------------------------------------------------------

## 3. Patrón Repositorio

El patrón repositorio actúa como un **mediador** entre el dominio y la capa de datos.

### Beneficios

-   Desacopla el dominio del ORM.
-   Facilita pruebas unitarias.
-   Centraliza reglas de acceso a datos.

### Contrato base

``` csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Remove(T entity);
}
```

------------------------------------------------------------------------

## 4. Entity Framework Core -- Database First

### ¿Qué es Database First?

Es un enfoque donde: 
- La **base de datos ya existe** 
- EF genera: 
	- DbContext 
  - Entidades 
  - El modelo se basa en el esquema real.

Ideal cuando: 
- Se trabaja con bases legadas. 
- El DBA controla el esquema.

------------------------------------------------------------------------

## 5. Creación del Proyecto (.NET 10)

``` bash
dotnet new sln -n ArquitecturaCapas
dotnet new classlib -n Infraestructura.Data
dotnet new classlib -n Dominio
dotnet new webapi -n Api

dotnet sln add Infraestructura.Data Dominio Api
```

------------------------------------------------------------------------

## 6. Instalación de Paquetes EF Core

``` bash
dotnet add Infraestructura.Data package Microsoft.EntityFrameworkCore
dotnet add Infraestructura.Data package Microsoft.EntityFrameworkCore.SqlServer
dotnet add Infraestructura.Data package Microsoft.EntityFrameworkCore.Design
```

------------------------------------------------------------------------

## 7. Scaffolding -- Database First

``` bash
dotnet ef dbcontext scaffold "Server=.;Database=MiBaseDatos;Trusted_Connection=True;" Microsoft.EntityFrameworkCore.SqlServer --project Infraestructura.Data --output-dir Models --context AppDbContext --use-database-names --no-onconfiguring
```

Esto genera: - `AppDbContext` - Entidades basadas en tablas

------------------------------------------------------------------------

## 8. DbContext generado

``` csharp
public partial class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }

    public virtual DbSet<Usuario> Usuarios { get; set; }
}
```

------------------------------------------------------------------------

## 9. Implementación del Repositorio

## Implementación del Repositorio Genérico

📁 **Infraestructura / Data / Repositories / Repository.cs**

``` csharp
using Microsoft.EntityFrameworkCore;

public class Repository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(AppDbContext context)
    {
        _context = context;
        _dbSet = _context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id)
    {
        return await _dbSet.FindAsync(id);
    }

    public async Task<IReadOnlyList<T>> GetAllAsync()
    {
        return await _dbSet.AsNoTracking().ToListAsync();
    }

    public async Task AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
    }

    public void Update(T entity)
    {
        _dbSet.Update(entity);
    }

    public void Remove(T entity)
    {
        _dbSet.Remove(entity);
    }
}
```


------------------------------------------------------------------------

## 10. Repositorio Específico

``` csharp
public interface IUsuarioRepository : IRepository<Usuario>
{
    Task<Usuario?> ObtenerPorEmailAsync(string email);
}
```

``` csharp
public class UsuarioRepository : Repository<Usuario>, IUsuarioRepository
{
    public UsuarioRepository(AppDbContext context) : base(context) {}

    public async Task<Usuario?> ObtenerPorEmailAsync(string email)
    {
        return await _context.Usuarios
            .FirstOrDefaultAsync(x => x.Email == email);
    }
}
```

------------------------------------------------------------------------

## 11. Registro en Dependency Injection

``` csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped<IUsuarioRepository, UsuarioRepository>();
```

------------------------------------------------------------------------

## 12. Uso desde la API

``` csharp
[ApiController]
[Route("api/usuarios")]
public class UsuariosController : ControllerBase
{
    private readonly IUsuarioRepository _repository;

    public UsuariosController(IUsuarioRepository repository)
    {
        _repository = repository;
    }

    [HttpGet]
    public async Task<IActionResult> Get()
    {
        var usuarios = await _repository.GetAllAsync();
        return Ok(usuarios);
    }
}
```

------------------------------------------------------------------------

## 13. Buenas Prácticas

-   No exponer DbContext directamente.
-   Usar interfaces.
-   Separar contratos y implementaciones.
-   No mezclar lógica de negocio en repositorios.
-   Mantener la DAL como infraestructura.

------------------------------------------------------------------------

## 14. Conclusión

La **Capa de Acceso a Datos** es clave para mantener un sistema limpio y
escalable.\
El uso del **patrón repositorio** junto con **Entity Framework Core
Database First** permite integrar bases existentes sin sacrificar
diseño.

------------------------------------------------------------------------

**Instructor: Juan Carlos De La Cruz**
