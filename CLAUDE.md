# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is LibreNMS

LibreNMS is a PHP/MySQL/SNMP-based auto-discovering network monitoring system. It is built on Laravel 12 with a large legacy codebase under `includes/` and `html/`. PHP 8.2+ is required.

## Commands

### Development Setup

```bash
./scripts/composer_wrapper.php install   # Install PHP dependencies
npm install                              # Install JS dependencies
./lnms serve                             # Start dev web server at http://localhost:8000
npm run dev                              # Start Vite dev server for assets
npm run build                            # Build production assets
```

### Validation (run before submitting PRs)

```bash
./lnms dev:check                         # Run all checks (linting, static analysis, tests)
./lnms dev:check unit                    # Run unit tests only
./lnms dev:check unit --db --snmpsim     # Full test suite (requires snmpsim)
./lnms dev:check unit -o osname          # Test a specific OS
./lnms dev:check unit -m modulename      # Test a specific module
```

### Polling and Discovery (debug)

```bash
./lnms device:discover -vv HOSTNAME      # Discover device (hide sensitive)
./lnms device:discover -vvv HOSTNAME     # Discover device (full debug)
./lnms device:poll -vv HOSTNAME          # Poll device (hide sensitive)
./lnms device:poll -vvv HOSTNAME         # Poll device (full debug)
```

### Database

```bash
php artisan make:model ModelName -m -c -r  # Create model, migration, controller, resource
php artisan migrate                         # Run migrations
```

### IDE helpers (re-run periodically)

```bash
./lnms ide-helper:generate
./lnms ide-helper:models -N
```

### Translations

```bash
./lnms translation:generate              # Rebuild JS translation files after editing lang/
```

## Architecture

### Dual-layer structure

LibreNMS has two parallel code layers that are intentionally isolated from each other:

1. **Laravel layer** (`app/`, `resources/`, `routes/`, `database/`) — handles the web UI, REST API, Eloquent models, and artisan commands. Follow standard Laravel conventions here.

2. **Legacy layer** (`includes/`, `html/`) — handles CLI polling, discovery, and the legacy web UI. This code cannot access Laravel internals directly. New web pages should use Laravel, not the legacy layer.

### Key directories

- `app/Models/` — Eloquent models. Extend `app/Models/BaseModel.php`.
- `app/Http/Controllers/` — Laravel controllers for the web UI.
- `app/Console/Commands/` — Artisan commands (e.g. `DeviceDiscover`, `DevicePoll`).
- `LibreNMS/` — Non-Laravel classes, namespaced to match directory structure (PSR-0). One class per file.
- `LibreNMS/OS/` — Per-OS PHP classes (implement discovery/polling traits).
- `LibreNMS/Modules/` — Discovery and polling modules (e.g. `Mempools`, `Netstats`).
- `includes/discovery/` — Legacy discovery module `.inc.php` files.
- `includes/polling/` — Legacy polling module `.inc.php` files.
- `includes/html/pages/` — Legacy web UI pages (URL routing via `html/index.php`).
- `resources/definitions/os_detection/` — YAML files for OS detection rules.
- `resources/definitions/os_discovery/` — YAML files for SNMP-based sensor/health discovery.
- `resources/definitions/config_definitions.json` — Defines all configurable settings.
- `tests/` — PHPUnit tests; `tests/snmpsim/*.snmprec` is captured device SNMP data; `tests/data/*.json` is expected database state.
- `mibs/` — SNMP MIB files. Standard MIBs at root, vendor-specific in subdirectories.

### SNMP access

Use `SnmpQuery` (not raw snmp functions) for all SNMP operations in new code:

```php
SnmpQuery::get('SNMPv2-MIB::sysName.0')->value();
SnmpQuery::walk('IF-MIB::ifTable')->table();
```

### Adding a new OS

1. Create `resources/definitions/os_detection/<os>.yaml` — detection rules (`sysObjectID` preferred).
2. Create `resources/definitions/os_discovery/<os>.yaml` — YAML-based sensor discovery (no PHP needed for simple cases).
3. Optionally create `LibreNMS/OS/<OsClass>.php` — for custom PHP discovery/polling logic.
4. Capture test data: `./scripts/collect-snmp-data.php -h HOSTNAME`
5. Save expected DB state: `./scripts/save-test-data.php -o osname`
6. Verify: `./lnms dev:check unit -o osname`

### Adding config settings

Define the setting in `resources/definitions/config_definitions.json`, add English translations to `resources/lang/en/settings.php`, then run `./lnms translation:generate`.

### Database migrations

Create with `php artisan make:model ModelName -m`. Never modify existing migrations — always add new ones.

## Testing

OS module tests require snmpsim (`pip3 install snmpsim`). The test database defaults to `librenms_phpunit_78hunjuybybh`; configure via `.env` with `DB_TEST_*` variables.

Review `tests/snmpsim/*.snmprec` and `tests/data/*.json` files for sensitive data before committing.

## PR checklist

Per `.github/PULL_REQUEST_TEMPLATE.md`: run `./lnms dev:check`, include WebUI screenshots for UI changes, add/update test data for discovery/polling/YAML changes.
