# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

This repository is currently in the **design phase** — no application code exists yet (no `.sln`/`.csproj`, no `src/`). Only design documentation exists: `README.md` (product overview) and `ARCHITECTURE.md` (technical architecture, decided). There are no build, lint, or test commands to run yet. Once the solution is scaffolded, this file must be updated with real commands (`dotnet build`, `dotnet test`, etc.) and a description of the actual source layout.

## What this project is

ClinicBot Core: a multi-channel conversational agent that automates billing/accounting for clinics. Staff report sessions and expenses via Telegram (photos of receipts or structured messages); the system validates and stores them as an immutable event log, then on a "cierre de mes" (month close) command applies configurable business rules to produce two parallel Excel ledgers (a real-time internal "Maestro" and a purged/invoiced "Oficial" for the accountant) plus generated PDF invoices filed into a cloud folder hierarchy.

Read `README.md` for the full functional spec (cash-matching/"cuadre de caja" algorithm, grouped invoicing by center, configurable payment-method rules) and `ARCHITECTURE.md` for the technical design — do not duplicate that content here; treat both as the source of truth and keep this file in sync with them if they change.

## Architecture essentials (see ARCHITECTURE.md for full detail and diagrams)

- **Stack**: ASP.NET Core (C#) backend, Vue.js frontend, PostgreSQL, hosted on Azure (Container Apps). AI calls go through Azure AI Foundry (provider-agnostic gateway), not a direct Anthropic/OpenAI SDK — starting model is the cheapest one available with adequate text+vision quality, swappable via config.
- **Pattern**: Hexagonal/Clean Architecture — domain (entities, business rules, cuadre de caja algorithm) must stay free of infrastructure dependencies (DB, storage, channel adapters, LLM client). Use MediatR for application use cases.
- **Immutability**: the ledger is an append-only event table, never mutated in place; the Maestro/Oficial exports are projections derived from it. This is required both for auditability and for Veri*Factu (Spanish e-invoicing) compliance.
- **Multi-tenancy = physical isolation of compute/storage/secrets per client, shared DB server**: each client (clinic group) gets its own Blob Storage account, Key Vault, and Container App instance, plus its own **database** (not just a `TenantId` column) — but that database lives on a **shared** PostgreSQL Flexible Server, not a dedicated server per client (changed from an earlier fully-dedicated-server design for cost reasons: one server bill instead of N). Non-data-bearing infrastructure is also shared across clients: frontend static app, container registry, CI/CD pipeline, Container Apps *Environment*, Entra External ID tenant. Do not reintroduce shared-*table* multi-tenancy (`TenantId` row filters) — that model was explicitly rejected; per-client database catalogs are the isolation boundary.
- **Compliance drives design, not just features**: Veri*Factu requires a per-tenant hash-chained, immutable invoice record; RGPD requires access control and data residency in the EU (Azure Spain Central, or West Europe if a service's free tier doesn't cover Spain Central). Any change touching invoice generation or the event log must preserve the hash chain and immutability guarantees.
- **Jobs**: month-close and other batch operations run via Hangfire on top of the same Postgres instance, must be idempotent (safe to retry without duplicating invoices), and the cash-matching algorithm's randomness must be seeded and the seed persisted for auditability.
- **Cost constraint (current phase)**: the project runs on an Azure for Students credit — finite, no auto-recharge, no backup card. No Azure Front Door / WAF (no free tier); rely on Container Apps/Static Web Apps native TLS ingress plus ASP.NET Core rate limiting, EF Core parameterized queries, and Entra's built-in brute-force protection instead. Container registry is GitHub Container Registry (free), not Azure Container Registry. Set an Azure Cost Management budget alert before any deployment — this is a hard requirement, not a suggestion, given the credit has no overage safety net. See `ARCHITECTURE.md` §11 for the full list of cost trade-offs; revisit them once the project has real budget.

## Commit Message Instructions

Generate commit messages following **Conventional Commits** with a **Gitmoji prefix**. This is the canonical source of truth for the gitmoji↔type mapping — `commitlint.config.js` (see `CI_CD.md`) enforces it programmatically, so keep the two in sync if this table changes.

### Format

```
<gitmoji> <type>(<scope>): <description>

[optional body — explain WHY, not what]

[optional footer — BREAKING CHANGE: description]
```

**Rules:**
- Subject max 100 characters
- Description can start with uppercase or lowercase
- Scope is optional but recommended (component, module, or feature area)
- Body only when the reason is non-obvious
- Use `BREAKING CHANGE:` footer for breaking changes, or `!` after type: `feat!:`

### Gitmoji by commit type

Pick the **most specific** emoji that matches the change. Default to the first one per type when in doubt.

#### `feat` — New functionality

| Emoji | Code | When |
|-------|------|------|
| ✨ | `:sparkles:` | General new feature |
| 💄 | `:lipstick:` | UI/style new component |
| ♿️ | `:wheelchair:` | Accessibility improvement |
| 🚸 | `:children_crossing:` | UX/usability improvement |
| 📱 | `:iphone:` | Responsive design |
| 🌐 | `:globe_with_meridians:` | Internationalization / i18n |
| 🚩 | `:triangular_flag_on_post:` | Feature flags |
| 🛂 | `:passport_control:` | Auth, roles, permissions |
| 💫 | `:dizzy:` | Animations / transitions |
| 💥 | `:boom:` | Breaking change feature (`feat!:`) |

#### `fix` — Bug fix

| Emoji | Code | When |
|-------|------|------|
| 🐛 | `:bug:` | General bug fix |
| 🚑️ | `:ambulance:` | Critical hotfix |
| 🩹 | `:adhesive_bandage:` | Minor / non-critical fix |
| 🔒️ | `:lock:` | Security or privacy fix |
| ✏️ | `:pencil2:` | Typo fix |
| 🥅 | `:goal_net:` | Error handling / catch errors |
| 👽️ | `:alien:` | Fix due to external API change |

#### `perf` — Performance

| Emoji | Code | When |
|-------|------|------|
| ⚡️ | `:zap:` | Any performance improvement |

#### `refactor` — Code restructure (no functional change)

| Emoji | Code | When |
|-------|------|------|
| ♻️ | `:recycle:` | General refactor |
| 🔥 | `:fire:` | Remove code or files |
| ⚰️ | `:coffin:` | Remove dead code |
| 🗑️ | `:wastebasket:` | Deprecate code |
| 🚚 | `:truck:` | Move or rename files / routes |
| 🏗️ | `:building_construction:` | Architectural changes |
| 🦺 | `:safety_vest:` | Add or improve validation logic |

#### `docs` — Documentation only

| Emoji | Code | When |
|-------|------|------|
| 📝 | `:memo:` | General documentation |
| 📄 | `:page_facing_up:` | License file |
| 💡 | `:bulb:` | Source code comments |
| 💬 | `:speech_balloon:` | Text content / literals |

#### `style` — Formatting, no logic change

| Emoji | Code | When |
|-------|------|------|
| 🎨 | `:art:` | Code structure / formatting |
| 🚨 | `:rotating_light:` | Fix compiler / linter warnings |

#### `test` — Tests

| Emoji | Code | When |
|-------|------|------|
| ✅ | `:white_check_mark:` | Add or update tests |
| 🧪 | `:test_tube:` | Add failing test |
| 📸 | `:camera_flash:` | Snapshot tests |

#### `build` — Build system / dependencies

| Emoji | Code | When |
|-------|------|------|
| 📦️ | `:package:` | Update compiled files or packages |
| ➕ | `:heavy_plus_sign:` | Add a dependency |
| ➖ | `:heavy_minus_sign:` | Remove a dependency |
| ⬆️ | `:arrow_up:` | Upgrade dependencies |
| ⬇️ | `:arrow_down:` | Downgrade dependencies |
| 📌 | `:pushpin:` | Pin dependency to specific version |
| 🏷️ | `:label:` | Add or update TypeScript types |

#### `chore` — Maintenance

| Emoji | Code | When |
|-------|------|------|
| 🔧 | `:wrench:` | Update configuration files |
| 🔨 | `:hammer:` | Update dev scripts |
| 🙈 | `:see_no_evil:` | Update .gitignore |
| 🍱 | `:bento:` | Add or update assets |
| 🔊 | `:loud_sound:` | Add or update logs |
| 🔇 | `:mute:` | Remove logs |

#### `ci` — CI/CD pipelines

| Emoji | Code | When |
|-------|------|------|
| 👷 | `:construction_worker:` | Add or update CI pipeline |
| 💚 | `:green_heart:` | Fix CI build |
| 🚀 | `:rocket:` | Deployment configuration |
| 🧱 | `:bricks:` | Infrastructure changes |
| 🩺 | `:stethoscope:` | Add or update health checks |

#### `revert` — Revert a commit

| Emoji | Code | When |
|-------|------|------|
| ⏪️ | `:rewind:` | Revert any previous commit |

### Examples

```
✨ feat(dashboard): Add real-time sensor chart with zoom
🐛 fix(auth): Resolve token refresh infinite loop
⚡️ perf(devices): Replace N+1 query with single join
♻️ refactor(alarms): Extract threshold validation to domain service
📝 docs(api): Add examples to device installation endpoints
⬆️ build(deps): Upgrade Vuetify from 3.4 to 3.5
👷 ci(pipelines): Add PR title validation for conventional commits
⏪️ revert: Revert "feat(sensors): add export button"
✨ feat!: Replace REST pagination with cursor-based

🚑️ fix(mqtt): Handle SSL reconnection after 5 consecutive failures

Exponential backoff added to avoid hammering the broker.
Max retry interval: 5 minutes.
```
