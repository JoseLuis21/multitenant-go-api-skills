---
name: multitenant-go-api-skills
description: Levanta desde cero (o completa) una API en Go + Fiber v2 multi-tenant con base de datos por tenant (database-per-tenant), plano de control separado, autenticación JWT y permisos/roles/usuarios centralizados. Pregunta primero si el store es PostgreSQL o MongoDB y genera el scaffold ejecutable para el elegido. Úsala siempre que se hable de arrancar un backend nuevo, multi-tenancy, SaaS multi-empresa, "una API como la de facturación", control plane + tenants, provisioning de bases por cliente, RBAC/ACL, permisos por rol, middleware de tenant, o migrar un proyecto de un solo tenant a multi-tenant — aunque no se nombre "Go" ni "Fiber" explícitamente, y aunque el usuario solo diga "necesito el boilerplate de siempre".
---

# API multi-tenant en Go (control plane + base por tenant)

Esta skill reproduce una arquitectura probada en producción: **cada empresa tiene su propia base de datos**, y una **base de control** guarda usuarios, empresas, el registro de bases de tenant y el ACL (roles y permisos). Sirve igual con PostgreSQL o con MongoDB porque todo lo específico del motor vive detrás de dos piezas: el _tenant manager_ (resuelve la conexión de la empresa) y los repositorios.

El aislamiento es **físico**, no un `WHERE company_id = ?`. Esa es la decisión de fondo: una tabla de tenant nunca lleva columna `company_id`, y una consulta mal escrita no puede filtrar datos de otro cliente porque literalmente no están en esa base.

Vida de un request:

```
JWT auth  →  tenant  →  permiso  →  handler  →  use case  →  repositorio
(userID,      (resuelve      (resource:action        (lee la conexión
 companyID,    la conexión     contra los perms       del contexto,
 role,         de la empresa   del token)             no de una global)
 permissions)  y la inyecta
               en el contexto)
```

## Paso 0 — Preguntar antes de escribir

Haz **una sola** ronda de preguntas (usa AskUserQuestion) y arranca. Lo que necesitas saber:

1. **Store**: PostgreSQL o MongoDB. Determina qué archivo de referencia sigues y la mitad del código.
2. **Módulo Go y nombre del proyecto**: p. ej. `github.com/usuario/mi-api`.
3. **Cómo se llama el tenant en el negocio**: `company`, `organization`, `tenant`, `clinic`… El nombre entra en tablas, rutas y claims. Por defecto `company`.
4. **Módulo de ejemplo**: si quieres un CRUD de negocio ya cableado (recomendado: `products`) para tener un endpoint de tenant real que probar de punta a punta.

Si el usuario ya tiene un repo empezado, no preguntes lo que puedas leer del `go.mod` y de la estructura existente.

## Invariantes

Estas son las reglas que hacen que la arquitectura funcione. Explícalas cuando las apliques, porque son las que se rompen solas cuando alguien agrega un módulo con prisa.

1. **Las tablas/colecciones del tenant no llevan `company_id`.** Si te encuentras escribiendo ese filtro en un repositorio de tenant, estás en la base equivocada.
2. **El orden del middleware es `auth → tenant → permiso → handler`.** Registrar una ruta de tenant sin el middleware de tenant la deja apuntando a la base de control: no falla, responde datos de otra base. Es el bug más caro de esta arquitectura.
3. **Los repositorios leen la conexión del contexto**, con la conexión inyectada solo como respaldo (`PoolFromContext(ctx, r.db)` / `DatabaseFromContext(ctx, r.db)`). Nunca uses una conexión global en un repositorio de tenant.
4. **En Fiber, pasa siempre `c.UserContext()`**, jamás `c.Context()`. El middleware de tenant inyecta la conexión con `c.SetUserContext()`; el contexto de fasthttp no la lleva y todo el módulo cae silenciosamente a la base de control.
5. **Dos juegos de migraciones, en carpetas separadas**: las de control corren al arranque sobre la base de control; las de tenant corren (a) al provisionar una empresa y (b) al arranque sobre cada tenant en estado `ready`. Ambas idempotentes y registradas en `schema_migrations`.
6. **El catálogo de permisos vive en el plano de control** y es la fuente de verdad: un rol solo puede tener permisos que existan en el catálogo. Cada recurso nuevo agrega sus permisos en una migración de control.
7. **Recursos y rutas en inglés**, aunque el dominio se hable en español (`vendedores` → recurso `sellers`, ruta `/api/v1/sellers`, permisos `sellers:*`). Sin esta convención el catálogo de permisos se vuelve una mezcla imposible de mantener.
8. **El ID lo decide el store, no el módulo.** Con PostgreSQL, `uuid.NewV7()` (ordenable por tiempo, no fragmenta el índice primario). Con MongoDB, el `_id` nativo: `bson.ObjectID` — que ya es ordenable por tiempo, ocupa 12 bytes en vez de 16 y es lo que espera cualquier herramienta de Mongo. **No metas UUID en Mongo ni ObjectID en Postgres.** Para que el resto del código no dependa de esa decisión, la generación vive en un solo archivo, `internal/shared/id/id.go`, que expone `id.New() string`; el dominio y los casos de uso siempre ven un `string` opaco.

