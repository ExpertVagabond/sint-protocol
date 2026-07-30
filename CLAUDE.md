# SINT Protocol — Agent Guide

SINT is a security enforcement layer for physical AI. It sits between AI agents and the physical world (robots, tool calls, actuators) ensuring every action is authorized, constrained, and audited.

## Quick Commands

```bash
pnpm install          # Install dependencies
pnpm run build        # Build all packages (required before test)
pnpm run test         # Run all tests
pnpm run typecheck    # Type-check without emitting
pnpm run bench        # PolicyGateway performance benchmarks (p50/p99 latency)
pnpm --filter @sint/gate-policy-gateway test   # single package
pnpm --filter @sint/gateway-server dev         # start gateway (port 3000)
```

## Monorepo Layout

```
apps/gateway-server/     → Hono HTTP API (port 3000)
packages/core/           → Shared types, Zod schemas, tier constants
packages/capability-tokens/ → Ed25519 token issuance, delegation, validation
packages/policy-gateway/ → THE choke point: tier assignment, constraints, combos
packages/evidence-ledger/ → SHA-256 hash-chained audit log
packages/bridge-mcp/    → MCP tool call → SINT request mapping
packages/bridge-ros2/   → ROS 2 topic/service → SINT request mapping
packages/persistence/   → Storage interfaces + in-memory implementations
packages/conformance-tests/ → Security regression suite (must pass on every PR)
```

## Architecture Rules

1. **Every action flows through `PolicyGateway.intercept()`** — no bridge adapter, route handler, or service makes authorization decisions independently.
2. **Result<T, E> — never throw.** All fallible operations return `{ ok, value } | { ok, error }` via `ok()`/`err()` helpers from `@sint/core`. Never use try/catch for control flow.
3. **Attenuation only.** Delegated capability tokens can only _reduce_ permissions. Never escalate.
4. **Append-only ledger.** Evidence ledger is INSERT-only, SHA-256 hash-chained. No updates, no deletes.
5. **Interface-first persistence.** Storage adapters implement interfaces from `@sint/persistence`; in-memory implementations for testing.
6. **Circuit breaker fail-open:** if a plugin throws, treat circuit as CLOSED.

## Approval Tiers (T0–T3)

| Tier | Enum | Auto? | When |
|------|------|-------|------|
| T0 | `T0_OBSERVE` | Yes | Read-only (sensors, queries) |
| T1 | `T1_PREPARE` | Yes | Low-impact writes (save waypoint, write file) |
| T2 | `T2_ACT` | No — escalate | Physical state change (move robot, operate gripper) |
| T3 | `T3_COMMIT` | No — human required | Irreversible (exec code, transfer funds, mode change) |

Tier rules: `packages/core/src/constants/tiers.ts`. Key request/decision types: `packages/core/src/` (`SintRequest`, `PolicyDecision`).

## Coding Conventions

- TypeScript strict mode (`noUncheckedIndexedAccess`, `noUnusedLocals`, `noUnusedParameters`)
- ES modules — `"type": "module"` with `.js` extensions in imports
- Readonly-by-default interface fields; Zod at boundaries; Vitest
- **@noble for crypto** (audited, zero-dep); **UUIDv7 for IDs**; ISO 8601 microsecond timestamps

## Adding a New Package

1. `packages/<name>/package.json` with `"name": "@pshkv/<name>"`
2. `tsconfig.json` extending `../../tsconfig.base.json` + `{ "path": "../<dep>" }` references
3. `vitest.config.ts`, `src/index.ts`, then `pnpm install` to link

## Multi-Agent Coordination

Multiple agents/developers may work concurrently. Before starting: `git pull --rebase`, then `pnpm run build` and `pnpm run test` — must be 0 failures before and after your change. Add a conformance test for any new security invariant.

### Known Collision Risks / Gotchas
- `SintDeploymentProfile` exists in `policy.ts` (site profiles); engine version was renamed `SintHardwareDeploymentProfile`. Do not re-add generic names in engine packages.
- requestId MUST be UUID v7 — `crypto.randomUUID()` produces v4 and fails schema validation. Use `generateUUIDv7()` from `@sint/gate-capability-tokens`.
- `CircuitBreakerPlugin.trip()` sets `manualTrip=true`, permanently preventing auto-HALF_OPEN. Tests exercising auto-recovery must open the circuit via `recordDenial`, not `trip()`.

## Status

Phases 1–9 complete (security wedge, bridges incl. MCP/ROS2/MAVLink/IoT, OWASP ASI01–ASI10 conformance, token registry, Python/Rust SDKs). Phase 10 next: npm publish, Constraint Language CL-1.0, sintctl registry CLI, Show HN. Check `git log --oneline -10` for what landed recently.
