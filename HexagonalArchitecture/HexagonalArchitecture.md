
# 📘 Wiki de Arquitectura de Software, Arquitectura Hexagonal y DDD  
**Instructor:** Juan Carlos de la Cruz  

---

## 🎯 Objetivo de esta Wiki

Este documento tiene como objetivo **explicar, de forma profunda y aplicada**, los conceptos fundamentales de:

- Arquitectura de Software
- Arquitectura Hexagonal (Ports & Adapters)
- Patrones de Diseño
- Domain-Driven Design (DDD)

con un **énfasis especial en DDD y Arquitectura Hexagonal**, mostrando **cómo se aplican en el mundo real** y **cómo se implementan en .NET** con ejemplos claros y alineados al negocio.

---

# 🧱 1. Arquitectura de Software

## 1.1 ¿Qué es Arquitectura de Software?

La **arquitectura de software** es el conjunto de **decisiones estructurales** que definen:

- Cómo se organiza el sistema
- Cómo se comunican sus partes
- Qué responsabilidades tiene cada componente
- Qué reglas no deben romperse

> La arquitectura no es código,  
> es el **marco que guía todo el código**.

### Problemas que busca resolver
- Complejidad
- Acoplamiento excesivo
- Cambios costosos
- Sistemas difíciles de mantener

---

## 1.2 Arquitectura vs Diseño

| Arquitectura | Diseño |
|--------------|--------|
| Visión global | Detalles locales |
| Define límites | Implementa soluciones |
| Decide estructura | Usa patrones |
| Larga duración | Cambia con frecuencia |

---

# 🧩 2. Patrones de Diseño

## 2.1 ¿Qué es un Patrón de Diseño?

Un **patrón de diseño** es una **solución probada** a un problema recurrente en el diseño de software.

No es:
- Una librería
- Un framework
- Código copiado

Es:
- Una **idea reusable**
- Una forma de pensar

---

## 2.2 Patrón Repository (aplicado a DDD)

### Problema
El dominio **no debe saber** cómo se guardan los datos.

### Solución
Definir un contrato en el dominio.

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id);
    Task SaveAsync(Order order);
}
```

📌 El dominio **define el qué**,  
📌 La infraestructura define el **cómo**.

---

# 🧠 3. Domain-Driven Design (DDD)

## 3.1 ¿Qué es DDD?

DDD es un **enfoque de diseño de software** que pone el **conocimiento del negocio** en el centro del sistema.

> Si el software no refleja el negocio,  
> el software está mal diseñado.

DDD no es:
- Un framework
- Una arquitectura
- Una moda

DDD es:
- Modelado
- Comunicación
- Diseño consciente

---

## 3.2 Dominio

El **dominio** es el área del conocimiento que el software busca resolver.

Ejemplo:
- Ventas
- Facturación
- Logística

El dominio contiene:
- Reglas
- Procesos
- Restricciones reales

---

## 3.3 Lenguaje Ubicuo

El **lenguaje ubicuo** es un vocabulario compartido entre:
- Negocio
- Desarrollo
- Documentación
- Código

Ejemplo de negocio:
> “Un pedido no puede confirmarse si no tiene ítems.”

Ejemplo en código:
```csharp
order.Confirm();
```

---

## 3.4 Entidades

### Concepto

Una **Entidad** es un objeto que:
- Tiene identidad
- Vive en el tiempo
- Cambia de estado

La identidad es más importante que sus atributos.

```csharp
public class Order
{
    public OrderId Id { get; }
    public OrderStatus Status { get; private set; }
}
```

---

## 3.5 Value Objects

### Concepto

Un **Value Object**:
- No tiene identidad
- Es inmutable
- Se compara por valor

Ejemplo de negocio:
> Dinero, Dirección, Rango de Fechas

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");

        return new Money(Amount + other.Amount, Currency);
    }
}
```

---

## 3.6 Aggregates

### Concepto Formal

Un **Aggregate** es un **límite de consistencia** que agrupa:
- Aggregate Root
- Entidades internas
- Value Objects
- Reglas de negocio

Todo lo que está dentro del aggregate:
- Se valida junto
- Se persiste junto
- Cambia en una sola transacción

---

## 3.7 Aggregate Root

### Concepto

El **Aggregate Root** es:
- La única puerta de entrada al aggregate
- El guardián de las reglas
- El responsable de la consistencia

Desde afuera:
- Solo se referencia al root
- Nunca a entidades internas

---

### Ejemplo de Aggregate (Order)

#### Vista conceptual
```
Order (Aggregate Root)
 ├── OrderItem (Entidad)
 │    ├── ProductId (VO)
 │    └── Money (VO)
 └── Reglas de negocio
```

#### Implementación

```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();

    public OrderId Id { get; }
    public OrderStatus Status { get; private set; }

    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    public void AddItem(ProductId productId, Money price, int quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Cannot modify confirmed order");

        _items.Add(new OrderItem(productId, price, quantity));
    }

    public void Confirm()
    {
        if (!_items.Any())
            throw new InvalidOperationException("Order cannot be empty");

        Status = OrderStatus.Confirmed;
    }
}
```

---

## 3.8 Bounded Context

### Concepto

Un **Bounded Context** define un **límite explícito** donde:
- Un modelo es válido
- Un lenguaje tiene significado

Ejemplo:
- Sales.Order
- Billing.Invoice
- Shipping.Shipment

Cada contexto:
- Tiene su propio modelo
- No comparte entidades

---

# 🔷 4. Arquitectura Hexagonal

## 4.1 Concepto

La **Arquitectura Hexagonal** propone que:
- El dominio esté en el centro
- Nada externo lo contamine
- Todo acceso pase por puertos

> El dominio no depende de nada.  
> Todo depende del dominio.

---

## 4.2 Puertos

Los **puertos** son interfaces que define el dominio.

```csharp
public interface IOrderRepository
{
    Task SaveAsync(Order order);
}
```

---

## 4.3 Adaptadores

Los **adaptadores** implementan los puertos.

```csharp
public class OrderRepository : IOrderRepository
{
    public Task SaveAsync(Order order)
    {
        // EF Core / SQL Server
        return Task.CompletedTask;
    }
}
```

---

## 4.4 Flujo Completo

1. API recibe request
2. Application ejecuta caso de uso
3. Dominio valida reglas
4. Infraestructura persiste
5. Respuesta al cliente

---

# 📁 5. Estructura de Solución

```
/src
 ├── Domain
 ├── Application
 ├── Infrastructure
 └── Api
```

---

# 🚫 6. Anti-patterns Comunes

- Dominio con EF Core
- Entidades anémicas
- Aggregates gigantes
- Repositorios por entidad

---

# 🏁 Conclusión

DDD + Arquitectura Hexagonal permiten construir:
- Software alineado al negocio
- Sistemas evolutivos
- Código mantenible y testeable

---

**Instructor:** Juan Carlos de la Cruz  
**Documento:** Wiki Técnica  