## Paso 1 — Estructura

Arquitectura hexagonal por módulo. El dominio nunca importa infraestructura.

```
cmd/api/main.go                      # arranque: carga env, bootstrap.NewApp(), listen, shutdown
internal/
  bootstrap/app.go                   # cableado secuencial de todos los módulos
  platform/
    database/
      connection.go                  # pool/cliente + ping
      migrations.go                  # runner idempotente (embed .sql o registry Go)
      tenant_manager.go              # provisiona la base de la empresa y cachea conexiones
      tenant_context.go              # inyecta/lee la conexión del tenant en el context.Context
      tenant_middleware.go           # resuelve companyID → conexión → SetUserContext
      control_migrations/            # esquema del plano de control
      tenant_migrations/             # esquema de cada empresa (sin company_id)
  shared/
    httpresponse/                    # envoltorio único de respuesta {success,message,data}
    crypto/                          # hash de password, cifrado de secretos
    id/                              # id.New(): uuid v7 en Postgres, bson.ObjectID en Mongo
  modules/
    auth/                            # usuarios, login, sesión, JWT, middlewares de auth y permiso
    <tenant>/                        # empresas: alta + provisioning de su base
    acl/                             # roles, catálogo de permisos, asignaciones
    <ejemplo>/                       # módulo de negocio de muestra, ya en la base del tenant
```

Cada módulo de negocio:

```
internal/modules/<modulo>/
  domain/            entidad + validación, errores centinela, interfaz Repository, eventos
  application/
    dto/             Input/Output structs
    usecases/        un archivo por caso de uso, método Execute(ctx, input, actorID)
  infrastructure/
    http/            handler.go, routes.go, validation.go, errors.go (mapeo a HTTP)
    persistence/     implementación del Repository
    events/          emisor (log por defecto)
  module.go          NewModule(deps) + Register(app, middlewares...)
```

## Paso 2 — Orden de construcción

Construye en este orden y compila (`go build ./...`) después de cada bloque; así los errores aparecen de a uno.

1. `go.mod`, `Makefile`, `.env.example`, `docker-compose.yml`, `Dockerfile` — están en `assets/`, cópialos y ajusta el nombre del módulo.
2. `shared/httpresponse`, `shared/crypto` y `shared/id` (la variante del store elegido).
3. `platform/database`: conexión, runner de migraciones, contexto de tenant, tenant manager, middleware de tenant.
4. Migraciones de control: usuarios, empresas, membresías, `tenant_databases`, roles, catálogo de permisos, `role_permissions`, `user_roles`.
5. Migraciones de tenant: la tabla/colección espejo de la empresa (`tenant_company`) y lo que necesite el módulo de ejemplo.
6. Módulo `auth`: registro, login, `select-company` (emite el JWT con `cid` + `perms`), middleware JWT, `RequirePermission`.
7. Módulo del tenant (`company`): crear empresa → provisionar su base → sembrar el rol `admin` con `*:*` y asignárselo al creador.
8. Módulo `acl`: CRUD de roles, listado del catálogo de permisos, asignación de rol a usuario.
9. Módulo de ejemplo, cableado con el middleware de tenant y `RequirePermission`.
10. `bootstrap/app.go` y `cmd/api/main.go`.

