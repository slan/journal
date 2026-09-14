# Twilight Struggle RL — Handover Brief for Claude Code

**Origin:** Voice-mode discussion with Claude, 14 September 2026
**Author of decisions:** Slan
**Purpose:** Turn this brief into a concrete technical implementation plan. Use a high-reasoning model. Nothing below is code — it is context, decisions, and an ordered plan with open questions to resolve.

---

## 1. Current state of the project

**Goal:** Train an AI opponent for the board game *Twilight Struggle*, and use the project as a vehicle for experimenting with semi-autonomous, agent-driven research loops.

**What exists today**

- A Python game engine that enforces the rules (forked from an open-source project whose original author was building an LLM-based Twilight Struggle opponent — repo name not recalled during the session; locate it in the codebase).
- PPO training via self-play. Bootstrapped from pure self-play, then an AlphaStar-style league: checkpoints are frozen and added to an opponent pool, and the current policy trains against the pool.
- Progress plateaued — the agent stopped improving against the league.
- **External baseline:** the official digital version (Playdek) ships its full game AI in an external DLL. Slan has hooked this DLL and uses it (a) as a rules oracle to validate the Python engine and (b) as a real search-based opponent.
- **Current result:** ~25% win rate against Playdek's *easy* AI with the standard two-handicap rules.

**Known failure mode:** early on the agent hit a "DEFCON suicide" wall — losing by triggering nuclear war. Slan worked around it by crafting expert opponents for the pool. Whether the policy truly internalised DEFCON safety, or whether it regresses without the experts, was not established. Treat as an open question.

**Bottleneck:** a training run takes ~2 hours. The Python engine is slow. This limits experiments to a few per day and is the single biggest obstacle to any automated research loop.

**Reusable infrastructure from a previous project (mobile RTS, Unity, server-authoritative):**

- A shared-memory tensor bridge between a fast native engine and Python. Ping-pong protocol using native shared memory and semaphores. Transport is engine-language-agnostic; buffer layout is an application-level agreement.
- A C# binding exists. A Rust binding is expected to be straightforward.
- Python side: Stable-Baselines3 with a custom shared-memory `VecEnv` adapter. Already scaled from 8 to 256 parallel environments without touching the transport.
- Conclusion from the session: swapping the engine behind the bridge is a port, not a research problem.

**Hardware:** RTX 4090. Can run quantized local models (e.g. Qwen3 ~8B class, and a 4B model comfortably).

**Model access:** Claude, Codex, local models.

---

## 2. Decisions taken during the session

1. **Stay on Twilight Struggle.** Commands & Colors: Napoleonics was considered as an easier target. Rejected: it is close to a solved-shape problem for plain PPO (small action space, dense rewards, short games), and it has no external opponent to serve as a scoreboard. The Playdek DLL baseline is the project's most valuable asset for automated research; don't give it up.

2. **Engine speed comes before agent orchestration.** Building a multi-agent research team on top of a 2-hour simulator produces five agents queuing for one slow engine. Fix the engine first.

3. **Rewrite the engine in Rust behind the existing shared-memory bridge.** Design for batching from the start: one contiguous buffer holding N games' observations, stepped in a single call. Use flat arrays for state rather than object-per-card.

4. **Do NOT base the new engine exclusively on the Playdek DLL.** Slan proposed dismissing the Python implementation and deriving rules from the DLL alone. Pushed back, and Slan accepted:
   - The DLL would then be both the reference implementation and the opponent baseline — its quirks would be trained against invisibly.
   - Hot-seat mode only exposes end-to-end games, so rule mismatches surface as mysterious divergences far from their cause.
   - **Instead:** keep the Python engine as the specification, port it to Rust, and use the DLL as a *differential oracle* — play identical move sequences through Rust and DLL, flag the first state where they disagree.

