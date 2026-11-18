# SuperHero, inc. — Local Hero API (SI730 PC2 2024-2)

Solución para la Práctica Calificada 2 de **Aplicaciones Web SI730**, implementada con **C# / .NET 8 / ASP.NET Core**, **Domain-Driven Design** y **CQRS**.

> **Importante antes de entregar:** reemplaza `000000000` por tu código de estudiante en:
> - Nombre de la solución: `si730pc2u000000000.API.sln`
> - Nombre del proyecto: `si730pc2u000000000.API.csproj`
> - Namespace raíz: `si730pc2u000000000.API` (usa *Find & Replace* en todo el proyecto)
> - Etiquetas `<remarks>` con tus nombres y apellidos (actualmente dice *Joaquín Aguilar*)
> - Nombre del `.zip`: `upc-pre-202402-si730-ws51-pc2-u000000000.zip`

---

## 1. Alcance implementado

- **Endpoint obligatorio del examen**: `POST /api/v1/heros/{heroId}`.
- **Endpoints GET adicionales** (agregados a pedido, fuera del alcance mínimo del enunciado):
  - `GET /api/v1/heros/total` → total de héroes registrados.
  - `GET /api/v1/heros/{id}` → héroe por su `id` autogenerado (`404` si no existe).
- El `heroId` viaja **en el path**, nunca en el body.
- El `id` es autogenerado por MySQL y se devuelve en el response.
- `registerDateAt` se genera en el servidor (`DateTime.UtcNow`), no lo envía el cliente.
- Persistencia en MySQL, base de datos / esquema **`local-hero`**, tabla **`heros`** (snake_case, plural).
- Documentación OpenAPI/Swagger completa con comentarios XML.
- Manejo global de excepciones con respuestas `ProblemDetails`.

## 2. Arquitectura

```
src/si730pc2u000000000.API/
├── Shared/                                  # Bounded context Shared
│   ├── Application/Cqrs/                    # ICommand, IQuery, dispatchers
│   ├── Domain/Model/
│   │   ├── Aggregates/AuditableEntity.cs    # id, created_at, updated_at
│   │   ├── Exceptions/                      # DomainException, BusinessRuleValidationException
│   │   ├── ValueObjects/EmailAddress.cs     # VO reutilizable
│   │   └── Guard.cs
│   ├── Domain/Repositories/                 # IRepository<T>, IUnitOfWork
│   ├── Infrastructure/Persistence/EFCore/   # AppDbContext + convenciones ORM
│   └── Interfaces/REST/                     # Middleware + extensiones DI
├── HERO/                                    # Bounded context HERO
│   ├── Domain/Model/
│   │   ├── Aggregates/Hero.cs               # Aggregate root
│   │   ├── ValueObjects/{HeroId,Continent}.cs
│   │   └── Exceptions/DuplicateHeroException.cs
│   ├── Domain/Repositories/                 # Write repo + read repo (CQRS)
│   ├── Application/
│   │   ├── Commands/CreateHeroCommand*.cs   # Write side
│   │   ├── Queries/HeroExistsQuery*.cs      # Read side (unicidad)
│   │   ├── Queries/GetHeroByIdQuery*.cs     # Read side (GET por id)
│   │   ├── Queries/GetHeroesTotalQuery*.cs  # Read side (GET total)
│   │   ├── Dtos/                            # Request / Resource
│   │   └── Mapping/HeroMapper.cs
│   ├── Infrastructure/Persistence/EFCore/   # HeroConfiguration + repositorios
│   └── Interfaces/REST/Controllers/         # HeroesController
├── Migrations/                              # Migración InitialCreate
└── Program.cs
```

### Decisiones DDD / CQRS

| Elemento | Decisión |
|---|---|
| `Hero` | Aggregate root; hereda de `AuditableEntity`. |
| `HeroId` | Value object del contexto HERO (`long > 0`). |
| `Continent` | *Enumeration* (smart enum) con ids 1..5 y nombres `AMERICAN`, `EUROPE`, `AFRICA`, `ASIA`, `OCEANIA`. Se persiste como `int`. |
| `EmailAddress` | Value object del contexto **Shared**, reutilizable, con regex y máximo 50 caracteres. |
| CQRS | Comando `CreateHeroCommand` (escritura) y queries `HeroExistsQuery`, `GetHeroByIdQuery` y `GetHeroesTotalQuery` (lectura), con repositorios e interfaces separadas. |
| Unicidad | Regla en el aggregate + query en el lado de lectura + **índice único** en MySQL como última barrera ante concurrencia. |
| Auditoría | `AppDbContext` sobrescribe `SaveChanges`/`SaveChangesAsync` y llena `created_at` / `updated_at`. |
| ORM | `AddSnakeCaseNames()` aplica snake_case a columnas y plural a tablas; `ConfigureAuditableEntities()` configura auditoría. |

