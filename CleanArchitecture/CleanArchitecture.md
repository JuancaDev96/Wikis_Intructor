# Arquitectura Limpia (Clean Architecture)

**Instructor:** Juan Carlos De La Cruz

------------------------------------------------------------------------

# Introducción

La **Arquitectura Limpia (Clean Architecture)** es un enfoque de diseño
de software que busca construir sistemas **mantenibles, escalables,
testeables y desacoplados de las tecnologías externas**.

Este modelo arquitectónico propone organizar el software de manera que
**las reglas del negocio permanezcan independientes de frameworks, bases
de datos, interfaces de usuario o servicios externos**.

La idea fundamental es que **el software debe sobrevivir a las
tecnologías que lo rodean**. Frameworks cambian, bases de datos
evolucionan, herramientas aparecen y desaparecen; sin embargo, **las
reglas del negocio suelen permanecer estables durante largos periodos de
tiempo**.

Por esta razón, la Arquitectura Limpia propone que:

-   El **dominio del negocio sea el centro del sistema**
-   Las **dependencias siempre apunten hacia el núcleo**
-   Las **tecnologías externas sean detalles intercambiables**

------------------------------------------------------------------------

# Origen de la Arquitectura Limpia

La Arquitectura Limpia fue propuesta por **Robert C. Martin**, conocido
en la comunidad de software como **Uncle Bob**.

Robert C. Martin es uno de los autores más influyentes dentro del
movimiento **Agile** y uno de los firmantes del **Manifiesto Ágil
(2001)**.

La arquitectura fue formalizada y difundida ampliamente en su libro:

**Clean Architecture: A Craftsman's Guide to Software Structure and
Design (2017)**

Sin embargo, el concepto no surgió de manera aislada. En realidad,
**Clean Architecture es una evolución de múltiples estilos
arquitectónicos previos** que buscaban resolver problemas similares
relacionados con el acoplamiento, la mantenibilidad y la escalabilidad
del software.

------------------------------------------------------------------------

# Arquitecturas que Influenciaron Clean Architecture

La Arquitectura Limpia es el resultado de la evolución de varias ideas
arquitectónicas importantes.

## Arquitectura Hexagonal (Ports and Adapters)

Propuesta por **Alistair Cockburn** alrededor del año 2005.

Esta arquitectura introdujo la idea de separar el sistema en:

-   **Núcleo de negocio**
-   **Adaptadores externos**

Los adaptadores permiten que el sistema interactúe con:

-   Interfaces de usuario
-   Bases de datos
-   APIs externas
-   Sistemas de mensajería

La idea principal era que el **dominio no dependiera de ninguna
tecnología externa**.

Esta arquitectura introdujo el concepto de:

**Ports (puertos) y Adapters (adaptadores)**

Los **puertos** definen interfaces del dominio y los **adaptadores**
implementan esas interfaces utilizando tecnologías específicas.

Clean Architecture adopta completamente esta idea.

------------------------------------------------------------------------

## Arquitectura Onion

Propuesta por **Jeffrey Palermo en 2008**.

La Arquitectura Onion introduce el concepto de **capas concéntricas**,
donde:

-   El dominio está en el centro
-   Las dependencias apuntan siempre hacia adentro
-   Las capas externas dependen de las internas

Las capas externas representan detalles de infraestructura como:

-   Bases de datos
-   Frameworks
-   Interfaces de usuario

Este modelo influyó directamente en el concepto de **capas circulares**
de Clean Architecture.

------------------------------------------------------------------------

## Arquitectura en Capas (Layered Architecture)

Es una de las arquitecturas más tradicionales dentro del desarrollo de
software empresarial.

Divide el sistema en capas como:

-   Presentación
-   Aplicación
-   Dominio
-   Infraestructura
-   Persistencia

Aunque este enfoque ayudó a organizar el software, muchas
implementaciones terminaron generando **dependencias incorrectas entre
capas**, donde el dominio terminaba dependiendo de tecnologías externas.

Clean Architecture toma la idea de capas pero **refuerza las reglas de
dependencia**.

------------------------------------------------------------------------

# Principio Fundamental: Dependency Rule

El principio más importante de Clean Architecture es conocido como:

**Dependency Rule (Regla de Dependencias)**

Esta regla establece que:

> Las dependencias del código fuente deben apuntar únicamente hacia el
> interior.

Esto significa que:

-   Las capas internas **no conocen las externas**
-   El dominio **no conoce frameworks**
-   El dominio **no conoce bases de datos**
-   El dominio **no conoce interfaces de usuario**

En cambio, son las capas externas las que dependen de las internas.

------------------------------------------------------------------------

# Estructura de Clean Architecture

