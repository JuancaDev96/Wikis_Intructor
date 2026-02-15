# 📚 Patrones de Diseño aplicados a Arquitectura Hexagonal

**Instructor:** Juan Carlos de la Cruz

------------------------------------------------------------------------

## 🧠 ¿Qué es un Patrón de Diseño?

Un **patrón de diseño** es una solución reutilizable a un problema común
dentro del desarrollo de software.\
No es código específico, sino una guía estructurada que describe:

-   Cuándo aplicar una solución
-   Cómo implementarla correctamente
-   Qué beneficios ofrece
-   Qué problemas evita

Los patrones permiten:

✅ Reducir el acoplamiento\
✅ Aumentar la mantenibilidad\
✅ Facilitar la escalabilidad\
✅ Mejorar la testabilidad\
✅ Separar responsabilidades

------------------------------------------------------------------------

## 🧱 ¿Qué es Arquitectura Hexagonal?

La **Arquitectura Hexagonal (Ports and Adapters)** busca aislar la
lógica de negocio del mundo exterior:

-   Base de datos
-   Frameworks
-   APIs externas
-   Interfaces de usuario

Esto permite que el **Dominio no dependa de la Infraestructura**.

             UI
              |
       -----------------
       |   Application |
       -----------------
          |        |
       Ports    Adapters
          |        |
       Infrastructure

------------------------------------------------------------------------

# 🧩 PATRONES DE DISEÑO

------------------------------------------------------------------------

## 🔹 Service Pattern

Encapsula la lógica de negocio dentro de servicios de aplicación.

### Interfaz

``` csharp
public interface IAccountService
{
    Task CreateAccountAsync(AccountDto account);
}
```

### Implementación

``` csharp
public class AccountService : IAccountService
{
    private readonly IAccountRepository _repository;

    public AccountService(IAccountRepository repository)
    {
        _repository = repository;
    }

    public async Task CreateAccountAsync(AccountDto account)
    {
        var entity = new Account(account.Name, account.Balance);
        await _repository.AddAsync(entity);
    }
}
```

------------------------------------------------------------------------

## 🔹 Repository Pattern

Abstrae el acceso a datos.

### Puerto

``` csharp
public interface IAccountRepository
{
    Task AddAsync(Account account);
    Task<Account?> GetByIdAsync(Guid id);
}
```

### Adaptador

``` csharp
public class AccountRepository : IAccountRepository
{
    private readonly BankingDbContext _context;

    public AccountRepository(BankingDbContext context)
    {
        _context = context;
    }

    public async Task AddAsync(Account account)
    {
        await _context.Accounts.AddAsync(account);
    }

    public async Task<Account?> GetByIdAsync(Guid id)
    {
        return await _context.Accounts.FindAsync(id);
    }
}
```

------------------------------------------------------------------------

## 🔹 Unit Of Work Pattern

Coordina múltiples repositorios bajo una misma transacción.

### Interfaz

``` csharp
public interface IUnitOfWork
{
    IAccountRepository Accounts { get; }
    Task<int> CommitAsync();
}
```

### Implementación

``` csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly BankingDbContext _context;

    public IAccountRepository Accounts { get; }

    public UnitOfWork(BankingDbContext context, IAccountRepository accountRepository)
    {
        _context = context;
        Accounts = accountRepository;
    }

    public async Task<int> CommitAsync()
    {
        return await _context.SaveChangesAsync();
    }
}
```

------------------------------------------------------------------------

## 🔹 Factory Pattern

Permite crear objetos complejos sin exponer su lógica de construcción.

``` csharp
public interface IAccountFactory
{
    Account Create(string name, decimal balance);
}
```

``` csharp
public class AccountFactory : IAccountFactory
{
    public Account Create(string name, decimal balance)
    {
        return new Account(name, balance);
    }
}
```

------------------------------------------------------------------------

## 🔹 Dependency Injection Pattern

Facilita la inversión de dependencias.

``` csharp
builder.Services.AddScoped<IAccountRepository, AccountRepository>();
builder.Services.AddScoped<IAccountService, AccountService>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

------------------------------------------------------------------------

## 🔹 Ports and Adapters Pattern

Define contratos para desacoplar el dominio.

### Puerto de salida

``` csharp
public interface IEmailSender
{
    Task SendAsync(string to, string message);
}
```

### Adaptador SMTP

``` csharp
public class SmtpEmailSender : IEmailSender
{
    public Task SendAsync(string to, string message)
    {
        return Task.CompletedTask;
    }
}
```

------------------------------------------------------------------------

## 🔹 Domain Events Pattern

Permite reaccionar a cambios dentro del dominio.

``` csharp
public record AccountCreatedEvent(Guid AccountId);
```

``` csharp
public class AccountCreatedHandler
{
    public Task Handle(AccountCreatedEvent domainEvent)
    {
        return Task.CompletedTask;
    }
}
```

------------------------------------------------------------------------

# 🎯 Beneficios en Arquitectura Hexagonal

  Beneficio           Descripción
  ------------------- -----------------------
  Bajo acoplamiento   Dominio independiente
  Testabilidad        Fácil mockeo
  Escalabilidad       Nuevos adaptadores
  Mantenibilidad      Cambios aislados

------------------------------------------------------------------------

# 📌 Conclusión

El uso de estos patrones dentro de Arquitectura Hexagonal permite
construir sistemas robustos, mantenibles y desacoplados, especialmente
en aplicaciones empresariales desarrolladas con .NET 10.
