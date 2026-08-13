# Plantilla de módulo de negocio

Ejemplo con `products`. Reemplaza el nombre y adapta los campos; la estructura no cambia.

## Checklist

1. `domain/` — entidad + validación, errores centinela, interfaz `Repository`, eventos
2. `application/dto/` — structs de entrada y salida
3. `application/usecases/` — un archivo por caso de uso
4. `infrastructure/persistence/` — implementación del repositorio
5. `infrastructure/http/` — handler, rutas, validación, mapeo de errores
6. `module.go` — `NewModule()` + `Register()`
7. Migración de **tenant** con la tabla/colección (sin `company_id`)
8. Migración de **control** con los permisos del recurso
9. Cableado en `bootstrap/app.go`
10. Tests de dominio y de casos de uso con repositorio mockeado

## Dominio

El dominio no importa Fiber, ni pgx, ni el driver de Mongo. Es la regla que mantiene el núcleo testeable sin base de datos.

```go
// internal/modules/products/domain/product.go
package domain

type Product struct {
    ID        string
    Code      string
    Name      string
    Price     int64
    IsActive  bool
    CreatedBy string
    CreatedAt time.Time
    UpdatedBy string
    UpdatedAt time.Time
    DeletedBy *string
    DeletedAt *time.Time
}

// El constructor normaliza y valida: si devuelve un Product, es un Product válido.
// Así ninguna capa de arriba necesita reconfirmar los invariantes.
func NewProduct(id, code, name string, price int64, createdBy string) (*Product, error) {
    now := time.Now().UTC()
    p := &Product{
        ID:        id,
        Code:      strings.TrimSpace(code),
        Name:      strings.TrimSpace(name),
        Price:     price,
        IsActive:  true,
        CreatedBy: createdBy,
        CreatedAt: now,
        UpdatedBy: createdBy,
        UpdatedAt: now,
    }
    if err := p.Validate(); err != nil {
        return nil, err
    }
    return p, nil
}

func (p *Product) Validate() error {
    if p.Code == "" {
        return ErrProductCodeRequired
    }
    if len(p.Code) > 50 {
        return ErrProductCodeTooLong
    }
    if p.Name == "" {
        return ErrProductNameRequired
    }
    if p.Price < 0 {
        return ErrProductPriceNegative
    }
    return nil
}
```

```go
// internal/modules/products/domain/errors.go
package domain

// Centinelas: el resto del código los compara con errors.Is, así que el texto
// puede cambiar sin romper nada.
var (
    ErrProductNotFound      = errors.New("product: not found")
    ErrProductCodeRequired  = errors.New("product: code is required")
    ErrProductCodeTooLong   = errors.New("product: code is too long")
    ErrProductNameRequired  = errors.New("product: name is required")
    ErrProductPriceNegative = errors.New("product: price cannot be negative")
    ErrProductCodeTaken     = errors.New("product: code already exists")
)
```

```go
// internal/modules/products/domain/repository.go
package domain

type Filters struct {
    Search   string
    IsActive *bool
    Limit    int
    Offset   int
}

// El puerto lo define el dominio y lo implementa la infraestructura: por eso la
// dependencia apunta hacia adentro y el caso de uso se puede testear con un mock.
type Repository interface {
    Create(ctx context.Context, product Product) error
    GetByID(ctx context.Context, id string) (*Product, error)
    GetByCode(ctx context.Context, code string) (*Product, error)
    List(ctx context.Context, filters Filters) ([]Product, int, error)
    Update(ctx context.Context, product Product) error
    Delete(ctx context.Context, id, deletedBy string) error
}

type EventEmitter interface {
    Emit(ctx context.Context, event Event)
}
```

Nota que ningún método recibe `companyID`: la empresa ya está en el contexto y determina la base.

