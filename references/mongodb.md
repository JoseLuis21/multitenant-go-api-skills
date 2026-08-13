# Store: MongoDB (mongo-go-driver)

Una **base de datos por empresa** dentro del mismo cluster. El driver ya maneja su propio pool de conexiones, así que hay un solo `*mongo.Client` compartido y lo que se resuelve por tenant es el `*mongo.Database`. Esto es más barato que en PostgreSQL: cambiar de tenant es elegir un handle, no abrir conexiones.

Si algún tenant necesita su propio cluster (un cliente grande, o requisitos de residencia de datos), `tenant_databases` guarda su URI y el manager abre un cliente aparte solo para él. El resto sigue usando el compartido.

Dependencias: `go.mongodb.org/mongo-driver/mongo`, `github.com/google/uuid`. Los ejemplos son para el driver v1.17; en v2 `mongo.Connect` ya no recibe `ctx` y los paquetes de opciones cambian de ruta, pero la estructura es idéntica.

## Conexión

```go
package database

import (
    "context"
    "errors"
    "fmt"
    "net/url"
    "strings"

    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
    "go.mongodb.org/mongo-driver/mongo/readpref"
)

type Info struct {
    Hosts    []string
    Database string
}

func ConnectionInfo(mongoURI string) (Info, error) {
    if mongoURI == "" {
        return Info{}, errors.New("DATABASE_URL is required")
    }
    parsed, err := url.Parse(mongoURI)
    if err != nil {
        return Info{}, fmt.Errorf("parse mongo uri: %w", err)
    }
    return Info{
        Hosts:    strings.Split(parsed.Host, ","),
        Database: strings.TrimPrefix(parsed.Path, "/"),
    }, nil
}

func NewMongoClient(ctx context.Context, mongoURI string) (*mongo.Client, error) {
    opts := options.Client().
        ApplyURI(mongoURI).
        SetMaxPoolSize(50).
        SetRetryWrites(true)

    client, err := mongo.Connect(ctx, opts)
    if err != nil {
        return nil, fmt.Errorf("connect mongo: %w", err)
    }

    // mongo.Connect es perezoso: no toca el servidor. Sin este ping, unas
    // credenciales malas recién explotan en el primer request.
    if err := client.Ping(ctx, readpref.Primary()); err != nil {
        _ = client.Disconnect(ctx)
        return nil, fmt.Errorf("ping mongo: %w", err)
    }
    return client, nil
}
```

## Contexto del tenant

```go
package database

type tenantDBContextKey struct{}
type tenantCompanyContextKey struct{}

func WithTenantDatabase(ctx context.Context, db *mongo.Database) context.Context {
    if ctx == nil || db == nil {
        return ctx
    }
    return context.WithValue(ctx, tenantDBContextKey{}, db)
}

func WithTenantCompanyID(ctx context.Context, companyID string) context.Context {
    if ctx == nil || companyID == "" {
        return ctx
    }
    return context.WithValue(ctx, tenantCompanyContextKey{}, companyID)
}

func DatabaseFromContext(ctx context.Context, fallback *mongo.Database) *mongo.Database {
    if ctx != nil {
        if db, ok := ctx.Value(tenantDBContextKey{}).(*mongo.Database); ok && db != nil {
            return db
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

Mongo no tiene DDL, pero sí tiene trabajo de esquema que debe correr una sola vez y en orden: crear colecciones con validador, crear índices, sembrar catálogos, hacer backfills de campos nuevos. Se resuelve con un **registry de funciones Go ordenado por nombre**, con el mismo contrato que las migraciones SQL: idempotente y registrado en `schema_migrations`.

```go
package database

import (
    "context"
    "fmt"
    "sort"
    "time"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
)

type Migration struct {
    Name string // "000001_init_products" — el orden lexicográfico es el de aplicación
    Up   func(ctx context.Context, db *mongo.Database) error
}

// Los dos registros están separados a propósito, igual que las dos carpetas de
// .sql: lo del plano de control nunca debe correr sobre la base de una empresa.
var controlMigrations []Migration
var tenantMigrations []Migration

