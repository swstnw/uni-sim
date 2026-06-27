# Project Summary — Universe Simulation

## Concept
A simulation that models a universe as many *n*-dimensional objects, each living in its own
**relativistic local time**. A Rust **engine** computes the world **optimistically** — an
event-driven scheduler that cuts corners and predicts — and corrects itself by **rewinding** when a
paradox (a causal/straggler violation) is detected. The world is observed and gently steered by
**agents**, which are *study instruments* (not players) bound one-to-one to objects through a single
API. A **web workbench** (IDE + sandbox + an *n*-D visualization frame) is the human interface.

## Philosophy & constraints
- **One small server** (2c / 4 GB / 60 GB), kept deliberately minimal as a project challenge;
  ~30% of effort goes to optimization (engine algorithms *and* build/ops tooling).
- **Architecture-first, maximum room to rework.** Swap-freedom comes from a single versioned API
  contract, *not* from process boundaries.
- **KISS for the MVP, but always leave a clean seam** for deferred richness.

## Key decisions
- **Contract-first.** Agents, client, editor, and admin are all clients of one versioned
  Observer/Controller API (protobuf, gRPC/tonic, `universe.v0`); the engine's internal model is kept
  strictly separate from the wire model.
- **Engine = an event-driven scheduler.** The heart is a **pending event set** ordered by each
  object's next-update local time; workers pop the most-due entry, process it, and the object
  re-inserts itself (a lone body still integrates its own trajectory); interactions are events into the
  same set. Fixed-timestep is just the trivial scheduling policy. Renamed from "core" to avoid the
  Rust `core` std-crate collision.
- **No global clock — three times kept separate.** Per-object **proper time** (`LocalClock {proper_time,
  rate}`, relativistic dilation, state not a driver); a **coordinate/"universe" time** as the common
  ordering axis (the hard one, deferred to v1); and **GVT** = the front of the pending set, the
  commit/reclaim frontier (executor bookkeeping, deferred — trivial in the MVP).
- **Optimistic speculative execution.** A single in-order worker is conservative and never rolls back;
  rollback is the price of going **out of order** (parallel or speculative). On a detected causal
  violation: a **localized rollback** over a causal graph (Time-Warp / optimistic-PDES) and a **guided
  re-derive** informed by a learned "no-good" summary (CDCL). Strong determinism is deferred; the model
  is authoritative, so a recompute need not reproduce the past. Unlimited rewind = unlimited *reach*
  (GVT-driven fossil collection + re-sim), not unlimited storage.
- **Agents.** Frame-local view only (own-state + observation, indexed by local time), no god's-eye
  world; one agent ⇄ one object via the object API with a turtle-graphics Python SDK; run as
  resource-limited subprocesses. **Present-tense control:** `SetObjectParams` applies at the object's
  current local time, so control is never a straggler. A rewindable instrument needs no commit signal.
- **Object model.** A **data-driven type registry over dense columnar storage** (fixed primitive value
  set; units as schema metadata; optional sparse fields) — flexibility in metadata, density in storage.
- **Regions.** A hierarchical region tree (Barnes–Hut / AMR) with weak far-field coupling, and
  per-region scale-aware checkpoints. The same structure bounds rollback *and* enables future
  multi-server sharding; the MVP's single pending set is the one-region base case of this tree.
- **Client.** No native app; a browser workbench with **all** visualization (projection, geometry,
  skin) client-side; the engine emits only raw physical data. Chrome/Chromium only → WebGPU.
- **Engine boundary.** A **synchronous in-process port** (enqueue command/event + outbound event
  channel). The gateway owns the gRPC socket and maps wire ⇄ internal; the engine and registry depend
  on no async runtime or transport (no tokio/tonic/prost).
- **Ops (lean for 4 GB).** Monorepo + Cargo workspace, co-located single binary, embedded DB, CI and
  dashboards offloaded.

## Documents
- **architecture.md** — decisions, seams, engine internals, the contract sketch, and the ADR backlog.
- **mvp-plan.md** — the slice-by-slice build sequence (Phase 0 → Phase 3).

## MVP shape
All parts present, minimum functionality — 1 object, 1 agent, one end-to-end thread (engine → API →
client dot; agent reads; admin start/stop) — built as a walking skeleton, with every advanced feature
(rollback, GVT, skip/predict, regions, the no-good store) present as a wired-but-inert seam.
