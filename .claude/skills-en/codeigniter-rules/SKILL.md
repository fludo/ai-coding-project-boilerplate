---
name: codeigniter-rules
description: Applies CodeIgniter 4 backend API conventions for controllers, models, services, validation, and error handling. Use when implementing or reviewing CodeIgniter 4 + MariaDB API code.
---

# CodeIgniter 4 Backend API Rules

## Prerequisite Detection

Inspect `composer.json`, `app/Config/*` (especially `App.php`, `Routes.php`, `Routing.php`, `Database.php`, `Filters.php`, `Validation.php`), the `.env`, and representative controllers/models before applying conventions. Treat a rule as project-specific only when configuration or an established pattern supports it. Label limited-pattern conclusions as inferred. When competing conventions change a public contract (a route, a response shape, an HTTP status), stop and name the source or user decision required.

Confirm the framework version with `php spark version`; APIs below assume CodeIgniter `^4.7` on PHP `^8.2`.

## Layering

Keep a clear request flow and one owner per concern:

Route -> Controller (HTTP boundary) -> Service (business rules, optional) -> Model (persistence) -> MariaDB

- **Controller**: translate HTTP to/from the application. No business rules, no raw SQL.
- **Service**: orchestrate business rules when logic spans multiple models or external calls. Plain PHP classes under `app/Services`, constructed with their dependencies. Skip this layer for straightforward CRUD.
- **Model**: persistence and data-shape validation. One model per table; no HTTP concerns.

Do not call `model()` or the query builder from a controller when a service owns the rule, and never emit HTTP responses from a model or service — return data or throw.

## Controllers (RESTful API)

Extend `CodeIgniter\RESTful\ResourceController` for resource endpoints and set the format explicitly.

```php
<?php

namespace App\Controllers\Api;

use App\Models\ArticleModel;
use CodeIgniter\RESTful\ResourceController;

class Articles extends ResourceController
{
    protected $modelName = ArticleModel::class;
    protected $format    = 'json';

    public function index(): ResponseInterface
    {
        return $this->respond($this->model->findAll());
    }

    public function show($id = null): ResponseInterface
    {
        $article = $this->model->find($id);
        if ($article === null) {
            return $this->failNotFound("Article {$id} not found");
        }

        return $this->respond($article);
    }

    public function create(): ResponseInterface
    {
        $data = $this->request->getJSON(true) ?? [];

        if (! $this->model->insert($data)) {
            return $this->failValidationErrors($this->model->errors());
        }

        return $this->respondCreated($this->model->find($this->model->getInsertID()));
    }

    public function update($id = null): ResponseInterface
    {
        if ($this->model->find($id) === null) {
            return $this->failNotFound("Article {$id} not found");
        }

        $data = $this->request->getJSON(true) ?? [];

        if (! $this->model->update($id, $data)) {
            return $this->failValidationErrors($this->model->errors());
        }

        return $this->respond($this->model->find($id));
    }

    public function delete($id = null): ResponseInterface
    {
        if ($this->model->find($id) === null) {
            return $this->failNotFound("Article {$id} not found");
        }

        $this->model->delete($id);

        return $this->respondDeleted(['id' => $id]);
    }
}
```

**Response rules**
- Use the `ResponseTrait` helpers so status codes stay consistent: `respond` (200), `respondCreated` (201), `respondDeleted`, `respondNoContent` (204), `failValidationErrors` (400), `failNotFound` (404), `failForbidden` (403), `failUnauthorized` (401), `failServerError` (500).
- Never echo or `print`; always return a `ResponseInterface`.
- Read JSON request bodies with `$this->request->getJSON(true)` (associative array); do not trust `$_POST` for API clients.
- Keep the response envelope consistent across endpoints — pick a shape (bare resource vs. `{ "data": ... }`) and apply it everywhere; changing it is a contract change.

## Routing

Define API routes explicitly in `app/Config/Routes.php`. Prefer a versioned group and point the resource at the namespaced controller.

```php
$routes->group('api', ['namespace' => 'App\Controllers\Api'], static function ($routes) {
    $routes->resource('articles', ['controller' => 'Articles']);
});
```

- Keep auto-routing disabled (CI 4 defaults to off). Do not re-enable `AutoRouting` for an API.
- Apply cross-cutting concerns (auth, CORS, throttling) via filters in `app/Config/Filters.php`, not inside controllers.

## Models

