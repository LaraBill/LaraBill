**LARABILL PROJECT RULES**
==========================

1. Purpose
----------

Establish a single, modular Laravel 11 monorepo standard for **LaraBill**—a configurable billing and provisioning platform built with **Eloquent**, **MySQL**, and **Docker**.  

**LaraBill is modular by design:**
* Use **Billing + Invoicing** modules alone for standalone billing management
* Enable the **Provisioning** module to connect "money to machines"—automating resource provisioning when payments are captured
* Modules and drivers are self-contained packages inside `app/Modules` and `app/Drivers`, each owning its own routes, migrations, views, assets, and tests
* Each module can be enabled or disabled via its `module.json` manifest—choose the features you need

* * *

2. Repository Layout
--------------------

```
app/
  Core/                               # cross-cutting code
    Contracts/
      Drivers/                        # driver capability contracts
        Provider.php
        Provisioner.php
        Metrics.php
        Inventory.php
        Webhooks.php
    Support/
    Events/
    Enums/
  Modules/
    Billing/                          # CORE - always available
      module.json
      src/
        Config/
        Database/{Migrations,Seeders,Factories}/
        Http/{Controllers,Middleware,Requests}/
        Jobs/
        Listeners/
        Mail/
        Policies/
        Providers/ModuleServiceProvider.php
        Resources/{views,lang,assets}
        Routes/{api.php,web.php}
        Services/
        Support/
        Tests/{Feature,Unit}/
    Invoicing/                        # CORE - always available
      ... same pattern ...
    Provisioning/                     # OPTIONAL - enable via module.json
      module.json
      src/
        Config/
        Database/{Migrations,Seeders}/
        Domain/
          Enums/                      # ResourceStatus, TaskStatus, ProviderType
          Events/                     # OrderPlaced, ResourceProvisioned, ...
          Exceptions/
          Models/                     # Resource, ProvisionTask, Credential, Region, PlanMap
          Policies/
          Support/ValueObjects/       # ResourceSpec, CredentialsRef, RegionCode
        Http/{Controllers,Requests}/
        Jobs/                         # KickProvision, PollTask, Deprovision, SyncState
        Listeners/                    # OnPaymentCaptured -> KickProvision
        Providers/ModuleServiceProvider.php
        Resources/{views,lang,assets}
        Routes/{api.php,web.php}
        Services/
          Orchestrator/               # state machine
          Mappers/                    # normalize provider payloads
          Audit/                      # append-only audit trail
        Tests/{Feature,Unit,Integration}/
  Drivers/                            # OPTIONAL - only needed if Provisioning enabled
    Panels/                           # hosting control panels
      Whm/
        module.json
        src/...
    Compute/                          # VPS/cloud providers
      Proxmox/
        module.json
        src/...
    Game/                             # game server panels
      Tcadmin/
        module.json
        src/...
    Cloud/                            # generic cloud/SSH
      GenericSsh/
        module.json
        src/...
bootstrap/
  modules.php                         # auto-registers module providers
config/
  modules.php                         # discovery config
public/build/                         # vite output
resources/
  js, css, layouts
tests/
  TestCase.php
docker/
  compose.yml
Makefile
composer.json
package.json
vite.config.ts
```

* * *

3. Module & Driver Rules
------------------------

* Each **module** or **driver** contains everything it needs: routes, migrations, factories, views, translations, tests.
    
* Each has a `module.json` manifest with an **`enabled`** field to control activation:
    
    **Module example (Billing):**
    ```json
    {
      "name": "Billing",
      "enabled": true,
      "providers": [
        "LaraBill\\Modules\\Billing\\src\\Providers\\ModuleServiceProvider"
      ],
      "requires": {
        "composer": { "brick/money": "^0.10" },
        "npm": { "chart.js": "^4.4.0" }
      }
    }
    ```
    
    **Driver example (Provisioning - Proxmox):**
    ```json
    {
      "name": "Proxmox",
      "enabled": true,
      "type": "Compute",
      "capabilities": ["provision","deprovision","suspend","resume","metrics","inventory"],
      "providers": [
        "LaraBill\\Drivers\\Compute\\Proxmox\\src\\ProxmoxServiceProvider"
      ],
      "requires": {
        "composer": { "guzzlehttp/guzzle": "^7.9" },
        "npm": {}
      }
    }
    ```
    
