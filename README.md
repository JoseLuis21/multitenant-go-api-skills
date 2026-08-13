# multitenant-go-api

Skill de [Claude Code](https://claude.com/claude-code) para levantar una API en **Go + Fiber v2** con arquitectura **multi-tenant de base de datos por empresa** (control plane + una base por tenant), autenticación JWT y permisos/roles/usuarios centralizados.

Funciona con **PostgreSQL** o con **MongoDB**: la skill pregunta cuál usas y genera el scaffold para ese motor. Toda la parte específica del store queda detrás del `TenantManager` y de los repositorios, así que el resto del código es idéntico en ambos casos.

## Qué genera

- Plano de control con `users`, `companies`, `company_users`, `tenant_databases` y el ACL completo (`roles`, `permissions`, `role_permissions`, `user_roles`).
- Una base de datos por empresa, provisionada al crearla: se crea, se migra y se siembra sola.
- `TenantManager` que resuelve y cachea la conexión de cada empresa, y middleware que la inyecta en el `context.Context` de cada request.
- Autenticación JWT con login en dos pasos (credenciales → elegir empresa) y middleware `RequirePermission(resource, action)`.
- Dos juegos de migraciones idempotentes: control y tenant, con runner incluido.
- Módulos hexagonales (`domain` → `application` → `infrastructure`) y un CRUD de ejemplo ya cableado.
- `Makefile`, `Dockerfile`, `docker-compose.yml` y `.env.example`.

El aislamiento es físico: las tablas de tenant **no llevan columna `company_id`**. Una consulta mal escrita no puede filtrar datos de otro cliente porque no están en esa base.

## Instalación

Para todos tus proyectos:

```bash
git clone https://github.com/<usuario>/multitenant-go-api-skill.git ~/.claude/skills/multitenant-go-api
```

Solo para un proyecto (queda versionada en el repo):

```bash
git clone https://github.com/<usuario>/multitenant-go-api-skill.git .claude/skills/multitenant-go-api
```

Verifica con `/skills` dentro de Claude Code.

## Uso

Se activa sola cuando pides algo del estilo:

- "necesito arrancar una API multi-tenant como la de facturación"
- "arma el backend con base por cliente y roles"
- "quiero pasar este proyecto a multi-tenant"

O invócala directamente: `/multitenant-go-api`.

Lo primero que hace es preguntar cuatro cosas —store (PostgreSQL o MongoDB), módulo Go, cómo se llama el tenant en tu negocio y si quieres un CRUD de ejemplo— y con eso arma el proyecto.

## Estructura

```
SKILL.md                        instrucciones y flujo de trabajo
references/
  architecture.md               control plane vs tenant, TenantManager, bootstrap
  postgres.md                   implementación con pgx v5
  mongodb.md                    implementación con mongo-go-driver
  acl.md                        usuarios, roles, permisos, JWT
  module-template.md            plantilla de módulo de negocio
assets/
  Makefile  Dockerfile  .env.example
  docker-compose.postgres.yml   docker-compose.mongo.yml
```

## Licencia

MIT
