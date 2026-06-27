# Universe Simulation — MVP Plan (v0.8)

> Status: living document. The architecture, seams, and ADRs live in **architecture.md**;
> this document is only the **build sequence**. Optimizes for the walking-skeleton goal:
> all parts present, minimal functionality each, every advanced feature wired-but-inert.

---

## Shape of the MVP

A **walking skeleton**: 1 object, 1 agent, one end-to-end thread (engine → API → client dot;
agent reads own-state + observation; admin start/stop), built so every advanced feature is present
as a **wired-but-inert seam**. The whole rollback / GVT / optimism cluster is **named, stubbed,
deferred** — in the MVP a single in-order worker over one object is conservative, so no rollback ever
fires (see architecture.md §5.4).

---

## Phase 0 — Foundations (start here)

**Exit criterion:** `docker compose up` boots the stack and every component completes a "hello" over
the real API; CI green; the same stack runs on the box under systemd.

- **Slice 0 — Dev environment & repo.** *(done)* rustup (+ clippy, rustfmt), `protoc`, Docker +
  compose, git, editor + rust-analyzer; hosted repo created; `rust-toolchain.toml` pinned (1.96.0)
  + `.gitignore`; smoke build verified.

- **Slice 1 — Repo & Cargo workspace.** *(next — Rust-structure-rich, code-light)* Monorepo +
  virtual workspace manifest (`resolver`, `[workspace.package]`, `[workspace.dependencies]`,
  edition 2024). Crate stubs under `crates/` matching the seams — `contract`, `engine`, `registry`,
  `gateway` (lib) and `server` (bin) — plus sibling dirs `proto/`, `agent-sdk/`, `web/`, `deploy/`,
  `.github/`. Declare the architecture edges now (even with empty stubs) so `cargo tree` reflects
  the seams: `server → gateway → {contract, engine, registry}`, `engine → registry`,
  `server → {engine, registry}`. **engine/registry must list no tokio/tonic/prost** (ADR-24).
  *Done: `cargo build` clean; `cargo tree` matches the crate graph; no cycles.*

- **Thin CI (right after Slice 1).** GitHub Actions: build + `fmt` + `clippy` on push, so every later
  slice is gated green. *Done: CI passes on push.*

- **Slice 2 — Contract v0 (`universe.proto`).** The architecture.md §6 API as real protobuf
  (`package universe.v0;`), request/response wrappers, `oneof Value` (+ tiny vector/list wrappers).
  Reflects the time decisions: per-object `local_time`, **no global tick**, `Invalidate { from_local_time }`,
  `Step(n)` = process N events. Rust codegen via `tonic-build`; Python via `grpcio-tools`; generated
  code not committed. *Done: proto compiles; Rust + Python stubs generate.*

- **Slice 3 — Co-located engine+API binary boots.** `server` (tokio): tonic serving a trivial `Hello`
  + empty `ListObjectTypes`; `/metrics` heartbeat on a separate HTTP port; JSON logs (`tracing`);
  env config. *Optional dev-only* observability profile (Prometheus + Grafana on the laptop).
  *Done: client gets a hello; `curl /metrics` ticks; logs are JSON.*

- **Slice 4 — Type-registry + columnar store (de-risk spike).** Manifest (RON/TOML) → one
  `ObjectTypeSpec`; registry answers `Describe`/`ListObjectTypes`; **hand-rolled columnar store**
  allocates packed columns (Scalar + Vector) and round-trips one object; `SetObjectParams` validator.
  *Done: registry serves one type; store round-trips an object; a write to a `ReadOnly` field is rejected.*

- **Slice 4b — State export (CSV/JSON).** Small `export` bin: load manifest → build store → write
  columns; CSV (vector flattens to `x,y,z…`) with schema-derived headers (name + unit); JSON for
  later nesting. *Done: `export --format csv` opens cleanly in a spreadsheet / pandas.*

- **Slice 5 — Seam stubs as no-ops.** Traits wired but inert: `TimeRatePolicy`→1.0,
  `SkipPredictPolicy`→compute, `StragglerDetector`→None, `NoGoodStore`→no-op, `Checkpoint`→trivial,
  plus the scheduler's `enqueue`/event channel and a scope-aware `rollback(affected_set, to_time)`
  that never fires. *Done: the scheduler constructs and drains with no-op seams; all compile; none do work.*

- **Slice 6 — Python agent-SDK skeleton.** Small generic lib (`uv`-managed): connect,
  `Describe`/`ListObjectTypes`, split properties (readable) / commands (writable), complete hello.
  (Dynamic turtle methods + streaming arrive in Phase 1.) *Done: SDK connects to the Slice-3 server
  and prints the type's fields.*

- **Slice 7 — Workbench shell.** `tonic-web` on the gateway (gRPC-web, no proxy; gateway also serves
  the static bundle); a minimal TS page (Chrome/WebGPU target) that connects and shows "connected" +
  the type. No rendering. *Done: page reaches the gateway over gRPC-web and shows connected.*

- **Slice 8 — Compose + CI + end-to-end hello + first deploy.** Multi-stage Rust Dockerfile
  (+ `cargo-chef`), agent image, one `docker-compose.yml` (co-located server [serving web] + agent);
  CI expands to build/push the image to **GHCR** and run the stack to assert the hello; server-prep
  (ssh-keys, non-root user, `ufw`, Docker, a **systemd** unit running `compose up -d`) + scripted
  `compose pull && up`. *Done — Phase-0 exit: local `docker compose up` boots and the hello completes
  end-to-end; CI green; the stack runs on the box under systemd.*

---

## Phase 1+ — after the skeleton boots

- **Phase 1 — Walking skeleton.** The scheduler advances **1 object** (single in-order worker, the
  object re-inserting itself each step); the stream carries its position; the frame renders **one dot**
  (WebGPU; wgpu→WASM if you want the renderer in Rust); the agent reads own-state + observation via the
  SDK; admin start/stop + "alive" metric; save/load that object.

- **Phase 2 — Harden the seams.** Real `LocalClock` (proper time + rate); the controller path
  affecting the engine (present-tense `SetObjectParams`); initial-state editor; dynamic turtle methods
  in the SDK.

- **Phase 3 — Optimization pass** (institutionalize the 30%). Build caching, benchmark harness +
  baseline algorithm, image slimming, CI time; release-cycle docs, runbook, alert rules. This is also
  where the deferred engine cleverness begins to land: the ordering coordinate (v1), bounded optimism /
  graceful backpressure, and real GVT-driven fossil collection.

---

## Dev-tooling requirements map

1. **Developer room** → monorepo + Cargo workspace + Compose + CI/CD (slices 0–2, 8).
2. **Tester** → a reproducible test universe (single-threaded replay mode), integration harness
   (spin engine+API, assert), saved API "query collections" (grpcurl / Bruno), and the CSV/JSON export
   for offline analysis.
3. **Admin** → lean: `/metrics` + JSON logs on the box, a dev-only Grafana stack, offloaded dashboards
   in prod; control API.
4. **Production + release cycle** → GHCR images, systemd, scripted pull-deploy, rollback runbook, ADRs.