* Providers register their own routes, migrations, views, policies, and config.
    
* **Drivers** (optional—only needed with Provisioning module):
    * Must declare `type` (Panels, Compute, Game, Cloud) and `capabilities`
    * Implement capability-based `Core\Contracts\Drivers\*` interfaces
    * Must implement `Webhooks` contract if they expose webhook endpoints; otherwise polling is mandatory
    * All provider operations must be **async via Jobs**—no long-running HTTP requests
    * Drivers are optional unless a billing plan requires them via `plan_maps`
    

* * *

4. Driver Contracts (Optional—Provisioning Only)
-------------------------------------------------

**Note:** This section only applies if you enable the Provisioning module. Drivers allow LaraBill to communicate with infrastructure providers.

### Core Contracts

All drivers implement capability-based contracts in `app/Core/Contracts/Drivers/`:

* **Provider.php** — base contract: `name()`, `type()`, `capabilities()`
* **Provisioner.php** — resource lifecycle: `provision()`, `deprovision()`, `resize()`, `suspend()`, `resume()`
* **Metrics.php** — (optional) usage data: `usage()`, `health()`, `costs()`
* **Inventory.php** — (optional) available resources: `regions()`, `images()`, `plans()`, `quotas()`
* **Webhooks.php** — (optional but recommended) webhook handling: `verifySignature()`, `handleWebhook()`

### Example: Provisioner Interface

```php
namespace LaraBill\Core\Contracts\Drivers;

use LaraBill\Modules\Provisioning\src\Domain\Models\Resource;
use LaraBill\Modules\Provisioning\src\Domain\Support\ValueObjects\ResourceSpec;

interface Provisioner extends Provider
{
    /**
     * Start provisioning a new resource
     * @return string Provider task/reference ID
     */
    public function provision(ResourceSpec $spec): string;
    
    /**
     * Poll task status
     * @return array ['status' => 'pending|completed|failed', 'details' => [...]]
     */
    public function poll(string $taskId): array;
    
    /**
     * Deprovision (destroy) a resource
     */
    public function deprovision(Resource $resource): void;
    
    /**
     * Suspend a resource (keep data, stop billing)
     */
    public function suspend(Resource $resource): void;
    
    /**
     * Resume a suspended resource
     */
    public function resume(Resource $resource): void;
}
```

### Capability-Based Querying

Before calling a driver method, check if it supports that capability:

```php
$driver = app(DriverManager::class)->get('proxmox');

if (in_array('metrics', $driver->capabilities())) {
    $usage = $driver->usage($resource);
}
```

* * *

5. Provisioning Architecture (Optional Module)
-----------------------------------------------

**Note:** This section describes the optional Provisioning module. Enable it in `module.json` to automate resource provisioning when payments are captured.

### Resource State Machine

Resources transition through a state machine defined by the **ResourceStatus** enum:

```
PENDING → QUEUED → PROVISIONING → ACTIVE
                                    ↓
                               SUSPENDED → RESUMING → ACTIVE
                                    ↓
                               UPDATING → ACTIVE
                                    ↓
                               FAILED (manual intervention)
                                    ↓
                            DEPROVISIONING → DEPROVISIONED
```

All state transitions are recorded in an **append-only** `provision_audits` table for compliance and debugging.

### Orchestrator Rules

The Provisioning Orchestrator (state machine) follows these rules:

* **All operations are async via Jobs**—never block HTTP requests
* Every provider API call carries an **idempotency key**: `{order_uuid}:attempt_{n}`
* Poll driver tasks until terminal state (completed/failed)
* Use exponential backoff with jitter between polls; cap total attempts
* Webhooks can short-circuit polling if driver supports them

### Database Tables

The Provisioning module creates these tables (via migrations in `Provisioning/Database/Migrations`):

* **`resources`**  
  `id, user_id, driver, driver_ref, plan_code, region, status, spec_json, billing_item_id, last_synced_at, created_at, updated_at`
    
* **`provision_tasks`**  
  `id, resource_id, driver_task_id, action, status, attempts, last_error, created_at, updated_at`
    