func RegisterControlMigration(m Migration) { controlMigrations = append(controlMigrations, m) }
func RegisterTenantMigration(m Migration)  { tenantMigrations = append(tenantMigrations, m) }

func RunControlMigrations(ctx context.Context, db *mongo.Database) error {
    return runMigrations(ctx, db, controlMigrations)
}

func RunTenantMigrations(ctx context.Context, db *mongo.Database) error {
    return runMigrations(ctx, db, tenantMigrations)
}

func runMigrations(ctx context.Context, db *mongo.Database, list []Migration) error {
    items := make([]Migration, len(list))
    copy(items, list)
    sort.Slice(items, func(i, j int) bool { return items[i].Name < items[j].Name })

    applied := db.Collection("schema_migrations")

    for _, item := range items {
        count, err := applied.CountDocuments(ctx, bson.M{"_id": item.Name})
        if err != nil {
            return fmt.Errorf("check migration %s: %w", item.Name, err)
        }
        if count > 0 {
            continue
        }

        // Sin transacción: en un cluster de un solo nodo no hay sesiones
        // transaccionales. Por eso cada Up debe ser idempotente — si falla a
        // mitad, el reintento tiene que poder repetir lo ya hecho sin romperse.
        if err := item.Up(ctx, db); err != nil {
            return fmt.Errorf("apply migration %s: %w", item.Name, err)
        }

        if _, err := applied.InsertOne(ctx, bson.M{"_id": item.Name, "appliedAt": time.Now().UTC()}); err != nil {
            return fmt.Errorf("record migration %s: %w", item.Name, err)
        }
    }
    return nil
}
```

Una migración de tenant típica: crear la colección explícitamente (Mongo la crearía sola al primer insert, pero crearla ahora hace que la base recién provisionada sea inspeccionable y que el validador aplique desde el primer documento) y sus índices.

```go
// internal/platform/database/tenant_migrations/000002_products.go
func init() {
    database.RegisterTenantMigration(database.Migration{
        Name: "000002_products",
        Up: func(ctx context.Context, db *mongo.Database) error {
            if err := createCollection(ctx, db, "products"); err != nil {
                return err
            }

            // CreateMany es idempotente si las claves y el nombre coinciden:
            // un índice ya existente no da error.
            _, err := db.Collection("products").Indexes().CreateMany(ctx, []mongo.IndexModel{
                {
                    Keys: bson.D{{Key: "code", Value: 1}},
                    Options: options.Index().SetName("ux_products_code").SetUnique(true).
                        // Índice parcial: permite reutilizar el código de un
                        // producto borrado, igual que el WHERE deleted_at IS NULL de SQL.
                        SetPartialFilterExpression(bson.M{"deletedAt": bson.M{"$eq": nil}}),
                },
                {Keys: bson.D{{Key: "name", Value: 1}}, Options: options.Index().SetName("ix_products_name")},
                {Keys: bson.D{{Key: "createdAt", Value: -1}}, Options: options.Index().SetName("ix_products_created_at")},
            })
            return err
        },
    })
}

