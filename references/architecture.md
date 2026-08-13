# Arquitectura: plano de control y planos de tenant

## Qué vive en cada base

**Base de control (una sola, siempre disponible):**

- `users` — identidad global. Un usuario existe una vez aunque pertenezca a cinco empresas.
- `companies` — el tenant (renómbrala si el negocio la llama de otra forma).
- `company_users` — membresías usuario↔empresa.
- `tenant_databases` — el registro: para cada empresa, dónde está su base y en qué estado.
- `roles`, `permissions`, `role_permissions`, `user_roles` — el ACL completo.
- Sesiones/refresh tokens, tokens de verificación de email y de reset de password.
- Cualquier cosa transversal al SaaS: planes, suscripciones, facturación propia, catálogos globales (bancos, comunas, tipos de documento).

**Base de tenant (una por empresa):**

- Todo el dato de negocio: productos, clientes, movimientos, bodegas, tesorería…
- Una copia espejo mínima de la empresa (`tenant_company`: id, slug, nombre) para que la base sea autoexplicativa en un dump o una restauración.
- **Sin columna `company_id`.** El aislamiento ya lo da la base.

La pregunta para decidir dónde va una tabla nueva: *¿el dato tiene sentido sin una empresa?* Si sí (usuario, plan, catálogo de países), es de control. Si no, es de tenant.

## El registro de tenants

```
tenant_databases
  company_id     PK / único        — a qué empresa pertenece
  database_name                    — nombre de la base
  database_url                     — URI de conexión (o solo el nombre si comparten cluster)
  status         provisioning | ready | failed
  last_error                       — el error del último intento fallido
  created_at, updated_at
```

`status` no es decorativo: el middleware de tenant solo resuelve conexiones de empresas en `ready`. Una empresa en `provisioning` o `failed` responde 503 en vez de conectarse a medias. Cuando el provisioning falla se guarda el motivo en `last_error`, que es lo único que después permite entender por qué una empresa quedó a medio crear.

Guardar la URI completa (y no solo el nombre) es lo que permite mover un tenant grande a otro servidor sin tocar código.

## Contrato del TenantManager

Es la única pieza que sabe de motores de base de datos. Todo lo demás habla con esta interfaz.

```go
type TenantManager interface {
    // Crea la base de la empresa, corre sus migraciones, siembra los datos
    // iniciales y la deja en 'ready'. Idempotente: correrla dos veces sobre la
    // misma empresa no debe romper nada.
    Provision(ctx context.Context, company Company) error

    // Devuelve la conexión de la empresa, creándola y cacheándola la primera vez.
    // Error si la empresa no existe o su base no está 'ready'.
    Resolve(ctx context.Context, companyID string) (Conn, error)

    // Cierra todas las conexiones cacheadas (shutdown).
    Close()
}
```

`Conn` es `*pgxpool.Pool` con PostgreSQL y `*mongo.Database` con MongoDB. La implementación concreta está en `references/postgres.md` y `references/mongodb.md`.

**Sobre el caché de conexiones:** el mapa `companyID → conexión` se protege con `sync.RWMutex` — lectura optimista con `RLock`, y al insertar bajo `Lock` hay que volver a chequear si otra goroutine ya insertó (si sí, cierra la que acabas de crear y devuelve la existente). Sin ese doble chequeo, dos requests simultáneos de la misma empresa recién despierta abren dos pools y uno queda huérfano.

Ojo con el crecimiento: N empresas activas = N pools abiertos. Con pocos cientos de tenants va bien; más arriba conviene un límite de conexiones por pool bajo (2–5) y evicción por inactividad.

## Contexto del tenant

La conexión viaja en el `context.Context`, no en el struct del repositorio. Así el mismo repositorio sirve para todas las empresas y no hay estado por request en las dependencias.

```go
type tenantConnKey struct{}
type tenantCompanyKey struct{}

func WithTenantConn(ctx context.Context, conn Conn) context.Context
func WithTenantCompanyID(ctx context.Context, companyID string) context.Context

// fallback es la conexión inyectada al construir el repositorio; solo se usa
// cuando el contexto no trae ninguna (tests, jobs fuera de un request HTTP).
func ConnFromContext(ctx context.Context, fallback Conn) Conn
func CompanyIDFromContext(ctx context.Context) string
```

