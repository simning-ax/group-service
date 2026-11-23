# Group Service

Group Service ansvarar för all hantering av:

- Gruppnivåer (`GroupLevel`)
- Grupper (`Group`)
- Gruppledare (`GroupLeader`)

Tjänsten exponeras som ett REST API och används av både frontend och andra mikrotjänster.

---

# Arkitektur

Tjänsten följer en action-baserad arkitektur byggd på Slim 4.

Varje request passerar följande lager:

```text
HTTP Request
    ↓
Action
    ↓
Validator
    ↓
Command
    ↓
Handler
    ↓
Repository
    ↓
Database
```

Responsen returneras därefter genom samma kedja tillbaka till klienten.

---

# Projektstruktur

```text
src
├── Application
│   ├── Commands
│   ├── Handlers
│   └── Validators
│
├── Domain
│   ├── DataTransportObjects
│   ├── Entities
│   ├── Enums
│   ├── Repositories
│   └── ValueObjects
│
├── Http
│   └── Actions
│
└── Infrastructure
    └── Database
```

---

# Actions

Varje endpoint har exakt en Action-klass.

Actionens ansvar är att:

- läsa route-parametrar
- läsa request-body
- anropa validator
- skapa command-objekt
- anropa handler
- returnera HTTP-svar

Exempel:

```php
UpdateGroupLevelAction
CreateGroupAction
UpsertGroupLeaderAction
```

Actions innehåller ingen affärslogik.

---

# Validators

Validators ansvarar för att validera inkommande request-data.

Exempel på regler:

- obligatoriska fält
- numeriska värden
- enum-värden
- ID-matchning mellan URL och body

Valideringsfel kastar:

```php
ValidationException
```

vilket returnerar:

```http
422 Unprocessable Entity
```

---

# Commands

Commands representerar en användares avsikt.

Exempel:

```php
CreateGroupCommand
UpdateGroupCommand
UpdateGroupLevelSortOrderCommand
```

Commands skapas normalt genom:

```php
Command::fromRequest(...)
```

Command-objekt innehåller normaliserad data och används som indata till handlers.

---

# Handlers

Handlers innehåller applikationslogik.

Exempel:

```php
CreateGroupHandler
UpdateGroupHandler
GetLeaderGroupsHandler
```

En handler kan:

- verifiera att relaterade resurser finns
- anropa flera repositories
- bygga DTO:er
- kasta domänspecifika exceptions

Handlers innehåller ingen SQL.

---

# Entiteter

Domänmodellen består av entiteter som representerar verksamhetsobjekt.

Exempel:

```php
Group
GroupLevel
GroupLeader
User
```

Entiteter ansvarar för:

- domändata
- domänbeteende
- serialisering mot databasen

Exempel:

```php
Group::fromDBRow()
Group::asDBRow()
```

---

# Value Objects

Value Objects används för stark typning och validering.

Exempel:

```php
GroupId
GroupLevelId
UserId
DateTimeValue
```

UUID-validering sker i respektive ID-objekt.

Exempel:

```php
new GroupId($id);
```

Ogiltiga UUID:er genererar:

```php
InvalidArgumentException
```

vilket returnerar:

```http
400 Bad Request
```

---

# Enums

Enums används för domänspecifika värden.

Exempel:

```php
Role
Venue
```

```php
enum Role : string {
    case LEADER = 'Ledare';
    case ASSISTANT = 'Assistent';
    case EDUCATOR = 'Utbildare';
}
```

Enums används i både:

- entiteter
- DTO:er
- validatorer
- OpenAPI-kontrakt

---

# DTO-er

DTO:er används för kommunikation mellan applikationslager och HTTP-lager.

Exempel:

```php
GroupDTO
GroupLevelDTO
UserDTO
GroupLevelSortOrderDTO
```

DTO:er skapas normalt från entiteter:

```php
UserDTO::fromEntity($user);
```

---

# Repositories

Repositories ansvarar för all databasåtkomst.

Exempel:

```php
DbalGroupRepository
DbalGroupLevelRepository
DbalGroupLeaderRepository
```

Repositories:

- innehåller SQL
- använder Doctrine DBAL
- returnerar entiteter
- känner inte till HTTP

---

# Databas

Databasåtkomst sker via Doctrine DBAL.

Scheman hanteras med:

```text
Phinx
```

Migrationer finns i:

```text
db/migrations
```

Ny migration skapas med:

```bash
vendor/bin/phinx create MigrationName
```

Migrationer körs med:

```bash
composer migrate
```

Migrationer körs också när containern startar så att databasen alltid är up-to-date

---

# OpenAPI

API:t dokumenteras i:

```text
openapi.yaml
```

OpenAPI används för:

- Swagger-dokumentation
- request-validering
- response-validering
- integrationstester

Vid endpointändringar ska alltid:

1. OpenAPI uppdateras
2. Tester uppdateras
3. VERSION uppdateras vid breaking changes

---

# Felhantering

Alla exceptions hanteras centralt.

Vanliga fel:

| Exception | HTTP |
|------------|------|
| InvalidArgumentException | 400 |
| ValidationException | 422 |
| NotFoundException | 404 |
| UnauthorizedException | 401 |
| ForbiddenException | 403 |

Fel returneras som:

```json
{
  "statusCode": 404,
  "error": {
    "type": "NotFoundException",
    "message": "Resursen finns inte"
  }
}
```

---

# Teststrategi

Projektet använder flera testnivåer.

## Unit-tester

Testar enskilda klasser:

```text
Actions
Validators
Commands
Handlers
Entities
DTO-er
Enums
Repositories
Value Objects
```

Körs med:

```bash
composer test
```

---

## Integrationstester

Verifierar hela requestkedjan:

```text
HTTP
 ↓
Action
 ↓
Validator
 ↓
Command
 ↓
Handler
 ↓
Repository
 ↓
Database
```

Integrationstester validerar även:

- OpenAPI request
- OpenAPI response

---

# Kodkvalitet

Projektet använder:

```text
PHPUnit
PHPStan
PHPCS
Infection
Rector
```

Körs via:

```bash
composer test
composer stan
composer fix
```

---

# Lokal utveckling

Starta tjänsten:

```bash
docker compose up --build -d
```

Installera beroenden:

```bash
composer install
```

Kör migrationer:

```bash
composer migrate
```

Starta tester:

```bash
composer test
```

Swagger finns på:

```text
http://localhost:8080/docs
```

Health-check:

```text
http://localhost:8080/health
```

---

# Designprinciper

Projektet följer följande principer:

- En Action per endpoint.
- En Handler per use-case.
- All SQL i repositories.
- All identifiering sker med Value Objects.
- Enums används i stället för magiska strängar.
- OpenAPI är kontraktet för API:t.
- Integrationstester verifierar kontraktet.
- Domänobjekt ska alltid vara giltiga när de används.
- Referenser mellan mikrotjänster sker endast via ID:n, aldrig objekt.

Det här dokumentet beskriver den faktiska arkitekturen i Group Service och ska uppdateras när arkitekturen förändras.