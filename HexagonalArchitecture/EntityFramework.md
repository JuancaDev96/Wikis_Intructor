# 🗄️ Entity Framework Core

**Instructor:** Juan Carlos de la Cruz

------------------------------------------------------------------------

## 📌 ¿Qué es Entity Framework Core?

Entity Framework Core (EF Core) es un ORM (Object Relational Mapper) que
permite trabajar con bases de datos usando objetos del lenguaje de
programación, evitando escribir consultas SQL manuales.

Permite:

-   Mapear tablas a clases
-   Columnas a propiedades
-   Relaciones a navegación de objetos
-   Migraciones automáticas

------------------------------------------------------------------------

# 🧭 ENFOQUES DE TRABAJO

------------------------------------------------------------------------

## 🔹 Code First

En este enfoque:

➡️ Primero se crean las clases del dominio\
➡️ Luego EF genera la Base de Datos

### 📄 Entidad

``` csharp
public class Account
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public decimal Balance { get; set; }

    public Address Address { get; set; }
}
```

------------------------------------------------------------------------

## 🔹 Database First

➡️ Primero existe la Base de Datos\
➡️ EF genera las entidades desde la BD

### 📦 Comando Scaffold

``` bash
dotnet ef dbcontext scaffold "connectionString"
Microsoft.EntityFrameworkCore.SqlServer
-o Models
```

------------------------------------------------------------------------

# 🧱 CONFIGURACIÓN DE ENTIDADES

EF recomienda usar:

## IEntityTypeConfiguration

Permite separar la configuración del modelo de la entidad.

``` csharp
public class AccountConfiguration :
IEntityTypeConfiguration<Account>
{
    public void Configure(EntityTypeBuilder<Account> builder)
    {
        builder.HasKey(x => x.Id);

        builder.Property(x => x.Name)
               .HasMaxLength(150)
               .IsRequired();
    }
}
```

------------------------------------------------------------------------

# 🏠 OWNED ENTITIES (OwnsOne)

Se usa cuando un objeto depende completamente de otro.

### Value Object

``` csharp
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
}
```

### Configuración

``` csharp
builder.OwnsOne(x => x.Address, address =>
{
    address.Property(a => a.Street)
           .HasColumnName("Street");

    address.Property(a => a.City)
           .HasColumnName("City");
});
```

------------------------------------------------------------------------

# 🔗 RELACIONES

------------------------------------------------------------------------

## 🔹 One To Many

``` csharp
builder.HasMany(x => x.Transactions)
       .WithOne(t => t.Account)
       .HasForeignKey(t => t.AccountId);
```

------------------------------------------------------------------------

## 🔹 One To One

``` csharp
builder.HasOne(x => x.Profile)
       .WithOne(p => p.Account)
       .HasForeignKey<Profile>(p => p.AccountId);
```

------------------------------------------------------------------------

## 🔹 Many To Many

``` csharp
builder.HasMany(x => x.Roles)
       .WithMany(r => r.Accounts);
```

------------------------------------------------------------------------

# 🧾 MIGRACIONES

------------------------------------------------------------------------

## 📦 Crear migración

``` bash
dotnet ef migrations add InitialCreate
```

------------------------------------------------------------------------

## 📤 Aplicar migración

``` bash
dotnet ef database update
```

------------------------------------------------------------------------

## 📄 Eliminar migración

``` bash
dotnet ef migrations remove
```

------------------------------------------------------------------------

## 📜 Script SQL

``` bash
dotnet ef migrations script
```

------------------------------------------------------------------------

# 🛠️ PUBLICAR CAMBIOS EN BD

Cuando se modifica una entidad:

1.  Crear migración

``` bash
dotnet ef migrations add UpdateAccount
```

2.  Aplicar cambios

``` bash
dotnet ef database update
```

------------------------------------------------------------------------

# 🧪 REGISTRO DE CONFIGURACIONES

``` csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly
    (typeof(AppDbContext).Assembly);
}
```

------------------------------------------------------------------------

# 🎯 BUENAS PRÁCTICAS

-   Usar IEntityTypeConfiguration
-   Separar Dominio de Infraestructura
-   No exponer DbContext en Application
-   Versionar migraciones
-   Usar Owned Entities para Value Objects

------------------------------------------------------------------------

# 📌 CONCLUSIÓN

Entity Framework Core permite trabajar de manera eficiente con bases de
datos usando programación orientada a objetos, facilitando el
mantenimiento, evolución y versionado del esquema de datos mediante
migraciones.
