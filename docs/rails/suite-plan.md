# Craft suite on Rails: analysis and plan

> **Status:** planning baseline, 2026-11-01 · **Decision record:** [ADR 0002](../adr/0002-rails-ui-layer.md)
> · **Measured on:** upstream snapshots of 2026-11-01. Numbers move daily upstream.

## 1. Goal

Port the storytold craft apps, one after another, to Rails 8 + Hotwire + ViewComponent (no
React). Keep each app's Rust engine and replace only its UI. Ship:

1. **Gems / mountable engines** that the owner's apps (railstation.dev, plai.cx / dreamengine,
   mosaik.marketing) can drop in, restyle to their design system, and use to replace their own
   weaker equivalents. **This is the primary aim.**
2. **A SaaS** where each app is a separately bookable plugin, plus a cheaper full-suite bundle.

The design principles follow from that order: the engines must not depend on any single host's
auth, tenancy, billing or CSS. SaaS concerns form an optional engine of their own.

## 2. What is being ported (measured)

All counts are non-test Rust lines, rounded. "UI to replace" is ui-egui plus the desktop and web
app shells. Readiness is each repo's own ROADMAP headline.

| Order | App | Upstream target | Rust total | UI to replace | Commands | Upstream stage |
|---|---|---|---|---|---|---|
| 1 | filmcraft | Premiere Pro | 450k | 73k | `Session::execute` | alpha; 400–700 h to beta |
| 2 | effectcraft | After Effects | 348k | 91k | `Session::execute` | alpha; 350–620 h |
| 3 | soundcraft | Pro Tools | 90k | 22k | `Session::execute` | pre-alpha; 110–180 h to alpha |
| 4 | photocraft | Photoshop | 498k | 182k | `Session::execute` | alpha; 1,100–1,800 h |
| 5 | lightcraft | Lightroom | 263k | 74k | `Session::execute` | alpha; 340–600 h |
| 6 | vectorcraft | Illustrator | 472k | 129k | `Session::execute` (about 710) | alpha; 170–270 h |
| 7 | designcraft | InDesign | 222k | 85k | `Session::execute` (446) | pre-alpha |
| 8 | figcraft | Figma | 199k | 62k | `Session::execute`, plus a collab server | alpha; 162–292 h |
| 9 | deckcraft | PowerPoint | 184k | 34k | `Session::execute` (320) | alpha; 120–210 h |
| 10 | pdfcraft | Acrobat | 396k | 73k | `Session` + `automation::call` | alpha; 400–700 h |
| 11 | wordcraft | Word | 232k | 61k | `Session::run` | alpha; 150–230 h |
| 12 | gridcraft | Excel | 144k | 49k | `Session::execute` | alpha; 160–240 h |
| 13 | cadcraft | AutoCAD | 161k | 31k | `Session::execute` | pre-alpha; 125–190 h to alpha |

Shared upstream libraries:

- **`craft-libs` (36 crates).** About 65k lines are egui- or OS-bound (ui, keymap, menu, window,
  update, font-manager, quit, platform). Those are replaced, not ported. The engine-side crates
  are reused: docpath, pdf, print, media, jpeg, cms, text, font-catalog and others.
- **`craft-i18n`.** TSV catalogs; only wordcraft uses them today.
- **`craft-ai-integrations`.** Provider adapters; no app depends on it yet.

Every app pins storytold forks of egui and winit. Those disappear with the UI.

**Ordering rationale.** Apps are grouped into families so that each family's interactive surface
is built once:

- **time-based (1–3):** shared timeline, transport, keyframes, mixer. effectcraft already pulls
  18 crates from filmcraft.
- **raster (4–5):** canvas, layers, brushes, histogram.
- **vector / layout (6–9):** artboard canvas, path tools, text frames, pages, collaboration.
- **documents (10–12):** page view, text, grids.
- **CAD (13).**

The order inside families favours the owner's hosts: plai's Sequence NLE, compositions, audio
and image editor (§3) map to apps 1–5.

## 3. Reuse from the owner's repos