## 3. Reglas de negocio

| Regla | Dónde se aplica | Error |
|---|---|---|
| `heroId` > 0 | `HeroId` VO | 400 |
| Email válido, no vacío, ≤ 50 | `EmailAddress` VO (Shared) | 400 |
| `heroPlace` ≤ 50, obligatorio | `Guard` en `Hero` | 400 |
| `heroCountry` ≤ 250, obligatorio | `Guard` en `Hero` | 400 |
| `heroAbility` ≤ 50, obligatorio | `Guard` en `Hero` | 400 |
| Continent obligatorio y entre 1..5 | `Continent.FromId` / `FromName` | 400 |
| Continent debe coincidir con un nombre válido | `Continent.FromName` | 400 |
| No repetir (heroId, place, country, continent) | Query + índice único `ux_heros_registration_key` | 409 |
| `registerDateAt` generado por servidor | `CreateHeroCommandHandler` | — |

## 4. Configuración y ejecución

### 4.1 Base de datos

El `appsettings.json` trae:

```json
"ConnectionStrings": {
  "LocalHeroDatabase": "Server=localhost;Port=3306;Database=local-hero;Uid=root;Pwd=root;"
}
```

Ajusta usuario/contraseña según tu MySQL. La base de datos se llama `local-hero` (con guion, por ello se escribe entre backticks en SQL):

```sql
CREATE DATABASE IF NOT EXISTS `local-hero`;
```

### 4.2 Dependencias

Versiones alineadas con el proyecto de ejemplo *Learning Center Platform* visto en clase:

| Paquete | Versión |
|---|---|
| `Microsoft.EntityFrameworkCore` | `8.0.4` |
| `Microsoft.EntityFrameworkCore.Design` | `8.0.4` |
| `Pomelo.EntityFrameworkCore.MySql` | `8.0.2` |
| `Swashbuckle.AspNetCore` | `6.5.0` |

### 4.3 Comandos

```bash
cd SuperHeroLocalHeroAPI
dotnet restore
dotnet build

# Aplicar la migración (crea la tabla heros)
dotnet tool install --global dotnet-ef --version 8.0.4   # si no lo tienes
dotnet ef database update --project src/si730pc2u000000000.API/si730pc2u000000000.API.csproj \
                          --startup-project src/si730pc2u000000000.API/si730pc2u000000000.API.csproj

# Ejecutar
dotnet run --project src/si730pc2u000000000.API/si730pc2u000000000.API.csproj
```

Swagger: **http://localhost:5104/swagger**

## 5. Ejemplo de uso

```bash
curl -i -X POST http://localhost:5104/api/v1/heros/123 \
  -H "Content-Type: application/json" \
  -d '{
        "registerEmailAddress": "owner@superhero.com",
        "heroPlace": "Lima",
        "heroCountry": "Peru",
        "heroContinent": "AMERICAN",
        "heroAbility": "Flight"
      }'
```

Respuesta `201 Created`:

```json
{
  "id": 1,
  "heroId": 123,
  "registerEmailAddress": "owner@superhero.com",
  "heroPlace": "Lima",
  "heroCountry": "Peru",
  "heroContinent": "AMERICAN",
  "heroAbility": "Flight",
  "registerDateAt": "2026-11-17T18:30:00Z",
  "createdAt": "2026-11-17T18:30:00Z",
  "updatedAt": null
}
```

Repitiendo el mismo request se obtiene `409 Conflict` con un `ProblemDetails`:

```json
{
  "status": 409,
  "title": "The requested resource already exists.",
  "detail": "A hero with heroId 123 is already registered in the same place, country and continent.",
  "instance": "/api/v1/heros/123"
}
```

### Endpoints GET adicionales

```bash
curl http://localhost:5104/api/v1/heros/total
# {"total":2}

curl http://localhost:5104/api/v1/heros/1
# { "id": 1, "heroId": 123, ... }

curl -i http://localhost:5104/api/v1/heros/9999
# 404 -> {"status":404,"title":"Hero not found.", ...}
```

> El enunciado menciona “la información del rating” en el response; no existe tal atributo en el modelo, por lo que se interpreta como un error de copiado y se devuelve el resource completo del hero con su `id` generado.

## 6. Empaquetado

```bash
zip -r upc-pre-202402-si730-ws51-pc2-u000000000.zip SuperHeroLocalHeroAPI \
  -x '*bin*' '*obj*'
```

Fuera de alcance según el enunciado: CORS, Security y Testing.
# pc2-web-practica