Usa claves de tipo struct vacío privado, no strings: evita colisiones con otros paquetes y hace imposible que alguien escriba la clave desde fuera.

## Middleware de tenant

```go
func NewTenantMiddleware(manager TenantManager) fiber.Handler {
    return func(c *fiber.Ctx) error {
        companyID, _ := c.Locals("companyID").(string)
        if companyID == "" {
            return httpresponse.Error(c, 400, "Company context is required", "COMPANY_CONTEXT_REQUIRED", nil)
        }

        conn, err := manager.Resolve(c.UserContext(), companyID)
        if err != nil {
            status := fiber.StatusServiceUnavailable
            if errors.Is(err, context.DeadlineExceeded) {
                status = fiber.StatusGatewayTimeout
            }
            return httpresponse.Error(c, status, "Tenant database is not available", "TENANT_DATABASE_UNAVAILABLE", nil)
        }

        ctx := WithTenantConn(c.UserContext(), conn)
        ctx = WithTenantCompanyID(ctx, companyID)
        c.SetUserContext(ctx)   // ← y por eso los handlers deben pasar c.UserContext()
        return c.Next()
    }
}
```

`companyID` lo puso el middleware de auth desde el claim `cid` del JWT. Que venga del token firmado y no de un header es lo que impide que un cliente pida datos de otra empresa cambiando un parámetro.

**Extensiones habituales del middleware** (agrégalas solo si el proyecto las pide): un *gate* de suscripción que corta con 402 antes de resolver la conexión cuando la empresa dejó de pagar. Si lo agregas, deja las rutas de auth, empresa y suscripción fuera de este middleware — si no, un cliente moroso no puede ni entrar a pagar. Y que un error del gate no bloquee (fail-open): un problema del plano de billing no debe tumbar la operación.

## Envoltorio de respuestas

Una sola forma de responder en toda la API, para que el frontend tenga un solo camino de manejo de errores:

```go
// éxito
{"success": true,  "message": "Product created successfully", "data": {...}}
// error
{"success": false, "message": "Product not found", "code": "NOT_FOUND", "errors": {...}}
```

```go
package httpresponse

func Success(c *fiber.Ctx, status int, message string, data any) error {
    return c.Status(status).JSON(fiber.Map{"success": true, "message": message, "data": data})
}

func Error(c *fiber.Ctx, status int, message, code string, details any) error {
    body := fiber.Map{"success": false, "message": message, "code": code}
    if details != nil {
        body["errors"] = details
    }
    return c.Status(status).JSON(body)
}
```

Los listados devuelven `data: {"total": 120, "items": [...]}`. Decidir esto el día uno evita la migración incómoda cuando el primer listado necesita paginar.

El mapeo de error de dominio a HTTP vive en cada módulo, en `infrastructure/http/errors.go`, traduciendo errores centinela:

```go
func MapErrorToHTTPStatus(err error) (int, string, string) {
    switch {
    case errors.Is(err, domain.ErrProductNotFound):
        return 404, "Product not found", "NOT_FOUND"
    case errors.Is(err, domain.ErrProductCodeTaken):
        return 409, "Product code already exists", "CONFLICT"
    case errors.Is(err, domain.ErrProductCodeRequired):
        return 422, "Product code is required", "VALIDATION_ERROR"
    default:
        return 500, "Internal server error", "INTERNAL_ERROR"
    }
}
```

Cuidado con el `default: 500`: un mapper genérico que traga todo hace que un 401, un 403 o un 409 legítimos lleguen al frontend como "error inesperado" y se reporten como bugs de backend que no existen. Mapea explícitamente cada error que el dominio pueda devolver.

## Bootstrap

`NewApp()` cablea todo en secuencia y devuelve `(*fiber.App, cleanup func(), error)`. El orden importa porque cada bloque depende del anterior:

