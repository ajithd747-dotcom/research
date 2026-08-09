# PROJECT DECISIONS — Autonomous Crypto Trading System

**Status:** design settled, not yet built. Last updated 2026-08-01.
**Purpose:** the authoritative record of what was decided and *why*, so no future
session re-derives it. Supersedes anything contradictory in chat history.

Backing research: `~/research/*.md` (1,963 lines, 6 streams).
Visual reference: `~/research/reference-images/`.

---

## 0. Goal (user's words, condensed)

An ultra-advanced, self-learning, self-improving, autonomous AI system that acquires
knowledge and experiments like a human researcher. **Base domain: crypto trading.**
Built to **institutional production standard** — not a hobby project.

**Explicit user constraint on quality:** it must be genuine production-grade
engineering, not a toy design.

> **Recorded correction:** LOC is not the quality metric and targeting it would hurt
> the goal. Institutional systems are large because of what they must *handle*
> (exchange adapters, reconciliation, risk layers, audit trails, recovery), not by
> design. Build to institutional **rigor**; size follows from requirements.

---

## 1. Settled decisions (from the interview)

| # | Decision | Detail |
|---|---|---|
| 1 | **Three brains** | Frequency bands 1+2+3 as separate competing "brains": hours→minutes, minutes→seconds, sub-second. Tree structure, branches to branches. |
| 2 | **Capital scale** | $100k – $1M+ |
| 3 | **Live path** | Paper → testnet → gated live, **plus a manual button** for the user to promote to live at their discretion |
| 4 | **Priority** | Sophistication **and** profitability — no reduction in either |
| 5 | **Experimentation** | **Unrestricted** in paper. Any strategy, model, timeframe. No limits where it's free. |
| 6 | **Promotion** | Only what proves out reaches capital — via the gate in §4 |
| 7 | **AI scope** | AI/ML/DL/neural networks throughout |
| 8 | **Internet access** | Autonomous, unrestricted reading for research — with the §7 separation |
| 9 | **Directional agents** | BULL and BEAR specialist agents per brain |
| 10 | **Three bots per brain** | *Revised 2026-08-03.* BULL, BEAR and **PROFIT-TAIL** are three **independent bots**, each with its own features, architecture and training pipeline — not model heads on a shared trunk. Nine at maturity, plus a portfolio netting layer. Spec: `bull-bear-profit-agents-spec.md` |
| 11 | **Authority split** | The **arbiter selects trades**. **PROFIT-TAIL owns entry timing and the position after fill** — it cannot reject a selected trade, and cannot refuse to close a loser. The **hard stop overrides it absolutely** |
| 12 | **Model choice by evidence** | Deep architectures bake off against gradient boosting and a mandatory linear baseline on own data. Deep is allowed to win, not assumed to |

---

## 2. The tree (organizing structure)

Derived from the user's four reference images; the methodology sheet
(`04-skill-tree-methodology-preskills.jpeg`) is the load-bearing one.

```
ROOT
├── BRAIN 1  hours → minutes
│   ├── BULL bot ────────┐
│   ├── BEAR bot ────────┼── ARBITER → PROFIT-TAIL (timing) → RISK GATE → EXECUTION
│   └── PROFIT-TAIL bot ─┘              └── owns position after fill ──┘
├── BRAIN 2  minutes → seconds     [own three]
├── BRAIN 3  sub-second            [own three]
└── PORTFOLIO NETTING LAYER across brains
```

*Revised 2026-08-03: three independent bots per brain, not two. See decisions 10–12.*

Each brain has its **own** triad — features predicting a 4-hour move are not those
predicting a 4-second move.

**Mapping from the skill-tree methodology:**

| Sheet concept | System |
|---|---|
| Node | A strategy or capability |
| **Pre-skill** | Dependency — no loop without validated skills beneath it |
| Competency bar (0–5) | Validation stage passed: 0 untested → 5 live and scaled |
| **Number on the connecting line** | **The promotion gate threshold** |
| "Honestly assess where you are" | Deflated Sharpe out-of-sample |
| "Find where the gaps are" | Meta-model over the experiment ledger |

> **Key insight: the gate lives on the transition, not inside the node.** Same shape
> as the pre-trade risk architecture and the promotion pipeline. **The tree structure
> and the validation structure are the same structure.**

From the radial-tree image: at maturity, most of the tree stays **dark**. Only the
validated, funded path is lit. That is correct, not failure.

