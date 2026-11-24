# 0002. Architecture style: modular monolith

- Status: Accepted
- Date: 2026-11-24

## Context
Tollgate has two main parts, an evaluation harness and an LLM gateway, plus shared storage. It is built by one developer, with a small load and a local demo environment.

## Options considered
1. **Modular monolith.** One codebase and deployable; modules with explicit interfaces and table ownership. Simple to run, test and debug.
2. **Microservices** (harness and gateway as separate services with separate databases). Independent scaling and deployment. Cons: network calls between services, distributed failures, more infrastructure; no need at this scale.
3. **Unstructured monolith.** Fastest to start. Cons: modules become entangled, which makes testing and later changes harder.

## Decision
Modular monolith: one repository, modules `harness`, `gateway` and `shared`, each owning its tables and exposing a public interface. There are two entry points (CLI and gateway server) from the same codebase.

## Consequences
- Easy local setup (one Docker Compose file) and end-to-end tests.
- Boundaries must be kept by discipline: no cross-module table access.
- The gateway can later be extracted into its own service if needed, thanks to the clear boundary.

## How we will know it was right
Modules stay testable in isolation. If the gateway needs independent scaling or deployment, revisit.
