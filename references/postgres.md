# Store: PostgreSQL (pgx v5)

Una base por empresa en el mismo servidor. `DATABASE_URL` apunta a la base de control y sirve de plantilla: para cada tenant se reemplaza el path de la URI por el nombre de su base.

Dependencias: `github.com/jackc/pgx/v5`, `github.com/google/uuid`.

## Conexión

```go
package database

import (
    "context"
    "errors"
    "fmt"
    "net/url"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

type Info struct {
    Host     string
    Port     int
    Database string
    SSLMode  string
}

// ConnectionInfo existe para poder loguear a dónde se está conectando sin
// filtrar la contraseña en los logs.
func ConnectionInfo(databaseURL string) (Info, error) {
    if databaseURL == "" {
        return Info{}, errors.New("DATABASE_URL is required")
    }

    config, err := pgx.ParseConfig(databaseURL)
    if err != nil {
        return Info{}, fmt.Errorf("parse database url: %w", err)
    }

    sslMode := "prefer"
    if parsed, err := url.Parse(databaseURL); err == nil {
        if mode := parsed.Query().Get("sslmode"); mode != "" {
            sslMode = mode
        }
    }

    return Info{Host: config.Host, Port: int(config.Port), Database: config.Database, SSLMode: sslMode}, nil
}

func NewPostgresPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
    info, err := ConnectionInfo(databaseURL)
    if err != nil {
        return nil, err
    }

    config, err := pgxpool.ParseConfig(databaseURL)
    if err != nil {
        return nil, fmt.Errorf("parse database url: %w", err)
    }

    pool, err := pgxpool.NewWithConfig(ctx, config)
    if err != nil {
        return nil, fmt.Errorf("connect postgres host=%s port=%d db=%s: %w", info.Host, info.Port, info.Database, err)
    }

    // El ping es lo que convierte un error de credenciales en un fallo de
    // arranque claro en vez de un 500 en el primer request.
    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("ping postgres host=%s port=%d db=%s: %w", info.Host, info.Port, info.Database, err)
    }

    return pool, nil
}
```

## Contexto del tenant

```go
package database

type tenantPoolContextKey struct{}
type tenantCompanyContextKey struct{}

func WithTenantPool(ctx context.Context, pool *pgxpool.Pool) context.Context {
    if ctx == nil || pool == nil {
        return ctx
    }
    return context.WithValue(ctx, tenantPoolContextKey{}, pool)
}

func WithTenantCompanyID(ctx context.Context, companyID string) context.Context {
    if ctx == nil || companyID == "" {
        return ctx
    }
    return context.WithValue(ctx, tenantCompanyContextKey{}, companyID)
}

func PoolFromContext(ctx context.Context, fallback *pgxpool.Pool) *pgxpool.Pool {
    if ctx != nil {
        if pool, ok := ctx.Value(tenantPoolContextKey{}).(*pgxpool.Pool); ok && pool != nil {
            return pool
        }
    }
    return fallback
}

func CompanyIDFromContext(ctx context.Context) string {
    if ctx == nil {
        return ""
    }
    companyID, _ := ctx.Value(tenantCompanyContextKey{}).(string)
    return companyID
}
```

## Migraciones

Dos carpetas embebidas, aplicadas en orden de nombre de archivo, idempotentes vía tabla `schema_migrations`. Agregar un `.sql` es todo lo que hace falta: no hay registro manual.