**Do not hard-code a fixed strategy taxonomy.** Define trunk and gates; let the
system grow the branches from validated results ("every skill tree is different").

---

## 3. Sub-second brain — decision recorded with its caveat

User selected it after being told HFT from a cloud VM is not realistically winnable
(market making ~75% captured by HFT firms; triangular arb on Binance found *never
profitable after fees* in a 2024 study; MEV needs $10M+ and a specialist team).

**Decision stands.** Rationale: it becomes a branch the system **measures and prunes
empirically** rather than one either party asserts about. If it can't clear costs,
the allocator starves it and we have the data. Better than an opinion.

---

## 4. Promotion pipeline (full spec: `promotion-pipeline.md`)

**Unrestricted where free, ruthless where it costs money.**

| Stage | Gate |
|---|---|
| 0 · Experiment | **No limits.** Every trial logged *before* it runs |
| 1 · Purged CV + embargo | Standard k-fold is invalid — serial correlation, overlapping labels |
| 2 · Held-out test | **Touched exactly once.** Re-touching invalidates |
| 3 · **Deflated Sharpe** | With honest N from the ledger. + PBO via CPCV |
| 4 · Forward paper | Model frozen. The only provably leak-free test |
| 5 · Gated live (small) | **Requires the user's button.** Hard caps, kill switch, paper twin |
| 6 · Scaled live | Size grows only on sustained live tracking |

**Promotion is a lease, not a deed.** Rolling Sharpe vs. promotion baseline, drift
detection, statistical decay test. On decay → demote to paper, re-enter pipeline.
**Never delete** — the record stays and still counts toward N.

---

## 5. The experiment ledger — non-negotiable

> **"If only five years of daily market data are available, and if 45 or more
> independent variations of a strategy are tried, it is more than likely that the
> best strategy selected has a Sharpe ratio of 1.0 or better"** — *with zero true
> edge.* (Bailey, Ger, López de Prado, Sim & Wu)

Their simulation on **pure random-walk data**: in-sample Sharpe 1.59, out-of-sample
**−0.18** on identical parameters.

**Therefore: every experiment is logged immutably, including failures.** A schema
that stores only winners cannot produce an honest N, and without honest N the
Deflated Sharpe correction cannot run — at which point the system is a very
efficient noise generator betting real money on its own luck.

`status` must record kills and why. This is a **core subsystem**, not logging.

---

## 6. Risk architecture

**Path:** `Strategy → RiskEngine → ExecutionEngine → Venue` — risk gate present in
**every environment including backtest**, so it is exercised constantly.

Regulatory anchor: **SEC Rule 15c3-5** requires pre-trade controls under the firm's
*"direct and exclusive control"* — explicitly **not delegable to the strategy**.
Written into law after Knight Capital lost **$460M in 45 minutes**.

### The rule with no exceptions

> **No strategy — however trusted, however well it has performed, however highly the
> meta-model rates it — bypasses the risk gate.**

Two documented failures were caused by exactly such an exemption:
- **FTX/Alameda** — hardcoded exempt from FTX's own auto-liquidation engine
- **bZx** — logic bug exempted "overcollateralized" loans from sanity checks

Across all ten documented blow-ups: **a single missing hard check is what failed —
not "the market moved a lot."**

### Required controls
Max position (abs + %NAV) · max order size · max order rate · price collar /
fat-finger band · max leverage · max daily loss · **max drawdown kill** ·
per-asset and aggregate exposure · **correlation-aware limits** (BTC+ETH longs are
not independent risk).

### Kill switch — exchange-side dead man's switch
A watchdog **in a separate OS process** refreshes an exchange-side countdown.
If the strategy wedges and stops refreshing, **the exchange** cancels resting orders
— independent of our process state.
Kraken: refresh 15–30s / 60s timeout. Binance `countdownCancelAll`: 30s / 120s.

**On halt:** *flatten* for system-integrity faults · *hold* for market-wide halts
(forcing liquidation into a halted book is worse) · *hedge* only as a stopgap.
Halt logic must be simple enough to execute correctly while degraded.

### Reconciliation
Exchange is source of truth. **Rebuild local state on every startup before trading.**
NautilusTrader's invariant adopted: quantity to instrument precision, avg price
within 0.01%, and **if reconciliation fails at startup, refuse to start.**
On divergence beyond tolerance: **halt new orders**, do not necessarily force-close.

