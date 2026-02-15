# 🐳 Docker para Implementaciones Modernas

**Instructor:** Juan Carlos de la Cruz

------------------------------------------------------------------------

## 📌 ¿Qué es Docker?

Docker es una plataforma de contenedorización que permite empaquetar
aplicaciones junto con todas sus dependencias en una unidad estándar
llamada **contenedor**, garantizando que se ejecuten de manera
consistente en cualquier entorno.

<img width="800" height="939" alt="image" src="https://github.com/user-attachments/assets/74199c1b-79b4-4bc1-be09-9f020986ca01" />

------------------------------------------------------------------------

## 🚀 Impacto en Implementaciones Modernas

Docker ha revolucionado la forma en que se desarrollan, prueban e
implementan aplicaciones:

-   🔁 Portabilidad entre entornos
-   ⚡ Despliegues rápidos
-   🧪 Consistencia entre DEV, QA y PROD
-   📦 Aislamiento de dependencias
-   🧱 Facilita arquitecturas basadas en microservicios
-   🔄 Integración con CI/CD

------------------------------------------------------------------------

## 🧱 Arquitectura de Docker

  Componente         Descripción
  ------------------ -----------------------------------
  Docker Engine      Motor que ejecuta contenedores
  Docker Image       Plantilla para crear contenedores
  Docker Container   Instancia de una imagen
  Dockerfile         Script para construir imágenes
  Docker Registry    Repositorio de imágenes
  Docker Compose     Orquestador multi-contenedor

------------------------------------------------------------------------

# 📜 COMANDOS MÁS USADOS

## 📦 Gestión de imágenes

``` bash
docker build -t miapp:1.0 .
docker images
docker rmi miapp:1.0
docker pull nginx
docker push usuario/miapp:1.0
```

## 📦 Gestión de contenedores

``` bash
docker run -d -p 8080:80 nginx
docker ps
docker ps -a
docker stop <container_id>
docker start <container_id>
docker rm <container_id>
```

## 🔎 Inspección

``` bash
docker logs <container_id>
docker inspect <container_id>
docker exec -it <container_id> bash
```

------------------------------------------------------------------------

# 🏗️ CREACIÓN DE UNA IMAGEN

## 📄 Dockerfile

``` dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY . .
ENTRYPOINT ["dotnet", "MiApp.dll"]
```

## 🛠️ Construir imagen

``` bash
docker build -t miapp:1.0 .
```

------------------------------------------------------------------------

# ☁️ USO DE REGISTRY

## 🔐 Login

``` bash
docker login
```

## 🏷️ Tag de imagen

``` bash
docker tag miapp:1.0 usuario/miapp:1.0
```

## 📤 Publicar imagen

``` bash
docker push usuario/miapp:1.0
```

## 📥 Descargar imagen

``` bash
docker pull usuario/miapp:1.0
```

------------------------------------------------------------------------

# 📦 CREAR CONTENEDORES

``` bash
docker run -d -p 5000:80 --name mi_contenedor miapp:1.0
```

------------------------------------------------------------------------

# 🔐 ACCEDER A CONTENEDORES

``` bash
docker exec -it mi_contenedor bash
docker attach mi_contenedor
docker logs mi_contenedor
```

------------------------------------------------------------------------

# 📁 VOLÚMENES

``` bash
docker volume create mi_volumen
docker run -v mi_volumen:/data miapp
```

------------------------------------------------------------------------

# 🌐 NETWORKS

``` bash
docker network create mi_red
docker network ls
```

------------------------------------------------------------------------

# 📦 DOCKER COMPOSE

## 📄 docker-compose.yml

``` yaml
version: '3.9'
services:
  web:
    image: usuario/miapp:1.0
    ports:
      - "8080:80"
  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
```

## ▶️ Ejecutar

``` bash
docker compose up -d
```

## ⛔ Detener

``` bash
docker compose down
```

------------------------------------------------------------------------

# 🔐 BUENAS PRÁCTICAS

-   Usar imágenes oficiales
-   Minimizar tamaño de imagen
-   No ejecutar como root
-   Usar multi-stage builds
-   Versionar imágenes

------------------------------------------------------------------------

# 🎯 CONCLUSIÓN

Docker permite construir aplicaciones desacopladas, portables y
escalables, siendo clave en arquitecturas modernas basadas en
microservicios y despliegues automatizados.