* **`credentials`**  
  `id, name, driver, secrets_encrypted, scope (user|system), created_by, created_at, updated_at`
    
* **`plan_maps`**  
  Bridge between Billing plans and provider plans/regions  
  `id, billing_plan_id, driver, provider_plan_code, region, config_json, created_at, updated_at`
    
* **`provision_audits`**  
  Append-only audit trail  
  `id, resource_id, user_id, action, status_before, status_after, metadata_json, created_at`
    

### Link to Billing Module

When Provisioning is enabled, `billing_items` table gains an optional foreign key:

* `billing_items.resource_id` → links invoiced items to provisioned resources

### Event Flow

When a payment is captured:

1. **Billing module** fires `PaymentCaptured` event
2. **Provisioning listener** `OnPaymentCaptured` checks if order requires provisioning
3. If yes, dispatch `KickProvision` job to `provisioning` queue
4. Job creates `Resource` record (status: PENDING), creates `ProvisionTask`, calls driver
5. Driver returns task ID; job dispatches `PollTask` with delay
6. `PollTask` polls driver until terminal state, updates resource status
7. Audit trail records every transition

### API Routes

Minimal routes in `Provisioning/Routes/api.php`:

```
POST   /api/provision/orders/{order}/kick
GET    /api/provision/resources/{id}
POST   /api/provision/resources/{id}/suspend
POST   /api/provision/resources/{id}/resume
DELETE /api/provision/resources/{id}
POST   /api/provision/webhooks/{driver}
```

### Key Jobs

* **KickProvision** — initiates provisioning flow
* **PollTask** — polls driver task status with backoff
* **SyncResourceState** — periodic sync from provider
* **DeprovisionResource** — teardown flow

All jobs carry `resource_uuid` and idempotency key.

* * *

6. Autoloading
--------------

Root `composer.json`:

```json
"autoload": {
  "psr-4": {
    "LaraBill\\Core\\": "app/Core/",
    "LaraBill\\Modules\\": "app/Modules/",
    "LaraBill\\Drivers\\": "app/Drivers/"
  }
}
```

Run `composer dump-autoload` after edits.

* * *

5. Module Registration
----------------------

**bootstrap/modules.php**

```php
use LaraBill\Core\Support\ModuleRegistrar;
foreach (ModuleRegistrar::providers() as $provider) app()->register($provider);
```

**config/modules.php**

```php
return [
  'paths' => [base_path('app/Modules'), base_path('app/Drivers')],
  'enabled' => [],
  'deferred' => false,
];
```

**app/Core/Support/ModuleRegistrar.php** scans `module.json` and loads providers.

* * *

6. Git Workflow
---------------

* Branches: `main`, `develop`, short-lived `feat/`, `fix/`, `refactor/`, `chore/`.
    
* Commit format:
    
    ```
    ✨ Add proration logic to Billing
    🚀 Optimize invoice query with composite index
    ```
    
* Pull requests: one feature per PR; include migrations, rollback notes, performance notes, and linked issues.
    

* * *

7. Code Style
-------------

* PHP 8.3+, strict types where possible.
    
* Tools: PHP-CS-Fixer, Larastan/PHPStan max level, Pint optional.
    
* No business logic in controllers or models—use Services or Actions.
    
* Money = `DECIMAL(19,4)` everywhere.
    
* Eager-load, paginate, no unbounded queries.
    

* * *

8. Testing
----------

* Framework: Pest or PHPUnit.
    
* Coverage gate ≥ 80% for domain/service code.
    
* Place module tests in `src/Tests`.
    
* Types: **Unit**, **Feature**, **Integration**, **Golden-Master** (PDF/HTML snapshots).
    
* **Integration tests with fake drivers** (Provisioning only):
    * Each driver ships a fake implementation in `Drivers/*/src/Testing/FakeDriver.php`
    * Fake drivers simulate provisioning flows without hitting real APIs
    * Golden-master tests validate webhook payload normalization
    * Matrix testing: `PROVISION_DRIVERS=fake,proxmox,tcadmin` (use `fake` only on PRs)
    * Provisioning tests only run if module is enabled
    
* CI runs `php artisan test` across all modules.
    

* * *

9. Environment & Docker
-----------------------

* Works bare-metal or Docker.
    
* `docker/compose.yml` provides app, mysql, redis, mailhog.
    
