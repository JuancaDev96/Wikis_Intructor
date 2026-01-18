# 🧠 Introducción a IdentityServer

**Instructor:** Juan Carlos De La Cruz Ch.

------------------------------------------------------------------------

## ¿Qué es IdentityServer?

IdentityServer es un servidor de identidad y autorización para
aplicaciones **ASP.NET Core / .NET 8/9** que implementa los protocolos
de seguridad estándar de la industria: - **OAuth 2.0** - **OpenID
Connect (OIDC)**

IdentityServer se encarga de **emitir y validar tokens** de acceso,
autenticación y autorización, de modo que las aplicaciones no deban
hacerlo directamente.

------------------------------------------------------------------------

## 🎯 ¿Para qué sirve IdentityServer?

Sirve para **centralizar la autenticación y autorización** de los
usuarios en un sistema. En lugar de que cada aplicación tenga su propio
login, todas confían en **IdentityServer**, que actúa como **proveedor
de identidad**.

### Funciones principales

| Función                          | Descripción |
|----------------------------------|--------------|
| 🔐 **Autenticación**             | Verifica quién es el usuario (login). |
| ⚙️ **Autorización**              | Controla qué puede hacer el usuario (roles, claims, permisos). |
| 🎟️ **Emisión de Tokens**         | Genera tokens JWT para validar usuarios en las APIs. |
| 🧭 **Federación de Identidades** | Permite autenticarse con proveedores externos (Azure Entra ID, Google, etc.). |
| 🧩 **Integración con ASP.NET Core Identity** | Administra usuarios, contraseñas, roles y perfiles locales. |

  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 🏗️ Arquitectura de IdentityServer

En una arquitectura distribuida, IdentityServer actúa como el **cerebro
de autenticación**:

    +---------------------+
    |  IdentityServer     |  ← Autenticación centralizada
    |  (emite tokens JWT) |
    +---------------------+
            ↑     ↑
            |     |
    +---------------+        +----------------+
    | Web App (MVC) |        | API REST .NET 9|
    +---------------+        +----------------+

------------------------------------------------------------------------

## 🔄 Integración con ASP.NET Core Identity

IdentityServer se integra con **ASP.NET Core Identity** para manejar usuarios, roles, contraseñas y claims.  
Esto combina el manejo de usuarios locales con una plataforma de emisión de tokens estándar.

| ASP.NET Core Identity             | IdentityServer                                     |
|----------------------------------|----------------------------------------------------|
| Administra usuarios y roles.     | Emite tokens y maneja autenticación federada.      |
| Guarda credenciales en base de datos. | Implementa flujos OAuth2/OIDC.                |
| Autenticación local.             | Autenticación centralizada para múltiples apps.    |

  -----------------------------------------------------------------------

------------------------------------------------------------------------

## ⚙️ Flujo general de autenticación

1.  El usuario intenta acceder a la aplicación.\
2.  Es redirigido a **IdentityServer** para autenticarse.\
3.  IdentityServer valida las credenciales con **ASP.NET Identity**.\
4.  Emite un **token JWT**.\
5.  La app usa el token para acceder a las APIs protegidas.\
6.  Las APIs validan el token (firma, issuer, audience).

------------------------------------------------------------------------

## 🧩 Tipos de tokens en IdentityServer

| Tipo                | Descripción                                | Validación                     |
|----------------------|--------------------------------------------|---------------------------------|
| **JWT (self-contained)** | Contiene toda la información (claims, firma). | Validación local.               |
| **Reference Token**       | Solo contiene un identificador.              | Validación remota en IdentityServer. |
           

-------------------------------------------------------------------------------

### 🔍 Comparación

| Característica              | JWT tradicional       | Reference Token         |
|-----------------------------|-----------------------|--------------------------|
| Validación                  | Local en cada API     | Remota en IdentityServer |
| Contiene info del usuario   | Sí                    | No                       |
| Revocación de token         | No inmediata          | Centralizada             |
| Conexión con IdentityServer | No necesaria          | Obligatoria              |
| Escalabilidad               | Alta                  | Media                    |


------------------------------------------------------------------------

## 🧠 Diferencias: JWT vs IdentityServer

| Característica             | **JWT + ASP.NET Identity (manual)** | **IdentityServer + Identity** |
|-----------------------------|-------------------------------------|-------------------------------|
| Emisión de tokens           | Manual                              | Estándar OAuth2/OIDC          |
| Validación                  | Local                               | Estándar (OIDC)               |
| Autenticación externa       | Manual                              | Integrada                     |
| Revocación / Introspección  | No disponible                       | Soportado                     |
| Single Sign-On (SSO)        | No                                  | Sí                            |
| Multi-cliente (web, móvil, API) | Difícil                         | Nativo                        |
| Escalabilidad               | Limitada                            | Alta                          |
| Cumplimiento de estándares  | No                                  | Sí                            |

  estándares                                      
  --------------------------------------------------------------------------

------------------------------------------------------------------------

## 🧭 Cuándo usar cada uno

### ✅ Usa **ASP.NET Identity + JWT** cuando:

-   Tienes una sola aplicación.
-   No necesitas SSO ni federación.
-   Quieres algo rápido y liviano.

### ✅ Usa **IdentityServer** cuando:

-   Tienes múltiples clientes o APIs.
-   Necesitas **Single Sign-On**.
-   Requieres autenticación con **proveedores externos**.
-   Deseas **revocación de tokens y control de sesiones**.
-   Buscas **cumplir estándares de seguridad (OIDC/OAuth2)**.

------------------------------------------------------------------------

## 💬 Conclusión

Sí, con **JWT puro** puedes integrar múltiples APIs y frontends,
validando tokens en cada una.\
Sin embargo, **IdentityServer** te da un nivel adicional de **seguridad,
centralización y cumplimiento** con los estándares modernos de
identidad.

**En sistemas grandes o empresariales, IdentityServer es la base ideal
para la autenticación moderna.**

------------------------------------------------------------------------

**Instructor:** *Juan Carlos De La Cruz Ch.*\
**Curso:** Seguridad con NET 9 - Modulo 2
