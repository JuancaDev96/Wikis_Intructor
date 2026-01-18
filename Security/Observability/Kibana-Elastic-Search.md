# 🧱 Guía de Implementación de Entorno Docker y Logging Centralizado en .NET 9

**Instructor:** Juan Carlos De La Cruz Ch.

------------------------------------------------------------------------

## 🚀 Descripción General

Este documento detalla la configuración de un entorno completo con
**PostgreSQL**, **Vault**, **Redis**, **CAP**, **Elasticsearch** y
**Kibana**, junto con la integración de **Serilog** en un proyecto
**.NET 9 (Arquitectura Limpia)** para centralizar logs en Elasticsearch
y visualizarlos en Kibana.

------------------------------------------------------------------------

## 🐋 1️⃣ Configuración del `docker-compose.yml`

Guarda el siguiente archivo en la raíz de tu proyecto:

``` yaml
version: '3.9'

services:
  
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.0
    container_name: elasticsearch_security
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - xpack.security.http.ssl.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports:
      - "9200:9200"
    volumes:
      - C:\Users\juanc\OneDrive\Documentos\Docker\Volumenes\elasticsearch_security:/usr/share/elasticsearch/data
    networks:
      - galaxy_network

  kibana:
    image: docker.elastic.co/kibana/kibana:8.15.0
    container_name: kibana_security
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - galaxy_network

networks:
  galaxy_network:
    driver: bridge
```

### 🔌 Puertos Expuestos

  Servicio        Puerto Host   Puerto Interno
  --------------- ------------- ----------------
  PostgreSQL      1500          5432
  Vault           8200          8200
  Redis           6379          6379
  CAP             3000          3000
  Elasticsearch   9200          9200
  Kibana          5601          5601

------------------------------------------------------------------------

## ⚙️ 2️⃣ Integración de Logging con Serilog y Elasticsearch en .NET 9

### a) Instalación de Paquetes NuGet

Ejecuta en la consola dentro del proyecto API o Infrastructure:

``` bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Elasticsearch
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Settings.Configuration
```

------------------------------------------------------------------------

### b) Creación del archivo `SerilogExtensions.cs`

Ubica este archivo dentro de **Infrastructure/Extensions**:

``` csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Serilog;
using Serilog.Sinks.Elasticsearch;
using System;

namespace Galaxy.Security.Infrastructure.Extensions
{
    public static class SerilogExtensions
    {
        public static void AddSerilogElastic(this IServiceCollection services, IConfiguration configuration)
        {
            var elasticUri = configuration["ElasticConfiguration:Uri"] ?? "http://localhost:9200";

            Log.Logger = new LoggerConfiguration()
                .Enrich.FromLogContext()
                .WriteTo.Console()
                .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri(elasticUri))
                {
                    AutoRegisterTemplate = true,
                    IndexFormat = $"galaxy-security-logs-{DateTime.UtcNow:yyyy-MM}"
                })
                .Enrich.WithProperty("Application", "Galaxy.Security.API")
                .CreateLogger();

            services.AddLogging(loggingBuilder =>
            {
                loggingBuilder.ClearProviders();
                loggingBuilder.AddSerilog(dispose: true);
            });
        }
    }
}
```

------------------------------------------------------------------------

### c) Configuración en `appsettings.json`

``` json
{
  "ElasticConfiguration": {
    "Uri": "http://localhost:9200"
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    }
  }
}
```

------------------------------------------------------------------------

### d) Registro en `Program.cs`

``` csharp
using Galaxy.Security.API.Middlewares;
using Galaxy.Security.Infrastructure.Extensions;

var builder = WebApplication.CreateBuilder(args);

// 🔹 Agregar Serilog y Elasticsearch
builder.Services.AddSerilogElastic(builder.Configuration);

// 🔹 Otros servicios de la arquitectura limpia
// builder.Services.AddApplication();
// builder.Services.AddInfrastructure(builder.Configuration);

var app = builder.Build();

// 🔹 Middleware global de excepciones
app.UseExceptionHandlingMiddleware();

app.MapControllers();

app.Run();
```

