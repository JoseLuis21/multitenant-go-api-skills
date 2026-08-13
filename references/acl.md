# Usuarios, roles y permisos (todo en el plano de control)

El ACL vive **entero** en la base de control, nunca en la base del tenant. La razón es práctica: un usuario pertenece a varias empresas y necesita autenticarse antes de que exista un tenant al que conectarse. Si los roles vivieran en la base de la empresa, el login sería un problema del huevo y la gallina.

## Modelo

```
users ──< company_users >── companies
  │                             │
  └──< user_roles >── roles ──< role_permissions >── permissions
                                                     (catálogo global)
```

- **`users`** — identidad global, email único. Existe una sola vez.
- **`companies`** — el tenant.
- **`company_users`** — membresía. Un usuario en tres empresas tiene tres filas.
- **`roles`** — por empresa. Cada empresa define los suyos; el nombre es único dentro de la empresa, no globalmente.
- **`permissions`** — catálogo **global**, no por empresa. Es la lista cerrada de lo que se puede otorgar.
- **`role_permissions`** — qué permisos tiene cada rol. **Esta tabla es la fuente de verdad.**
- **`user_roles`** — qué rol tiene el usuario en cada empresa (uno por empresa).

### Una sola fuente de verdad para los permisos del rol

Es tentador guardar también los permisos como un arreglo JSON dentro de `roles`, para leerlos de un tirón. Si lo haces, define desde el minuto uno cuál manda y haz que la otra sea derivada.

Con dos fuentes independientes pasa esto: alguien agrega un recurso nuevo y otorga sus permisos escribiendo en la columna JSON, mientras el código de autorización lee de `role_permissions`. Compila, los tests pasan, la migración corre limpia — y el permiso no se otorga nunca. El síntoma es un 403 que nadie logra explicar, meses después.

Recomendación: `role_permissions` es la fuente de verdad; si necesitas la vista JSON, constrúyela al leer.

## Esquema (PostgreSQL)

```sql
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY,
    email TEXT NOT NULL,
    email_normalized TEXT NOT NULL,        -- lower(trim(email)); es el que se busca al hacer login
    password_hash TEXT,                    -- nulo mientras el usuario fue invitado y no fijó su clave
    password_set_at TIMESTAMPTZ,
    first_name TEXT,
    last_name TEXT,
    status TEXT NOT NULL,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    CONSTRAINT ux_users_email UNIQUE (email),
    CONSTRAINT ux_users_email_normalized UNIQUE (email_normalized),
    CONSTRAINT ck_users_status CHECK (status IN ('invited','active','blocked','pending_email_verification'))
);

CREATE TABLE IF NOT EXISTS companies (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT ux_companies_slug UNIQUE (slug)
);

CREATE TABLE IF NOT EXISTS company_users (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    status TEXT NOT NULL DEFAULT 'invited',
    joined_at TIMESTAMPTZ,
    invited_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT ux_company_users_user_company UNIQUE (user_id, company_id)
);

CREATE TABLE IF NOT EXISTS tenant_databases (
    company_id UUID PRIMARY KEY REFERENCES companies(id) ON DELETE CASCADE,
    database_name TEXT NOT NULL,
    database_url TEXT NOT NULL,
    status TEXT NOT NULL,
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT ck_tenant_databases_status CHECK (status IN ('provisioning','ready','failed'))
);

CREATE TABLE IF NOT EXISTS roles (
    id UUID PRIMARY KEY,
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_by UUID REFERENCES users(id) ON DELETE RESTRICT,
    deleted_at TIMESTAMPTZ,
    -- Clave compuesta: permite que role_permissions y user_roles referencien
    -- (company_id, role_id) juntos, y así la base impide asignar el rol de una
    -- empresa a un usuario en otra.
    CONSTRAINT ux_roles_company_id_id UNIQUE (company_id, id)
);

CREATE UNIQUE INDEX IF NOT EXISTS ux_roles_company_name
    ON roles (company_id, name) WHERE deleted_at IS NULL;

CREATE TABLE IF NOT EXISTS permissions (
    key TEXT PRIMARY KEY,                  -- "products:create"
    resource TEXT NOT NULL,                -- "products"
    action TEXT NOT NULL,                  -- "create"
    resource_label TEXT NOT NULL DEFAULT '', -- nombre visible: "Productos"
    description TEXT NOT NULL DEFAULT '',  -- "Crear productos"
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS ux_permissions_resource_action ON permissions (resource, action);

CREATE TABLE IF NOT EXISTS role_permissions (
    company_id UUID NOT NULL,
    role_id UUID NOT NULL,
    permission_key TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (company_id, role_id, permission_key),
    CONSTRAINT fk_role_permissions_role FOREIGN KEY (company_id, role_id)
        REFERENCES roles(company_id, id) ON DELETE CASCADE,
    -- RESTRICT y no CASCADE: desactivar un permiso del catálogo no debe borrar
    -- en silencio lo que los roles ya tenían otorgado.
    CONSTRAINT fk_role_permissions_permission FOREIGN KEY (permission_key)
        REFERENCES permissions(key) ON DELETE RESTRICT
);

CREATE TABLE IF NOT EXISTS user_roles (
    company_id UUID NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (company_id, user_id),      -- un rol por usuario y empresa
    CONSTRAINT fk_user_roles_company FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE CASCADE,
    CONSTRAINT fk_user_roles_role FOREIGN KEY (company_id, role_id) REFERENCES roles(company_id, id) ON DELETE CASCADE
);
```

