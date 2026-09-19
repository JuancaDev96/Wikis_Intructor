# Wiki — Angular 21 · Módulo 2 Front-End

**Curso:** Especialización Full-Stack .NET 10 & Angular 21 — Galaxy Training

**Día:** Día 1 · 12 de septiembre de 2026

**Sesiones:** Sesión A — Introducción a Angular 21 · Sesión B — Arquitectura y Modelos

**Instructor:** Juan Carlos De La Cruz Ch.

**Proyecto del curso:** TechStore — ERP de tienda construido con Angular 21 + .NET 10 Minimal APIs

---

## Índice

**Sesión A — Introducción a Angular 21**
1. [¿Qué es Angular 21?](#1-qué-es-angular-21)
   
3. [Angular vs React vs Vue](#2-angular-vs-react-vs-vue)
   
5. [Herramientas de desarrollo](#3-herramientas-de-desarrollo)
   
7. [Arquitectura interna de Angular 21](#4-arquitectura-interna-de-angular-21)
   
9. [Nuevas features de Angular 21](#5-nuevas-features-de-angular-21)
    
11. [Instalación y verificación del entorno](#6-instalación-y-verificación-del-entorno)
    
13. [Creación del proyecto base](#7-creación-del-proyecto-base)

**Sesión B — Arquitectura y Modelos**

8. [Estructura feature-based del proyecto TechStore](#8-estructura-feature-based-del-proyecto-techstore)

9. [Standalone Components — sin NgModule](#9-standalone-components--sin-ngmodule)
    
11. [Modelos TypeScript — auth.model.ts](#10-modelos-typescript--authmodelts)
    
13. [Interfaces vs Types vs Clases](#11-interfaces-vs-types-vs-clases)
    
15. [Environments — Dev vs Producción](#12-environments--dev-vs-producción)
    
17. [tsconfig.json — Modo estricto](#13-tsconfigjson--modo-estricto)
    
19. [Resumen del Día 1 y próxima sesión](#14-resumen-del-día-1-y-próxima-sesión)

---

## 1. ¿Qué es Angular 21?

Angular 21 es un **framework SPA (Single Page Application)** desarrollado y mantenido por Google. A diferencia de una librería como React, Angular es un framework "todo incluido": trae de fábrica Router, HttpClient, Forms, Inyección de Dependencias (DI) e internacionalización (i18n), por lo que un equipo no necesita elegir ni integrar piezas externas para empezar a construir una aplicación completa.

**Características clave:**

| Característica | Detalle |
|---|---|
| Lenguaje | 100% TypeScript — tipado estático, POO, decoradores |
| Versión actual | Angular 21, lanzada en 2025 |
| Ciclo de releases | Una versión *major* cada 6 meses |
| Tipo de framework | Opinionado y enterprise-grade (no una librería como React) |
| Casos de uso | Dashboards empresariales, SPAs complejas, PWA, SSR |
| Empresas que lo adoptan | Google, Microsoft, IBM, Santander, entre otras |

**¿Por qué importa que sea "opinionado"?** Un framework opinionado impone una forma de estructurar el código (dónde van los servicios, cómo se inyectan dependencias, cómo se navega). Esto reduce las decisiones arbitrarias del equipo y genera consistencia entre proyectos y desarrolladores — algo especialmente valioso en equipos grandes o con alta rotación.

**Proyecto del curso:** a lo largo de esta especialización se construye **TechStore**, un ERP de tienda con backend en .NET 10 Minimal APIs y frontend en Angular 21. Todos los ejemplos de código de este día (modelos, environments, estructura de carpetas) pertenecen a este proyecto.

---

## 2. Angular vs React vs Vue

No existe un framework "mejor" en abstracto — la elección depende del tipo de proyecto, el equipo y el ecosistema con el que se integra.

| | **Angular 21** | **React 19** | **Vue 3** |
|---|---|---|---|
| Tipo | Full framework (todo incluido) | Librería de UI (ecosistema propio) | Framework progresivo |
| TypeScript | Obligatorio | Opcional | Opcional |
| Inyección de dependencias | Nativa, con decoradores | No nativa — se resuelve con Context o librerías externas | No nativa |
| Filosofía | Opinionado → menos decisiones para el equipo | Flexible → más decisiones para el equipo | Flexible con curva suave |
| Reactividad / estado | Signals, RxJS | Hooks, Context, Redux (externo) | Options API / Composition API |
| Curva de aprendizaje | Media-alta | Media | Baja |
| Ideal para | Enterprise, equipos grandes | Proyectos flexibles, alta personalización | Proyectos medianos |

**¿Por qué Angular en este curso?**

- **Consistencia en proyectos grandes** — todos los desarrolladores estructuran el código de la misma forma.
- **Contrato fijo con el equipo** — TypeScript obligatorio evita ambigüedades en los tipos de datos que viajan entre frontend y backend.
- **Integración natural con .NET** — ambos ecosistemas comparten una filosofía fuertemente tipada y orientada a objetos.
- **Tipado robusto desde el diseño** — los errores de tipos se detectan en tiempo de compilación, no en producción.

---

## 3. Herramientas de desarrollo

Antes de escribir la primera línea de Angular, el entorno debe tener instalado:

| Herramienta | Para qué sirve | Cómo se verifica |
|---|---|---|
| **Node.js 22 LTS** | Entorno de ejecución de JavaScript en el servidor; requerido por npm y Angular CLI | Descargar desde `nodejs.org` → versión LTS |
| **npm 11+** | Gestor de paquetes de Node, incluido con la instalación de Node.js | `npm --version` |
| **Angular CLI 21** | Herramienta de línea de comandos para crear, servir y compilar proyectos Angular | `npm install -g @angular/cli@21` → `ng version` |
| **VS Code + extensiones** | Editor recomendado, con Angular Language Service, ESLint, Prettier, GitLens y Thunder Client | — |
| **Docker Desktop** *(opcional)* | Permite correr SQL Server en un contenedor durante el desarrollo local | — |

> **Nota didáctica:** Angular CLI es la herramienta más importante de esta lista para el día a día — genera componentes, servicios, módulos y build de producción con un solo comando, siguiendo siempre las convenciones oficiales del framework.

---

## 4. Arquitectura interna de Angular 21

Angular se construye a partir de seis piezas fundamentales que se combinan entre sí:

| Pieza | Rol | Ejemplo de sintaxis |
|---|---|---|
| **Componentes** | Bloque fundamental de la UI: combina una clase TypeScript con un template HTML | `@Component({ selector, template, styles })` |
| **Servicios** | Encapsulan lógica de negocio y comunicación HTTP; se inyectan donde se necesiten | `@Injectable({ providedIn: "root" })` |
| **Router** | Navegación declarativa entre vistas | `Routes[]`, `RouterLink`, `RouterOutlet`, Guards, Lazy Loading |
| **HttpClient** | Peticiones HTTP tipadas al backend | `get<T>()`, `post<T>()`, `put<T>()`, `delete<T>()` → `Observable<T>` |
| **Formularios** | Manejo reactivo de formularios | `FormBuilder`, `FormGroup`, `FormControl`, `Validators` |
| **Dependency Injection (DI)** | Mecanismo nativo para inyectar dependencias sin instanciarlas manualmente | función `inject()` (Angular 14+) o inyección por constructor |

**Cómo se relacionan:** un **componente** típico inyecta un **servicio** (vía DI) para pedir datos a través de **HttpClient**, valida la entrada del usuario con **Formularios**, y navega a otra vista usando el **Router**. Entender esta cadena es la base para leer cualquier código Angular.

---

## 5. Nuevas features de Angular 21

Angular evoluciona rápido. Estas son las features que cambian la forma de escribir código hoy en día respecto a versiones anteriores a Angular 14-16:

| Feature | Disponible desde | Qué resuelve |
|---|---|---|
| **Standalone Components** | Default desde Angular 17 | `standalone: true` — elimina la necesidad de `NgModule`; simplifica el árbol de dependencias y habilita lazy loading nativo |
| **Signals** | Estabilizados en v18 | `signal<T>()`, `computed()`, `effect()` — reactividad de grano fino sin depender de Zone.js, con mejor rendimiento |
| **Deferrable Views (`@defer`)** | Angular 17+ | Carga diferida declarativa de bloques pesados de UI: `@defer (on viewport) { <heavy-component/> }` |
| **Control Flow (`@if`, `@for`, `@switch`)** | Angular 17+ | Reemplaza las directivas estructurales `*ngIf` / `*ngFor` con sintaxis nativa del compilador, más rápida y legible |
| **Zoneless Mode** | Experimental desde v18 | Elimina la dependencia de Zone.js, mejorando el rendimiento y la integración con Web APIs modernas |
| **SSR mejorado** | Hydration completa, v16-21 | Renderizado en servidor con hidratación total del cliente, mejorando SEO y tiempo de carga inicial |

**Idea central para transmitir en clase:** Angular está migrando de un modelo de detección de cambios "global" (Zone.js) hacia un modelo de **reactividad explícita y granular** (Signals). Esa es la dirección de toda la plataforma en los próximos años.

---

## 6. Instalación y verificación del entorno

Pasos para dejar el entorno listo antes de crear el primer proyecto:

1. **Instalar Node.js 22 LTS** desde `nodejs.org` (elegir la versión LTS, no la "Current").
2. **Verificar instalación:**
   ```bash
   node --version    # v22.x.x
   npm --version     # 11.x.x
   ```
3. **Instalar Angular CLI globalmente:**
   ```bash
   npm install -g @angular/cli@21
   ```
4. **Verificar Angular CLI:**
   ```bash
   ng version
   ```
   Debe mostrar la versión de Angular CLI y confirmar que Node está correctamente enlazado.

> Un error común en este punto es tener una versión de Node.js desactualizada (Angular 21 requiere Node 20.19+ o 22+). Si `ng version` falla, lo primero a revisar es `node --version`.

---

## 7. Creación del proyecto base

```bash
ng new techstore-frontend
# ? Add Angular routing?          → YES
# ? Which stylesheet format?      → SCSS

cd techstore-frontend
ng serve --open    # abre http://localhost:4200
```

**Estructura generada por el CLI:**

```
src/
  app/
    app.ts              ← Componente raíz
    app.html            ← Template raíz
    app.routes.ts       ← Definición de rutas
    app.config.ts       ← Proveedores de la app (DI, router, etc.)
  environments/
    environment.ts               ← Configuración de desarrollo
    environment.production.ts    ← Configuración de producción
  index.html            ← Entry point HTML
  main.ts               ← Bootstrap de la aplicación
```

**Punto didáctico:** desde Angular 17, `ng new` genera un proyecto **standalone por defecto** — no aparece ningún `app.module.ts`. Es importante que los estudiantes no busquen ese archivo pensando que "algo salió mal": simplemente ya no existe.

---

## 8. Estructura feature-based del proyecto TechStore

```
src/app/
  core/            ← Servicios singleton, guards, interceptors, modelos
    models/        ← auth.model.ts, product.model.ts, etc.
    services/      ← auth.service.ts, product.service.ts, etc.
    guards/        ← auth.guard.ts, role.guard.ts
    interceptors/  ← auth.interceptor.ts, error.interceptor.ts
  shared/          ← Componentes y módulos reutilizables
  features/        ← Módulos de negocio (products, customers, sales)
  layout/          ← Shell, Toolbar, Sidenav
```

**Ventajas de organizar el proyecto así:**

- **Separación de responsabilidades (SRP)** — cada carpeta tiene un propósito único y predecible.
- **Lazy loading por feature** — es trivial cargar bajo demanda el módulo de `products` sin tocar `customers` o `sales`.
- **Escalabilidad** — se pueden agregar nuevas features sin modificar el `core` del proyecto, lo que reduce el riesgo de romper código existente.

**Regla práctica para dar en clase:** si un archivo se usa en **una sola feature**, vive dentro de `features/`. Si se usa en **más de una**, sube a `shared/`. Si es **infraestructura transversal** (autenticación, interceptores HTTP, modelos globales), vive en `core/`.

---

## 9. Standalone Components — sin NgModule

**Antes (Angular < 14):** cada componente debía declararse dentro de un `NgModule` contenedor.

```typescript
@NgModule({
  declarations: [ProductListPage],
  imports: [CommonModule, MatTableModule],
  exports: [ProductListPage]
})
export class ProductsModule {}
```

**Ahora (Angular 17+):** cada componente declara sus propias dependencias — no hay módulo contenedor.

```typescript
@Component({
  selector: 'app-products-list',
  standalone: true,   // ← clave
  imports: [
    CommonModule,
    RouterLink,
    PageHeaderComponent,   // otro standalone
    HasRoleDirective       // directiva standalone
  ],
  templateUrl: './list.page.html',
})
export class ProductsListPage {}

// El bootstrap en main.ts también cambia:
bootstrapApplication(App, appConfig);   // sin NgModule
```

**Por qué importa:** con NgModules, entender de dónde venía una dependencia de un componente obligaba a rastrear varios archivos. Con Standalone Components, **el propio componente es la fuente de verdad** de todo lo que necesita — se lee de arriba hacia abajo, sin saltos entre archivos.

---

## 10. Modelos TypeScript — auth.model.ts

Archivo real del proyecto TechStore (`src/app/core/models/auth.model.ts`):

```typescript
export type Role = 'Admin' | 'Customer';   // union type

export interface LoginRequest {
  userName: string;
  password: string;
}

// Respuesta del backend: POST /api/auth/login
export interface LoginResponse {
  token: string;
  expirationDate: string;
  role: Role;
}

// Patrón Result del backend — envuelve toda respuesta de la API
export interface ApiResult<T = unknown> {
  value?: T;
  isSuccess: boolean;
  isFailure: boolean;
  message: string | null;
  errors: string[];
}

// Estado de sesión: se guarda en memoria (signal) y en sessionStorage
export interface CurrentUser {
  userName: string;
  role: Role;
  token: string;
  expiresAt: string;
}
```

**Conceptos clave de este archivo:**

- **`Role` — Union Type:** solo `'Admin'` o `'Customer'` son valores válidos; TypeScript rechaza cualquier otro string en tiempo de compilación.
- **`ApiResult<T>` — Result Pattern:** en lugar de que el backend lance excepciones HTTP genéricas, envuelve **toda** respuesta con un contrato uniforme: éxito/fracaso, valor y lista de errores. Esto simplifica enormemente el manejo de errores en el frontend.
- **`CurrentUser` — estado de sesión:** se mantiene en memoria mediante un `signal` para que la UI reaccione a cambios de sesión en tiempo real, y se persiste en `sessionStorage` para sobrevivir a un refresh de página.

---

## 11. Interfaces vs Types vs Clases

Una de las dudas más frecuentes de quien empieza en TypeScript. Regla general: **interfaces para modelar datos, types para combinarlos, clases cuando se necesita lógica o inyección de dependencias.**

| | **Interfaces** | **Types** | **Clases** |
|---|---|---|---|
| Existencia | Solo en compile time — se borran al compilar a JS | Solo en compile time | Existen en runtime |
| Uso principal | Modelar DTOs y respuestas de API | Uniones, intersecciones, alias de tipos | Lógica, métodos, DI en Angular |
| Ejemplo | `interface Product { id: number; name: string; price: number; }` | `type Role = 'Admin' \| 'Customer';` | `@Injectable() export class AuthService {}` |
| Cuándo usarla | Al recibir datos de una API | Al necesitar combinar o restringir valores posibles | Al necesitar que Angular inyecte la dependencia |

**Generics** combinan estas piezas para escribir código reutilizable y tipado: `ApiResult<T>`, `Observable<Product[]>`, `signal<CurrentUser | null>()`. El tipo genérico `T` se reemplaza según el contexto sin duplicar código.

---

## 12. Environments — Dev vs Producción

```typescript
// src/environments/environment.ts (DESARROLLO)
export const environment = {
  production: false,
  apiUrl: '',   // vacío → el proxy del dev server intercepta /api/*
};
// proxy.conf.json → target: "https://localhost:7263"
```

```typescript
// src/environments/environment.production.ts (PRODUCCIÓN)
export const environment = {
  production: true,
  apiUrl: 'https://api.mitienda.com',   // URL real del backend en Azure
};
```

**Flujo de trabajo:**

`ng serve` (usa `environment.ts`) → `ng build --configuration=production` (reemplaza el archivo por `environment.production.ts`) → la app compilada apunta al `apiUrl` real.

**Cómo se consume en un servicio:**

```typescript
import { environment } from '../../../environments/environment';

private readonly baseUrl = environment.apiUrl; // '' en dev, URL real en prod
```

**Punto didáctico clave:** el código del servicio **nunca cambia** entre entornos — solo cambia qué archivo de configuración se empaqueta en el build. Esto evita el error clásico de "olvidé cambiar la URL antes de subir a producción".

---

## 13. tsconfig.json — Modo estricto

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "useDefineForClassFields": false,
    "experimentalDecorators": true,
    "forceConsistentCasingInFileNames": true,
    "strictPropertyInitialization": false
  }
}
```

**¿Qué activa `"strict": true`?** Es un interruptor maestro que activa de una sola vez varios checks de TypeScript, entre ellos `noImplicitAny` y `strictNullChecks`.

- **Detección temprana de errores** — atrapa bugs de tipado en tiempo de compilación, antes de que lleguen a producción.
- **Código más mantenible** — obliga a declarar tipos explícitos en todo el proyecto, eliminando ambigüedades.
- **Menos sorpresas en runtime** — errores como "no se puede leer una propiedad de `undefined`" se detectan antes de ejecutar el código.

---

## 14. Resumen del Día 1 y próxima sesión

- Angular 21 es un full framework SPA enterprise-grade basado en TypeScript.
- Standalone Components eliminan la necesidad de `NgModule` (default desde Angular 17).
- Signals proveen reactividad más eficiente que Zone.js.
- La estructura feature-based (`core/`, `shared/`, `features/`, `layout/`) es escalable y mantenible.
- Interfaces se usan para DTOs, types para uniones, clases para DI.
- Los environments permiten configuración diferenciada dev/prod sin cambiar código.

**Próxima sesión:** Servicios Core, Routing y Autenticación JWT.

---

*Material elaborado para la Especialización Full-Stack .NET 10 & Angular 21 — Galaxy Training. Instructor: Juan Carlos De La Cruz Ch.*
