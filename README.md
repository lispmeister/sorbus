# Agent Payments via L402, Lightning (LND), and Fedimint

A service architecture that lets AI agents pay for L402-gated APIs without
holding any keys. The user holds long-term spending authority in a Fedimint
wallet on their phone. When the user approves a session, e-cash moves from
the phone into custody at the service for the lifetime of that session; the
agent then spends within that session budget automatically. External APIs
see a single anonymous payment node. Internally, every satoshi is attributed
to the user, session, and agent that spent it.

## The Problem

An AI agent acting on a user's behalf needs to call APIs that charge per
request via [L402](https://github.com/lightninglabs/L402) (HTTP 402 +
Lightning invoice + macaroon). The agent should:

- Never hold wallet keys or long-term e-cash.
- Spend only what the user has explicitly budgeted.
- Operate autonomously within that budget — no per-call phone taps.
- Explain itself when it needs a new or larger budget.
- Let the user revoke access instantly from their phone.
- Be anonymous to every external API it calls.

## Technology Choices

| Layer | Choice | Reason |
|---|---|---|
| Payment protocol | **L402** (Lightning Labs) | Standard HTTP 402 + bolt11 + macaroon; stateless verification; credential reuse; macaroon caveats for scoped access |
| Lightning node | **LND** | Mature Fedimint gateway integration; gRPC `RouterSendPaymentV2` returns the preimage directly; advanced payment routing |
| E-cash / federation | **Fedimint** (operator-run) | Shared custody, privacy, Lightning bridge; compatible path to independent guardians in future versions |
| Phone wallet | **Companion app** (MVP) | Fastest path to a demo; push via APNs/FCM; deep-link for session creation and budget approvals |

## Design Rationale

1. Fedimint gateways already pool Lightning channel liquidity and bridge e-cash ↔ Lightning. Multi-user channel sharing is what gateways do; this design adds a session-and-policy layer on top.
2. L402's preimage-as-credential model maps cleanly onto the flow: LND pays the bolt11, returns the preimage, Coordinator hands `macaroon:preimage` to the agent as its Authorization credential. The agent reuses it until the API server revokes it.
3. L402 credentials are **reusable** — the agent pays once per tier/access grant, not per call. Most API calls consume no sats and never touch the Coordinator.
4. Running LND as a shared service lets the operator centralize channel management, routing-fee optimization, and capital allocation. Individual users never need to think about channels, liquidity, or routing fees.

## Full Specification

The complete protocol and service specification — trust model, system components, API surface, state machines, error codes, end-to-end flows, operating guidance, and open questions — is in [`spec/SORBUS-SPECIFICATION.md`](spec/SORBUS-SPECIFICATION.md).
