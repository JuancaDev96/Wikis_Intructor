# 📘 Domain Events e Integration Events en .NET

## Arquitectura Hexagonal, DDD y Sistemas Event‑Driven

👨‍🏫 **Instructor:** Juan Carlos De La Cruz\
🎯 **Objetivo:** Comprender profundamente el uso de **eventos en
arquitecturas modernas** utilizando .NET.

------------------------------------------------------------------------

# 🌍 1. Introducción

En sistemas modernos basados en:

-   Arquitectura Hexagonal
-   Domain Driven Design (DDD)
-   Microservicios
-   Event Driven Architecture

los **eventos** permiten construir sistemas **desacoplados, escalables y
mantenibles**.

Un **evento** representa:

> 📢 Algo importante que ocurrió dentro del sistema.

Ejemplos reales:

-   🧑 Usuario registrado
-   💳 Pago realizado
-   🏦 Préstamo creado
-   📦 Pedido generado
-   📧 Email enviado

En lugar de que los módulos se llamen directamente entre sí, el sistema
puede **emitir un evento** y permitir que otros componentes reaccionen.

Esto produce:

✅ Bajo acoplamiento\
✅ Alta extensibilidad\
✅ Sistemas reactivos\
✅ Arquitecturas distribuidas

------------------------------------------------------------------------

# 🧠 2. Concepto Fundamental

En arquitecturas modernas existen **dos tipos principales de eventos**.

| Tipo de Evento | Propósito |
| :--- | :--- |
| **Domain Event** | Representa algo importante que ocurrió dentro del dominio (interno). |
| **Integration Event** | Comunica un hecho hacia otros sistemas o microservicios (externo). |

⚠️ **No son lo mismo y tienen responsabilidades distintas.**

------------------------------------------------------------------------

# 🏛️ 3. Domain Events (Eventos de Dominio)

## 📌 ¿Qué son? (La Campana Interna)
Un **Domain Event** es un hecho que ocurrió dentro de tu lógica de negocio y que es importante para el sistema, pero **solo de forma interna**.

> 💡 **Analogía:** Imagina que en un banco, cuando se crea un préstamo, suena una **campana interna**. Esa campana avisa a los otros departamentos del banco (como contabilidad o notificaciones) para que actúen, pero nadie fuera del banco escucha esa campana.

### ¿Por qué usarlos?
1.  **Consistencia:** Permite que una acción desencadene otras de forma organizada.
2.  **Desacoplamiento:** El código que crea el préstamo no necesita saber cómo se envía un email o cómo se genera un cronograma de pagos.
3.  **Lenguaje Ubicuo:** Los eventos usan el lenguaje del negocio (`PrestamoCreado`, `PagoRealizado`).

------------------------------------------------------------------------

# 🏦 4. Ejemplo Didáctico: El Flujo del Préstamo

Imagina este proceso cuando un cliente solicita un préstamo:

1.  **Acción:** Se crea el préstamo en el sistema.
2.  **Evento:** Se dispara `LoanCreatedDomainEvent`.
3.  **Reacciones Internas:**
    *   **Contabilidad:** Registra el asiento contable.
    *   **Email Service:** Prepara un correo de bienvenida.
    *   **Gestor de Riesgos:** Actualiza el perfil del cliente.

Si mañana quieres añadir un "Logs de Auditoría", solo creas un nuevo **Handler** que escuche el mismo evento. **¡No tocas el código original del préstamo!**

------------------------------------------------------------------------

# 🧱 5. Estructura Base (Código Esencial)

Para implementar esto, necesitamos una base mínima:

```csharp
public interface IDomainEvent { DateTime OccurredOn { get; } }

public abstract class BaseEntity 
{
    private readonly List<IDomainEvent> _events = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _events;

    protected void AddEvent(IDomainEvent @event) => _events.Add(@event);
    public void ClearEvents() => _events.Clear();
}
```

# 🚀 6. Creación y Emisión del Evento

Así se ve un evento y cómo la entidad lo genera:

