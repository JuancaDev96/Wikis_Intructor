# ⚙️ Gestión de Recursos en Contenedores Docker

**Instructor:** Juan Carlos De La Cruz Ch.

---

## 📌 Introducción

Uno de los errores más comunes en arquitecturas basadas en Docker, especialmente en etapas iniciales o entornos de desarrollo, es **no definir límites de recursos para los contenedores**.

Por defecto, Docker permite que un contenedor consuma **todos los recursos disponibles del host** (CPU, memoria y procesos). Si esto no se controla, puede convertirse rápidamente en un riesgo para la estabilidad del sistema.

---

## 🧠 Principio Arquitectónico Clave

> **Un contenedor sin límites de recursos es un contenedor fuera de control.**

La gestión de recursos no es una optimización tardía, sino una **decisión arquitectónica fundamental** que impacta directamente en la robustez, escalabilidad y confiabilidad del sistema.

---

## 🔥 ¿Qué ocurre si NO se configuran límites?

Cuando un contenedor no tiene restricciones explícitas:

- Puede consumir el **100% de la CPU** del host
- Puede agotar toda la **memoria RAM disponible**
- El sistema operativo puede activar el **OOM Killer**
- Otros contenedores pueden degradarse o fallar
- Los problemas son difíciles de diagnosticar

Este escenario es especialmente crítico en:
- Arquitecturas con múltiples servicios
- Entornos compartidos
- Sistemas con alta concurrencia

---

## 🧱 Recursos que Deben Controlarse

Todo despliegue profesional debe definir límites para:

- **CPU** → evita monopolización de procesamiento
- **Memoria (RAM)** → previene caídas del host
- **Procesos (PIDs)** → evita saturación del sistema

Estos límites convierten a Docker en una **plataforma controlada**, no solo en un runtime.

---

## 🐳 Configuración de Recursos en Docker Compose

Ejemplo recomendado de configuración:

```yaml
services:
  galaxy_pedidos_ui:
    image: galaxy.pedidos.ui:1.0.0
    container_name: galaxy_pedidos_ui
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
```

### 📌 Explicación

- **limits**: consumo máximo permitido
- **reservations**: recursos mínimos garantizados
- Evita interferencias entre servicios
- Facilita el escalado controlado

---

## 🧪 Casos Especiales: Bases de Datos

Servicios como **SQL Server**, **PostgreSQL** o motores de búsqueda requieren especial atención.

Ejemplo:

```yaml
deploy:
  resources:
    limits:
      memory: 1G
```

Sin este control, una base de datos puede consumir toda la memoria del host bajo carga.

---

## 🧠 Impacto Arquitectónico

Una correcta gestión de recursos permite:

- Mayor estabilidad del sistema
- Rendimiento predecible
- Escalado horizontal real
- Menor riesgo de fallos en cascada

Además, alinea la arquitectura con buenas prácticas empresariales.

---

## 🚀 Preparación para Orquestadores

Definir límites desde Docker Compose prepara naturalmente el camino hacia:

- Kubernetes (`requests` / `limits`)
- AWS ECS / Fargate
- Azure AKS
- Google GKE

> **Quien define recursos desde el inicio, escala sin dolor.**

---

## ✅ Conclusión

- Docker sin límites consume todo lo que encuentra
- Los recursos deben definirse explícitamente
- Es una práctica obligatoria en sistemas profesionales
- La gestión de recursos es parte del diseño arquitectónico

---

📘 *Documento elaborado por Juanca Dev*