```go
package database

import (
    "context"
    "embed"
    "fmt"
    "io/fs"
    "sort"
    "strings"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

//go:embed control_migrations/*.sql
var controlMigrationFiles embed.FS

//go:embed tenant_migrations/*.sql
var tenantMigrationFiles embed.FS

type migration struct {
    name string
    sql  string
}

func RunControlMigrations(ctx context.Context, db *pgxpool.Pool) error {
    return runEmbeddedMigrations(ctx, db, controlMigrationFiles, "control_migrations")
}

func RunTenantMigrations(ctx context.Context, db *pgxpool.Pool) error {
    return runEmbeddedMigrations(ctx, db, tenantMigrationFiles, "tenant_migrations")
}

func runEmbeddedMigrations(ctx context.Context, db *pgxpool.Pool, files embed.FS, dir string) error {
    if _, err := db.Exec(ctx, `
        CREATE TABLE IF NOT EXISTS schema_migrations (
            name TEXT PRIMARY KEY,
            applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
        )`); err != nil {
        return fmt.Errorf("ensure migrations table: %w", err)
    }

    entries, err := fs.ReadDir(files, dir)
    if err != nil {
        return fmt.Errorf("read migrations dir: %w", err)
    }

    names := make([]string, 0, len(entries))
    for _, entry := range entries {
        if !entry.IsDir() && strings.HasSuffix(entry.Name(), ".sql") {
            names = append(names, entry.Name())
        }
    }
    sort.Strings(names)

    for _, name := range names {
        var applied bool
        if err := db.QueryRow(ctx, `SELECT EXISTS(SELECT 1 FROM schema_migrations WHERE name = $1)`, name).Scan(&applied); err != nil {
            return fmt.Errorf("check migration %s: %w", name, err)
        }
        if applied {
            continue
        }

        content, err := files.ReadFile(dir + "/" + name)
        if err != nil {
            return fmt.Errorf("read migration %s: %w", name, err)
        }

        // Cada migración va en su propia transacción: o entra completa o no entra.
        if err := applyMigration(ctx, db, migration{name: name, sql: string(content)}); err != nil {
            return fmt.Errorf("apply migration %s: %w", name, err)
        }
    }

    return nil
}

func applyMigration(ctx context.Context, db *pgxpool.Pool, item migration) error {
    tx, err := db.BeginTx(ctx, pgx.TxOptions{})
    if err != nil {
        return err
    }
    defer tx.Rollback(ctx)

    if _, err := tx.Exec(ctx, item.sql); err != nil {
        return err
    }
    if _, err := tx.Exec(ctx, `INSERT INTO schema_migrations (name) VALUES ($1)`, item.name); err != nil {
        return err
    }
    return tx.Commit(ctx)
}
```

**Convenciones de las migraciones**

- Nombre: `000001_init_control_plane.sql`, `000002_companies.sql`, … Cinco/seis dígitos, orden lexicográfico = orden de aplicación.
- Siempre `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, `INSERT … ON CONFLICT DO UPDATE`. Una migración ya aplicada no se vuelve a correr, pero la idempotencia salva cuando alguien restaura un dump o corre el runner sobre una base a medio migrar.
- **Nunca edites una migración ya desplegada.** Ya está registrada en `schema_migrations` y no volverá a ejecutarse: el cambio no llega a producción y las bases quedan divergentes. Agrega una migración nueva.
- Los datos de tenant van a `tenant_migrations/` **sin columna `company_id`**; el ACL y los catálogos globales van a `control_migrations/`.

## TenantManager

```go
package database

const (
    TenantDatabaseStatusProvisioning = "provisioning"
    TenantDatabaseStatusReady        = "ready"
    TenantDatabaseStatusFailed       = "failed"
)

type TenantDatabase struct {
    CompanyID    string
    DatabaseName string
    DatabaseURL  string
    Status       string
    LastError    *string
}

type TenantManager struct {
    controlDB        *pgxpool.Pool
    baseDatabaseURL  string
    adminDatabaseURL string // apunta a la base 'postgres': CREATE DATABASE no corre desde la base que se está creando

    mu    sync.RWMutex
    pools map[string]*pgxpool.Pool
}