------------------------------------------------------------------------

## 🧩 3️⃣ Middleware de Manejo de Excepciones

Tu middleware actual manejará los errores de dominio, autenticación y
excepciones generales, registrando los eventos en **Elasticsearch**
gracias a la integración de Serilog.

Los logs se enviarán automáticamente a la consola y al índice:

    galaxy-security-logs-YYYY-MM

------------------------------------------------------------------------

## 📊 4️⃣ Visualización en Kibana

1.  Abre Kibana en tu navegador: <http://localhost:5601>

2.  Navega a: **Management → Index Patterns → Create Index Pattern**

3.  Crea un nuevo patrón con el nombre:

        galaxy-security-logs-*

4.  Selecciona `@timestamp` como campo de tiempo.

Podrás visualizar los logs con los siguientes campos: - `message` -
`level` - `Application` - `exception` - `timestamp`

------------------------------------------------------------------------

## ✅ 5️⃣ Resultado Final

-   Logs centralizados en Elasticsearch, visualizados desde Kibana.
-   Middleware de excepciones funcional con Serilog.
-   Servicios de infraestructura completos para seguridad, caché,
    mensajería y monitoreo.
-   Persistencia local garantizada mediante volúmenes Docker en Windows.

---

## 📊 ElasticSearch, Kibana y Observabilidad

### 🔍 ¿Qué es ElasticSearch?
ElasticSearch es un **motor de búsqueda y análisis de texto completo** basado en **Lucene**.  
Se utiliza para almacenar, buscar y analizar grandes volúmenes de datos en tiempo real.  
Es ideal para **logs**, **métricas** y **monitorización de aplicaciones**.

**Características principales:**
- Almacenamiento de datos en formato JSON.
- Búsquedas rápidas y precisas.
- Escalabilidad horizontal (clusterización).
- API RESTful nativa.

### 📈 ¿Qué es Kibana?
Kibana es una herramienta visual que trabaja sobre ElasticSearch.  
Permite **explorar, visualizar y analizar datos** de manera interactiva.  

**Usos comunes:**
- Dashboards de monitoreo.
- Visualización de logs.
- Análisis de tendencias y seguridad.
- Alertas basadas en métricas.

### 🧠 Observabilidad: concepto clave
La **observabilidad** es la capacidad de comprender el estado interno de un sistema a partir de los datos que genera.  
Incluye tres pilares fundamentales:
1. **Logs:** Registros detallados de eventos.  
2. **Métricas:** Indicadores numéricos sobre el rendimiento del sistema.  
3. **Traces (trazas):** Seguimiento del flujo de una solicitud a través de los componentes del sistema.

**Beneficios:**
- Mejora el tiempo de respuesta ante incidentes.  
- Facilita la detección temprana de fallos.  
- Incrementa la eficiencia operativa.

### 🔒 Impacto en la seguridad
La observabilidad también tiene un **rol crítico en la ciberseguridad**.  
Permite detectar comportamientos anómalos, accesos no autorizados o actividades sospechosas mediante el análisis de logs y métricas.

**Ejemplos de uso:**
- Monitoreo de intentos de acceso fallidos.  
- Análisis de patrones de tráfico inusual.  
- Correlación de eventos en tiempo real.  
- Generación de alertas automáticas mediante Elastic Stack (ELK).

---

## 📦 Conclusiones

- Docker permite aislar entornos de desarrollo, garantizando consistencia entre equipos.  
- PostgreSQL y pgAdmin facilitan la administración de bases de datos dentro de contenedores.  
- ElasticSearch y Kibana agregan valor en la observación y seguridad del sistema.  
- La observabilidad es un componente esencial de la **operación moderna** y la **seguridad proactiva**.

---


**Instructor:** Juan Carlos De La Cruz Ch.\
© 2025 -- Curso de Arquitectura Limpia y Seguridad en .NET 9