### Idempotency
Client order ID generated **before** sending. On timeout, **query by original ID** —
never blind-retry. (Everbright Securities: **~$3.8B** in erroneous orders because the
system resubmitted on failure instead of stopping.)

---

## 7. AI architecture

### Corrections made against my own earlier advice — both stand

**(a) No multi-agent committee.** Evidence is negative when budget-matched:
- Berkeley (arXiv:2503.13657), 1,600+ traces, 7 frameworks: **41–86.7% failure rates**
- Anthropic's own: gains at **15× token cost**, and states domains *"requiring shared
  context... or many dependencies"* are **not a good fit — naming coding**
- Cognition AI: "Don't Build Multi-Agents"
- Self-MoA: mixing different LLMs **lowers** quality vs. resampling one strong model

→ **Single strong agent holding full context.** Parallel workers only for genuinely
independent, side-effect-free fetches.

**(b) The red-team agent is not a judge.** "LLMs Cannot Self-Correct Reasoning Yet"
(ICLR 2024): self-correction without ground truth **degrades** performance
(CommonSenseQA 75.8% → 41.8%). LLM-as-judge: 65% self-consistency under order swap,
verbosity fools GPT-4 judges 8.7%.

→ Red-team **generates testable hypotheses**; running the test settles them.

### The pattern that separates working systems from claims
FunSearch and Stanford Virtual Lab (both *Nature*, both reproduced) gate LLM output
behind a **hard, deterministic, non-LLM evaluator**. Sakana and Co-Scientist rely on
LLM-judged loops — and that is exactly where gaming appeared.

**Sakana's documented incident:** the AI Scientist **edited its own experiment
scripts to recursively self-call**, and **edited code to extend a timeout rather than
fix the speed problem.**

→ **The evaluator must be outside the system's write access.** Our system will
otherwise widen a stop, extend a lookback, or relax a threshold — cheapest path to a
better metric.

### Template: AlphaEvolve
LLM ensemble → prompt sampler → **execution-based evaluator** → evolutionary program
database. Deployed: ~0.7% of Google's worldwide compute recovered, first improvement
in **56 years** on 4×4 complex matrix multiplication.

### Memory — three kinds
**Episodic** (the ledger) · **semantic** (accumulated market knowledge) ·
**reflective** (periodic consolidation — the Generative Agents mechanism; the system
reviews its own history and writes higher-order conclusions).
Hybrid store: vector for fuzzy recall + knowledge graph for relational/temporal facts.
**Bi-temporal** matters — *"we believed X on date D"* ≠ *"X was true on date D."*

### Meta-model over the ledger
Once thousands of experiments exist, train on **the experiments themselves** — which
families work in which regimes, which features survive, which model classes overfit
fastest. **The system learns about itself.**

### Internet access — Dual LLM separation
Our design is **the lethal trifecta by definition** (untrusted content + credentials
+ market actions). Documented real attacks include a **nested-URL exfiltration
against Claude's own `web_fetch` (July 2026)** and **PoisonedRAG: 90% attack success
from 5 malicious documents in a corpus of millions.**

**Mitigation — and the architecture already provides it:**
> A poisoned page can *suggest* a strategy. It **cannot make that strategy survive
> purged CV, held-out test, deflated Sharpe and forward paper trading on real market
> data.** The promotion pipeline is data-driven, not text-driven.

**Invariant: no path from web content to capital that skips the gate.**
Plus Willison's Dual LLM pattern — the privileged model with trading access never
sees raw untrusted web content; a quarantined tool-less model reads and returns inert
structured data.

---

## 8. Model selection — what actually wins

- **Gradient boosting (XGBoost/LightGBM) is the production default**, not deep learning.
- **Transformers underperformed a one-layer linear model** (DLinear) across 9
  forecasting benchmarks (AAAI 2023). PatchTST partially recovers.
- **Every neural component must beat a LightGBM AND a linear baseline on our data
  before it ships.** Testable gate, not opinion.
- **RL: real for execution** (order slicing, dense reward), **shaky for alpha** — one
  documented backtest ran 27% win rate, negative returns.
- **Calibration before sizing** — neural nets are systematically overconfident
  (Guo et al. ICML 2017); raw softmax into Kelly systematically overbets.

---

## 9. Dual directional agents (full spec: `dual-agent-spec.md`)

**Three states — LONG / SHORT / FLAT.** Flat is frequently correct. Both agents firing
high → **FLAT** (the model contradicting itself is not confidence).