* `.env.example` covers both modes.
    
* Secrets never committed; use `.env` or secret manager.
    

* * *

10. Database
------------

* Migrations per module under `Database/Migrations`.
    
* Laravel auto-loads them via `loadMigrationsFrom()`.
    
* Composite indexes for `user_id,status`, `status,due_date`, etc.
    
* One concern per migration; never edit history.
    
* **Provisioning tables** (created only if Provisioning module enabled):
    * `resources` — provisioned infrastructure resources
    * `provision_tasks` — driver task tracking
    * `credentials` — encrypted provider credentials (per-customer or system-wide)
    * `plan_maps` — bridge between billing plans and provider plans/regions
    * `provision_audits` — **append-only** audit trail for compliance (never delete, never update)
    
* **Optional foreign key**: `billing_items.resource_id` links invoices to provisioned resources (when Provisioning enabled).
    

* * *

11. Performance & Caching
-------------------------

* Kill N+1s with `with()`/`loadCount()`.
    
* Cache read-mostly data (plans, taxes) via Redis tags.
    
* Queue heavy work (renewals, invoices) with Horizon + idempotency keys.
    
* **Provisioning queue** (when module enabled):
    * Dedicated `provisioning` queue for resource lifecycle jobs
    * Horizon tags: `resource:{id}`, `driver:{name}`
    * Per-driver **rate limits** to avoid hitting provider API limits
    * **Circuit breaker** in Orchestrator—pause driver calls if failure rate exceeds threshold
    
* **Metrics emission** (Provisioning):
    * Track task latency, success rate, retries, driver API response time
    * Expose for Prometheus/Grafana integration (optional)
    
* Use `php artisan larabill:doctor` and `npm run doctor:npm` to verify deps.
    

* * *

12. CI/CD & Makefile
--------------------

**Required CI Steps**

```
composer validate
composer install --no-interaction
php artisan larabill:doctor
npm ci && npm run doctor:npm
npm run build
php artisan migrate --env=testing
php artisan module:discover
php artisan test
# Conditional: if Provisioning enabled
php artisan test --testsuite=Provisioning
```

**Makefile Snippets**

```
up: docker compose up -d
down: docker compose down
test: php artisan test
stan: vendor/bin/phpstan analyse
fix: vendor/bin/php-cs-fixer fix --dry-run
queue: php artisan horizon                                    # optional - for provisioning
doctor: php artisan larabill:doctor && npm run doctor:npm
seed-fakes: php artisan module:seed Provisioning --class=FakeDriversSeeder  # optional
```

* * *

13. Frontend Assets (Vite)
--------------------------

* Single root `vite.config.ts` auto-discovers:
    
    ```
    app/Modules/*/src/Resources/assets/app.ts
    app/Modules/*/src/Resources/assets/app.scss
    app/Drivers/*/src/Resources/assets/app.ts
    app/Drivers/*/src/Resources/assets/app.scss
    ```
    
* Build once: `npm run build` → `public/build`.
    
* Each module loads its assets in its own views.
    
* Optional Blade directive `@viteIfExists('app/Modules/Billing/...')` guards missing assets.
    
* **Drivers** can ship admin config UI in `Resources/views/admin/*` for driver-specific settings (Provisioning only).
    

* * *

14. Dependency Rules
--------------------

* All PHP deps live in **root composer.json**.
    
* All JS deps live in **root package.json**.
    
* Each module’s `module.json` lists its desired deps.
    
* `larabill:doctor` and `npm run doctor:npm` ensure they’re satisfied; they do **not** install automatically.
    

* * *

15. Security & Data
-------------------

* No PII in logs.
    
* Webhooks verified by signature & timestamp.
    
* Sensitive fields encrypted with Laravel's encrypter.
    
* Transactions wrap renewals, invoices, payments.
    
* **Provisioning-specific security** (when module enabled):
    * **Credentials vault**: `credentials` table stores encrypted provider API keys/secrets
    * Per-customer credential storage—**never** store in `.env` for multi-tenant scenarios
    * Credentials encrypted using Laravel's encrypter; decrypt only when calling driver
    * **RBAC abilities**: `provision.*`, `resource.*`, `credentials.*` for fine-grained access control
    * **Strict no-PII in audit logs**: mask API tokens, hash provider resource IDs when logging
    * All provisioning operations require authenticated user; system operations use service account
    