5. **The hybrid architecture is the research direction of interest:** an LLM supplies *strategic* judgement (card evaluation, headline choice, regional priority, risk assessment), a small distilled network makes that judgement cheap, and PPO handles the *tactical / numeric* layer (influence placement, ops arithmetic). Rationale: LLMs are strong at linguistic strategic reasoning and weak at counting; PPO is the reverse. The DEFCON wall is a strategic self-preservation failure — exactly the layer an LLM might supply.

6. **The LLM never sits in the training loop.** Latency would kill throughput. Label offline, distil, run at engine speed.

7. **Ablation is non-negotiable.** Every addition (LLM prior, search) is measured against an identical run without it, on the same playback baseline. If the hybrid doesn't beat plain PPO, the LLM added nothing. This is the first thing that gets dropped under excitement; don't let it be.

8. **Search (MuZero/AlphaZero-style) is phase four, not phase one.** Three research risks stacked at once are unmeasurable. Add them in order.

9. **Multi-agent research automation:** cross-vendor (Claude + Codex) is a stronger lever than persona prompting — different models fail differently, so disagreement carries real signal. Adversarial personas help but tend to produce contrarian objections rather than different hypotheses. Local Qwen: use for grunt work (log parsing, run summaries, config permutations), not ideation. Every proposal must become a runnable experiment. Define a stopping rule (target win rate, compute budget, or N failed experiments before escalating to Slan).

---

## 3. Ordered plan

### Phase 1 — Fast engine

- Port the Python rules engine to Rust.
- Expose it behind the existing shared-memory bridge (write the Rust binding).
- Batch-shaped buffer layout from day one; target hundreds of parallel games.
- Build the **differential test harness**: random legal playouts through Rust, Python, and the Playdek DLL; compare full state after every step; report first divergence with the move sequence that produced it.
- Done when thousands of random games run with zero divergence against both oracles.
- Also worth doing: a fixed regression suite of scripted move sequences covering tricky rules (DEFCON transitions, scoring cards, event/ops interactions, realignments, coups, space race, Wargames, etc.).

### Phase 2 — Re-baseline

- Retrain the existing PPO setup, unchanged, on the fast engine.
- Record win rate vs Playdek easy (two-handicap) at several sample budgets.
- This is the **control** for everything after. Know what plain PPO does at 10–100× the samples before adding anything clever. It is possible the plateau was partly a sample-count problem.
- Revisit the DEFCON question here: does the policy hold DEFCON discipline when the expert opponents are removed from the pool?

### Phase 3 — LLM strategic prior

1. **State collection:** run self-play on the fast engine, dump a large diverse set of positions. Weight toward high-value decision points: headline phase, high-ops cards in hand, DEFCON 2–3 situations, scoring-card timing.
2. **Offline labelling:** query a strong LLM (Claude / Codex) for *judgements*, not moves. Candidate label schema:
   - regional priority ranking
   - per-card: play for ops vs. event, and rough value
   - strategic risk level / DEFCON danger assessment
   - headline recommendation and rationale
   Design the schema to be structured (JSON), consistent, and cheap to distil.
3. **Distillation:** train a small network to predict the labels from raw state. Plain supervised learning. Report held-out accuracy per label type.
4. **Integration into PPO** — two variants to compare:
   - (a) distilled head outputs appended as extra observation channels;
   - (b) distilled head used as a bias on the action prior (KL-regularised toward it, decaying over training).
5. **Ablation:** identical PPO run without the prior. Compare vs Playdek baseline.

### Phase 4 — Search at inference

- Add MCTS at inference time using the PPO value network for leaf evaluation, and the (LLM-shaped) policy head as the search prior — AlphaZero shape with an informed prior instead of a random-init head.
- **Hidden information:** vanilla MCTS on the hidden opponent hand will be overconfident. Use *determinization* (sample plausible opponent hands consistent with public info, search each, aggregate). Crude but standard for card games. ReBeL / Stratego-style approaches are the more principled alternatives if determinization proves inadequate.
- Measure separately against the Playdek baseline, with and without the LLM prior.

