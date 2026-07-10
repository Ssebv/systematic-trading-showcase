# Systematic Trading Research Platform

*🇪🇸 [Versión en español](README.es.md)*

> Multi-strategy algorithmic trading infrastructure built to **learn the truth about an edge
> before risking capital** — not to show off a vanity P&L.


---

## TL;DR

I designed and operated, end to end, a systematic trading system (12 strategies across crypto +
prediction markets, 20 processes on a VPS) that trades against real venues in paper mode to
**validate executable edge before committing money**. The emphasis is not "a bot that makes
money," but the **rigor infrastructure**: multi-phase backtesting, execution simulation with real
slippage, layered risk management, and statistical discipline to **kill, with data, the strategies
that don't survive costs**.

**The punchline:** most retail edges get competed away at retail scale. I built the system that let
me *prove it with data* and act on it — harvest the one robust edge, and stop investing in the ones
that don't pay. Intellectual discipline over vanity P&L.

---

## Architecture

```
                         ┌───────────────────────────────┐
                         │        VPS (Docker, hardened)  │
                         │   read-only · non-root · CI    │
                         └───────────────┬───────────────┘
                                         │ supervisord (20 processes)
        ┌────────────────────┬───────────┼────────────────────┬────────────────────┐
        ▼                    ▼            ▼                    ▼                    ▼
 ┌────────────┐      ┌────────────┐ ┌──────────┐      ┌──────────────┐    ┌──────────────┐
 │  Crypto    │      │ Prediction │ │  Macro   │      │  RISK layer   │    │ Observability │
 │ strategies │      │  markets   │ │  context │      │               │    │  + alerting   │
 │ (spot/fut) │      │ (wallet    │ │ (radar)  │      │ circuit break.│    │ real-time     │
 └─────┬──────┘      │ copy-trade)│ └──────────┘      │ forward-guard │    │  dashboard    │
       │             └─────┬──────┘                   │ (auto-retire) │    │ 31 watchers   │
       │                   │                          │ sizing/gates  │    │ → Telegram    │
       │                   │                          │ daily CB      │    │ auto-auditor  │
       ▼                   ▼                          └───────┬───────┘    └──────┬────────┘
 ┌──────────────────────────────┐                            │                   │
 │  Decision engine + ML         │◄───────────────────────────┴───────────────────┘
 │  (calibration, winrate-opt)   │         feedback loop: every decision measured,
 └──────────────┬───────────────┘         audited, and compared vs forward data
                │
                ▼
 ┌──────────────────────────────┐
 │  Multi-phase backtester       │
 │  filter-replay · Monte Carlo  │
 │  param-sweep · OHLCV engine    │
 └──────────────────────────────┘
```

<!-- screenshot pendiente: real-time dashboard — paste a screenshot of the status panel here -->

---

## Engineering highlights

- **Forward-guard with auto-retire** — strategies that fail their deadline + profit-factor retire
  themselves (weight → 0) with no manual intervention. Risk design that applies to itself.
- **Live-shadow simulator** — measures *executable* edge by simulating real fills (slippage against
  the order book, fees, gas) on live signals, instead of the theoretical edge that deceives.
- **Counterfactual analysis** — when risk gates blocked trades, I replayed the blocked signals
  against real market prices and **proved that taking them would have lost −339% over 5 days**
  (186 of 204 trades losing). *I quantified that "not trading" was the correct decision.*
- **Production systems debugging** — (1) an OOM-loop where the kernel repeatedly SIGKILL'd a worker
  loading a 946 MB model → env-gated it to a lightweight scorer, RAM 87%→63%; (2) a process the
  orchestrator reported as *"healthy"* but which had been hung for 7 days after an uncaught
  exception, caught by inspecting `/proc` (0s CPU, no heartbeat).
- **Research discipline** — killed candidates with data: market-making (0.1¢ spreads, venue already
  efficient), mean-reversion (edge was an upper bound, not executable), CLV (contaminated sample).