func NewTenantManager(controlDB *pgxpool.Pool, baseDatabaseURL string) (*TenantManager, error) {
    if controlDB == nil {
        return nil, errors.New("tenant manager requires control database")
    }
    if baseDatabaseURL == "" {
        return nil, errors.New("tenant manager requires base database url")
    }
    return &TenantManager{
        controlDB:        controlDB,
        baseDatabaseURL:  baseDatabaseURL,
        adminDatabaseURL: adminDatabaseURLFrom(baseDatabaseURL),
        pools:            make(map[string]*pgxpool.Pool),
    }, nil
}
```

### Provisioning

Se llama al crear la empresa. Cada paso que falla marca el registro como `failed` con el motivo, para que quede rastro de por qué esa empresa no quedó utilizable.

```go
func (m *TenantManager) Provision(ctx context.Context, company Company) error {
    databaseName := tenantDatabaseName(company.ID, company.Slug)
    databaseURL, err := replaceDatabaseName(m.baseDatabaseURL, databaseName)
    if err != nil {
        return fmt.Errorf("build tenant database url: %w", err)
    }

    // 1. Registrar primero: si el proceso muere a mitad, la empresa queda visible
    //    como 'provisioning' y se puede reintentar en vez de desaparecer.
    if err := m.upsertTenantDatabase(ctx, TenantDatabase{
        CompanyID: company.ID, DatabaseName: databaseName,
        DatabaseURL: databaseURL, Status: TenantDatabaseStatusProvisioning,
    }); err != nil {
        return fmt.Errorf("register tenant database: %w", err)
    }

    fail := func(step string, err error) error {
        _ = m.markFailed(ctx, company.ID, err)
        return fmt.Errorf("%s: %w", step, err)
    }

    if err := m.ensureDatabaseExists(ctx, databaseName); err != nil {
        return fail("ensure tenant database exists", err)
    }

    tenantPool, err := NewPostgresPool(ctx, databaseURL)
    if err != nil {
        return fail("connect tenant database", err)
    }
    defer tenantPool.Close() // no se cachea: se abrirá on-demand al primer request

    if err := RunTenantMigrations(ctx, tenantPool); err != nil {
        return fail("run tenant migrations", err)
    }
    if err := seedTenantCompany(ctx, tenantPool, company); err != nil {
        return fail("seed tenant company", err)
    }

    return m.upsertTenantDatabase(ctx, TenantDatabase{
        CompanyID: company.ID, DatabaseName: databaseName,
        DatabaseURL: databaseURL, Status: TenantDatabaseStatusReady,
    })
}

func (m *TenantManager) ensureDatabaseExists(ctx context.Context, databaseName string) error {
    adminPool, err := NewPostgresPool(ctx, m.adminDatabaseURL)
    if err != nil {
        return fmt.Errorf("connect admin database: %w", err)
    }
    defer adminPool.Close()

    var exists bool
    if err := adminPool.QueryRow(ctx,
        `SELECT EXISTS(SELECT 1 FROM pg_database WHERE datname = $1)`, databaseName).Scan(&exists); err != nil {
        return fmt.Errorf("check database existence: %w", err)
    }
    if exists {
        return nil
    }

    // CREATE DATABASE no acepta parámetros; por eso el nombre se normaliza en
    // tenantDatabaseName() a [a-z0-9_] y nunca sale de input libre del usuario.
    if _, err := adminPool.Exec(ctx, fmt.Sprintf(`CREATE DATABASE "%s"`, databaseName)); err != nil {
        return fmt.Errorf("create database %s: %w", databaseName, err)
    }
    return nil
}
```

### Resolución y caché

```go
func (m *TenantManager) Resolve(ctx context.Context, companyID string) (*pgxpool.Pool, error) {
    if companyID == "" {
        return nil, errors.New("company id is required")
    }

    m.mu.RLock()
    if pool, ok := m.pools[companyID]; ok {
        m.mu.RUnlock()
        return pool, nil
    }
    m.mu.RUnlock()

    record, err := m.getTenantDatabase(ctx, companyID)
    if err != nil {
        return nil, err
    }
    if record.Status != TenantDatabaseStatusReady {
        return nil, fmt.Errorf("tenant database for company %s is %s", companyID, record.Status)
    }

    pool, err := NewPostgresPool(ctx, record.DatabaseURL)
    if err != nil {
        return nil, fmt.Errorf("connect tenant database: %w", err)
    }

    m.mu.Lock()
    defer m.mu.Unlock()
    // Doble chequeo: otro request de la misma empresa pudo ganar la carrera
    // mientras abríamos el pool. Sin esto queda un pool huérfano por cada carrera.
    if existing, ok := m.pools[companyID]; ok {
        pool.Close()
        return existing, nil
    }
    m.pools[companyID] = pool
    return pool, nil
}

