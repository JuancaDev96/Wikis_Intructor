# Observabilidad con Seq en Arquitecturas .NET

**Instructor:** Juan Carlos De La Cruz

---

## 📌 Introducción

En sistemas modernos, especialmente aquellos orientados a **arquitecturas distribuidas, escalables y basadas en microservicios**, la observabilidad deja de ser un lujo y se convierte en una **necesidad crítica**.

Este documento tiene como objetivo explicar:
- Qué es la **observabilidad**
- Por qué es clave en sistemas robustos
- Cómo **Seq** actúa como una puerta de entrada simple y poderosa
- Cómo integrar Seq con **.NET 10 + Serilog**
- Cómo esta base prepara el camino hacia arquitecturas distribuidas

Este material sigue la misma línea conceptual que los documentos de **arquitectura en capas** vistos previamente.

---

## 🔍 ¿Qué es Observabilidad?

La **observabilidad** es la capacidad de un sistema para permitirnos entender **qué está ocurriendo internamente**, basándonos únicamente en lo que el sistema expone.

Se apoya en tres pilares fundamentales:

### 1️⃣ Logs
Eventos que describen lo que ocurre en el sistema.

### 2️⃣ Métricas
Valores numéricos que representan el estado del sistema (CPU, latencia, throughput, etc.).

### 3️⃣ Trazas (Tracing)
Seguimiento de una solicitud a través de múltiples componentes o servicios.

> ❗ Sin observabilidad, depurar un sistema distribuido es como intentar arreglar un avión en pleno vuelo con los ojos vendados.

---

## 🧠 ¿Por qué la Observabilidad es Clave en Sistemas Robustos?

Un sistema robusto debe:
- Detectar errores rápidamente
- Permitir análisis post-mortem
- Reducir el MTTR (Mean Time To Recovery)
- Escalar sin perder visibilidad

Sin observabilidad:
- Los errores solo se ven cuando el usuario se queja
- El debugging depende de reproducir escenarios
- El crecimiento del sistema genera caos

Con observabilidad:
- Los errores son visibles antes de impactar al negocio
- Se identifican cuellos de botella
- Se toman decisiones basadas en datos reales

---

## 🧩 Seq como Ventana de Observabilidad

### ¿Qué es Seq?

**Seq** es una plataforma de ingestión y visualización de **logs estructurados**, diseñada especialmente para el ecosistema .NET.

### ¿Por qué Seq?

- Integración nativa con **Serilog**
- Curva de aprendizaje baja
- UI intuitiva
- Búsqueda avanzada por propiedades
- Ideal como primer paso hacia observabilidad completa

Seq se convierte en la **ventana inicial** para observar el comportamiento del sistema.

---

## 🧱 Seq dentro de una Arquitectura en Capas

En una arquitectura por capas:

- **Presentación** → genera solicitudes
- **Aplicación** → orquesta reglas de negocio
- **Dominio** → define comportamientos
- **Infraestructura** → ejecuta persistencia y servicios externos

Seq actúa de forma **transversal**, recolectando información de todas las capas sin acoplarse a ninguna.

```text
[ API ]
   ↓
[ Application ]  → Logs estructurados → Seq
   ↓
[ Domain ]
   ↓
[ Infrastructure ]
```

---

## 🐳 Integración de Seq con Docker Compose

```yaml
services:
  seq:
    image: datalust/seq:latest
    container_name: seq
    restart: unless-stopped
    environment:
      - ACCEPT_EULA=Y
    ports:
      - "5341:80"
    volumes:
      - seq-data:/data

volumes:
  seq-data:
```

- UI disponible en `http://localhost:5341`
- Persistencia de logs garantizada

---

## ⚙️ Integración con .NET 10 + Serilog

### Paquetes necesarios

- Serilog.AspNetCore
- Serilog.Sinks.Seq
- Serilog.Settings.Configuration

### Configuración en Program.cs

```csharp
builder.Host.UseSerilog((context, services, configuration) =>
{
    configuration
        .ReadFrom.Configuration(context.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithProperty("Service", "Pedidos.API")
        .Enrich.WithProperty("Environment", context.HostingEnvironment.EnvironmentName)
        .WriteTo.Console()
        .WriteTo.Seq(context.Configuration["Seq:ServerUrl"]);
});
```

### appsettings.json