## Paso 3 — Verificar que corre de verdad

Un scaffold que compila pero no atiende un request no está terminado. Levanta la base (`make db-up`), corre la API (`make run`) y recorre el flujo completo con curl:

```bash
POST /api/v1/auth/register        # crea usuario → token sin empresa
POST /api/v1/companies            # crea empresa → provisiona su base + rol admin
POST /api/v1/auth/select-company  # token nuevo con cid + role + perms
GET  /api/v1/permissions          # catálogo de permisos (plano de control)
POST /api/v1/products             # escribe en la base del tenant
GET  /api/v1/products             # lee de la base del tenant
```

La prueba que de verdad importa: **crea una segunda empresa con el mismo usuario, escribe un producto en cada una y confirma que cada listado ve solo lo suyo.** Un tenant vacío hace pasar cualquier cosa; el aislamiento solo queda demostrado con dos tenants con datos distintos.

Verifica también que la base de control **no** tiene la tabla/colección `products`. Si la tiene, alguna ruta se registró sin el middleware de tenant.

## Errores que ya costaron caro

Vale la pena revisarlos antes de dar el trabajo por cerrado; todos son bugs que compilan, pasan los tests y se ven bien hasta que están en producción.

- **`c.Context()` en vez de `c.UserContext()`** — un módulo entero leyendo la base de control sin error visible.
- **Ruta literal después de la paramétrica** — en Fiber, `/:id` registrado antes que `/summary` se traga `/summary` y devuelve un 500 de parseo. Las rutas literales van primero, siempre.
- **Permisos del rol guardados en dos lugares** — si el rol tiene una columna JSON de permisos _y_ una tabla `role_permissions`, define cuál manda (la tabla) y haz que la otra sea derivada o no exista. Con dos fuentes, los `INSERT` a la que nadie lee no otorgan nada y el bug tarda meses en aparecer.
- **Migración puesta en la carpeta que no se ejecuta** — la tabla nunca se crea y el error aparece recién al primer request.
- **En Mongo, filtrar por `_id` con el string en vez del `ObjectID`** — `bson.M{"_id": "68a1…"}` compila, no da error y devuelve cero documentos, porque en BSON un string y un ObjectID son tipos distintos. Se ve como "el registro desapareció". Lo mismo con las referencias (`companyId`, `roleId`): guardadas como string, ningún `$lookup` las cruza.
- **Escalas distintas entre tablas** — si guardas dinero o cantidades en micro-unidades en un lado y en unidades en otro, los informes cuadran en los tests y mienten en producción. Documenta la escala junto a la columna.
- **Validar solo contra un tenant vacío** — prueba que el código corre, no que está bien. Siembra datos sintéticos.

## Referencias

Lee el archivo del store elegido **completo** antes de escribir la capa de datos; los otros, cuando toques esa parte.

| Archivo                         | Cuándo leerlo                                                                                                                                                           |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `references/architecture.md`    | Siempre. Plano de control vs tenant, contrato del `TenantManager`, envoltorio de respuestas, cableado del bootstrap.                                                    |
| `references/postgres.md`        | Store = PostgreSQL. Pool pgx, provisioning con `CREATE DATABASE`, runner de migraciones `.sql` embebidas, repositorios.                                                 |
| `references/mongodb.md`         | Store = MongoDB. Driver v2 (`mongo-driver/v2`), cliente compartido, base por empresa, migraciones como registry de funciones Go, índices, repositorios.                 |
| `references/acl.md`             | Al construir `auth` y `acl`: esquema de usuarios/roles/permisos, claims del JWT, `RequirePermission`, siembra del rol admin, cómo agregar permisos de un recurso nuevo. |
| `references/module-template.md` | Cada vez que agregues un módulo de negocio. Checklist completo de las tres capas.                                                                                       |

`assets/` trae `Makefile`, `.env.example`, `docker-compose.yml` y `Dockerfile` listos para copiar; solo cambia el nombre del módulo y del proyecto.