```csharp
// El Evento
public record LoanCreatedEvent(Guid LoanId, Guid CustomerId) : IDomainEvent 
{
    public DateTime OccurredOn => DateTime.UtcNow;
}

// La Entidad (en la Capa de Dominio)
public class Loan : BaseEntity 
{
    public static Loan Create(Guid customerId, decimal amount) 
    {
        var loan = new Loan(customerId, amount);
        // El dominio "grita" que algo pasó
        loan.AddEvent(new LoanCreatedEvent(loan.Id, customerId));
        return loan;
    }
}
```

------------------------------------------------------------------------

# ⚙️ 7. Procesando el Evento (Handlers)

El **Handler** es quien reacciona. Es como el empleado que escucha la campana y sabe qué hacer.

```csharp
public class LoanCreatedHandler : IDomainEventHandler<LoanCreatedEvent>
{
    public async Task HandleAsync(LoanCreatedEvent @event)
    {
        // Ejemplo: Guardar en logs o preparar un Integration Event
        Console.WriteLine($"Procesando préstamo: {@event.LoanId}");
    }
}
```

------------------------------------------------------------------------

# 🌐 8. Integration Events (Eventos de Integración)

## 📌 ¿Qué son? (La Carta al Exterior)
Mientas que el Domain Event es una "campana interna", un **Integration Event** es una **carta enviada a otro sistema**. 

Se usan para comunicar que algo importante pasó a otros microservicios o sistemas externos que no comparten la misma base de datos.

> 💡 **Analogía:** Una vez que el banco aprobó tu préstamo (campana interna), el banco envía una carta a la Notaría para que preparen los documentos. La Notaría es un sistema externo.

------------------------------------------------------------------------

# 🐇 9. RabbitMQ: El Mensajero
Para enviar estas "cartas", usamos un **Message Broker** como **RabbitMQ**.

### Conceptos Clave (Didáctico):
1.  **Producer (Productor):** Tu aplicación .NET enviando la carta.
2.  **Exchange (Intercambiador):** La oficina de correos que decide a dónde va la carta.
3.  **Queue (Cola):** El buzón donde la carta espera a ser leída.
4.  **Consumer (Consumidor):** El otro sistema que lee la carta y actúa.

### Ejemplo de flujo con RabbitMQ:
```
[App .NET] -> (Publica: LoanCreatedIntegrationEvent) -> [RabbitMQ Exchange] -> [Queue] -> [App Notaría]
```

------------------------------------------------------------------------

# 🧱 10. Implementación Concisa (RabbitMQ)

Un Integration Event es un objeto plano (DTO) que viaja por la red.

```csharp
// El Evento de Integración
public record LoanCreatedIntegrationEvent(Guid LoanId, decimal Amount);

// Publicando el evento (Usando una abstracción como MassTransit o RabbitMQ Client)
public class IntegrationEventService
{
    private readonly IBus _bus; // Ejemplo con MassTransit

    public async Task PublishAsync(LoanCreatedIntegrationEvent @event)
    {
        // Esto envía el mensaje a RabbitMQ
        await _bus.Publish(@event);
    }
}
```

------------------------------------------------------------------------

# 🔁 11. El Gran Cuadro (Flujo Completo)

1.  **Dominio:** Se crea el `Loan` -> Se dispara `LoanCreatedDomainEvent`.
2.  **Handler Interno:** Escucha el evento de dominio y decide que otros sistemas deben saberlo.
3.  **Infraestructura:** El Handler llama al servicio de RabbitMQ y publica `LoanCreatedIntegrationEvent`.
4.  **Exterior:** RabbitMQ entrega el mensaje a quien esté interesado.

------------------------------------------------------------------------

# 🎯 12. Conclusión

*   **Domain Events:** Para sincronizar cosas **dentro** de tu propio microservicio (Capa de Aplicación/Dominio).
*   **Integration Events:** Para avisar al **mundo exterior** (Capa de Infraestructura).
*   **RabbitMQ:** Es la infraestructura que hace posible que los mensajes lleguen a su destino de forma segura y asíncrona.

---
👨‍🏫 **Guía mejorada por Juanca Dev** | *Arquitecturas Limpias y Reactivas*