Agrega además las tablas de tokens: `email_verifications`, `password_setup_tokens`, `password_reset_tokens` (todas con `token_hash` único — se guarda el hash, nunca el token en claro — más `expires_at` y `used_at`/`consumed_at`), y `sessions` o `refresh_tokens` si el proyecto usa refresh.

## Esquema (MongoDB)

Las mismas colecciones, con estas diferencias:

- `users`: `_id` = uuid string; índices únicos en `email` y `emailNormalized`.
- `roles`: índice único parcial en `{companyId: 1, name: 1}` con `partialFilterExpression: {deletedAt: null}`.
- `permissions`: `_id` = la key (`"products:create"`); índice único en `{resource, action}`.
- `role_permissions`: un documento por trío, con índice único en `{companyId, roleId, permissionKey}`. Alternativa: un arreglo `permissionKeys` dentro del documento del rol — es legítimo en Mongo y evita el join, **siempre que sea el único lugar donde viven**. La regla de una sola fuente de verdad no cambia con el motor.
- `user_roles`: índice único en `{companyId, userId}`.
- Mongo no tiene claves foráneas: la integridad que en SQL da la FK compuesta `(company_id, role_id)` hay que hacerla cumplir en el caso de uso — al asignar un rol, verifica que el rol pertenezca a esa empresa antes de escribir.

## Formato de los permisos

`recurso:acción` en inglés, en minúsculas y plural para el recurso:

```
products:create   products:read   products:update   products:delete
sellers:read      reports:sales   *:*
```

- `*:*` es el comodín total, reservado para el rol `admin`.
- `products:*` otorga todas las acciones sobre el recurso.
- Recursos siempre en inglés aunque el dominio se hable en español (`vendedores` → `sellers`). Mezclar idiomas en el catálogo lo vuelve inmanejable en cuanto pasa de veinte recursos.
- `resource_label` y `description` **sí** van en el idioma del usuario: son lo que se muestra en la pantalla de configuración de roles y en el buscador del catálogo.

## Claims del JWT

El token lleva el contexto completo de autorización para que la ruta caliente no consulte la base:

```go
type accessTokenClaims struct {
    Email       string        `json:"eml"`
    SessionID   string        `json:"sid,omitempty"`
    Role        string        `json:"role,omitempty"`
    CompanyID   string        `json:"cid,omitempty"`   // ← lo que usa el middleware de tenant
    Permissions []interface{} `json:"perms,omitempty"`
    jwt.RegisteredClaims                               // sub = userID, iss, exp
}
```

**El contrapeso:** los permisos quedan congelados hasta que el token expira. Si le quitas un permiso a un rol, quien ya tiene token lo conserva hasta el vencimiento. Por eso el TTL del access token debe ser corto (15–60 minutos) y, si el proyecto necesita revocación inmediata, se agrega un chequeo de versión de sesión contra la base — pero eso vuelve a poner una consulta en cada request, así que hazlo sólo si el requisito existe de verdad.

**Login en dos pasos.** El usuario puede pertenecer a varias empresas, así que:

1. `POST /auth/login` → valida credenciales y devuelve un token **sin `cid`** más la lista de empresas del usuario. Con ese token solo se puede listar empresas y elegir una.
2. `POST /auth/select-company` → verifica la membresía, lee el rol y sus permisos, y emite el token definitivo con `cid`, `role` y `perms`.

Este diseño evita el error clásico de tomar la empresa de un header o del body: al venir del token firmado, el cliente no puede cambiarla.

## Middleware de autenticación

