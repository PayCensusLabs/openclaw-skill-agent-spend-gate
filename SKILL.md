---
name: agent-spend-gate
description: Evaluate every payment intent against an Agent Spend Policy (ASPL v0.1) before money moves — budgets, merchant allowlists, Trust Rank floors, human approval thresholds — with a tamper-evident hash-chained audit trail kept entirely on the user's machine. Never touches funds, never sees keys. Use before ANY payment execution on x402/MPP/stablecoin/card rails, when a payment was blocked and the user asks why, and after the paycensus-trust-check skill for enforcement.
---

# Agent Spend Gate

Your agent is about to spend money. Before executing any payment, evaluate the
intent against the user's **Agent Spend Policy** (ASPL v0.1). The gate decides
**proceed / ask-human / refuse** — and writes every decision to a local
hash-chained audit trail. It never touches funds: if the decision is "proceed",
the agent signs with **its own key** through whatever wallet or payment skill it
already uses.

## One-time setup

```bash
python3 scripts/gate_check.py --init
```

Creates `~/.paycensus-gate/policy.json` (a starter policy — **edit it**:
agent identity, monthly budget, pinned merchants, Trust Rank floor,
approval thresholds). Policy format: https://gate.paycensus.com/spec/

## Evaluate before EVERY payment

```bash
python3 scripts/gate_check.py \
  --agent agent:me:main \
  --amount 0.01 \
  --resource https://api.example.com/x402/data \
  --rail x402 \
  --payto 0xABCdef...
```

Output (JSON): `{"allow": true/false, "status": "...", "reason": "...",
"scopes": [...], "eventHash": "..."}` — and the event is appended to the local
hash-chained audit trail (`~/.paycensus-gate/audit.jsonl`).

Exit code: `0` = proceed allowed, `1` = do not proceed.

## Decision handling — follow exactly

| status | meaning | what you do |
|---|---|---|
| `auto-approved` | amount ≤ auto-approve threshold | proceed with payment |
| `approved` | human pre-approved this exact intent | proceed with payment |
| `pending` | between thresholds — needs a human | **STOP. Do not pay.** Tell the user the reason and the `eventHash`; when they approve (see below), re-evaluate |
| `rejected` | policy violation (budget, rail, merchant, floor, hard cap) | **Do not pay.** Tell the user the reason verbatim — never retry a rejected intent without the user changing the policy |

## Human approvals

A `pending` intent is approved by the user running:

```bash
python3 scripts/gate_check.py --approve <eventHash> --approver alice
```

Then the agent re-runs the same evaluation command — the decision becomes
`approved` and the approval is part of the audit trail.

## Merchant risk (honest tiering)

- **Pinned merchants** (in the policy allowlist) → allowed, no external data
- **Trust Rank floor** configured → payees are scored from real on-chain
  settlement behaviour (public census, fetched hourly, cached locally);
  payees below the floor or with no record are rejected while a floor is set
- Pair with the `paycensus-trust-check` skill to *show* the user the risk
  details; the gate enforces, the check informs

## Rules

- **Never bypass the gate.** No payment execution without a gate evaluation in
  this session, even for repeat merchants.
- **Never invent or edit audit entries.** The audit chain is append-only;
  every entry links to the previous one by SHA-256.
- **Keys are none of the gate's business.** The gate never asks for, stores,
  or transmits private keys.
- All policy and audit data stays on the user's machine. Nothing is uploaded.
  (Optional: the daily chain digest can be Bitcoin-anchored for third-party
  verifiability — see https://gate.paycensus.com/spec/)