| Repo | State (2026-11-01) | What we take | What a craft engine replaces there |
|---|---|---|---|
| **dreamengine** (plai.cx), `develop` | Mature Rails 8.1. 308 components; 790 Minitest files; Devise; user-owned projects with `ProjectMembership` roles. | `ApplicationComponent` conventions (keyword args, typed props, slots, `fragment_cache`); the `Ui::*` primitives (about 27) as the seed of `craft_ui`; the token pattern (`--plai-*`, `--st-*`); billing ledger patterns (`Wallet`/`WalletEntry` append-only ledger, `UsageReservation`/`UsageEntry`, `GenerationCreditTransaction`, versioned price policies, Stripe webhook/reconcile); MCP server with OAuth (`gem "mcp"`, `app/tools/mcp/`); Studio editor registry (`Studio::Editors`); the agent protocol (one worktree and branch per issue, serial merges); spec-kit skills. | Sequence NLE (`app/services/sequences/`, ffmpeg-subprocess renderer, legacy `Timeline`/EDL exporter) → filmcraft. Compositions (ffmpeg renderer) → effectcraft. Audio mix / Wavelength → soundcraft. `ImageDocument`/`ImageMask` editor → photocraft. Storyboard / webtoon → designcraft (to evaluate). The domain mapping must be verified per app in its spec. |
| **railstation** (railstation.dev) | Early starter kit, 1 commit. Devise, Stripe Checkout/Portal/webhooks, CMS, RSpec (83 examples). One plan per user, no tenancy, no tokens, 5 components that nothing renders. | Stripe Checkout / Portal / webhook flow. As a sold Rails starter it is the natural **SaaS host** for the suite (its README v2 list already includes teams and AI). | Its planned WYSIWYG (v2) → wordcraft (to evaluate). Blog images use ActiveStorage with no editing → photocraft (optional). |
| **mosaik** (mosaik.marketing) | Gems installed, almost no code: no models, components or specs. The design lives in `.claude/rules/*` (Organization tenant with acts_as_tenant, Pundit roles, tiers Free/€9/€29/€99, credits at 1 = €0.01 cost / €0.02 price, Fibonacci spacing, brand tokens). | Its rules confirm the host contract has to support an **Organization** tenant and Pundit. Its credit model matches dreamengine's ledger. | Planned Studio video generation and storyboard canvas → filmcraft / figcraft. PDF (prawn/grover) → pdfcraft output. |

**Shared conventions, verified in all three:**

- Rails 8.1, Ruby 3.3, propshaft + importmap + tailwindcss-rails 4.
- Turbo and Stimulus; ViewComponent 4 (mosaik pins 4.6.0, railstation 4.12.0).
- Solid Queue, Solid Cache and Solid Cable; PostgreSQL; Kamal; Devise 5; Stripe.

The engines target exactly this stack.

**Differences the contract must absorb:**

| | Tenant model | Authorization | Tests |
|---|---|---|---|
| railstation | user only | hand-rolled `admin?` | RSpec |
| dreamengine | user plus per-project membership | `ProjectAuthorization` | Minitest |
| mosaik | Organization via acts_as_tenant (planned) | Pundit | RSpec (planned) |

**Workflow note.** The coordinator / worker / issue / worktree workflow is not written down in
mosaik. Its closest written form is dreamengine's `docs/studio/agent-protocol.md` plus the
`spec-kit-maqa*` repos. Phase R0 adopts it as written in the owner's preferences.

## 4. Architecture

### 4.1 Gems