```json
{
  "Seq": {
    "ServerUrl": "http://localhost:5341"
  }
}
```

---

## 🧾 Logs Estructurados: la Clave Real

```csharp
Log.Information(
    "Pedido creado {PedidoId} para cliente {ClienteId} por {Total}",
    pedido.Id,
    pedido.ClienteId,
    pedido.Total
);
```

Esto permite:
- Filtrar por PedidoId
- Correlacionar eventos
- Crear dashboards funcionales

---

## 🌐 Seq como Puerta a Sistemas Distribuidos

Cuando el sistema crece:
- Microservicios
- Mensajería
- APIs externas

Seq permite:
- Correlación por TraceId
- Visualización por servicio
- Identificación de fallos en cascada

Es el **primer paso natural** antes de integrar:
- OpenTelemetry
- Grafana
- Prometheus
- Jaeger / Tempo

---

## 🔐 Seguridad y Autenticación en Seq

A partir de versiones recientes, **Seq exige una configuración explícita de seguridad en el primer arranque**. Esto refuerza el concepto de que la observabilidad también forma parte de la **superficie de seguridad del sistema**.

Si no se define correctamente, Seq **no iniciará** y mostrará un error similar a:

```text
No default admin password was supplied
```

### 🧠 ¿Por qué ocurre esto?

Seq requiere que el arquitecto o equipo tome una decisión consciente:
- Definir una contraseña de administrador
- O indicar explícitamente que no habrá autenticación (solo recomendado para desarrollo)

Esto evita exponer información sensible por error.

---

### 🧪 Opción 1: Desactivar autenticación (DEV / LOCAL)

Recomendada **únicamente para entornos locales o de aprendizaje**.

```yaml
services:
  seq:
    image: datalust/seq:latest
    container_name: seq
    restart: unless-stopped
    environment:
      - ACCEPT_EULA=Y
      - SEQ_FIRSTRUN_NOAUTHENTICATION=true
    ports:
      - "5341:80"
    volumes:
      - seq-data:/data
```

Con esta configuración:
- Seq arranca sin login
- Acceso directo a la UI
- Ideal para desarrollo rápido

---

### 🔐 Opción 2: Definir password admin (PRODUCCIÓN)

Recomendada para **entornos productivos y corporativos**.

```yaml
services:
  seq:
    image: datalust/seq:latest
    container_name: seq
    restart: unless-stopped
    environment:
      - ACCEPT_EULA=Y
      - SEQ_FIRSTRUN_ADMINPASSWORD=SuperPasswordSegura123!
    ports:
      - "5341:80"
    volumes:
      - seq-data:/data
```

Credenciales iniciales:
- **Usuario:** admin
- **Password:** definida en la variable

📌 En producción, esta contraseña debe gestionarse mediante:
- Secrets
- Parameter Store
- Variables seguras del orquestador

---

### 🧹 Limpieza de Volúmenes (First Run)

Si Seq ya intentó arrancar sin estas variables, es necesario limpiar el volumen:

```bash
docker compose down -v
docker compose up -d
```

Esto garantiza que Seq vuelva a ejecutar el proceso de inicialización.

---

### 🛡️ Observabilidad y Seguridad

Los logs pueden contener:
- Identificadores de usuarios
- Información de negocio
- Errores internos

Por ello:
- Seq **no debe exponerse sin autenticación en PROD**
- La seguridad forma parte del diseño arquitectónico

---

## 🚀 Evolución de la Observabilidad

| Nivel | Herramienta |
|-----|------------|
| Básico | Seq + Serilog |
| Intermedio | Seq + Grafana |
| Avanzado | OpenTelemetry Stack |

Nada se pierde, todo se acumula.

---

## 🧑‍🏫 Rol del Instructor

Como instructor, el objetivo no es solo enseñar herramientas, sino:
- Formar criterio arquitectónico
- Entender el *por qué* antes del *cómo*
- Preparar sistemas que escalen sin dolor

Seq no es solo un visor de logs:
> Es la primera ventana a la salud real del sistema.

---

## ✅ Conclusión

- La observabilidad es obligatoria en sistemas modernos
- Seq ofrece una entrada simple y poderosa
- Su integración con .NET es natural
- Prepara el camino hacia arquitecturas distribuidas

Implementar observabilidad desde el inicio es una **decisión arquitectónica**, no técnica.

---

📘 *Documento elaborado por Juanca Dev*

