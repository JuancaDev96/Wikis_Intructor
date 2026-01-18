# Despliegue con Docker en Arquitecturas .NET

**Instructor:** Juan Carlos De La Cruz Chinga

---

## 📌 Introducción

En arquitecturas modernas, el **despliegue** deja de ser una tarea operativa aislada y se convierte en una **decisión arquitectónica estratégica**.

Docker permite que una solución:
- Sea **reproducible**
- Sea **portable**
- Escale sin fricción
- Se integre naturalmente con **CI/CD**, **microservicios** y **observabilidad**

Este documento introduce los conceptos clave de **Docker**, **Dockerfile**, **Docker Compose**, **CI/CD** y cómo todo esto potencia arquitecturas .NET robustas y escalables, tomando como referencia una solución real.

---

## 🐳 ¿Qué es Docker?

**Docker** es una plataforma de contenedorización que permite empaquetar una aplicación junto con:
- Su runtime
- Dependencias
- Configuración

Todo dentro de una **unidad autocontenida** llamada *contenedor*.

### Beneficios clave
- "Funciona en mi máquina" deja de existir
- Mismo artefacto en DEV, QA y PROD
- Arranque rápido
- Aislamiento entre servicios

---

## 📦 Contenedores vs Máquinas Virtuales

| Contenedores | Máquinas Virtuales |
|-------------|-------------------|
| Ligeros | Pesados |
| Arranque en segundos | Arranque lento |
| Comparten kernel | Kernel por VM |
| Ideales para microservicios | Ideales para monolitos legacy |

---

## 🧱 ¿Qué es un Dockerfile?

Un **Dockerfile** define cómo se construye una imagen.

En .NET, normalmente se usa **multi-stage build** para:
- Compilar
- Publicar
- Ejecutar

### Ejemplo real (.NET 10)

```dockerfile
# Etapa de build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY ["Galaxy.Pedidos.UI/Galaxy.Pedidos.UI.csproj", "Galaxy.Pedidos.UI/"]
COPY ["Galaxy.Pedidos.DataAccess/Galaxy.Pedidos.DataAccess.csproj", "Galaxy.Pedidos.DataAccess/"]
COPY ["Galaxy.Pedidos.Business/Galaxy.Pedidos.Business.csproj", "Galaxy.Pedidos.Business/"]
COPY ["Galaxy.Pedidos.Repositories/Galaxy.Pedidos.Repositories.csproj", "Galaxy.Pedidos.Repositories/"]
COPY ["Galaxy.Pedidos.Common/Galaxy.Pedidos.Common.csproj", "Galaxy.Pedidos.Common/"]
COPY ["Galaxy.Pedidos.DTO/Galaxy.Pedidos.DTO.csproj", "Galaxy.Pedidos.DTO/"]
COPY . .

RUN dotnet restore "Galaxy.Pedidos.UI/Galaxy.Pedidos.UI.csproj"
RUN dotnet publish "Galaxy.Pedidos.UI/Galaxy.Pedidos.UI.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Runtime
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
ENV ASPNETCORE_URLS=http://+:80
COPY --from=build /app/publish .
EXPOSE 80
ENTRYPOINT ["dotnet", "Galaxy.Pedidos.UI.dll"]
```

---

## 🧩 ¿Qué es Docker Compose?

**Docker Compose** permite definir y orquestar **múltiples servicios** que conforman una arquitectura completa.

Se define mediante un archivo `docker-compose.yml`.

---

## 🏗️ Arquitectura Desplegada con Docker Compose

Servicios involucrados:

- SQL Server (datos de negocio)
- PostgreSQL (seguridad / autenticación)
- Seq (observabilidad)
- API / UI .NET

Todos conectados mediante una **red interna compartida**.

---

## 🐳 Docker Compose (Arquitectura Real)

```yaml
services:
  sqlserver_arq_capas:
    image: mcr.microsoft.com/mssql/server:2019-latest
    container_name: bd_arq_capas
    restart: always
    environment:
      MSSQL_SA_PASSWORD: Password2025
      MSSQL_PID: Express
      ACCEPT_EULA: Y
    volumes:
      - C:/Users/juanc/OneDrive/Documentos/Docker/Volumenes/layared_volumen:/var/opt/mssql/data
    ports:
      - "1402:1433"
    networks:
      - layared_network

  postgres_arq_capas:
    image: postgres:latest
    container_name: bd_seguridad_arq_capas
    restart: always
    environment:
      POSTGRES_DB: bdseguridad
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password2025
    volumes:
      - C:/Users/juanc/OneDrive/Documentos/Docker/Volumenes/layared_postgres_volumen:/var/lib/postgresql/data
    ports:
      - "1403:5432"
    networks:
      - layared_network

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
      - C:/Users/juanc/OneDrive/Documentos/Docker/Volumenes/seq_volumen:/data
    networks:
      - layared_network

  galaxy_pedidos_ui:
    build:
      context: .
      dockerfile: Dockerfile
    image: galaxy.pedidos.ui:1.0.0
    container_name: galaxy_pedidos_ui
    restart: unless-stopped
    environment:
      ASPNETCORE_ENVIRONMENT: Development
    ports:
      - "5000:80"
    depends_on:
      - postgres_arq_capas
    networks:
      - layared_network

networks:
  layared_network:
    driver: bridge
```

---

## 🔁 Integración Continua (CI)

**CI** valida automáticamente:
- Compilación
- Tests
- Build de imagen Docker

Cada commit:
- Reduce errores humanos
- Asegura calidad
- Genera artefactos listos para despliegue

---

## 🚀 Despliegue Continuo (CD)

**CD** automatiza:
- Publicación de imágenes
- Despliegue a entornos
- Versionado

Docker permite que el mismo contenedor:
- Pase de DEV → QA → PROD

---

## 🌐 Docker como Base para Microservicios

Docker habilita naturalmente:
- Separación por servicio
- Escalado independiente
- Aislamiento de fallos

Cada contenedor se convierte en una **unidad de despliegue**.

---

## 📈 Camino de Evolución

| Etapa | Tecnología |
|-----|-----------|
| Inicial | Docker + Compose |
| Intermedia | CI/CD Pipelines |
| Avanzada | Kubernetes / ECS / AKS |

Docker es el **primer gran paso**.

---

## 🧠 Visión Arquitectónica

Contenerizar no es solo una decisión técnica:

- Reduce fricción entre equipos
- Acelera entregas
- Prepara el sistema para escalar

Docker abre la puerta a:
- Observabilidad
- Automatización
- Resiliencia

---

## 🧑‍🏫 Rol del Instructor

Como instructor, el objetivo es:
- Enseñar fundamentos sólidos
- Construir criterio
- Preparar arquitecturas reales

Docker no es una moda:
> Es el lenguaje estándar del despliegue moderno.

---

## ✅ Conclusión

- Docker estandariza despliegues
- Docker Compose orquesta arquitecturas completas
- CI/CD se vuelve natural
- El sistema queda listo para microservicios y escala

Diseñar pensando en despliegue es diseñar pensando en el futuro.

---

📘 *Documento elaborado por Juanca Dev*