```
craft-rails (new monorepo, gems released together)
├─ craft_core      engine: host contract, CraftDocument/owner model, CommandsController,
│                  command bus → Turbo Streams, ActionCable job progress, ActiveStorage media
│                  adapter, automation ids, i18n, generators (craft:install, craft:theme)
├─ craft_ui        ViewComponent kit: primitives (from dreamengine Ui::*), workspace shell
│                  (docking, tabs, panels), menus/shortcuts from the command registry,
│                  inspector forms from param schemas, Lookbook previews
├─ craft_runtime   importmap-pinned ES modules: WASM loader + worker, canvas surface,
│                  AudioWorklet, WebCodecs, the window.craft API, base Stimulus controllers
├─ craft_bridge    Rust crate + Ruby glue template (magnus/rb-sys): generic over a
│                  `CraftSession` trait (execute, commands, events, render, jobs);
│                  catch_unwind; GVL released for long jobs
├─ craft_saas      optional engine: products, plans, bundles, entitlements, seats, credits /
│                  usage ledger, Stripe (from dreamengine + railstation patterns)
└─ craft_mcp       registers each mounted app's commands as tools in the host's MCP server

<app>-rails (one fork per craft app, upstream-syncable)
├─ crates/ruby-bridge   implements CraftSession for the app's Session
├─ crates/wasm-core     headless wasm-bindgen build of the engine (no eframe)
└─ rails/               gem `<app>_rails`: Engine, app-specific components and Stimulus
                        controllers (e.g. timeline), native ext, prebuilt .wasm
```

The families share components through `craft_ui` once a second app needs them. Examples: a
timeline for film, effect and sound; a raster canvas for photo and light. The first app builds
the component in its own gem, and the second app's spec moves it into the shared gem.

### 4.2 Host contract (`craft_core`)

```ruby
# config/initializers/craft.rb (written by `rails g craft:install`)
Craft.configure do |c|
  c.parent_controller = "::ApplicationController"        # Devise helpers, layout, CSRF
  c.current_user  = ->(ctrl) { ctrl.current_user }
  c.current_owner = ->(ctrl) { ctrl.current_user }       # mosaik: ActsAsTenant.current_tenant
  c.authorize     = ->(user, action, record) { record.owner == user }   # or Pundit / ProjectAuthorization
  c.entitled      = ->(owner, app_key) { true }          # craft_saas overrides
  c.meter         = ->(owner, operation, quantity) {}    # craft_saas overrides
  c.layout        = "application"                        # or the host's page shell
  c.storage_service = :default                           # ActiveStorage service
end
```

`CraftDocument` belongs to a polymorphic `owner` (User, Organization, Experience…) and stores the
engine's project file. Media are ActiveStorage attachments.

### 4.3 Styling (host adapts the look, functionality stays)

1. **Tokens only.** `craft_ui` styles everything through CSS custom properties: `--craft-bg`,
   `--craft-panel`, `--craft-line`, `--craft-text`, `--craft-accent`, `--craft-radius`,
   `--craft-font-ui` and others. The gem has defaults for light and dark.
2. **The gem ships prebuilt CSS** that needs neither Node nor the host's Tailwind build, so it
   works in every host. Hosts map tokens in one file. For example, dreamengine:
   `--craft-bg: var(--st-bg); --craft-accent: var(--plai-accent)`.
3. **Component substitution.** `Craft.ui.override(:button, Ui::ButtonComponent)` lets a host
   render its own primitives inside craft panels, as long as they accept the same keyword
   interface.
4. **Layout slots.** The workspace shell renders inside the host layout. A host can use the
   full-screen editor layout or embed a single panel (for example only the timeline) in its own
   page.

Whether tailwindcss-rails 4 can compile engine stylesheets inside the host is **not verified**.
R1 checks it as an optional second route.

### 4.4 Runtime split and efficiency

- **Browser (WASM):** preview, scrubbing, playback, canvas tools. The `.wasm` is served by
  propshaft with immutable caching. Cost per user is close to zero.
- **Server (native gem):** each command is validated and journaled against the stored document.
  Export, render and proxy run as Solid Queue jobs with progress over Solid Cable.
- **Command flow:**
  1. Edits apply optimistically in the browser engine.
  2. The same command id and params post to `CommandsController`.
  3. The server replays them on its `Session`.
  4. The response is a Turbo Stream for the non-canvas panels.
  5. If the server result diverges, the client resyncs from the server document.