// createCollection ignora el error 48 (NamespaceExists) para poder reintentar.
func createCollection(ctx context.Context, db *mongo.Database, name string) error {
    err := db.CreateCollection(ctx, name)
    var cmdErr mongo.CommandError
    if errors.As(err, &cmdErr) && cmdErr.Code == 48 {
        return nil
    }
    return err
}
```

Los archivos van en `internal/platform/database/tenant_migrations/*.go` y `control_migrations/*.go`, registrándose con `init()`. Importa ambos paquetes con `_` desde `bootstrap` para que los `init()` corran.

**Nunca edites una migración ya desplegada:** su nombre está en `schema_migrations` y no volverá a ejecutarse. Agrega una nueva.

## TenantManager

```go
package database

const (
    TenantDatabaseStatusProvisioning = "provisioning"
    TenantDatabaseStatusReady        = "ready"
    TenantDatabaseStatusFailed       = "failed"
)

type TenantDatabase struct {
    CompanyID    string  `bson:"_id"`
    DatabaseName string  `bson:"databaseName"`
    // Vacío = vive en el cluster compartido. Con valor = cluster propio.
    DatabaseURI  string  `bson:"databaseUri,omitempty"`
    Status       string  `bson:"status"`
    LastError    *string `bson:"lastError,omitempty"`
    CreatedAt    time.Time `bson:"createdAt"`
    UpdatedAt    time.Time `bson:"updatedAt"`
}

type TenantManager struct {
    controlDB     *mongo.Database
    sharedClient  *mongo.Client

    mu       sync.RWMutex
    dbs      map[string]*mongo.Database // companyID → handle
    clients  map[string]*mongo.Client   // uri → cliente dedicado
}

func NewTenantManager(sharedClient *mongo.Client, controlDB *mongo.Database) (*TenantManager, error) {
    if sharedClient == nil || controlDB == nil {
        return nil, errors.New("tenant manager requires mongo client and control database")
    }
    return &TenantManager{
        sharedClient: sharedClient,
        controlDB:    controlDB,
        dbs:          make(map[string]*mongo.Database),
        clients:      make(map[string]*mongo.Client),
    }, nil
}
```

### Provisioning

```go
func (m *TenantManager) Provision(ctx context.Context, company Company) error {
    databaseName := tenantDatabaseName(company.ID, company.Slug)

    // Se registra antes de tocar nada: si el proceso muere a mitad, la empresa
    // queda visible como 'provisioning' y el reintento es posible.
    if err := m.upsertTenantDatabase(ctx, TenantDatabase{
        CompanyID: company.ID, DatabaseName: databaseName,
        Status: TenantDatabaseStatusProvisioning,
    }); err != nil {
        return fmt.Errorf("register tenant database: %w", err)
    }

    fail := func(step string, err error) error {
        _ = m.markFailed(ctx, company.ID, err)
        return fmt.Errorf("%s: %w", step, err)
    }

    // Mongo crea la base al primer write, así que "crear la base" es correr sus
    // migraciones: ahí se crean las colecciones y los índices de verdad.
    tenantDB := m.sharedClient.Database(databaseName)

    if err := RunTenantMigrations(ctx, tenantDB); err != nil {
        return fail("run tenant migrations", err)
    }
    if err := seedTenantCompany(ctx, tenantDB, company); err != nil {
        return fail("seed tenant company", err)
    }

    return m.upsertTenantDatabase(ctx, TenantDatabase{
        CompanyID: company.ID, DatabaseName: databaseName,
        Status: TenantDatabaseStatusReady,
    })
}

func seedTenantCompany(ctx context.Context, db *mongo.Database, company Company) error {
    _, err := db.Collection("tenant_company").UpdateOne(ctx,
        bson.M{"_id": company.ID},
        bson.M{"$set": bson.M{
            "slug": company.Slug, "name": company.Name, "updatedAt": time.Now().UTC(),
        }, "$setOnInsert": bson.M{"createdAt": time.Now().UTC()}},
        options.Update().SetUpsert(true),
    )
    return err
}
```

### Resolución y caché

```go
func (m *TenantManager) Resolve(ctx context.Context, companyID string) (*mongo.Database, error) {
    if companyID == "" {
        return nil, errors.New("company id is required")
    }

    m.mu.RLock()
    if db, ok := m.dbs[companyID]; ok {
        m.mu.RUnlock()
        return db, nil
    }
    m.mu.RUnlock()

    record, err := m.getTenantDatabase(ctx, companyID)
    if err != nil {
        return nil, err
    }
    if record.Status != TenantDatabaseStatusReady {
        return nil, fmt.Errorf("tenant database for company %s is %s", companyID, record.Status)
    }

    client := m.sharedClient
    if record.DatabaseURI != "" {
        // Tenant con cluster propio: cliente dedicado, cacheado por URI para que
        // dos empresas del mismo cluster externo compartan conexiones.
        client, err = m.clientFor(ctx, record.DatabaseURI)
        if err != nil {
            return nil, fmt.Errorf("connect dedicated tenant cluster: %w", err)
        }
    }

    db := client.Database(record.DatabaseName)

    m.mu.Lock()
    defer m.mu.Unlock()
    if existing, ok := m.dbs[companyID]; ok {
        return existing, nil
    }
    m.dbs[companyID] = db
    return db, nil
}

func (m *TenantManager) clientFor(ctx context.Context, uri string) (*mongo.Client, error) {
    m.mu.RLock()
    if client, ok := m.clients[uri]; ok {
        m.mu.RUnlock()
        return client, nil
    }
    m.mu.RUnlock()

    client, err := NewMongoClient(ctx, uri)
    if err != nil {
        return nil, err
    }

    m.mu.Lock()
    defer m.mu.Unlock()
    if existing, ok := m.clients[uri]; ok {
        _ = client.Disconnect(ctx) // perdimos la carrera; cerramos el nuestro
        return existing, nil
    }
    m.clients[uri] = client
    return client, nil
}

func (m *TenantManager) Close() {
    m.mu.Lock()
    defer m.mu.Unlock()
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    for uri, client := range m.clients {
        _ = client.Disconnect(ctx)
        delete(m.clients, uri)
    }
    m.dbs = make(map[string]*mongo.Database)
    // sharedClient lo cierra el cleanup del bootstrap, que es quien lo creó.
}
```

### Nombre de la base

```go
// tenant_<slug>_<id sin guiones>. Límite duro de Mongo: 63 bytes, y no admite
// / \ . " $ * < > : | ? ni espacios. Normalizar a [a-z0-9_] deja fuera todo eso.
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
```

### Registro en la base de control

```go
func (m *TenantManager) upsertTenantDatabase(ctx context.Context, record TenantDatabase) error {
    now := time.Now().UTC()
    _, err := m.controlDB.Collection("tenant_databases").UpdateOne(ctx,
        bson.M{"_id": record.CompanyID},
        bson.M{
            "$set": bson.M{
                "databaseName": record.DatabaseName,
                "databaseUri":  record.DatabaseURI,
                "status":       record.Status,
                "lastError":    record.LastError,
                "updatedAt":    now,
            },
            "$setOnInsert": bson.M{"createdAt": now},
        },
        options.Update().SetUpsert(true),
    )
    return err
}

func (m *TenantManager) getTenantDatabase(ctx context.Context, companyID string) (*TenantDatabase, error) {
    var record TenantDatabase
    err := m.controlDB.Collection("tenant_databases").
        FindOne(ctx, bson.M{"_id": companyID}).Decode(&record)
    if errors.Is(err, mongo.ErrNoDocuments) {
        return nil, fmt.Errorf("tenant database for company %s not found", companyID)
    }
    if err != nil {
        return nil, fmt.Errorf("get tenant database: %w", err)
    }
    return &record, nil
}
```

## Migraciones de tenant al arranque

```go
func runReadyTenantMigrations(ctx context.Context, client *mongo.Client, controlDB *mongo.Database) error {
    cursor, err := controlDB.Collection("tenant_databases").
        Find(ctx, bson.M{"status": database.TenantDatabaseStatusReady})
    if err != nil {
        return fmt.Errorf("list ready tenants: %w", err)
    }
    var records []database.TenantDatabase
    if err := cursor.All(ctx, &records); err != nil {
        return err
    }

    group, groupCtx := errgroup.WithContext(ctx)
    group.SetLimit(4)
    for _, record := range records {
        record := record
        group.Go(func() error {
            db := client.Database(record.DatabaseName)
            if err := database.RunTenantMigrations(groupCtx, db); err != nil {
                // Una empresa con problemas no debe impedir que la API levante.
                slog.Error("tenant migration failed", "companyID", record.CompanyID, "error", err)
            }
            return nil
        })
    }
    return group.Wait()
}
```

## Repositorios

```go
package mongodb

type ProductRepository struct {
    db *mongo.Database // fallback; en un request gana el handle del contexto
}

func NewProductRepository(db *mongo.Database) *ProductRepository {
    return &ProductRepository{db: db}
}

func (r *ProductRepository) collection(ctx context.Context) *mongo.Collection {
    return database.DatabaseFromContext(ctx, r.db).Collection("products")
}

func (r *ProductRepository) Create(ctx context.Context, product domain.Product) error {
    doc := toModel(product) // _id = product.ID (uuid v7 string)

    if _, err := r.collection(ctx).InsertOne(ctx, doc); err != nil {
        // Traducir el error del driver a un error de dominio aquí evita que el
        // resto del código tenga que saber qué es un E11000.
        if mongo.IsDuplicateKeyError(err) {
            return domain.ErrProductCodeTaken
        }
        return fmt.Errorf("create product: %w", err)
    }
    return nil
}

func (r *ProductRepository) GetByID(ctx context.Context, id string) (*domain.Product, error) {
    var model productModel
    err := r.collection(ctx).FindOne(ctx, bson.M{"_id": id, "deletedAt": nil}).Decode(&model)
    if errors.Is(err, mongo.ErrNoDocuments) {
        return nil, nil // "no existe" no es un error de infraestructura; lo decide el caso de uso
    }
    if err != nil {
        return nil, fmt.Errorf("get product: %w", err)
    }
    product := model.toDomain()
    return &product, nil
}

func (r *ProductRepository) List(ctx context.Context, filters domain.Filters) ([]domain.Product, int, error) {
    // Sin companyID en el filtro: la base ya es la de la empresa.
    filter := bson.M{"deletedAt": nil}
    if filters.Search != "" {
        filter["$or"] = []bson.M{
            {"code": bson.M{"$regex": filters.Search, "$options": "i"}},
            {"name": bson.M{"$regex": filters.Search, "$options": "i"}},
        }
    }

    total, err := r.collection(ctx).CountDocuments(ctx, filter)
    if err != nil {
        return nil, 0, fmt.Errorf("count products: %w", err)
    }

    opts := options.Find().
        SetSort(bson.D{{Key: "createdAt", Value: -1}}).
        SetSkip(int64(filters.Offset)).
        SetLimit(int64(filters.Limit))

    cursor, err := r.collection(ctx).Find(ctx, filter, opts)
    if err != nil {
        return nil, 0, fmt.Errorf("list products: %w", err)
    }
    var models []productModel
    if err := cursor.All(ctx, &models); err != nil {
        return nil, 0, fmt.Errorf("decode products: %w", err)
    }
    ...
}
```

## Particularidades de Mongo que hay que decidir explícitamente

- **`_id` es el ID de dominio**, un string UUID v7, no un `ObjectID`. Así los IDs son iguales entre el plano de control y los tenants, y una API que los devuelve no cambia de formato según la tabla. UUID v7 además es ordenable por tiempo, lo que mantiene sano el índice.
- **Transacciones sólo con replica set.** Un Mongo local de un nodo no las soporta. Si la lógica de negocio las necesita (mover stock y registrar el movimiento como una unidad), levanta el entorno de desarrollo como replica set de un nodo (`--replSet rs0` + `rs.initiate()`); el `docker-compose.yml` de `assets/` ya viene así. Si no las necesitas, prefiere operaciones atómicas de un solo documento: `$inc`, `$set` condicionado y `findOneAndUpdate` con filtro son atómicos por documento y resuelven la mayoría de las carreras.
- **Unicidad = índice único.** No existe la constraint declarativa: si un campo debe ser único, hay un índice único en una migración, y el repositorio traduce el duplicado con `mongo.IsDuplicateKeyError`. Sin ese índice, la validación "ya existe" es un chequeo previo con una carrera en el medio.
- **Borrado lógico**: `deletedAt: null` en los documentos vivos, e índice único parcial con `partialFilterExpression: {deletedAt: null}` para que el código de un producto borrado se pueda reutilizar.
- **camelCase en los documentos** (`createdAt`, `isActive`), igual que en el JSON de la API. Ahorra una capa de traducción y hace legible un `find()` en la consola contra la respuesta HTTP.
- **Escalas de dinero**: Mongo guarda doubles con `float64`, que no sirve para dinero. Usa `Decimal128` (`primitive.NewDecimal128FromString`) o enteros en una escala fija documentada. Elige una y aplícala a todas las colecciones.
- **Validadores de esquema**: opcionales, pero un `$jsonSchema` en la migración que crea la colección atrapa en la base los documentos malformados que la aplicación deja pasar por un bug. Vale la pena en las colecciones críticas.