La arquitectura se representa comúnmente mediante **círculos
concéntricos**.

Desde el centro hacia afuera encontramos:

1.  Entities
2.  Use Cases
3.  Interface Adapters
4.  Frameworks & Drivers

------------------------------------------------------------------------

# Entities (Entidades)

Las entidades representan **las reglas de negocio más puras del
sistema**.

Contienen:

-   Modelos de dominio
-   Reglas empresariales críticas
-   Comportamientos del negocio

Las entidades deben ser **independientes de cualquier tecnología**.

Ejemplos:

-   Cliente
-   Producto
-   Pedido
-   Factura

Las entidades pueden ser utilizadas por **cualquier aplicación dentro de
la organización**.

------------------------------------------------------------------------

# Use Cases (Casos de Uso)

Los casos de uso representan **la lógica de aplicación**.

Definen cómo interactúan las entidades para cumplir objetivos del
sistema.

Un caso de uso responde preguntas como:

-   Crear un pedido
-   Registrar un cliente
-   Procesar un pago
-   Generar un reporte

Los casos de uso:

-   Orquestan entidades
-   Aplican reglas del negocio
-   Definen flujos de aplicación

Pero **no dependen de frameworks ni bases de datos**.

------------------------------------------------------------------------

# Interface Adapters

Esta capa adapta los datos entre:

-   El mundo externo
-   El dominio interno

Aquí encontramos componentes como:

-   Controllers
-   Presenters
-   Gateways
-   DTOs
-   Mappers

Su responsabilidad es **transformar datos entre formatos externos y
formatos del dominio**.

Por ejemplo:

-   Convertir JSON en objetos de dominio
-   Mapear entidades a DTOs
-   Adaptar respuestas del dominio para una API REST

------------------------------------------------------------------------

# Frameworks & Drivers

Esta es la capa más externa del sistema.

Incluye:

-   Frameworks web
-   ORMs
-   Bases de datos
-   UI frameworks
-   Sistemas de mensajería
-   APIs externas

Ejemplos:

-   ASP.NET
-   Spring Boot
-   Express
-   Entity Framework
-   Hibernate
-   MongoDB
-   PostgreSQL

En Clean Architecture estas herramientas son consideradas **detalles de
implementación**.

------------------------------------------------------------------------

# Cómo se Implementa Clean Architecture

Implementar Clean Architecture implica organizar el proyecto respetando
las dependencias.

Una estructura común podría ser:

    src
     ├── Domain
     │   ├── Entities
     │   └── ValueObjects
     │
     ├── Application
     │   ├── UseCases
     │   ├── Interfaces
     │   └── DTOs
     │
     ├── Infrastructure
     │   ├── Persistence
     │   ├── ExternalServices
     │   └── Implementations
     │
     └── Presentation
         ├── API
         ├── Controllers
         └── Views

------------------------------------------------------------------------

# Beneficios de Clean Architecture

Adoptar este enfoque arquitectónico ofrece múltiples beneficios.

## Alta mantenibilidad

El código es más fácil de modificar y extender.

## Independencia de frameworks

El sistema puede cambiar de tecnología sin afectar el dominio.

## Alta testabilidad

Las reglas del negocio pueden probarse sin necesidad de bases de datos o
interfaces externas.

## Bajo acoplamiento

Las capas están claramente separadas.

## Mayor vida útil del software

El sistema puede evolucionar durante años sin reescrituras completas.

------------------------------------------------------------------------

# Desafíos de Clean Architecture

Aunque es muy poderosa, también presenta ciertos desafíos.

## Mayor complejidad inicial

La arquitectura requiere mayor planificación.

## Más capas y abstracciones

Esto puede aumentar la cantidad de código.

## Curva de aprendizaje

Los equipos deben comprender principios como:

-   Inversión de dependencias
-   Separación de responsabilidades
-   Diseño orientado al dominio

------------------------------------------------------------------------

# Conclusión

La Arquitectura Limpia representa una evolución natural en la forma de
diseñar software profesional.

Su principal objetivo es **proteger las reglas del negocio de los
cambios tecnológicos**, permitiendo que los sistemas sean más duraderos,
mantenibles y escalables.

En un mundo donde las tecnologías cambian constantemente, diseñar
software con este enfoque permite que **el valor del negocio permanezca
estable y protegido dentro del núcleo del sistema**.

Adoptar Clean Architecture no significa únicamente reorganizar carpetas
o capas; implica **cambiar la forma en que se piensa el diseño del
software**, colocando el dominio en el centro y tratando las tecnologías
como simples herramientas intercambiables.

------------------------------------------------------------------------

**Instructor:** Juan Carlos De La Cruz