```go
func NewJWTAuthMiddleware(secret, issuer string) fiber.Handler {
    return func(c *fiber.Ctx) error {
        tokenString := bearerToken(c.Get("Authorization"))
        if tokenString == "" {
            return httpresponse.Error(c, 401, "Unauthorized", "UNAUTHORIZED", nil)
        }

        claims := &accessTokenClaims{}
        token, err := jwt.ParseWithClaims(tokenString, claims, func(t *jwt.Token) (interface{}, error) {
            // Sin este chequeo, un token firmado con alg "none" o con RS256 y una
            // clave pública conocida pasaría la validación. Es el agujero clásico.
            if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, errors.New("unexpected signing method")
            }
            return []byte(secret), nil
        })
        if err != nil || !token.Valid {
            return httpresponse.Error(c, 401, "Unauthorized", "UNAUTHORIZED", nil)
        }
        if issuer != "" && claims.Issuer != issuer {
            return httpresponse.Error(c, 401, "Unauthorized", "UNAUTHORIZED", nil)
        }
        if claims.Subject == "" {
            return httpresponse.Error(c, 401, "Unauthorized", "UNAUTHORIZED", nil)
        }

        c.Locals("userID", claims.Subject)
        c.Locals("email", claims.Email)
        c.Locals("sessionID", claims.SessionID)
        c.Locals("role", claims.Role)
        c.Locals("companyID", claims.CompanyID)

        permissions := make([]string, 0)
        for _, p := range claims.Permissions {
            if s, ok := p.(string); ok {
                permissions = append(permissions, s)
            }
        }
        c.Locals("permissions", permissions)

        return c.Next()
    }
}
```

## Middleware de permisos

```go
func RequirePermission(resource, action string) fiber.Handler {
    return func(c *fiber.Ctx) error {
        permissions := PermissionFromContext(c)
        if len(permissions) == 0 {
            return httpresponse.Error(c, 403, "Forbidden", "FORBIDDEN", nil)
        }
        if MatchesPermission(permissions, resource, action) {
            return c.Next()
        }
        return httpresponse.Error(c, 403, "Forbidden", "FORBIDDEN", nil)
    }
}

func MatchesPermission(permissions []string, resource, action string) bool {
    required := resource + ":" + action
    for _, p := range permissions {
        if p == "*:*" || p == required {
            return true
        }
        // "products:*" cubre cualquier acción sobre products
        if strings.HasSuffix(p, ":*") && strings.HasPrefix(p, resource+":") {
            return true
        }
    }
    return false
}

// El claim puede llegar como []string o como []interface{} según cómo se haya
// deserializado; soportar ambos evita un 403 fantasma difícil de diagnosticar.
func PermissionFromContext(c *fiber.Ctx) []string {
    if perms, ok := c.Locals("permissions").([]string); ok {
        return perms
    }
    if raw, ok := c.Locals("permissions").([]interface{}); ok {
        result := make([]string, 0, len(raw))
        for _, p := range raw {
            if s, ok := p.(string); ok {
                result = append(result, s)
            }
        }
        return result
    }
    return nil
}
```

Uso en las rutas — el orden es el contrato:

```go
func (h *Handler) RegisterRoutes(app *fiber.App, authMiddleware, tenantMiddleware fiber.Handler) {
    products := app.Group("/api/v1/products", authMiddleware, tenantMiddleware)

    products.Get("/summary", httpauth.RequirePermission("products", "read"), h.Summary) // literales primero
    products.Post("/",       httpauth.RequirePermission("products", "create"), h.Create)
    products.Get("/",        httpauth.RequirePermission("products", "read"),   h.List)
    products.Get("/:id",     httpauth.RequirePermission("products", "read"),   h.Get)
    products.Put("/:id",     httpauth.RequirePermission("products", "update"), h.Update)
    products.Delete("/:id",  httpauth.RequirePermission("products", "delete"), h.Delete)
}
```

## Alta de empresa y rol admin

Crear una empresa es una secuencia con un orden obligado, porque cada paso depende del anterior:

```go
func (uc *CreateCompanyUseCase) Execute(ctx context.Context, input dto.CreateCompanyInput, userID string) (*dto.CreateCompanyOutput, error) {
    // 1. Empresa en el plano de control
    company, err := domain.NewCompany(id, input.Name, slugify(input.Name), userID)
    if err := uc.companies.Create(ctx, *company); err != nil { ... }

    // 2. Membresía del creador
    if err := uc.companies.CreateMembership(ctx, company.ID, userID); err != nil { ... }

    // 3. Base del tenant: crear, migrar, sembrar. Es el paso que puede tardar.
    if err := uc.tenantManager.Provision(ctx, *company); err != nil { ... }

    // 4. Rol admin con *:* y asignación al creador. Sin esto, quien acaba de
    //    crear la empresa entra y recibe 403 en todo.
    if err := uc.roleSeeder.SeedDefaultAdminRole(ctx, company.ID, userID); err != nil { ... }

    return output, nil
}
```

