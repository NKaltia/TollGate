# 0003. The harness treats the system under test as a black box over HTTP

- Status: Accepted
- Date: 2026-11-24

## Context
The harness must evaluate the author's RAG assistant now, and potentially other systems later, without depending on their internal code.

## Options considered
1. **Black box over HTTP.** The SUT implements a small contract (`POST /ask`) and the harness only sees inputs and outputs. Cons: the SUT must implement the endpoint and expose retrieved document IDs.
2. **Import the SUT as a library.** Direct function calls, so no HTTP overhead. Cons: tight coupling; the harness would break when the SUT's internals change; it only works for TypeScript systems.

## Decision
Black box over HTTP, through an SUT adapter with the contract defined in the detailed design (§5.5.2).

## Consequences
- The harness is reusable for any system that implements the contract.
- A mock SUT is trivial to build, which makes tests deterministic.
- Network timeouts and errors must be handled (retries, `error` trials).