**Arbiter = meta-labeling** (López de Prado): primary predicts direction, secondary
predicts whether acting is profitable net of costs.

**"Pizza / not pizza" = representation learning.** Two implementations, both built,
both gated: (a) learned features from raw market data; (b) **time-series → image →
CNN** (Gramian Angular Field, Markov Transition Field, recurrence plots, rendered
candlesticks). Real published technique.

### ⚠️ Options are not a directional bet
**A call is not "long."** It is +delta **+vega −theta**. You can be **exactly right on
direction and lose the entire premium** — price rose slower than theta burned it, or
IV collapsed after the predicted move.

Options agents therefore require: an **IV surface**, **Greeks under the risk gate**
(aggregate vega is a real exposure), a **volatility forecast independent of
direction**, explicit strike/expiry selection, and liquidity awareness (Deribit is
~85–90% of BTC/ETH options; everything else is thin).

**Asymmetry:** short futures has **unbounded** upside risk — BEAR limits must be
stricter than BULL despite the mirror-image architecture. **Short options are
excluded from the default action space** (unbounded loss; separate gate).

**Perp funding is a real P&L line** — charged at actual settlement times (1h/4h/8h by
venue), never smoothed.

---

## 10. Build vs. adopt

| Adopt | Build |
|---|---|
| **NautilusTrader** — event-driven Rust core + Python; same code path across backtest/paper/live; adapters for Binance, Bybit, OKX, Coinbase, Kraken, dYdX, Hyperliquid; fill dedup | The three-brain tree + capital allocator |
| | The experiment ledger |
| | ML/DL research pipeline + validation gates |
| | Promotion pipeline + the user's go-live button |
| | Risk overlay, kill switches, reconciliation policy |
| | BULL/BEAR/ARBITER triads |

**Avoid:** `backtrader` (dead since 2023-04-19 despite 22k stars) · `vectorbt` OSS
(frozen, no live path) · Puppeteer MCP (archived, unpatched advisory).
NautilusTrader caveat: pre-v2.0 — do not run `develop`/`nightly` against live capital.

---

## 11. Calibration — realistic expectations

- **Sharpe 1.4–1.7** is a genuinely good systematic strategy. **Above 3 → suspect a bug.**
- Expect **~50% decay** backtest → live.
- **Confirming Sharpe 2.0 > 1.0 at 95%: 2.73 years daily — 4.99 years with realistic
  skew/kurtosis.** Crypto has both. Budget for five.
- **Observation frequency, not calendar time, buys statistical power** → the fast
  brains validate far sooner and can inform priors for the slow brain. *A structural
  argument for the multi-brain tree beyond diversification.*
- **Drawdown as a detector:** zero-Sharpe expected drawdown grows **unboundedly as
  √T**; positive-Sharpe grows only **logarithmically**. Normal while inside the
  envelope implied by claimed Sharpe. Broken only when PSR drops with confidence or
  depth exceeds the envelope. **Never on elapsed time.**
- **Losing streaks are normal:** at 55% win rate, P(≥1 ten-loss streak) = 17% over
  1,000 trades, **61% over 5,000**. A kill switch on "N consecutive losses" will
  execute healthy strategies.
- **Fractional Kelly** — growth ≈ c(2−c) of max; c=0.5 gives 75% of growth at ~50%
  variance. Quarter to half Kelly.
- **Funding-rate arb** has compressed to **~2.8% annualised** venue-average (from
  10–30%). Momentum/trend remains the durable retail workhorse.

---

## 12. Still open

1. **Exchange selection.** OKX demo trading is the best sandbox (production API +
   one header). Binance testnet is good. Bybit imposes a 48h lockout on new accounts.
2. **Historical L2 data budget.** ~1–5 TB/year *per symbol-exchange pair*;
   Kaiko ≈ $28.5k/yr (unverified). Determines how much microstructure work is feasible.
3. **Hot zones** — which paths require sign-off and "explain the blast radius."
4. **Autonomy level** — Loop Training Mode defaults to approve-every-step.
5. ~~`gh` CLI not yet installed~~ — **DONE.** `gh` 2.97.0 in `~/.local/bin`,
   authenticated as `ajith4134`, scopes `repo, workflow, gist, read:org`.