- **Replacing ffmpeg pipelines** (plai's Sequence and Composition renderers): the native export
  runs in-process. There is no subprocess, and it has exact `Tick` timing. Migration goes
  through the engine's existing interchange importers (OTIO / FCPXML), with an adapter on the
  host side.

### 4.5 Generated UI

Menus, shortcuts and enabled states come straight from `CommandSpec`.

**Measured blocker:** `CommandSpec.params` is a free-text "JSON-ish" doc, not a schema.
Generating dialogs and inspector forms needs a structured, serde-derived parameter schema per
command, which is an L4 change. It is proposed upstream to storytold (it also improves their MCP
schemas) rather than carried as a fork patch. Until it lands, dialogs are hand-built components.
This is the biggest single efficiency lever across 13 apps.

## 5. SaaS (`craft_saas`)

| Model | Purpose |
|---|---|
| `Craft::Product` | one per app key (`film`, `photo`, …), plus a `suite` product |
| `Craft::Plan` | price, interval, Stripe price id; has many `PlanEntitlement` (app key, seats, quotas) |
| `Craft::Subscription` + items | one Stripe subscription with an item per booked app, or the suite plan; owner is polymorphic |
| `Craft::Entitlement` | resolved and cached per owner; `Craft.config.entitled` reads it |
| `Craft::Ledger` / `Craft::LedgerEntry` | append-only credits for server work (exports, renders, AI), following dreamengine's wallet ledger and usage reservations; mosaik's 1 credit = €0.01 cost / €0.02 price fits |

- **Pricing shape:** each app is a subscription item; the suite is a single plan priced below
  the sum of its parts. Prices and the Stripe tax setup are owner decisions and are not set here.
- **Unit economics:** preview runs on the client, so server cost is mainly storage plus metered
  exports and AI. That keeps a cheap suite plan viable.
- **Host:** a thin `craftsuite` app based on railstation (auth, Stripe, CMS, legal pages) that
  mounts all engines plus `craft_saas`.

## 6. Estimate

The unit is Opus 5.5 agent wall-clock hours, consistent with upstream's calibration of about 4–5
hours per checklist point.

**Basis.** FilmCraft's app-specific UI is estimated at 600–1,100 h for full parity over 73k UI
lines, which is 8.2–15 h per 1k lines. Later apps are scaled by their measured UI size times a
reuse factor of 0.60–0.85, for the shared kit and family components. The minimum viable product
(MVP: a core workflow that is usable and sellable) is taken as 30% of full parity. The reuse
factor and the 30% are judgment, not measurement. Recalibrate them after the FilmCraft pilot.

| Item | MVP | Full UI parity |
|---|---|---|
| Shared gems (`craft_core`, `craft_ui`, `craft_runtime`, `craft_bridge`, `craft_mcp`) | 250–450 | included |
| `craft_saas` and the craftsuite host | 80–160 | included |
| Host integration (dreamengine, mosaik, railstation; plai sequence migration) | 60–120 | included |
| filmcraft UI | 200–350 | 600–1,100 |
| Bridge, WASM core and packaging for apps 2–13 (template, 20–40 each) | 240–480 | included |
| effectcraft | 140–340 | 450–1,150 |
| soundcraft | 30–80 | 110–280 |
| photocraft | 270–700 | 890–2,320 |
| lightcraft | 110–280 | 370–950 |
| vectorcraft | 190–490 | 630–1,640 |
| designcraft | 130–320 | 420–1,080 |
| figcraft (collab extra not included) | 90–240 | 310–790 |
| deckcraft | 50–130 | 170–430 |
| pdfcraft | 110–280 | 360–930 |
| wordcraft | 90–230 | 300–780 |
| gridcraft | 70–190 | 240–620 |
| cadcraft | 40–120 | 150–390 |
| **Total** | **≈ 2,150–4,970** | **≈ 5,600–13,700** |

Not included in these totals:

- **Upstream sync:** 2–5 h a week per ported app.
- **Upstream's own gaps** (beta and parity work), which happens upstream.
- **figcraft multiplayer collaboration.**

**Elapsed time.** About 65% of the work can run in parallel, more across apps. With 5 workers,
elapsed time is about 0.48× the hours, so the MVP of the whole suite takes about 1,000–2,400 h of
continuous agent time. FilmCraft's MVP, including the shared foundation, takes about 300–550
hours of work, or about 150–260 h elapsed.

**Claude usage.**

| Model | Input / 1M tokens | Output / 1M tokens | Cache reads / 1M tokens |
|---|---|---|---|
| Opus 5.5 | $4 | $20 | $0.20 |
| Sonnet 5.5 | $2 | $10 | $0.20 |
| Haiku 5.5 | $0.10 | $0.50 | — |

These are first-party API list prices, cached 2026-10-06. Tokens per agent-hour have **not been
measured** for this workflow. Phase R0 measures them on its first issues. After that:

`cost = Σ hours(model) × tokens per hour(model) × price(model)`

Expected model mix:

- Haiku: the coordinator.
- Opus: bridge, WASM core, canvas and timeline surfaces, shared architecture.
- Sonnet: panels, dialogs, ViewComponents, tests and docs; most of the hours.

## 7. Phases (FilmCraft first)

Each phase becomes one spec-kit feature, then `tasks`, then `taskstoissues`, run with the
coordinator / worker workflow.

| Phase | Spec | Content | Gate |
|---|---|---|---|
| R0 | Governance | Fork rule amendments (ADR 0002 §Consequences); upstream-sync procedure; create `craft-rails` repo; Ruby CI; precompiled-gem matrix; token-usage measurement | rules merged, CI green on an empty gem |
| R1 | Bridge | `craft_bridge` + `crates/ruby-bridge`: `Session` in Ruby, execute/commands/events/undo/export, panic → exception, GVL | bridge specs pass; `filmcraft_rails` loads in a dummy app |
| R2 | WASM core | `crates/wasm-core` split from `apps/filmcraft-web` without eframe; `craft_runtime` JS API; render to canvas; audio | a frame renders and plays in a Stimulus-hosted canvas |
| R3 | Engine skeleton | `craft_core`: contract, `CraftDocument`, CommandsController, Turbo Streams, ActiveStorage import, Solid Queue export with Cable progress, `craft:install` | import → cut → export works headless through HTTP in the dummy app |
| R4 | Shell and kit | `craft_ui` primitives (from dreamengine `Ui::*`), tokens, prebuilt CSS, docking workspace, menus and shortcuts from the registry, Lookbook | the shell renders in dreamengine, mosaik and railstation with their tokens |
| R5 | Editor MVP | project bin, source and program monitors, timeline (canvas plus Stimulus), effect controls, export dialog | the upstream alpha core workflow passes as Rails system tests |
| R6 | Host adoption | dreamengine: replace the Sequence NLE (OTIO migration); mosaik and railstation mounts | plai sequences open in filmcraft and export natively |
| R7 | SaaS | `craft_saas` and the craftsuite host; per-app items and the suite bundle; metering | Stripe test-mode purchase unlocks an app |
| R8+ | Parity | remaining panels in priority order from upstream `docs/gaps.md`; automation and MCP parity | per-panel system and screenshot tests |

Proposed upstream contributions (filed after R0): structured command parameter schemas; a
headless `wasm-core` crate (if storytold wants it).

## 8. Risks

| Risk | Mitigation |
|---|---|
| The upstream UI grows daily (moving target) | Generate UI from the registry (§4.5); sync weekly; take parity targets from upstream `docs/gaps.md` |
| Native gem builds across five platforms | rb-sys cross-compile images; a pure-WASM fallback for preview |
| WASM threads need cross-origin isolation (COOP/COEP), which can break host pages | Isolate only the editor route; the single-threaded build works today |
| Engine memory per server `Session` | Sessions are short-lived per request or job, rebuilt from the stored document; Solid Cache for hot projects |
| Host contract is too narrow for one host | R4 gate requires all three hosts |
| Upstream licence or name changes | All repos are MIT / Apache-2.0 today; name use is an open item (ADR 0002 §6) |

## 9. Open questions for the owner

See ADR 0002 §6. Additional questions:

1. Prices and tiers for apps and the suite.
2. Whether railstation becomes the SaaS host (recommended) or the SaaS gets a new repo.
3. Whether dreamengine's Minitest suite and the RSpec hosts both need contract tests (proposed:
   the gems test with RSpec and ship a Minitest-compatible test helper).