```go
func NewApp() (*fiber.App, func(), error) {
    app := fiber.New()
    app.Use(recover.New())
    app.Use(logger.New(logger.Config{Next: func(c *fiber.Ctx) bool { return c.Path() == "/healthz" }}))
    app.Get("/healthz", healthHandler)

    // 1. Contexto de arranque con timeout: si la base no responde, el proceso
    //    muere con un error claro en vez de quedarse colgado en el deploy.
    ctx, cancel := context.WithTimeout(context.Background(), startupTimeout())
    defer cancel()

    // 2. Conexión de control + migraciones de control
    controlDB, err := database.Connect(ctx, os.Getenv("DATABASE_URL"))
    if err := database.RunControlMigrations(ctx, controlDB); err != nil { ... }

    // 3. Migraciones de tenant sobre cada empresa 'ready'
    //    (así una empresa creada hace meses recibe las tablas nuevas)
    if err := runReadyTenantMigrations(ctx, controlDB); err != nil { ... }

    // 4. Middlewares transversales
    authMiddleware := httpauth.NewJWTAuthMiddleware(secret, issuer)
    tenantManager, err := database.NewTenantManager(controlDB, os.Getenv("DATABASE_URL"))
    tenantMiddleware := database.NewTenantMiddleware(tenantManager)

    // 5. Módulos. Cada uno: NewModule(deps) + Register(app, middlewares...)
    authModule, _ := auth.NewModule(auth.Dependencies{ControlDB: controlDB, ...})
    authModule.Register(app, authMiddleware)

    companyModule, _ := company.NewModule(company.Dependencies{
        ControlDB: controlDB, TenantManager: tenantManager, RoleSeeder: roleSeeder,
    })
    companyModule.Register(app, authMiddleware)

    productsModule, _ := products.NewModule(controlDB)
    productsModule.Register(app, authMiddleware, tenantMiddleware)

    cleanup := func() { tenantManager.Close(); controlDB.Close() }
    return app, cleanup, nil
}
```

**Migraciones de tenant al arranque:** recorre `tenant_databases` en estado `ready`, abre cada base y corre el runner. Es lo que mantiene sincronizadas las empresas viejas. Con muchos tenants conviene hacerlo con concurrencia acotada (un `errgroup` con límite ~4) y que el fallo de una empresa se registre pero no impida arrancar la API.

**Dependencias por struct, no posicionales.** Un módulo con seis parámetros posicionales es una bomba: el día que agregas el séptimo, cualquier llamada mal ordenada compila igual si los tipos coinciden.

```go
type Dependencies struct {
    ControlDB     database.Conn
    TenantManager *database.TenantManager
    Mailer        mail.Sender
    Encryptor     crypto.Encryptor
}
```

**Colaboración entre módulos por interfaces chicas.** Cuando el módulo A necesita algo de B, A declara la interfaz mínima en su propio dominio y bootstrap le pasa la implementación de B. Evita el import cruzado y los ciclos:

```go
// en el dominio de sales
type StockDispatcher interface {
    Reserve(ctx context.Context, productID string, qty int64) error
}
```

## Orden de rutas en Fiber

Fiber resuelve por orden de registro, así que una ruta paramétrica declarada antes se come a las literales:

```go
// mal: GET /products/summary entra por :id y revienta al parsear el ID
products.Get("/:id", h.Get)
products.Get("/summary", h.Summary)

// bien
products.Get("/summary", h.Summary)
products.Get("/export", h.Export)
products.Get("/:id", h.Get)
```

Este error no da un 404 evidente sino un 500 de parseo dentro del handler equivocado, y por eso cuesta encontrarlo.

## Variables de entorno

| Variable | Para qué |
|---|---|
| `APP_ENV` | `development` / `production` |
| `APP_PORT` | Puerto del servidor (por defecto 3000) |
| `DATABASE_URL` | Conexión de la base de control **y** plantilla de las bases de tenant |
| `JWT_SECRET` | Firma HMAC de los access tokens |
| `JWT_ISSUER` | Emisor esperado; se valida al parsear el token |
| `JWT_EXPIRATION_MINUTES` | TTL del access token |
| `STARTUP_TIMEOUT_SECONDS` | Corte del arranque si la base no responde |
| `PUBLIC_API_URL` | URL pública de la API, para callbacks de terceros |

Valida al arrancar todo lo que sea obligatorio y falla con un mensaje explícito. Un secreto vacío que solo revienta en el primer login es mucho peor que un proceso que no levanta.