**El `id string` del puerto es opaco a propósito.** El dominio nunca sabe si detrás hay un `uuid.UUID` (PostgreSQL) o un `bson.ObjectID` (MongoDB); esa traducción vive entera en `infrastructure/persistence/`. En el repositorio de Mongo eso significa un `bson.ObjectIDFromHex` al entrar y un `.Hex()` al salir, y tratar el hex inválido como "no existe" en vez de propagar un error de parseo. En el de Postgres, un `uuid.Parse` con el mismo criterio. Ver `references/mongodb.md` (sección **IDs**) y `references/postgres.md` (**Convenciones de esquema**).

## Aplicación

```go
// internal/modules/products/application/dto/create_product.go
package dto

type CreateProductInput struct {
    Code  string
    Name  string
    Price int64
}

type CreateProductOutput struct {
    ID        string    `json:"id"`
    Code      string    `json:"code"`
    Name      string    `json:"name"`
    Price     int64     `json:"price"`
    IsActive  bool      `json:"isActive"`   // camelCase en el JSON
    CreatedAt time.Time `json:"createdAt"`
}
```

```go
// internal/modules/products/application/usecases/create_product.go
package usecases

type CreateProductUseCase struct {
    repository   domain.Repository
    eventEmitter domain.EventEmitter
}

func NewCreateProductUseCase(repository domain.Repository, eventEmitter domain.EventEmitter) *CreateProductUseCase {
    return &CreateProductUseCase{repository: repository, eventEmitter: eventEmitter}
}

func (uc *CreateProductUseCase) Execute(ctx context.Context, input dto.CreateProductInput, createdBy string) (*dto.CreateProductOutput, error) {
    // 1. Unicidad. Este chequeo da el mensaje bonito, pero no es la garantía:
    //    entre el SELECT y el INSERT hay una carrera. La garantía es el índice
    //    único, y el repositorio traduce su violación a ErrProductCodeTaken.
    existing, err := uc.repository.GetByCode(ctx, input.Code)
    if err != nil {
        return nil, fmt.Errorf("check product code: %w", err)
    }
    if existing != nil {
        return nil, domain.ErrProductCodeTaken
    }

    // 2. Entidad. El ID sale de shared/id, nunca del driver: con PostgreSQL es
    //    un uuid v7 y con MongoDB el hex de un ObjectID, y el caso de uso no
    //    necesita saber cuál — para él es un string opaco.
    newID := id.New()
    product, err := domain.NewProduct(newID, input.Code, input.Name, input.Price, createdBy)
    if err != nil {
        return nil, err
    }

    // 3. Persistir
    if err := uc.repository.Create(ctx, *product); err != nil {
        return nil, err
    }

    // 4. Evento
    uc.eventEmitter.Emit(ctx, domain.ProductCreated(*product))

    return &dto.CreateProductOutput{...}, nil
}
```

`ctx` se propaga siempre tal cual: es lo que lleva la conexión del tenant hasta el repositorio.

## Infraestructura HTTP

```go
// internal/modules/products/infrastructure/http/handler.go
func (h *Handler) Create(c *fiber.Ctx) error {
    var req CreateProductRequest
    if err := c.BodyParser(&req); err != nil {
        return httpresponse.Error(c, 400, "Invalid request body", "VALIDATION_ERROR", nil)
    }

    if err := ValidateCreateRequest(req); err != nil {
        status, message, code := MapErrorToHTTPStatus(err)
        return httpresponse.Error(c, status, message, code, nil)
    }

    userID, _ := c.Locals("userID").(string)

    // c.UserContext(), NUNCA c.Context(): el pool/handle del tenant lo inyectó
    // el middleware con SetUserContext. Con c.Context() el módulo entero escribe
    // en la base de control sin dar ningún error.
    output, err := h.createProduct.Execute(c.UserContext(), input, userID)
    if err != nil {
        status, message, code := MapErrorToHTTPStatus(err)
        return httpresponse.Error(c, status, message, code, nil)
    }

    return httpresponse.Success(c, 201, "Product created successfully", output)
}
```

