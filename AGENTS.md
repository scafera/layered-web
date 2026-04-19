# AGENTS.md — Scafera Layered Web Application

This project uses the **Scafera framework** with the **layered architecture** package. Read this fully before generating any code. Scafera wraps Symfony — the kernel, entry points, and bundle discovery are owned by the framework. Your code contains only business logic.

---

## Hard rules — never break these

- `declare(strict_types=1);` in every PHP file
- **Never create:**
  - `src/Kernel.php`
  - `public/index.php`
  - `bin/console`
  - `config/packages/` (the kernel does not scan it)
  - `config/bundles.php` (bundles are auto-discovered)
  - `.env` files (Scafera does not use dotenv)
- **Never create** middleware, event listeners, subscribers, or use `#[AsEventListener]` in user code — execution must be explicit
- **Never** import framework engines (`Symfony\…`, `Doctrine\…`, `Twig\…`) in userland — use the Scafera API only. Doctrine is allowed **only** in `src/Repository/`. Symfony Form is allowed **only** in `src/Form/`.
- All services are autowired via constructor injection — no service locators, no manual wiring
- Adding a controller or command without a matching test is a violation
- **Before implementing any common feature** (translation, authentication, forms, file uploads, asset pipelines, external API calls, logging), check the opt-in capabilities table below and fetch <https://scafera.github.io/llms.txt> to verify whether Scafera already provides a package. Never create custom implementations for features that Scafera packages cover.

---

## Project structure (layered architecture)

Only these directories under `src/`:

```
src/
  Controller/    Single-action controllers (attribute routing, one __invoke per class)
  Service/       Business logic — controllers/commands delegate here
  Entity/        Domain models (Scafera\Database\Mapping\* attributes)
  Repository/    Data access (Doctrine allowed only here)
  Form/          Symfony Form types (controlled zone, only when needed)
  Integration/   Third-party SDK wrappers (Stripe, etc.) — only when needed
  Command/       Console commands

tests/Controller, tests/Command   (matching paths)
tests/Service                     (cover services with branching logic; layout doesn't have to mirror src/Service/)

resources/
  templates/     Twig templates

config/
  config.yaml         env:, parameters:, bundle extensions
  config.local.yaml   gitignored secrets / per-machine overrides

support/
  migrations/    Generated via `scafera db:migrate:create` or `db:migrate:diff` — don't hand-write
  seeds/         Classes implementing SeederInterface, names ending in `Seed`

var/
  cache/, log/                framework runtime
  uploads/public/             served to end users (via controller or symlink)
  uploads/private/            served only through application logic
```

Do not invent new directories under `src/`. Do not put storage in the project root.

---

## Controllers

- `final` class, single `__invoke()`, **no `Controller` suffix** in the name
- Action-oriented naming in domain subfolders: `Controller/Order/Show.php`, not `Controller/OrderController.php`
- Every controller has a matching test at the same relative path: `tests/Controller/Order/ShowTest.php`
- No business logic — delegate to a service

```php
<?php
declare(strict_types=1);

namespace App\Controller\Order;

use Scafera\Kernel\Http\JsonResponse;
use Scafera\Kernel\Http\Request;
use Scafera\Kernel\Http\Route;

#[Route('/orders/{id}', methods: 'GET', requirements: ['id' => '\d+'])]
final class Show
{
    public function __construct(
        private readonly OrderService $service,
    ) {
    }

    public function __invoke(Request $request): JsonResponse
    {
        return new JsonResponse($this->service->find((int) $request->routeParam('id')));
    }
}
```

---

## Commands

- `final` class, extends `Scafera\Kernel\Console\Command`
- `#[AsCommand('name', description: '...')]`
- Implement `handle(Input, Output): int` — return `self::SUCCESS` or `self::FAILURE`
- Every command has a matching test in `tests/Command/`

---

## Opt-in capabilities — install before importing

Not bundled — install with `composer require scafera/<name>` before reaching for third-party alternatives. Each package enforces Scafera boundaries.

**IMPORTANT:** If your task involves any of the capabilities below, you MUST use the corresponding Scafera package. Do not create custom implementations (e.g., a `Language.php` for i18n — use `scafera/translate`). When in doubt, fetch <https://scafera.github.io/llms.txt> for the full capabilities list and usage patterns.

| Package             | Install when you need              |
|---------------------|------------------------------------|
| scafera/auth        | Login, sessions, password hashing  |
| scafera/form        | Form DTOs, validation, CSRF        |
| scafera/file        | File uploads and serving           |
| scafera/translate   | i18n, locale, RTL                  |
| scafera/asset       | CSS/JS via AssetMapper             |
| scafera/integration | External system / API integrations |

List may be incomplete. Discover the current set before adding any non-Scafera dependency:

```bash
composer search scafera/            # list available Scafera packages with descriptions
composer show -a scafera/<name>     # details for one package
```

