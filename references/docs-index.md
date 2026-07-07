# Filament docs index

This skill bundles the full upstream Filament 4 documentation so it works standalone in any project, not just inside the Filament monorepo. All paths below are relative to **this skill's own root directory** (the folder containing `SKILL.md`): `docs/` (panel-level concerns) and `packages/{package}/docs/` (per-package component docs). Use this index to jump straight to the right file instead of grepping the whole tree.

If this skill is running from inside a checkout of the `filamentphp/filament` repository itself, these same relative paths also happen to resolve at the repo root — but don't rely on that; always resolve paths from the skill directory.

## Introduction / setup

- `docs/01-introduction/01-overview.md` — what Filament is
- `docs/01-introduction/02-installation.md`
- `docs/01-introduction/03-ai.md` — Laravel Boost + Filament Blueprint (paid AI planning/security-audit tooling)
- `docs/01-introduction/04-optimizing-local-development.md` — local dev perf (OPcache, antivirus, Debugbar/Xdebug)
- `docs/02-getting-started.md`

## Resources (CRUD)

- `docs/03-resources/01-overview.md` — resource anatomy, authorization, `getEloquentQuery()`, protecting model attributes
- `docs/03-resources/02-listing-records.md` — table page, deferred badges
- `docs/03-resources/03-creating-records.md`
- `docs/03-resources/04-editing-records.md`
- `docs/03-resources/05-viewing-records.md`
- `docs/03-resources/06-deleting-records.md`
- `docs/03-resources/07-managing-relationships.md`
- `docs/03-resources/08-nesting.md`
- `docs/03-resources/09-singular.md`
- `docs/03-resources/10-global-search.md`
- `docs/03-resources/11-widgets.md`
- `docs/03-resources/12-custom-pages.md`
- `docs/03-resources/13-code-quality-tips.md` — schema/table/component class extraction (see `best-practices.md`)

## Panel configuration / navigation / users

- `docs/05-panel-configuration.md`
- `docs/06-navigation/*`
- `docs/07-users/01-overview.md` — `FilamentUser`, `canAccessPanel()`
- `docs/07-users/02-multi-factor-authentication.md`
- `docs/07-users/03-tenancy.md` — multi-tenancy scoping

## Styling

- `docs/08-styling/01-overview.md`, `02-css-hooks.md`, `03-colors.md`, `04-icons.md`

## Advanced

- `docs/09-advanced/01-render-hooks.md`
- `docs/09-advanced/02-assets.md`
- `docs/09-advanced/03-enums.md`
- `docs/09-advanced/04-file-generation.md`
- `docs/09-advanced/05-modular-architecture.md`
- `docs/09-advanced/06-security.md` — **the** security reference (see `security.md` for a curated summary)

## Testing

- `docs/10-testing/01-overview.md` through `06-testing-notifications.md` (see `testing.md` for curated summary)

## Plugins

- `docs/11-plugins/01-getting-started.md` through `05-configurable-resources-and-pages.md`

## Shared UI components

- `docs/12-components/*` — Action, Form, Infolist, Notifications, Schema, Table, Widget, and small components (Avatar, Badge, Button, Modal, Section, Select, Tabs, etc.)

## Deployment

- `docs/13-deployment.md` — `filament:optimize`, `canAccessPanel()` requirement in production (see `performance.md` + `security.md`)

## Upgrade guide

- `docs/14-upgrade-guide.md`

## Per-package component docs

- `packages/forms/docs/*` — every field type (`TextInput`, `Select`, `FileUpload`, `RichEditor`, `Repeater`, `Builder`, validation, custom fields)
- `packages/tables/docs/*` — columns, filters, actions, layout, summaries, grouping, custom data
- `packages/actions/docs/*` — modals, grouping, Create/Edit/View/Delete/Replicate/Import/Export actions
- `packages/infolists/docs/*` — entry types
- `packages/schemas/docs/*` — layout components (Section, Tabs, Wizard), custom components
- `packages/notifications/docs/*` — database + broadcast notifications
- `packages/widgets/docs/*` — stats overview, charts

## Contribution conventions (only relevant when extending Filament core / publishing a plugin)

- `CLAUDE.md` (bundled at this skill's root) — naming, fluent API, boolean-method conventions, `composer test`/`composer cs` commands (see `conventions.md`). This is Filament's own contributor guide, included here for reference; it does not apply to ordinary application code that just uses Filament.
