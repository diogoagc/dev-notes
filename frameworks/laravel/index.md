---
title: Laravel Best Practices
category: Frameworks / Laravel
tags:
  - php
  - laravel
  - best-practices
  - code-quality
  - tooling
updated_at: 2025-10-20
---

# Laravel Best Practices

> A curated collection of Laravel conventions, quality tools, useful packages, and configuration ideas. **This is a reference, not a mandatory checklist:** adopt each practice according to the project's complexity, requirements, and Laravel/PHP versions.

**Last updated:** 20 October 2025

## Contents

- [1. Guiding principles](#1-guiding-principles)
- [2. Project setup and authentication](#2-project-setup-and-authentication)
- [3. Code quality and automation](#3-code-quality-and-automation)
- [4. Application configuration](#4-application-configuration)
- [5. Validation, security, and authorization](#5-validation-security-and-authorization)
- [6. Logging, monitoring, and auditing](#6-logging-monitoring-and-auditing)
- [7. Debugging](#7-debugging)
- [8. Performance and optimization](#8-performance-and-optimization)
- [9. Optional packages and starter kits](#9-optional-packages-and-starter-kits)
- [10. Suggested project checklist](#10-suggested-project-checklist)
- [11. References](#11-references)

## 1. Guiding principles

- **Use Laravel's built-in features first.** Add dependencies when they solve a real problem.
- **Prefer clarity over cleverness.** Consistent, readable code is easier to maintain.
- **Validate, authorize, and test important operations.** They address different concerns.
- **Fail early during development.** Strict models, static analysis, and tests help expose mistakes.
- **Automate repeatable checks.** Formatting, analysis, and tests should be simple to run locally and in CI.
- **Measure before optimizing.** Additional infrastructure or execution models introduce operational costs.
- **Check compatibility.** Tooling, package APIs, and starter kits vary by Laravel and PHP version.

**Classification used below:**

| Label | Meaning |
| --- | --- |
| **Recommended** | A sensible default for most projects. |
| **Consider** | Useful when a project's needs justify it. |
| **Situational** | Adopt only for a specific requirement or with suitable safeguards. |
| **Explore** | Worth evaluating; not an automatic recommendation. |

## 2. Project setup and authentication

### Authentication starter kits — Recommended when authentication is needed

Prefer the **official Laravel starter kits** for new applications. Choose a stack that matches the team and product (for example, Livewire or Inertia with React/Vue).

Earlier options worth recognizing in existing projects:

- **Laravel Breeze:** lightweight authentication scaffolding.
- **Laravel Jetstream:** more feature-rich scaffolding, including options such as two-factor authentication and team management.

> **Compatibility note:** Laravel introduced new first-party starter kits in Laravel 12 and stopped adding updates to Breeze and Jetstream. Treat Breeze/Jetstream primarily as legacy references rather than the default starting point for new Laravel projects.

References: [Laravel Starter Kits](https://laravel.com/docs/starter-kits) · [Breeze](https://github.com/laravel/breeze) · [Jetstream](https://jetstream.laravel.com/)

### General project setup — Recommended

- Use supported PHP and Laravel versions and commit `composer.lock` for applications.
- Keep secrets in environment configuration; never commit actual `.env` credentials.
- Establish consistent local setup instructions and a small, reproducible test suite.
- Separate application-level code from HTTP concerns when complexity justifies it (for example, actions, services, jobs, or DTOs); avoid unnecessary abstractions.

## 3. Code quality and automation

### Laravel Pint — Recommended

Official opinionated PHP formatter based on **PHP-CS-Fixer**. Prefer it to maintaining two overlapping formatters unless there is a specific need.

```bash
composer require laravel/pint --dev
./vendor/bin/pint
./vendor/bin/pint --test
```

Reference: [Laravel Pint](https://laravel.com/docs/pint)

### Larastan — Recommended

Laravel-aware static analysis built on PHPStan. Helps detect type problems, incorrect calls, and other potential bugs without executing the application.

```bash
composer require larastan/larastan --dev
./vendor/bin/phpstan analyse
```

Configure `phpstan.neon` and increase strictness gradually, keeping existing codebases practical to maintain.

Reference: [Larastan](https://github.com/larastan/larastan)

### Rector — Consider

Automated refactoring and upgrade assistance. Complements Pint (formatting) and Larastan (analysis); it does not replace either.

```bash
composer require rector/rector --dev
./vendor/bin/rector process --dry-run
```

Review proposed changes and run tests after applying transformations.

Reference: [Rector](https://getrector.com/)

### Peck — Consider

Finds spelling and wording mistakes in code and documentation. May require **GNU Aspell**, depending on the environment.

```bash
composer require peckphp/peck --dev
./vendor/bin/peck --init
./vendor/bin/peck
```

Reference: [Peck](https://github.com/peckphp/peck)

### Tests — Recommended

- Use **Pest** or PHPUnit, choosing one consistent approach across the project.
- Prioritize tests for business rules, validation, permissions, integrations, and critical workflows.
- Prefer meaningful coverage of behavior to an arbitrary coverage percentage.
- Run the same checks in development and CI.

Reference: [Laravel Testing](https://laravel.com/docs/testing) · [Pest](https://pestphp.com/)

### Composer scripts — Recommended

Provide memorable entry points so developers and CI can run the same checks. Example fragment for `composer.json` (adapt paths and dependencies to the project):

```json
{
  "scripts": {
    "test": "@php artisan test",
    "lint": "@php vendor/bin/pint",
    "lint:check": "@php vendor/bin/pint --test",
    "analyse": "@php vendor/bin/phpstan analyse",
    "refactor:check": "@php vendor/bin/rector process --dry-run"
  }
}
```

```bash
composer test
composer lint:check
composer analyse
composer refactor:check
```

**CI baseline:** dependency installation, formatter check, static analysis, and tests. Add spelling checks or refactoring checks if they provide value.

## 4. Application configuration

### AppServiceProvider defaults — Consider individually

The following illustrates **independent options**, not a block that must always be copied into every project:

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Carbon\CarbonImmutable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Date;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\URL;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Detect common Eloquent mistakes in non-production environments.
        Model::shouldBeStrict(! app()->isProduction());

        // Prevent dangerous database reset/wipe commands in production.
        DB::prohibitDestructiveCommands(app()->isProduction());

        // Favor immutable date objects.
        Date::use(CarbonImmutable::class);

        // Use only when the application is intended to run over HTTPS
        // and reverse proxies / trusted headers are configured correctly.
        // URL::forceHttps();

        // OPTIONAL, SECURITY-SENSITIVE: see the explanation below.
        // Model::unguard();
    }
}
```

#### `Model::shouldBeStrict()` — Recommended during development/testing

Combines safeguards against:

- Accidental relationship lazy loading (including possible N+1 queries).
- Silently discarding attributes during mass assignment.
- Reading model attributes that were not loaded or do not exist as expected.

Enabling strict mode in production is also possible, but assess exception handling and existing behavior first.

#### `Model::unguard()` — Situational; not a general default

Disables mass-assignment restrictions globally, so `$fillable` is no longer required. **This also removes an important defense against unintended attribute updates.**

Never write `Model::create($request->all())` or pass untrusted data directly to `fill()`, `create()`, or `update()` when models are unguarded. Explicitly construct the attributes to persist, even after validation. For most projects, maintaining `$fillable`/`$guarded` or explicit assignment is safer.

#### `DB::prohibitDestructiveCommands()` — Recommended as a production safeguard

Prevents certain destructive Artisan database commands, such as `db:wipe` and migration reset/refresh commands. This **does not replace backups, deployment controls, or database permissions**.

#### `URL::forceHttps()` — Situational

Forces Laravel-generated URLs to use HTTPS. Apply when needed, after ensuring the deployment, proxy trust settings, and HTTPS termination are correct. Prefer correct infrastructure configuration to masking a proxy problem.

#### `Date::use(CarbonImmutable::class)` — Consider

Makes Laravel's date factory use immutable Carbon dates, reducing accidental mutation. Check expectations in existing application code and custom casts before changing a mature project.

Reference: [Laravel Eloquent](https://laravel.com/docs/eloquent) · [Carbon](https://carbon.nesbot.com/)

## 5. Validation, security, and authorization

### Validate every external input — Recommended

Validate form requests, API payloads, and other untrusted inputs at their boundaries. Use **Form Request** classes when validation rules or authorization become substantial.

```php
public function store(StoreProjectRequest $request): RedirectResponse
{
    $data = $request->validated();

    Project::create([
        'name' => $data['name'],
        'description' => $data['description'] ?? null,
    ]);

    return redirect()->route('projects.index');
}
```

Keep in mind: **validation controls data shape; authorization controls who may act; mass-assignment protection controls which attributes may be written.** None replaces the others.

### Use Laravel authorization — Recommended

- Use **Policies** for authorization related to a model or resource.
- Use **Gates** for abilities that do not naturally belong to a model.
- Check authorization on the server for every protected action; hiding buttons is not sufficient.

### Roles and permissions — Consider

**Spatie Laravel Permission** is useful when roles and permissions must be stored and managed dynamically (for example, admin-configurable RBAC).

For a very small, fixed role model, an enum plus native Policies/Gates may be simpler. If using Spatie, prefer checking abilities (`can`) rather than spreading role-name comparisons through application code.

Reference: [Spatie Laravel Permission](https://spatie.be/docs/laravel-permission) · [Laravel Authorization](https://laravel.com/docs/authorization)

## 6. Logging, monitoring, and auditing

These solve related but **different** problems:

| Tool | Purpose | Adoption |
| --- | --- | --- |
| **Laravel logs** | Application errors, warnings, and diagnostic messages. | Recommended |
| **Laravel Nightwatch** | Hosted performance and application monitoring (requests, exceptions, jobs, etc.). | Consider |
| **OPcodes Log Viewer** | Search and inspect application/server logs through a UI. | Consider |
| **Laravel Auditing** | Track changes to selected Eloquent model records. | Situational |

### Implementation notes

- Use structured, actionable log messages. Avoid logging passwords, tokens, and unnecessary personal information.
- Protect any log-viewer interface with appropriate authentication/authorization; production logs may contain sensitive details.
- **Laravel Auditing** (`owen-it/laravel-auditing`) generally requires opting models into auditing (for example, with its `Auditable` trait). Do not assume it automatically records every CRUD operation for every model.
- Configure audit event selection, retention, data sensitivity, and storage before enabling extensive model-change history.
- Nightwatch is a separate monitoring service; evaluate its cost and operational fit.

References: [Nightwatch](https://laravel.com/nightwatch) · [OPcodes Log Viewer](https://github.com/opcodesio/log-viewer) · [Laravel Auditing](https://github.com/owen-it/laravel-auditing)

## 7. Debugging

### Xdebug — Recommended for deeper investigations

Use breakpoints, step-through debugging, stack traces, and variable inspection through an IDE.

### LaraDumps — Consider

Helpful desktop-oriented dump/debugging experience for Laravel and other PHP projects. Use it as an aid rather than replacing tests or proper logging.

References: [Xdebug](https://xdebug.org/) · [LaraDumps](https://laradumps.dev/)

## 8. Performance and optimization

### Laravel Octane — Situational

Runs Laravel with persistent application workers (for example, with FrankenPHP, RoadRunner, or Swoole) and can improve throughput/latency for suitable workloads.

**Do not introduce Octane by default.** Start with profiling and straightforward improvements such as indexes, efficient queries, eager loading, caching, queueing slow work, and correct infrastructure sizing.

When adopting Octane:

- Benchmark against the current deployment to confirm a meaningful improvement.
- Understand the long-lived worker lifecycle: singleton services and static state can accidentally retain request-specific data.
- Watch for stale request/config references, memory growth, and necessary worker reloads on deployment.

Reference: [Laravel Octane](https://laravel.com/docs/octane)

## 9. Optional packages and starter kits

### Nuno Maduro Essentials — Explore

[Essentials](https://github.com/nunomaduro/essentials) provides configurable, opinionated Laravel defaults such as strict models, immutable dates, safe console behavior, and other quality-of-life options.

Review its configuration and version requirements carefully. Avoid unknowingly applying the same behavior twice through both the package and manual `AppServiceProvider` configuration.

### Nuno Maduro Laravel Starter Kit — Explore

[Nuno Maduro's Laravel Starter Kit](https://github.com/nunomaduro/laravel-starter-kit) is an opinionated skeleton centered on strict typing, automated checks, and consistent engineering standards. Useful as a reference even if you do not choose it as a project base.

Check its supported PHP/Laravel versions before use.

### Laravel Wayfinder — Consider for Inertia + TypeScript

[Laravel Wayfinder](https://github.com/laravel/wayfinder) generates type-safe TypeScript route helpers from Laravel routes/controllers. Particularly useful when working with Inertia and a React/Vue TypeScript frontend.

Use it where shared route types simplify frontend/backend integration; it is not necessary for a Blade-only project.

## 10. Suggested project checklist

Choose only the items relevant to the application:

- [ ] Select supported PHP/Laravel versions and an appropriate starter kit.
- [ ] Configure authentication if required.
- [ ] Establish request validation and server-side authorization.
- [ ] Configure Laravel Pint and formatter checks in CI.
- [ ] Add meaningful tests (Pest or PHPUnit).
- [ ] Enable Larastan with a maintainable strictness level.
- [ ] Consider Rector for refactoring or upgrades.
- [ ] Configure model strictness in development/testing.
- [ ] Add production safeguards against destructive database commands.
- [ ] Decide explicitly whether immutable dates and forced HTTPS fit the project.
- [ ] Preserve safe mass-assignment behavior; use global unguarding only deliberately.
- [ ] Set up actionable logs and protect production diagnostics.
- [ ] Add monitoring or auditing only where their value justifies the overhead.
- [ ] Profile performance before considering Octane.
- [ ] Document the chosen decisions and provide simple Composer scripts.

## 11. References

**Learning material**

- [Laravel practices — example video](https://www.youtube.com/live/8RNPkDd4-TA)
- [Roles and permissions — example video](https://www.youtube.com/live/IYyQAQrcySo)
- [Example project: aamimi/offerly-backend](https://github.com/aamimi/offerly-backend)

**Core documentation**

- [Laravel Documentation](https://laravel.com/docs)
- [Laravel Starter Kits](https://laravel.com/docs/starter-kits)
- [Laravel Pint](https://laravel.com/docs/pint)
- [Laravel Testing](https://laravel.com/docs/testing)
- [Laravel Authorization](https://laravel.com/docs/authorization)

**Tooling and additional projects**

- [Larastan](https://github.com/larastan/larastan)
- [Rector](https://getrector.com/)
- [Peck](https://github.com/peckphp/peck)
- [Nightwatch](https://laravel.com/nightwatch)
- [Log Viewer](https://github.com/opcodesio/log-viewer)
- [Laravel Auditing](https://github.com/owen-it/laravel-auditing)
- [LaraDumps](https://laradumps.dev/)
- [Nuno Maduro Essentials](https://github.com/nunomaduro/essentials)
- [Nuno Maduro Laravel Starter Kit](https://github.com/nunomaduro/laravel-starter-kit)
- [Laravel Wayfinder](https://github.com/laravel/wayfinder)
- [Spatie Laravel Permission](https://spatie.be/docs/laravel-permission)

---

*Living notes: review each recommendation when upgrading Laravel, PHP, or the relevant package. The `updated_at` field is maintained manually.*
