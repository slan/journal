# Pure self-play league and training dashboard

date: 2026-09-17 12:57
mode: voice
tags: [twilight-struggle, rl, ppo, self-play, league, elo, rust, dashboard]

## Context
Voice session continuing the Twilight Struggle AI thread. The Rust engine rewrite is effectively settled, so the session was mostly about how far pure self-play PPO can be pushed with no prior at all, how to measure that progress credibly, and how to visualise it while training runs.

## Ideas
### pure-self-play-ceiling-probe
Push PPO with self-play and an AlphaStar-style league as far as it goes on the Rust engine, deliberately introducing no prior. The point is to find the wall: where it plateaus tells you what an LLM prior actually needs to supply later. Gives the prior work a baseline it has to beat. Connects to the existing staged plan (Rust engine, re-baseline PPO, then LLM prior distilled into PPO). Status: exploring.

### league-elo-eval
Three populations: main agent training continuously, periodic frozen snapshots of it, and exploiters whose only objective is to beat the current main and expose holes. All match results feed an Elo pool. League-internal Elo against the frozen ladder is the real progress curve; DLL win rate sits alongside as an absolute anchor. Frozen checkpoints as fixed opponents also detect cycling. Status: exploring.

### league-dashboard
Simple web dashboard streaming aggregate league state over websockets: Elo curves per league member, match counts, and the win matrix between roles. Trainer emits events to a lightweight queue, dashboard subscribes. No live game replay — only aggregates, so it is a handful of numbers per match and cheap to push on every match completion. Status: seed.

### checkpoint-naming-convention
Filename-based lineage: run identifier, role, generation, short parent hash — e.g. run03-main-gen42-parent-abc123. Exploiters carry the identity of the opponent they target in the name rather than in a side file. The dashboard rebuilds the whole lineage tree by parsing filenames, no database needed. Status: seed.

## Decisions
- Push pure self-play PPO to its ceiling before introducing any prior.
- Gate progress on league-internal Elo, not on DLL win rate; DLL is a sanity check and absolute anchor only.
- Exploiters count as pure self-play, not priors — they are copies of the agent with a narrow objective, no outside knowledge injected.
- Keep a single shared network with a side indicator rather than one network per side; spend the budget on capacity instead.
- First acceptance test on the Rust engine is a throughput benchmark, which then doubles as a regression guard as card effects land.
- Dashboard streams aggregate league progression only; no live replay of individual games.

## Open questions
- What steps-per-second does the Rust engine actually buy over the Python engine? Needs measuring.
- What the concrete stopping rule is once Elo is the gate — flattening over how many steps.
- Matchmaking weights within the league were not worked out.

## Next actions
- [ ] Benchmark Rust engine throughput and compare against the Python engine baseline.
- [ ] CC: implement the throughput benchmark as the first acceptance test, wired as a regression guard.
- [ ] CC: draft the league structure — main, frozen snapshot ladder, exploiters — with an Elo pool over match results.
- [ ] CC: implement the checkpoint naming convention and a lineage parser that reads it from filenames alone.
- [ ] CC: prototype the websocket dashboard for Elo curves, match counts, and the role win matrix.

## Quotes
- "I push the simulation as far as I can, using self-play and then the league system, a la AlphaStar. And then when I hit the wall, I start introducing priors."
- "Pure PPO can only reach 25% win rate, and that's not good enough."
- "I don't want the live replay of games, I just the progression of the leagues."
