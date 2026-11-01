# ADR 0002: A Rails 8 / Hotwire / ViewComponent UI layer on the unchanged Rust engine

- **Status:** proposed. The project owner approved the direction on 2026-11-01 in the planning
  session. The open items in §6 need a decision before phase R1 starts.
- **Scope:** this fork (`GenieRobot/filmcraft-rails`), and the same pattern for every storytold
  craft app ([docs/rails/suite-plan.md](../rails/suite-plan.md)).

## Context

The goal is to ship the craft apps as Rails gems or engines that any Rails 8 app can mount and
restyle, and as a SaaS. The video editor comes first. The UI must be Rails, Hotwire and
ViewComponent, with no React.

Facts measured on 2026-11-01:

- FilmCraft has about 450k lines of Rust in 50 crates. Of that, about 73k is the egui UI and
  the app shells (L5–L6); the rest is codecs, engine and rendering (L0–L4).
- The engine is headless. Everything goes through `Session::execute(id, params) -> Result<Value>`
  with a command registry (`CommandSpec`: id, label, menu path, shortcut, params doc, `enabled`,
  `run`). The CLI, the control channel and MCP all use it, and `cargo xtask layers` keeps egui
  out of L0–L4.
- `apps/filmcraft-web` already runs the engine in the browser on WASM, with WebCodecs,
  AudioWorklet and OPFS.
- All 12 craft app repos have the same shape: a `Session` with a string-id command registry, a
  WASM web app, a CLI and MCP. All are dual-licensed MIT / Apache-2.0.

## Decision

1. **Keep L0–L4 as they are; replace only L5–L6.** We do not rewrite codecs, rendering or edit
   algebra in Ruby. They would be orders of magnitude slower, and the rewrite would repeat about
   375k lines of spec work.
2. **Two runtimes for one engine:**
   - **Server:** a native Ruby extension (`magnus` + `rb-sys`) that wraps `Session`. It handles
     project state, validation, export, proxies, and headless and agent use.
   - **Browser:** a headless WASM build of the engine, without eframe, behind a small JS API.
     Stimulus controllers drive it for real-time preview, scrubbing and playback.

   Frame-accurate playback cannot depend on a server round trip per frame.
3. **The UI is a mountable Rails engine** (`isolate_namespace`) shipped as a gem. It contains
   ViewComponents for panels and dialogs, Stimulus for the interactive surfaces (timeline,
   monitors, docking), and Turbo Streams for state that changes after a command. The engine owns
   no auth, tenancy or billing; it asks the host through a configuration contract (suite plan §4).
4. **Upstream stays mergeable.** New code lives only in `rails/` (gem), `crates/ruby-bridge`,
   `crates/wasm-core` and `docs/rails/`. Changes to L0–L4 go upstream to storytold as pull
   requests (for example structured parameter schemas) rather than living as fork-only patches.
5. **Shared pieces live in one separate gem monorepo** (`craft-rails`, to be created), so every
   craft app reuses them: host contract, workspace shell, UI kit, bridge template, WASM runtime,
   SaaS entitlements and MCP adapter.

## Consequences

The rules in `CLAUDE.md` / `AGENTS.md` inherited from upstream need these fork-local amendments
before phase R1. Each is its own task:

| Upstream rule | Amendment for this fork |
|---|---|
| "Pure Rust in the product" | L0–L4 stay pure Rust. The L5–L6 replacement is Ruby, ERB, CSS and Stimulus JS. No React and no JS bundler: importmap-compatible ES modules only. |
| `unsafe` only in `crates/platform` | `crates/ruby-bridge` uses `magnus` / `rb-sys` and keeps `unsafe_code = "forbid"` in our code. Every Ruby entry point runs under `catch_unwind`, and a panic becomes a Ruby exception. This needs an ADR 0001-style containment section if `unsafe` turns out to be needed. |
| CI only through `storytold/craft-common-actions` | The Rails gem needs Ruby CI: system tests and precompiled native gems for linux x64/arm64, macOS x64/arm64 and Windows. That is fork-local CI, decided in §6. |
| "Never crash", hostile input | Applies to the bridge and to Ruby code too: no `raise` escapes to the user without being rescued and turned into a recoverable UI error. Command params from the browser are hostile. |
| Clean-room; openly licensed assets with attribution | Unchanged, and it extends to icons, CSS and JS in the gem. Every vendored asset gets a `.attribution` sidecar. |

**Upstream sync** becomes a recurring job, estimated at 2–5 agent-hours a week per app. Upstream
lands about 50 commits a day.

## Alternatives rejected

- **Pure Ruby rewrite:** not viable, for the performance and repeated-work reasons in §Decision 1.
- **Server-only rendering** (send frames over the network): no smooth scrubbing, high server
  cost per user. It is kept as a fallback for clients without WebGPU / WASM.
- **Embedding the existing egui WASM canvas in a Rails page:** quick, but the result cannot be
  styled by the host or built from ViewComponents, and has no Hotwire integration. That defeats
  the goal.

## 6. Open items (owner decisions)

1. The name of the shared monorepo, and permission to create it (`GenieRobot/craft-rails`
   proposed).
2. Fork-local CI provider and runners for precompiled native gems.
3. Whether to ask storytold for upstream acceptance of structured parameter schemas and a
   headless `wasm-core` crate, or to carry them in the fork.
4. Product names for the SaaS. Use of the "…Craft" / ArtCraft names commercially has not been
   checked.