func (m *TenantManager) Close() {
    m.mu.Lock()
    defer m.mu.Unlock()
    for companyID, pool := range m.pools {
        pool.Close()
        delete(m.pools, companyID)
    }
}
```

### Nombre y URI de la base

```go
// tenant_<slug>_<id sin guiones>, en minúsculas, solo [a-z0-9_], máx 63 chars
// (límite de identificador de PostgreSQL). El slug hace legible el nombre en
// psql; el id lo hace único cuando dos empresas eligen el mismo nombre.
func tenantDatabaseName(companyID, slug string) string {
    normalizedSlug := normalizePart(slug)
    normalizedID := normalizePart(strings.ReplaceAll(companyID, "-", ""))
    if normalizedSlug == "" {
        normalizedSlug = "company"
    }
    name := fmt.Sprintf("tenant_%s_%s", normalizedSlug, normalizedID)
    if len(name) > 63 {
        name = name[:63]
    }
    return strings.TrimRight(name, "_")
}

func normalizePart(value string) string {
    value = strings.ToLower(strings.TrimSpace(value))
    value = regexp.MustCompile(`[^a-z0-9]+`).ReplaceAllString(value, "_")
    value = strings.Trim(value, "_")
    if len(value) > 24 {
        value = value[:24]
    }
    return value
}

func replaceDatabaseName(rawURL, databaseName string) (string, error) {
    parsed, err := url.Parse(rawURL)
    if err != nil {
        return "", fmt.Errorf("parse database url: %w", err)
    }
    parsed.Path = "/" + databaseName
    return parsed.String(), nil
}

// La base 'postgres' siempre existe y sirve para emitir el CREATE DATABASE.
func adminDatabaseURLFrom(baseDatabaseURL string) string {
    adminURL, err := replaceDatabaseName(baseDatabaseURL, "postgres")
    if err != nil {
        return baseDatabaseURL
    }
    return adminURL
}
```

### Registro en la base de control

```go
func (m *TenantManager) upsertTenantDatabase(ctx context.Context, record TenantDatabase) error {
    _, err := m.controlDB.Exec(ctx, `
        INSERT INTO tenant_databases (company_id, database_name, database_url, status, last_error, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, NOW(), NOW())
        ON CONFLICT (company_id) DO UPDATE SET
            database_name = EXCLUDED.database_name,
            database_url  = EXCLUDED.database_url,
            status        = EXCLUDED.status,
            last_error    = EXCLUDED.last_error,
            updated_at    = NOW()
    `, record.CompanyID, record.DatabaseName, record.DatabaseURL, record.Status, record.LastError)
    return err
}

func (m *TenantManager) getTenantDatabase(ctx context.Context, companyID string) (*TenantDatabase, error) {
    var r TenantDatabase
    err := m.controlDB.QueryRow(ctx, `
        SELECT company_id, database_name, database_url, status, last_error
        FROM tenant_databases WHERE company_id = $1
    `, companyID).Scan(&r.CompanyID, &r.DatabaseName, &r.DatabaseURL, &r.Status, &r.LastError)
    if err != nil {
        return nil, fmt.Errorf("get tenant database: %w", err)
    }
    return &r, nil
}
```

## Migraciones de tenant al arranque

```go
func runReadyTenantMigrations(ctx context.Context, controlDB *pgxpool.Pool) error {
    rows, err := controlDB.Query(ctx,
        `SELECT company_id, database_url FROM tenant_databases WHERE status = $1`,
        database.TenantDatabaseStatusReady)
    if err != nil {
        return fmt.Errorf("list ready tenants: %w", err)
    }
    defer rows.Close()

    type target struct{ companyID, url string }
    targets := make([]target, 0)
    for rows.Next() {
        var t target
        if err := rows.Scan(&t.companyID, &t.url); err != nil {
            return err
        }
        targets = append(targets, t)
    }

    // Concurrencia acotada: con 200 tenants, secuencial hace el arranque eterno
    // y sin límite abre 200 conexiones de golpe.
    group, groupCtx := errgroup.WithContext(ctx)
    group.SetLimit(4)
    for _, t := range targets {
        t := t
        group.Go(func() error {
            pool, err := database.NewPostgresPool(groupCtx, t.url)
            if err != nil {
                // Que una empresa no esté disponible no debe impedir arrancar:
                // se registra y el resto sigue.
                slog.Error("tenant migration: connect failed", "companyID", t.companyID, "error", err)
                return nil
            }
            defer pool.Close()
            if err := database.RunTenantMigrations(groupCtx, pool); err != nil {
                slog.Error("tenant migration failed", "companyID", t.companyID, "error", err)
            }
            return nil
        })
    }
    return group.Wait()
}
```

## Repositorios

La única regla que importa: la conexión sale del contexto.

```go
package postgres