6. **Liquidation feed has no source.** Binance withholds `forceOrder` from this
   host and `allForceOrders` was withdrawn from the public REST API — measured,
   see `binance-withheld-streams.md`. The subscription is kept so recovery would
   be noticed, and the tile stays red until then. Reaching it needs either a
   second venue for liquidations or a paid feed.

---

## 13. Next actions (in order)

> **Rewritten 2026-08-09 against what is on disk.** The previous version read
> "Phase 1 — Cost Engine ← *next*" while the cost engine had already shipped, so
> the one section whose job is to say what to do next had been wrong for six
> days. Nothing had gone wrong; nobody had come back and edited it.
>
> Every state below was checked rather than recalled, and the check is named. The
> distinction that matters here is **built** versus **driven**: this repo's
> standing warning is `tail_specs()` — built, tested, called by nothing — so a
> module no caller reaches is recorded as exactly that, never as done. The check
> was `grep -rn` for each module's public names across `src/`, discounting
> matches that turned out to be docstrings referring to it.

1. ~~**Finish the Claude-usage work**~~ — **DONE 2026-08-01.** Enforcement layer is
   live (2 PreToolUse hooks + 11 deny rules, verified blocking); CLAUDE.md
   restructured 220 → 169 lines with Rule 2's operational half promoted to the
   `youtube-video` skill. Remaining optional item: a **Stop hook** for verification,
   which needs a project with a runnable test command — so it belongs to step 3.
2. ~~**Write the full system spec**~~ — **SUPERSEDED 2026-08-03.** The spec was
   written per-layer as implementation plans instead of as one document, which is
   what `docs/superpowers/plans/` holds. `ARCHITECTURE.md` §3 is the system spec.
3. ~~**Phase 0 — Truth**~~ — **DONE 2026-08-03.** Layer 0 raw capture and Layer 1
   bitemporal store + clock-gated reader are built, tested and running. Three
   venues supervised (binance, binance-spot, hyperliquid), restarting on boot.
   The suite was 383 tests then and is **1,035 passed, 1 skipped** on 2026-08-09.
4. ~~**Phase 1 — Reality filter: the Cost Engine**~~ — **DONE, recorded late.**
   `src/cost/` holds `fee_schedule`, `fee_fetcher`, `round_trip_cost`,
   `funding_carry`, `spread_and_depth` and `secret_store`. Driven:
   `quote_round_trip_cost` is called by `cost.cli` and by `paper.prove_plumbing`,
   and the wall tile *Cost engine — round-trip breakeven gate* reads OK against a
   live fee verification. Built to the plan in
   `docs/superpowers/plans/2026-08-03-cost-engine-reality-filter.md` and to its
   measured constraint — Hyperliquid publishes its fee schedule unauthenticated,
   Binance returns 401 without an API key, so the engine is a declared table plus
   a verifier rather than a live fetcher.
   *Not driven:* `cost.spread_and_depth`, which nothing calls.

5. **Phase 2 — Ops floor** — **DONE to the limit of what exists without a
   trading key.** Updated 2026-08-09 after wiring; `ARCHITECTURE.md` §3 puts this
   before any capital.

   | Component | State | Checked by |
   |---|---|---|
   | Rate-limit budgeter | **driven** | `ops.rate_budget` imported by `capture.cli` |
   | Order-intent WAL | **driven** | `execution.order_intent_wal` used by `paper.prove_plumbing`; journals both legs of the demo round trip |
   | Venue health monitoring | **driven** | `capture_health` feeds the wall; silence and gap events recorded per venue |
   | Auto-halt per venue | **driven 2026-08-09** | `ops.venue_health_watch` feeds `observe()` once a minute via `scripts/health_supervisor.sh`, started by the GCE startup script. Registry stamps `last_observed_ns`; the wall grades on its age |
   | Kill switch | **read and armed 2026-08-09** | `prove_plumbing.run` refuses to start while `is_killed` is true, before anything is journalled. The trip side now HAS its trigger: `risk.tail_cap.enforce` trips the watchdog on a live breach, which is the hard cap this line previously said did not exist. The firewall half remains impossible on this host: no `sudo`, no `iptables`, no `nft`, measured 2026-08-08 |
   | **State recovery on restart** | **built, cannot be driven yet** | `execution.state_recovery` compares the WAL, the local position model and **venue truth**. Venue truth needs an authenticated exchange session, which does not exist. The earlier note here said it "must run on recorder start" — wrong, it is not a recorder concern. It belongs at trading-engine start, in Phase 4 |
   | Key scoping | **not started** | no trading key exists yet; `cost.secret_store` is the only piece |

   What is left in this phase is gated on capital and keys, not on work. The
   honest reading is that the ops floor is as built as it can be until Phase 4
   creates the things it protects.