```go
// internal/modules/products/infrastructure/http/routes.go
func (h *Handler) RegisterRoutes(app *fiber.App, authMiddleware, tenantMiddleware fiber.Handler) {
    products := app.Group("/api/v1/products", authMiddleware, tenantMiddleware)

    // Literales antes que /:id — Fiber resuelve por orden de registro y /:id
    // se tragaría /summary y /export.
    products.Get("/summary", httpauth.RequirePermission("products", "read"), h.Summary)
    products.Get("/export",  httpauth.RequirePermission("products", "read"), h.Export)

    products.Post("/",      httpauth.RequirePermission("products", "create"), h.Create)
    products.Get("/",       httpauth.RequirePermission("products", "read"),   h.List)
    products.Get("/:id",    httpauth.RequirePermission("products", "read"),   h.Get)
    products.Put("/:id",    httpauth.RequirePermission("products", "update"), h.Update)
    products.Delete("/:id", httpauth.RequirePermission("products", "delete"), h.Delete)
}
```

## module.go

```go
package products

type Module struct {
    handler *productshttp.Handler
}

func NewModule(db database.Conn) (*Module, error) {
    // db es la conexión de respaldo; en cada request gana la del contexto.
    repository := persistence.NewProductRepository(db)
    emitter := events.NewLogEventEmitter()

    handler := productshttp.NewHandler(
        usecases.NewCreateProductUseCase(repository, emitter),
        usecases.NewGetProductUseCase(repository),
        usecases.NewListProductsUseCase(repository),
        usecases.NewUpdateProductUseCase(repository, emitter),
        usecases.NewDeleteProductUseCase(repository, emitter),
    )
    return &Module{handler: handler}, nil
}

func (m *Module) Register(app *fiber.App, authMiddleware, tenantMiddleware fiber.Handler) {
    m.handler.RegisterRoutes(app, authMiddleware, tenantMiddleware)
}
```

Cuando el módulo pase de tres o cuatro dependencias, cambia los parámetros posicionales por un struct `Dependencies`: con parámetros posicionales del mismo tipo, invertir dos por error compila igual.

## Tests

```go
// dominio: sin base de datos, sin mocks
func TestNewProduct_TrimsAndActivates(t *testing.T) {
    product, err := NewProduct("id-1", "  P-001  ", "Martillo", 1990, "user-1")
    assert.NoError(t, err)
    assert.Equal(t, "P-001", product.Code)
    assert.True(t, product.IsActive)
}

func TestNewProduct_RejectsNegativePrice(t *testing.T) {
    _, err := NewProduct("id-1", "P-001", "Martillo", -1, "user-1")
    assert.ErrorIs(t, err, ErrProductPriceNegative)
}

// caso de uso: repositorio mockeado
func TestCreateProduct_RejectsDuplicateCode(t *testing.T) {
    repo := new(MockRepository)
    repo.On("GetByCode", mock.Anything, "P-001").Return(&domain.Product{ID: "existing"}, nil)

    uc := NewCreateProductUseCase(repo, &noopEmitter{})
    _, err := uc.Execute(context.Background(), dto.CreateProductInput{Code: "P-001", Name: "X"}, "user-1")

    assert.ErrorIs(t, err, domain.ErrProductCodeTaken)
    repo.AssertExpectations(t)
}
```

Lo que de verdad conviene testear del multi-tenant es la resolución del tenant y el aislamiento: que `Resolve` devuelva error si la empresa no está `ready`, que el caché devuelva la misma conexión dos veces, y — si hay bases de prueba disponibles — que dos empresas con el mismo código de producto no se pisen. Ese último test es el que justifica toda la arquitectura.

## Importaciones masivas

Si el módulo acepta cargas de archivos, léelo **en streaming** (fila a fila, con `encoding/csv` sobre un `Reader` o el lector de filas de excelize) e inserta en lotes de 500–1000. `csv.ReadAll()` más un insert por fila funciona con el archivo de prueba y se cae con el archivo real del cliente: la memoria crece con el tamaño del archivo y cada fila es un round-trip a la base.

El caso de uso expone `ExecuteStream` y el repositorio, `CreateMany`. Devuelve el resultado por fila (número de línea + error) para que el usuario pueda corregir su archivo, en vez de un "falló la importación" sin detalle.