- **Capital-allocation bug in production** — two auto-retired strategies silently held **62.7% of
  the capital split** they could never use: still members of the rebalancer's universe, plus a stale
  manual override the persist layer honored even though the risk layer hard-blocked them. Found by
  cross-checking the allocation state against the risk layer — two sources of truth nobody compared.
  Fix returned the survivor strategy its share (26%→69%). *Audit the money path, not just the code path.*
- **Monitor redesigned as a bidirectional state machine** — a watcher alerted "alpha source went
  quiet" then self-disabled to avoid spam. Correct — until the source *came back* and the actionable
  signal (resume harvesting) went unnoticed for 4 days. Redesigned to watch both transitions
  (quiet→alert→standby→revival→re-arm). *A monitor for a recoverable condition can't be one-shot.*
- **Test the failure path before wiring the alert** — a new on-chain reconciliation cron's first
  firing revealed that "not configured yet" (expected state) fell into the same alert path as "RPC
  broken" (real failure) → 4 noise pings/day. Fixed same day: silent on the expected state, loud on
  the real one. *Simulate the system's current state against the script before scheduling it.*
- **Adversarial multi-agent code review** — ran a 23-agent review harness (4 independent review
  lenses + skeptic verifiers that must *unanimously fail to refute* a finding before it's reported)
  over an already hand-verified change batch: 16 raw findings → 9 confirmed real (6 false positives
  filtered before reaching a human). Among the confirmed: a deploy pattern whose unconditional
  `git stash pop` could silently regress production data files, and a health panel that could never
  report "dead" because `pgrep -f` matched its own shell wrapper. *Redundancy finds bugs; adversarial
  verification keeps them honest.*
- **Health probes must measure success, not activity** — the dashboard builder's liveness probe used
  its log's mtime; since cron piped stderr to the same log, a crash-looping builder refreshed its own
  heartbeat every minute and stayed "healthy" while the output froze. Moved the probe to the built
  artifact, which only updates on success. *A component that can write its own heartbeat while
  failing will.*
- **Counterfactuals lie unless you make them executable** — the nightly "missed wins" audit scored
  blocked signals with intrabar-wick fills, zero costs, and retired strategies included: a structural
  upper bound that read as regret. Rebuilt the defaults around executable fills (bar-close triggers),
  measured round-trip costs, and live strategies only — the honest number *still* vindicated the risk
  gates (0% win rate on recent rejections), but now it's a number you can act on. *An optimistic
  counterfactual is a standing invitation to loosen the wrong gate.*

<!-- screenshot pendiente: research / strategy registry panel — optional -->

---

## Stack

`Python` · `Docker (multi-stage, hardened)` · `supervisord` · `Linux/VPS` · `Tailscale` ·
`launchd/cron` · `CI` · `pandas/numpy` · `ML (calibration, backtesting)` · `SQLite/JSON ledgers` ·
`web dashboard (Chart.js, design tokens)` · `Telegram alerting`

---

## What it demonstrates (mapped to roles)

| Capability | Role that values it |
|---|---|
| Production Python + systems architecture | Backend / Platform Engineering |
| Market microstructure, slippage, sizing, risk | Quant Dev / Trading Systems |
| Applied ML + calibration + backtesting | Quant Research / ML Engineering |
| Docker / infra / observability / CI | DevOps / SRE / Platform |
| **Statistical rigor + intellectual honesty** | **All of them** — the rarest one |

---

## What I learned

- Retail edges get competed away (fractions-of-a-cent spreads, alpha sources that dry up).
- **Activity ≠ edge**: a system that trades "always" loses; good ones sit still without an advantage.
- Validate-before-building saves weeks; killing your own idea with data is a skill.
- **Knowing when to stop**: once the bottleneck is no longer technical, continuing to tinker
  destroys value. Recognizing that is engineering judgment.

---

*Source code is private (it handles credentials and proprietary strategies). This writeup and the
screenshots are the public evidence. Happy to give a technical walkthrough in a conversation.*