type ProductRepository struct {
    db *pgxpool.Pool // fallback; en un request siempre gana el pool del contexto
}

func NewProductRepository(db *pgxpool.Pool) *ProductRepository {
    return &ProductRepository{db: db}
}

func (r *ProductRepository) Create(ctx context.Context, product domain.Product) error {
    db := database.PoolFromContext(ctx, r.db)   // ← siempre así en repos de tenant

    _, err := db.Exec(ctx, `
        INSERT INTO products (id, code, name, price, is_active, created_by, updated_by, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6, $6, NOW(), NOW())
    `, product.ID, product.Code, product.Name, product.Price, product.IsActive, product.CreatedBy)
    if err != nil {
        // 23505 = unique_violation. Traducir aquí el error del motor a un error
        // de dominio evita que el código de PostgreSQL se filtre a las capas de arriba.
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) && pgErr.Code == "23505" {
            return domain.ErrProductCodeTaken
        }
        return fmt.Errorf("create product: %w", err)
    }
    return nil
}

func (r *ProductRepository) List(ctx context.Context, filters domain.Filters) ([]domain.Product, int, error) {
    db := database.PoolFromContext(ctx, r.db)
    // Sin company_id: la base ya es la de la empresa.
    ...
}
```

**Inserciones masivas:** `pgx.CopyFrom` para cargas grandes, o `INSERT … VALUES` en lotes de 500–1000 filas. Combinado con un lector en streaming (fila a fila, sin cargar el archivo entero en memoria) el uso de RAM queda constante sin importar el tamaño del archivo.

**Concurrencia:** un `UPDATE stock SET reserved = reserved + $1 WHERE id = $2 AND available - reserved >= $1` es atómico por sí solo y no necesita `SELECT FOR UPDATE`; si afecta 0 filas, no había disponibilidad. La carrera real suele estar en el `INSERT` de la fila que todavía no existe: resuélvela con `ON CONFLICT DO UPDATE`.

## Convenciones de esquema

- **IDs**: `uuid.NewV7()` — ordenables por tiempo, así el índice primario no se fragmenta como con v4.
- **Auditoría**: `created_by`, `created_at`, `updated_by`, `updated_at` en toda tabla de negocio.
- **Borrado lógico**: `is_active BOOLEAN`, `deleted_at TIMESTAMPTZ`, `deleted_by UUID`; los índices únicos van con `WHERE deleted_at IS NULL` para poder reutilizar un código liberado.
- **Índices**: todo campo por el que se filtre u ordene.
- **Dinero y cantidades**: `NUMERIC`, o enteros en una escala fija. Si eliges escala fija (por ejemplo precios ×10⁶), aplícala a **todas** las tablas que toquen ese valor y déjala escrita en un comentario de la columna. Mezclar escalas produce informes que compilan, pasan los tests y dan cifras falsas.
- **JSON en camelCase** en los DTOs (`json:"isActive"`), snake_case en las columnas. La traducción vive en el modelo de persistencia.
