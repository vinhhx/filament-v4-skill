---
name: filament-4
description: "Use this skill whenever writing, reviewing, or planning code for a Filament 4 application (Laravel admin panels, resources, forms, tables, actions, infolists, widgets, notifications). Triggers include: any mention of 'Filament', 'FilamentPHP', creating/editing a Resource, Panel, Page, RelationManager, Widget, Action, Schema/Form/Infolist/Table component; questions about Filament authorization, performance, security hardening, or writing Pest/Livewire tests for a panel. Use this to make sure Filament code follows the framework's idiomatic conventions instead of generic Laravel/Livewire patterns, and to catch common mistakes around authorization, N+1 queries, and XSS. Do not use for non-Filament Laravel code or front-end work unrelated to a Filament panel."
license: Reference material derived from the Filament 4 documentation (MIT-licensed project docs).
---

# Filament 4 development

## Overview

Filament is a full-stack UI framework for Laravel/Livewire, composed of packages: `support` → `schemas` → `forms`, `infolists`, `tables`, `actions`, `notifications`, `widgets` → `panels`. This skill packages the framework's own documentation into task-focused references so an implementing agent writes idiomatic, secure, performant, and tested Filament code on the first try — without needing to re-read the entire docs tree every time.

This skill is self-contained: it bundles a full copy of the upstream Filament 4 docs under `docs/` and `packages/{package}/docs/` (relative to this skill's own root, i.e. the folder containing this `SKILL.md`), plus Filament's contributor guide as `CLAUDE.md`, so it works in any project — not just inside the `filamentphp/filament` repository. The reference files below are curated summaries with the exact method names and code patterns; `references/docs-index.md` maps every topic back to the full bundled doc file if more detail is needed.

## When to load which reference

| Situation | Read |
|---|---|
| Structuring a Resource/Page/Action into files, avoiding giant `configure()` methods, naming schema/table/component classes | `references/best-practices.md` |
| Writing PHP that matches Filament's own style (fluent setters, boolean method conventions, closures, `app()` vs `new`) — mainly relevant when extending/customizing core classes | `references/conventions.md` |
| Table/query feels slow, N+1 queries, large datasets, production deploy checklist, local dev slowness | `references/performance.md` |
| Authorization, policies, XSS/URL validation, HTML sanitization, file uploads, multi-tenancy scoping, anything touching user input | `references/security.md` |
| Writing or reviewing Pest/Livewire tests for resources, tables, actions, schemas, notifications | `references/testing.md` |
| Need the exact upstream doc for something not covered above | `references/docs-index.md` (path map into `docs/` and `packages/*/docs/`) |

Load only the reference(s) relevant to the current task — don't load all of them into context at once.

## Non-negotiable defaults (read this even if nothing else)

Apply these by default in every Filament resource/action/table you write, unless the user explicitly says otherwise:

1. **Authorize everything custom.** Filament auto-checks model policies (`viewAny`, `view`, `create`, `update`, `delete`) for standard resource CRUD only. Any custom action, custom page, or Livewire component you add must be authorized explicitly with `visible()`/`hidden()`/`authorize()` — it is not automatic. See `references/security.md`.
2. **Scope every custom query.** `getEloquentQuery()` / `modifyQueryUsing()` must reflect the current user's/tenant's permissions for any query you write yourself. Filament does not scope custom queries for you outside of its built-in tenancy feature.
3. **Never interpolate unvalidated user input into `url()`, `icon()`, or `extra*Attributes()`.** Wrap URLs from user/DB data in `Illuminate\Support\Str::sanitizeUrl()`. Never render unsanitized HTML with `{!! !!}` — use `->sanitizeHtml()`.
4. **Use dot notation and relationship aggregates (`counts()`, `exists()`, `avg()`, `sum()`) instead of manual `withCount`/accessor hacks** so Filament eager-loads correctly and you avoid N+1 queries in tables.
5. **Split large `form()`/`table()`/`infolist()` methods into dedicated `Schemas`/`Tables` classes and component classes** (see `references/best-practices.md`) rather than one huge inline array.
6. **Write a Pest/Livewire test for every resource page and every custom action you add** — at minimum `assertOk()` plus the behavior-specific assertion (`assertCanSeeTableRecords`, `assertHasFormErrors`, `assertNotified`, etc.).
7. **Never use abbreviated variable names, `final`, or `readonly` classes when writing code that extends/overrides core Filament classes** — see `references/conventions.md`. (This does not apply to normal application code that merely *uses* Filament components.)

## Quick facts

- Package boot order: `support` → `schemas` → `forms`/`infolists`/`tables`/`actions`/`notifications`/`widgets` → `panels`.
- Livewire components in a panel: pages (incl. resource pages), relation managers, widgets. NOT Livewire components: resource classes, schema components, actions — test these by unit-testing the class directly or through the Livewire component that uses them.
- Production checklist: `php artisan filament:optimize` (caches components + icons), `php artisan optimize`, ensure `FilamentUser::canAccessPanel()` is implemented (required outside `local` env).
- `composer test` runs the full Filament test suite (SQLite + commands + PHPStan) if you are contributing to Filament core itself, not to an app using it.