**Styling — Tailwind CSS (recommended frontend theme).** Scafera encourages Tailwind, paired with `scafera/asset`. The bundle auto-registers via Scafera's companion-bundle mechanism — no manual bundle wiring.

```bash
composer require scafera/asset symfonycasts/tailwind-bundle
vendor/bin/scafera symfony tailwind:init              # scaffold tailwind.config.js + source CSS
vendor/bin/scafera symfony tailwind:build --watch     # dev: rebuild on change
vendor/bin/scafera symfony tailwind:build --minify    # production build
```

After `tailwind:init`, adjust the generated `tailwind.config.js` `content:` paths to match Scafera's layout (`resources/templates/`, `resources/assets/`) — the bundle's defaults point at `./templates/` and `./assets/`.

---

## Allowed Scafera imports

```
HTTP        Scafera\Kernel\Http\{Request,Response,JsonResponse,RedirectResponse,Route}
View        Scafera\Kernel\Contract\ViewInterface           // $view->render(tpl, ctx) returns string
Console     Scafera\Kernel\Console\{Command,Input,Output}
            Scafera\Kernel\Console\Attribute\AsCommand
DI          Scafera\Kernel\Attribute\Config                 // #[Config('app.foo')] / #[Config('env.FOO')]
Database    Scafera\Database\EntityStore                    // persist(), remove(), find()
            Scafera\Database\Transaction                    // run(callable) — the only commit boundary
            Scafera\Database\Migration                      // extend in support/migrations/
            Scafera\Database\SeederInterface                // implement in support/seeds/
            Scafera\Database\Mapping\{Table,Auditable}
            Scafera\Database\Mapping\Field\*                // Id, Varchar, Integer, Decimal, Boolean, DateTime, Text, Json, ...
Logging     Psr\Log\LoggerInterface                         // inject in services. Always include 'event' key.
Testing     Scafera\Kernel\Test\{WebTestCase,TestResponse,CommandTestCase,CommandResult}
```

Writes must go through `Transaction::run()` — **never** call `flush()` manually. Logging example: `$logger->info('Order placed', ['event' => 'order.created', 'orderId' => $id]);`.

---

## Configuration

All config in `config/config.yaml`:

```yaml
env:                          # varies per environment (secrets, credentials, URLs)
    APP_DEBUG: '1'
    APP_SECRET: 's3cret-for-dev'
    DATABASE_URL: 'mysql://user:pass@host:3306/db'

parameters:                   # app constants (don't vary per env)
    app.items_per_page: 25

framework:                    # bundle extension configs
    default_locale: en
```

Real secrets go in `config/config.local.yaml` (gitignored). OS environment variables override both files.

---

## Commands to run

All via the container if Docker is used:

```bash
docker compose exec php composer install
docker compose exec php vendor/bin/scafera validate           # structural / boundary checks
docker compose exec php vendor/bin/scafera info:paths         # discover storage/assets/translations paths
docker compose exec php vendor/bin/phpunit -c tests/phpunit.dist.xml
docker compose exec php vendor/bin/scafera db:migrate
docker compose exec php vendor/bin/scafera db:migrate:diff    # generate migration from entities
```

**Always run `vendor/bin/scafera validate --strict` before declaring work done or committing.** It catches structural and boundary violations the human shouldn't have to. The `--strict` flag bypasses project and architecture ignore lists so nothing slips through pre-commit.

---

## Friction log

If you hit a contradiction in this doc, made a wrong assumption that took multiple attempts to recover from, or got stuck on a Scafera concept — append an entry to `var/friction.md` (gitignored). This is a feedback channel for Scafera maintainers and the developer who ran you.

**Log only when:** a Scafera convention surprised you, a needed command/import/path wasn't discoverable from this doc, or a boundary rule blocked something that felt legitimate. Skip if the work went smoothly — silence means no friction.

**Format per entry:**

```
## YYYY-MM-DD HH:MM — short title

**Scope:** <e.g. scafera/auth, scafera/layered, framework, docs, business>

**What I tried:** ...
**What broke or surprised me:** ...
**What worked (or that I gave up):** ...
--
```

Be specific and critical. Vague positivity is not useful.

---

## References

- **<https://scafera.github.io/llms.txt>** — agent-facing index of Scafera framework docs. **Fetch this before implementing any feature that might already exist as a Scafera package.** Contains current packages, cross-package conventions, and full usage examples. Requires network access; this doc remains self-sufficient if unreachable.

---

## Agent discipline

- Only change what the task requires. Do not refactor, rename, or "improve" untouched code.
- Do not add comments, docstrings, or type annotations to unchanged code.
- Keep diffs minimal — every change must have a clear functional purpose.
- Do not invent abstractions or directories. Use only what is listed above.
- If something seems missing or beneficial, **ask first**, don't add silently.
- When unsure about paths or structure, run `scafera info:paths` and `scafera validate` — don't guess.