### Phase 5 — Automated research loop

- Only now build the agent team, on top of: a fast engine, a fixed external scoreboard, and a known-good ablation protocol.
- Roles to consider: ideation (cross-vendor), engineering/experiment execution, analysis/summarisation (local model), reviewer/adversary.
- Hard constraints: every proposal becomes a runnable experiment; every experiment reports against the baseline; stopping rule enforced; Slan intervenes only on direction.

---

## 4. References surfaced during the session

**Hanabi (candidate small testbed for imperfect-information card play — deprioritised in favour of staying on Twilight Struggle, but relevant to the LLM-prior idea):**

- Bard et al., *The Hanabi Challenge: A New Frontier for AI Research*, arXiv:1902.00506. Introduces the Hanabi Learning Environment. Finding still cited as current: learned agents reach reasonable self-play scores but fall short of the best hand-coded agents, and learn brittle policies unreliable for ad-hoc teams.
- Ramesh et al., *Sparks of Cooperative Reasoning: LLMs as Strategic Hanabi Agents*, arXiv:2601.18077 (Jan 2026, v2 Mar 2026, ICML 2026 poster). SFT and RLVR (GRPO) fine-tuning of Qwen3-Instruct 4B on released Hanabi datasets; +21% (SFT) and +156% (RL) cooperative play, within ~3 points of o4-mini. **Directly reproducible on a 4090.** Releases the "HanabiRewards" dataset.

**LLMs as priors for RL (the pattern behind Phase 3):**

- *Efficient Reinforcement Learning with Large Language Model Priors*, ICLR 2025 (OpenReview id e2NRNQ0sZe). Treats LLMs as prior action distributions integrated into RL via Bayesian inference (variational inference / posterior sampling).
- Survey-level: LLM-generated policies used as informative priors or behaviour regularisers via KL penalties, improving sample efficiency and suppressing catastrophic exploration (see emergentmind "LLM-Guided Reinforcement Learning" topic page and citations therein, incl. Zhang et al. 2024).
- On-policy distillation (Thinking Machines blog; arXiv:2604.00626 survey) — relevant if the distilled head is later trained on-policy rather than from a static labelled set.

**Gap identified:** no work found applying LLM-distilled strategic priors + PPO + determinized search to a complex hidden-information wargame with an external search-based baseline. That combination is the candidate contribution.

*(Verify all references before citing; these were retrieved via web search during a voice session and not read in full.)*

---

## 5. Open questions for Claude Code to resolve or flag

1. Which open-source repo is the Python engine forked from? Document its licence and how far the fork has diverged.
2. Exact Playdek DLL interface: what state can be read/written, can games be seeded, can arbitrary positions be loaded (needed for differential testing beyond full playouts)?
3. Observation encoding for Twilight Struggle in the batched buffer — define the flat layout (map influence, control, DEFCON, VP, turn/AR, hand, discard knowledge, space race, milops, headline state, opponent-hand belief features).
4. Action space encoding and legal-action masking in Rust — how does the current Python version do it, and does it need to change?
5. Label schema for Phase 3 — finalise before collecting states.
6. Ablation protocol — fixed seeds, fixed sample budget, fixed evaluation set of N games vs Playdek easy, statistical test to declare a difference.
7. Where does the earlier "expert opponent" pool live, and should it be retained as a diagnostic rather than a training aid?
8. Stopping rule for the automated loop (Phase 5).

---

## 6. Instructions for Claude Code

- Produce a phased technical implementation plan from this brief, with concrete tasks, file/module layout, and acceptance criteria per phase.
- Treat Phase 1 → 2 → 3 → 4 → 5 as strictly ordered. Do not start Phase 3 work until Phase 2 has a recorded control result.
- Preserve the ablation protocol in every phase. Any experiment without a control is not an experiment.
- Flag anything in this brief that contradicts the actual codebase — the brief was reconstructed from a spoken conversation and may be wrong on details.