Extend `CodeIgniter\Model`. Declare the table contract and validation on the model so every write path is validated.

```php
<?php

namespace App\Models;

use CodeIgniter\Model;

class ArticleModel extends Model
{
    protected $table            = 'articles';
    protected $primaryKey       = 'id';
    protected $returnType       = 'array';
    protected $useTimestamps    = true;
    protected $allowedFields    = ['title', 'body', 'published'];
    protected $useSoftDeletes   = false;

    protected $validationRules = [
        'title'     => 'required|string|max_length[255]',
        'body'      => 'required|string',
        'published' => 'permit_empty|in_list[0,1]',
    ];

    protected $validationMessages = [
        'title' => ['required' => 'A title is required.'],
    ];
}
```

- **`$allowedFields` is mandatory** for mass-assignment safety; never add the primary key or `created_at`/`updated_at` to it.
- Keep `$returnType` consistent with what controllers expect (`'array'` for JSON APIs is simplest; use an Entity class only when rows carry behavior).
- Validate on the model (`$validationRules`) rather than per-controller, so inserts and updates share one rule set. Add request-specific checks via the `validation` service only when they do not belong to the data itself.
- Use model events (`beforeInsert`, `afterFind`, …) for cross-cutting data transforms (hashing, normalizing); keep them pure and documented.

## Entities

Use `CodeIgniter\Entity\Entity` only when a row needs behavior or computed/casting logic. Define `$casts` for typed access and keep business methods on the entity, persistence on the model.

```php
protected $casts = ['published' => 'boolean', 'id' => 'integer'];
```

## Validation & Error Handling

**Every failure has one owning outcome**: return a typed failure response, recover per a named requirement, or let it propagate to the framework exception handler with context.

- **Expected, client-caused failures** (bad input, missing resource, forbidden): return via the `ResponseTrait` `fail*` helpers with an actionable message. These are not exceptions.
- **Programming/infrastructure failures**: throw. Use the framework's HTTP exceptions where they fit:
  - `PageNotFoundException::forPageNotFound()`
  - `\CodeIgniter\Exceptions\PageNotFoundException`, `\CodeIgniter\Database\Exceptions\DatabaseException`
- Define domain exceptions under `app/Exceptions` extending a base you control; convert them to responses at the controller boundary, not deeper.
- Configure production behavior in `app/Config/Exceptions.php` and set `CI_ENVIRONMENT=production` in deployed `.env` so stack traces are never returned to clients. `DBDebug` must be `false` in production.
- Do not swallow errors with an empty `catch` or a silent default that hides a failure the caller requires.

```php
try {
    $this->orderService->place($payload);
} catch (InsufficientStockException $e) {
    return $this->fail($e->getMessage(), 409);
}
// Let unexpected exceptions propagate to the framework handler.
```

## Database Access

Routine persistence goes through models. When a query is too complex for the model API, use the Query Builder — never string-concatenated SQL. See [[mariadb-data-access]] for schema, migrations, query-builder, transaction, and MariaDB-specific rules.

## Configuration & Secrets

- Read environment via `.env` / `getenv()` / `env()`; never hardcode credentials. `.env` is git-ignored.
- Keep config classes (`app/Config/*`) free of secrets; override per-environment values from `.env` (e.g. `database.default.*`, `app.baseURL`).
- `app.baseURL` must be set correctly per environment; an empty value breaks generated URLs.

## Coding Conventions

- **PSR-12** formatting and **PSR-4** autoloading (`App\` -> `app/`). Classes `PascalCase`, methods/variables `camelCase`, constants `UPPER_SNAKE_CASE`.
- Declare parameter and return types on every method; use nullable (`?Type`) and union types deliberately. Prefer `declare(strict_types=1)` in new PHP files when the project already uses it.
- 0–2 parameters per method; group 3+ related inputs into a value object or associative array with a documented shape.
- Inject dependencies through constructors so they can be substituted in tests; avoid reaching for global singletons inside business logic.
- Remove dead code, debug `var_dump`/`log_message('debug', …)` scaffolding, and commented-out blocks within the current change. Comments explain "why", not "what".

## Logging

- Log at the observability-owning boundary (typically the controller or a dedicated handler) so one failure is not logged repeatedly. Use `log_message($level, $message)` with levels from `app/Config/Logger.php`.
- Never log credentials, tokens, full request bodies with secrets, or personal data. Redact before logging.
