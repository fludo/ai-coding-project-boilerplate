---
name: codeigniter-testing
description: Applies PHPUnit testing rules for CodeIgniter 4 APIs - feature/HTTP tests, database tests with migrations and seeds, and mocking. Use when writing or reviewing tests for a CodeIgniter 4 + MariaDB backend.
---

# CodeIgniter 4 Testing Rules

## Prerequisite Detection

Inspect `phpunit.dist.xml` (or `phpunit.xml`), the `tests/` layout, `app/Config/Database.php` `$tests` group, and existing tests before applying conventions. Confirm the runner with `vendor/bin/phpunit --version` and that a `tests` database group exists. Treat a rule as project-specific only when configuration or an established pattern supports it; label inferences.

## What to Test (priority order)

1. **Feature/HTTP tests** for each API endpoint — the real contract clients depend on (status code, body shape, persistence side effects).
2. **Model/database tests** for validation rules, custom finders, and events.
3. **Unit tests** for services and pure helpers with injected dependencies.

Test behavior through public entry points, not private methods. A route's HTTP response is the primary assertion for an API.

## Test Database

- Use the dedicated `tests` connection group; never run tests against development or production data. The suite must set `CI_ENVIRONMENT=testing`, which switches `defaultGroup` to `tests` (see `Database.php::__construct`).
- Point the `tests` group at a disposable MariaDB schema via `.env` (`database.tests.*`). An in-memory SQLite group is acceptable only when the code under test uses no MariaDB-specific SQL — prefer a real MariaDB test schema so behavior matches production.

## Database Tests

Use `DatabaseTestTrait` to run migrations and seeds and to reset state between tests.

```php
<?php

namespace Tests\Feature;

use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\DatabaseTestTrait;
use CodeIgniter\Test\FeatureTestTrait;

final class ArticlesApiTest extends CIUnitTestCase
{
    use DatabaseTestTrait;
    use FeatureTestTrait;

    protected $migrate     = true;     // run migrations before the suite
    protected $migrateOnce = false;    // re-migrate per test for isolation
    protected $refresh     = true;     // refresh the database between tests
    protected $seed        = 'ArticleSeeder'; // optional seeder class

    public function testIndexReturnsArticles(): void
    {
        $result = $this->get('api/articles');

        $result->assertStatus(200);
        $result->assertJSONFragment(['title' => 'Seeded article']);
    }

    public function testCreateRejectsInvalidPayload(): void
    {
        $result = $this->withBodyFormat('json')
            ->post('api/articles', ['body' => 'missing title']);

        $result->assertStatus(400);
    }

    public function testCreatePersistsArticle(): void
    {
        $result = $this->withBodyFormat('json')
            ->post('api/articles', ['title' => 'New', 'body' => 'Content']);

        $result->assertStatus(201);
        $this->seeInDatabase('articles', ['title' => 'New']);
    }
}
```

- `$refresh = true` with `$migrate = true` gives each test a clean schema; rely on it instead of manual teardown.
- Assert persistence with `seeInDatabase` / `dontSeeInDatabase`, not by re-querying through the code under test.
- Keep seeders small and purpose-built; do not depend on production seed data.

## Feature / HTTP Tests

`FeatureTestTrait` calls routes through the full framework stack (routing, filters, controller, response).

- Set the body format explicitly for JSON APIs: `->withBodyFormat('json')` then pass an array as the body.
- Assert on the contract: `assertStatus`, `assertJSONFragment`, `assertJSONExact`, `assertHeader`.
- Cover the failure paths you return deliberately — 400 validation, 404 not found, 401/403 auth — not only the happy path.
- Test filters (auth, throttling) by exercising the route with and without the required precondition.

## Unit Tests & Mocking

- For services, inject collaborators through the constructor and pass test doubles. Prefer hand-written fakes or PHPUnit `createMock()` over reaching into the framework.
- Use framework fakes where they exist: `Services::injectMock()` to replace a service in the container, and the `mock()` helpers for sessions, etc. Reset injected mocks in `tearDown`.
- Do not mock the database when a `DatabaseTestTrait` test against the `tests` schema is cheap and more faithful.

## Determinism & Isolation

- Each test is independent and order-free; never rely on state left by a previous test.
- Control time, randomness, and IDs — assert on structure (`assertArrayHasKey`) rather than on auto-increment values or timestamps you did not set.
- No real network calls; stub external HTTP at the service boundary.
- Name tests for the behavior and expected outcome (`testCreateRejectsInvalidPayload`), not the method name.

## Running

- Full suite: `composer test` or `vendor/bin/phpunit`.
- A failing test must fail for the behavioral reason, not a setup error — read the first failure before changing code. Keep the suite green before handing work off.