* * *

16. Configuration & Module Control
-----------------------------------

Modules are controlled via their `module.json` manifest. Set `"enabled": true` to activate, `false` to disable.

### Example: Billing Only

To run LaraBill as a billing-only system, ensure Provisioning module is disabled:

```json
// app/Modules/Provisioning/module.json
{
  "name": "Provisioning",
  "enabled": false,
  ...
}
```

### Example: Billing + Provisioning

Enable the Provisioning module and required drivers:

```json
// app/Modules/Provisioning/module.json
{
  "name": "Provisioning",
  "enabled": true,
  ...
}

// app/Drivers/Compute/Proxmox/module.json
{
  "name": "Proxmox",
  "enabled": true,
  "type": "Compute",
  ...
}
```

The `ModuleRegistrar` in `app/Core/Support/ModuleRegistrar.php` scans `module.json` files and registers only enabled modules. See `config/modules.php` for discovery paths.

* * *

17. Code Examples (Provisioning Module)
----------------------------------------

**Note:** These examples apply only if you enable the Provisioning module.

### Provisioning ModuleServiceProvider

```php
namespace LaraBill\Modules\Provisioning\src\Providers;

use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;
use LaraBill\Modules\Provisioning\src\Domain\Models\Resource;
use LaraBill\Modules\Provisioning\src\Domain\Policies\ResourcePolicy;

class ModuleServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Load migrations
        $this->loadMigrationsFrom(__DIR__.'/../Database/Migrations');
        
        // Load views
        $this->loadViewsFrom(__DIR__.'/../Resources/views', 'provisioning');
        
        // Load translations
        $this->loadTranslationsFrom(__DIR__.'/../Resources/lang', 'provisioning');
        
        // Load routes
        $this->loadRoutesFrom(__DIR__.'/../Routes/api.php');
        $this->loadRoutesFrom(__DIR__.'/../Routes/web.php');
        
        // Register policies
        Gate::policy(Resource::class, ResourcePolicy::class);
        
        // Register event listeners
        Event::listen(
            \LaraBill\Modules\Billing\src\Events\PaymentCaptured::class,
            \LaraBill\Modules\Provisioning\src\Listeners\OnPaymentCaptured::class
        );
    }
    
    public function register(): void
    {
        // Register services, bindings, etc.
        $this->app->singleton(
            \LaraBill\Modules\Provisioning\src\Services\Orchestrator\ProvisionOrchestrator::class
        );
    }
}
```

### Event Listener: Kick Provisioning on Payment

```php
namespace LaraBill\Modules\Provisioning\src\Listeners;

use LaraBill\Modules\Billing\src\Events\PaymentCaptured;
use LaraBill\Modules\Provisioning\src\Jobs\KickProvision;

final class OnPaymentCaptured
{
    public function handle(PaymentCaptured $event): void
    {
        // Only provision if order requires it
        if (!$event->order->requiresProvisioning()) {
            return;
        }
        
        // Dispatch async provisioning job with idempotency key
        dispatch(new KickProvision($event->order->id))
            ->onQueue('provisioning')
            ->delay(now()->addSeconds(1));
    }
}
```

### Idempotency Key Pattern

```php
// In KickProvision job
public function handle(): void
{
    $idempotencyKey = sprintf(
        '%s:attempt_%d',
        $this->order->uuid,
        $this->attempts()
    );
    
    $driver = app(DriverManager::class)->get($this->order->driver);
    
    // Driver receives key and won't duplicate if called again
    $taskId = $driver->provision($spec, $idempotencyKey);
}
```

* * *

18. Contribution Standards
--------------------------

1. Fork and branch off `develop`.
    
2. `make up` (Docker) or bare install.
    
3. Run `php artisan key:generate && php artisan migrate --seed`.
    
4. Follow commit + PR conventions.
    
5. Don’t add global packages without documenting in `module.json`.
    

* * *

19. Release
-----------

* Semantic Versioning.
    
* Tag and build Docker image with SHA + version.
    
* Run migrations on deploy, restart Horizon, verify health checks.
    
* Update changelog and docs.
    