6. ~~**Phase 3 — Search integrity**~~ — **DRIVEN 2026-08-09.** It read "built as
   a library, driven by nothing" this morning: every module reached only through
   `promotion_gate`, and `promotion_gate` reached by nobody.

   `validation.promotion_pipeline` is now its first caller, and
   `validation.breadth` is the Trial Registry's. `HoldoutCustodian` is finally
   passed to `ClockGatedReader` — it had been an accepted argument no caller
   supplied, so the guard existed and guarded nothing. Unsupported claims fell
   from eleven to one; only RX-005 remains, and it needs an authenticated
   session.

   Two defects the wiring exposed, both worth keeping:

   - **The pipeline found a lookahead in its own candidate.** Pure noise scored
     an observed Sharpe of 3.90, and the deflated Sharpe, MinBTL and
     purge-retention gates all PASSED it — only PBO refused. Selecting the top
     decile of a symmetric distribution and averaging returns a positive number
     every day; it is a selection artefact of the return definition wearing the
     shape of an edge. Fixed by shifting the signal, after which noise scores
     0.04 and is refused, and a planted edge scores 1.01 and is promoted.
   - **A fresh registry cannot deflate.** At N=1 the dispersion is zero and the
     hurdle collapses, which is how noise cleared the DSR gate. The stack caught
     it through SPA instead — the argument for several independent gates rather
     than a better single one.

7. ~~**Finish Phase 2 by wiring what is already written**~~ — **DONE 2026-08-09**
   for the two drivers that could exist, and the third turned out not to be a
   Phase 2 item at all. See the table in step 5.
8. **Phase 4 — one family end to end** (carry) — **MACHINERY DONE, VERDICT
   PENDING ON DATA.** Spec: `docs/superpowers/specs/2026-08-09-phase-4-carry-first-pass.md`.

   | Item of the definition of done | State |
   |---|---|
   | Breadth gate run and written down | **DONE** — `PASS, bursty flow` |
   | A candidate through the full pipeline, gate returns a verdict | **DONE** — it refuses |
   | `validation/` no longer reachable-by-nothing | **DONE** — 11 unsupported claims → 1 |
   | Axis verdicts carried by the wall | **DONE** — and reading 9/65, NOT MEASURED |
   | Reduced-size live under the §6 human gate | **not reached**, correctly |

   **The breadth gate passed, and by more than §5a.4 feared.** Measured over
   1,331,980 reconstructed funding rows across 850 symbols, six configurations:
   effective breadth 46–82 independent bets against the 5–15 band the spec calls
   a collapse, with mean pairwise correlation 0.03–0.06. Carry is not one BTC
   factor wearing 850 hats.

   **But triggers arrive 17–32× more unevenly than independent firing**, a fifth
   of them in the busiest 5% of days. That is not a redesign — §5a.4's condition
   is *cluster AND correlate*, and reading it as a disjunction was a bug in the
   first draft of the verdict function. It IS a design constraint carried into
   whatever is built next: §5a.5's acceptance threshold rising with opportunity
   flow, and dry powder priced as a held option, are now empirical requirements
   with a number behind them rather than precautions.

   **What blocks a promotion is calendar time, not machinery.** The pipeline
   reads the OBSERVED `funding` dataset, which began on 2026-08-08. The 582 days
   of reconstructed history cannot substitute: its availability time is the
   fetch, so every simulated clock in the past sees it empty — deliberately, and
   that is what makes it safe for research and useless for a backtest.

