# FerreteriaLujan-API

Backend de un sistema de gestión para una ferretería (**Ferretería Luján**), construido como una arquitectura de **microservicios** con Spring Boot. Cubre inventario, compras, ventas y autenticación, y expone todo a través de un único punto de entrada (API Gateway) con Swagger unificado.

## Qué hace

El sistema permite administrar la operación diaria de una ferretería:

- **Inventario**: productos, marcas, categorías, depósitos, stock por depósito y transferencias entre depósitos (con procesamiento transaccional y ajustes de stock).
- **Compras**: proveedores, facturas de compra y sus detalles. El servidor calcula el IVA, el subtotal y el `importeTotal`.
- **Ventas**: clientes, presupuestos, facturas de venta y sus detalles.
- **Seguridad**: login, refresh de token y gestión de usuarios y roles (ADMIN, VENDEDOR, etc.) con JWT.

## Arquitectura

```
                    ┌────────────────┐
   Cliente ───────▶ │  api-gateway   │  :8080  (validación JWT + ruteo + Swagger único)
                    └───────┬────────┘
                            │  lb:// (descubrimiento vía Eureka)
        ┌───────────┬───────┴───┬──────────────┐
        ▼           ▼           ▼              ▼
  ms-inventario  ms-ventas   ms-compras    ms-seguridad
     :8081         :8082       :8083          :8085
        └───────────┴─────┬─────┴──────────────┘
                          ▼
              PostgreSQL  +  Redis (caché)

  eureka-server :8761  ← todos los servicios se registran aquí
```

| Módulo | Puerto | Responsabilidad |
|---|---|---|
| `eureka-server` | 8761 | Registro y descubrimiento de servicios |
| `api-gateway` | 8080 | Entrada única: valida el JWT, enruta a cada microservicio y sirve el Swagger común |
| `ms-seguridad` | 8085 | Emite y renueva tokens, administra usuarios y roles |
| `ms-inventario` | 8081 | Productos, marcas, categorías, depósitos, inventario y transferencias |
| `ms-ventas` | 8082 | Clientes, presupuestos, facturas de venta |
| `ms-compras` | 8083 | Proveedores, facturas de compra |

Las entidades y DTOs compartidos viven en un JAR `commons` (`libs/commons-0.0.1-SNAPSHOT.jar`) que cada microservicio toma directamente desde su carpeta `libs/`.

### Rutas expuestas por el gateway

| Servicio | Rutas |
|---|---|
| Inventario | `/api/productos`, `/api/marcas`, `/api/categorias-productos`, `/api/depositos`, `/api/inventarios`, `/api/transferencias`, `/api/detalles-transferencia` |
| Ventas | `/api/clientes`, `/api/presupuestos`, `/api/presupuestos-detalle`, `/api/facturas-ventas`, `/api/detalles-facturas-ventas`, `/api/estados` |
| Compras | `/api/proveedores`, `/api/facturas-compras`, `/api/detalles-compras` |
| Seguridad | `/api/auth/login`, `/api/auth/refresh`, `/api/auth/me`, `/api/usuarios` |
| Documentación | `http://localhost:8080/swagger-ui.html` (Inventario, Ventas y Compras en una sola interfaz) |

## Tecnologías

- **Java 21**
- **Spring Boot** 4.1 (microservicios) y 3.5 (gateway y Eureka)
- **Spring Cloud 2025** — Gateway (MVC) y Netflix Eureka (server y client)
- **Spring Security + JWT** (firma HS256, validación con `spring-security-oauth2-jose`)
- **Spring Data JPA** + **PostgreSQL**
- **Redis** como caché compartida (`spring-boot-starter-cache`), con degradación elegante: si Redis falla, las peticiones siguen contra PostgreSQL
- **MapStruct** y **Lombok** para el mapeo DTO ↔ entidad
- **springdoc-openapi** (Swagger UI)
- **Logback** con logs a disco
- **JUnit / Spring Boot Test** para pruebas unitarias (servicios, filtros de autenticación, caché, manejo de errores, cálculo de IVA)
- **Maven** (con Maven Wrapper en gateway y Eureka)

## Autores y contribuciones

Proyecto desarrollado en equipo en el marco de FIUNI.

### Diogo Delvalle (`diogodelvalleFiuni`)

Trabajo centrado en la **integración del API Gateway y la exposición del backend a clientes web**:

- **Ruteo de ventas** (11/11/2026): agregó la ruta `/api/estados/**` al gateway para que los estados de ventas queden accesibles a través del punto de entrada único.
- **Ruteo de compras** (11–12/11/2026): corrigió y completó las rutas del servicio de compras en el gateway, reemplazando `/api/compras/**` por las rutas reales de los recursos: `/api/proveedores/**`, `/api/facturas-compras/**` y `/api/detalles-compras/**`.
- **Refactor de CORS** (12/11/2026): eliminó las clases `CorsConfig` duplicadas de `ms-inventario`, `ms-ventas` y `ms-compras` (que tenían el origen `localhost:8080` fijo en el código) y centralizó el manejo en el gateway con el filtro por defecto `RemoveRequestHeader=Origin`, de modo que los microservicios ya no ven orígenes de navegador.
- ├── ms-ventas/        # Clientes, presupuestos, facturas de venta
- └── ms-compras/       # Proveedores, facturas de compra