El sembrado del admin debe ser idempotente: si el rol ya existe, se reutiliza y solo se asigna.

```go
func (s *RoleSeederService) SeedDefaultAdminRole(ctx context.Context, companyID, userID string) error {
    existing, err := s.roles.GetByName(ctx, companyID, "admin")
    if err != nil {
        return fmt.Errorf("seed admin role: get role: %w", err)
    }
    if existing != nil {
        return s.assignments.AssignRole(ctx, userID, existing.ID, companyID)
    }

    roleID := uuid.Must(uuid.NewV7()).String()
    role, err := domain.NewRole(roleID, companyID, "admin", nil, []string{"*:*"}, userID)
    if err != nil {
        return fmt.Errorf("seed admin role: create entity: %w", err)
    }
    if err := s.roles.Create(ctx, *role); err != nil {
        return fmt.Errorf("seed admin role: persist: %w", err)
    }
    return s.assignments.AssignRole(ctx, userID, roleID, companyID)
}
```

**Si el provisioning falla a mitad**, la empresa queda registrada con estado `failed` y el usuario sin poder trabajar. Decide qué hacer y hazlo explícito: o el alta es transaccional en el plano de control y se revierte, o queda en `failed` con un endpoint de reintento. Lo que no sirve es dejar el hueco sin decidir — es el tipo de estado a medias que después nadie sabe cómo limpiar.

## Endpoints del ACL

```
GET    /api/v1/permissions              catálogo (filtros: resource, search)
GET    /api/v1/roles                    roles de la empresa
POST   /api/v1/roles                    crear rol (valida las keys contra el catálogo)
GET    /api/v1/roles/:id
PUT    /api/v1/roles/:id
DELETE /api/v1/roles/:id                borrado lógico
GET    /api/v1/role-assignments         qué rol tiene cada usuario de la empresa
POST   /api/v1/role-assignments         asignar rol a usuario
PUT    /api/v1/role-assignments/:userId
DELETE /api/v1/role-assignments/:userId
```

Al crear o actualizar un rol, valida las keys recibidas contra el catálogo (`SELECT key FROM permissions WHERE is_active AND key = ANY($1)`) y rechaza con 422 las que no existan. Sin esa validación se otorgan permisos con typos que nunca coinciden con ningún `RequirePermission` y el 403 resultante parece un bug del backend.

Estas rutas usan `authMiddleware` pero **no** el de tenant: el ACL vive en el plano de control.

## Agregar los permisos de un recurso nuevo

Una migración de control por recurso, idempotente:

```sql
INSERT INTO permissions (key, resource, action, resource_label, description, is_active)
VALUES
    ('warehouses:create', 'warehouses', 'create', 'Bodegas', 'Crear bodegas', true),
    ('warehouses:read',   'warehouses', 'read',   'Bodegas', 'Ver bodegas',   true),
    ('warehouses:update', 'warehouses', 'update', 'Bodegas', 'Editar bodegas', true),
    ('warehouses:delete', 'warehouses', 'delete', 'Bodegas', 'Eliminar bodegas', true)
ON CONFLICT (key) DO UPDATE SET
    resource_label = EXCLUDED.resource_label,
    description    = EXCLUDED.description,
    is_active      = EXCLUDED.is_active,
    updated_at     = NOW();
```

El `DO UPDATE` (y no `DO NOTHING`) es lo que permite corregir después una etiqueta o una descripción.

Los roles existentes **no** reciben el permiso nuevo automáticamente: hay que otorgarlo desde la pantalla de roles. El único que lo tiene desde el primer día es `admin`, porque `*:*` cubre lo que todavía no existe. Si quieres que un rol concreto lo reciba, agrégalo explícitamente en la misma migración insertando en `role_permissions` — y asegúrate de escribir en la tabla que el código realmente lee.

## Contraseñas y tokens

- Hash con **bcrypt** (`golang.org/x/crypto/bcrypt`, coste ≥ 12) o **argon2id**. Nunca SHA sin salt.
- Los tokens de verificación de email y de reset se guardan **hasheados** (SHA-256 basta, son de un solo uso y corta vida); el token en claro solo viaja en el correo.
- El reset de password responde igual exista o no el email. Si responde distinto, el endpoint se vuelve un enumerador de usuarios.
- Al cambiar la contraseña, invalida las sesiones activas del usuario.
- Guarda `email_normalized` (lower + trim) y busca por ahí: `Jose@x.com` y `jose@x.com` deben ser el mismo usuario.