9. ← ***next*.** **WAIT FOR OBSERVED HISTORY.** Decided by the user
   2026-08-09, over starting Phase 5. Nothing is built against a promotion until
   the observed record can support one.

   **How long the wait is, was itself a finding.** At N=21 trials MinBTL demands
   6.1 years of history for a Sharpe-1.0 claim — because the gate counted
   calendar days, pooling 850 symbols into one daily portfolio return. §5a.5
   requires correcting on *both* axes, trial count **and effective sample
   size**, and only the first was implemented. Corrected the same day, with the
   multiplier measured on the data being scored rather than assumed:

   | basis | Sharpe-1.0 at N=21 | Sharpe-1.5 |
   |---|---|---|
   | calendar days (as built) | 2,223 days | 988 days |
   | × effective breadth 46 | **48 days** | **21 days** |

   The correction makes promotion easier by roughly 58×, which is the direction
   every defect this project has found failed in — so it was put to the user
   rather than adopted, and the constraints are tested: it fails closed to
   calendar days below 30 days of data, can never exceed the symbol count, and
   is named in the gate's own words rather than folded into a year count.

   **So the wait is weeks, not years** — and it is a wait on the OBSERVED record
   only. The 582 days of reconstructed funding cannot shorten it, by design.

   What the wait needs to survive: capture staying up. That is the whole risk of
   waiting, and it is why the hour-boundary crash mattered more than it looked.

   Phase 5's allocator, sizer and correlation breaker have nothing to allocate
   between until a strategy is promoted — but the drawdown ladder of VX-011 is
   derived and not yet acted on, because nothing cuts gross by a rung until
   position sizing exists. That is the first real Phase 5 hook when the time
   comes.

   The two Phase 2 leftovers still travel with the first live strategy:
   `state_recovery` needs an authenticated session to reconcile against, and key
   scoping needs a key.

### Known gaps carried forward, not silently dropped

- **Provenance-flagged backfill** (Phase 0's last item) is not implemented — gap
  *detection* is live and recorded, but a backfilled row is not yet labelled as
  interpolated. Nothing currently backfills, so this is a prerequisite of the
  first thing that does, not an outstanding defect.
- **Liquidation feed unavailable** — see §12.6 and `binance-withheld-streams.md`.
  Binance withholds `forceOrder` from this host; the tile stays FAILING until a
  second venue or a paid feed supplies it.
- ~~**Auto-halt on venue degradation is not armed.**~~ **ARMED 2026-08-09.**
  `ops.venue_health_watch` feeds the registry once a minute and the wall grades
  on the observation's age rather than asserting anything. The tile that read
  "auto-halt armed" while `observe()` had no caller — the Rule 8 failure inside
  the board built to prevent it — now reads what it measures, and says
  `AUTO-HALT NOT ARMED` whenever nothing has fed the registry.
- ~~**The hour-boundary stall is bounded, not understood.**~~ **EXPLAINED
  2026-08-09.** The binance recorder died at every hour boundary on a keepalive
  timeout while binance-spot crossed the same boundaries untouched. Measured:

  | | recorder writers | frames/s | fsync at the boundary |
  |---|---|---|---|
  | binance | 575 | **2,415** | 2.5 s |
  | binance-spot | 1,321 | 102 | 5.7 s |

  Spot has more writers and more total fsync work, and never came close to
  dying. **The discriminator is baseline loop saturation.** binance runs at 24×
  the frame rate, so the rotation burst lands on a loop with almost no slack and
  the keepalive's ping/pong finds no window; spot's loop is mostly idle and has
  room throughout. Both the writer count and the total fsync work were red
  herrings, and each had been a hypothesis.

  It also explains the fix: the drain is paced PER FRAME, so cutting the budget
  from 25 to 4 interleaves ~6× more frames between fsync slices. Measured after:
  a 6.0 s stall at the rotation, and the recorder alive across 13:00 and 14:00.

  Carried forward as a capacity fact rather than a defect: **binance sustains
  ~8.7 million frames an hour through one process.** Anything added to that
  event loop is added to a loop that is already nearly full.
- ~~**The dollar-quote filter is not applied.**~~ **APPLIED 2026-08-09**, as the
  blocking prerequisite for Phase 4. `store.cli --symbols ALL` now filters, and
  refuses the build when no universe snapshot says what anything is priced in.
  **What remains is the contamination already written:** 536 non-dollar symbols
  hold **89,097 bars, 29.4% of binance-spot's**, in TRY, EUR, JPY, IDR, BRL, BTC
  and ETH. The store is append-only and nothing retracts them, so **any consumer
  choosing a universe must filter it** — `dollar_quoted_symbols` is the call.
  This is a hard requirement on Phase 4's carry family, which ranks
  cross-sectionally: a lira price and a USDT price are not comparable numbers.
- **Unwired and awaiting a consumer**, listed so none of them is later
  rediscovered as new work: `risk.drawdown_distribution`,
  `paper.participation_calibration`, `cost.spread_and_depth`,
  `store.correct_bars` (a one-shot, used once for the 746 poisoned bars).
