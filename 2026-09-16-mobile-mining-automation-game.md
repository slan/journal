# Mobile mining automation game

date: 2026-09-16 19:50
mode: voice
tags: [game-design, mobile, automation, factory-game, puzzle, mvp]

## Context
Voice session while making dinner, exploring a new game idea from scratch. Premise: factory/automation games are underrepresented on mobile, and there's a market opportunity if the input problem and the scaling problem can be solved. The session moved from a vague "casual automation game" to a concrete structure: a vertical Boulder Dash-style mining game with pneumatic tube logistics, restructured mid-session into short puzzle levels with a zoomed-out logistics meta map. Ended with a defined MVP.

## Ideas

### vertical-mining-automation-core
Mobile automation game on a vertical scrolling shaft rather than a 2D sprawl. Swipe-to-move, tile-based, Boulder Dash feel. The player is the logistics before machines are: you dig, then you build the thing that replaces you. Depth is the scarce axis instead of area, giving natural chapter breaks (minerals, then liquids, then gases) and keeping routing one-dimensional, which suits a thumb. Status: exploring.

### pneumatic-tube-logistics
Logistics are tubes with small air-propelled vehicles, not belts. A tube is two endpoints, so no tile-by-tile drawing on a touchscreen. Manual route and vehicle assignment, Transport Fever style with priorities, rather than a rules editor. Scarcity is the fleet cap, not the pipes: dig as many parallel routes as you like, the number of vehicles is fixed. Congestion is legible on screen as containers backing up, no UI needed. Status: exploring.

### rubble-buffer-and-surface-pile
Digging produces rubble that must be hauled, so expansion and production compete for the same fleet. Spoil goes to a local buffer at the dig face; you only stall when it fills, so there's choice without micromanagement. Rubble is hauled to a visible pile on the surface, fixed width (screen width), which grows and encroaches on finite surface plots where the sorter, depot and repair facilities go. Later tech turns rubble into low-grade ore (Captain of Industry angle), so a backlog becomes an asset. A percentage indicator shows pile status while deep so scrolling up isn't forced. Status: exploring.

### power-as-light-and-fog-of-war
Power is routed down the existing tube network rather than being a new system. Light is fog of war: an unpowered tunnel is a dark tunnel, so the network is your vision. Introduced around layer two or three as depth-gated infrastructure, not a minute-one resource. "How deep my grid reaches" becomes a legible progress marker, and the mid-game push-pull is extend the grid down versus thicken it to feed existing sorters. Status: exploring.

### puzzle-levels-plus-logistics-meta-map
Restructure from open sandbox to short, interruptible, puzzle-style vertical-slice levels, roughly two to three minutes each, better suited to a mobile audience. Hand-authored pacing can guarantee a decision every twenty seconds where a procedural sandbox can't. Each cleared site becomes a node on a zoomed-out logistics map; levels teach and unlock, the map is where unlocks compound. Shippable as the level game alone, with the map added later. Status: exploring.

### generator-plus-solver-curation
At two minutes a level, thirty-minute sessions need hundreds of levels, so generate and curate rather than hand-author everything. A solver scores difficulty and solvability ("solvable in ninety seconds with three vehicles"), which requires the sim to be headless from day one. Handcrafted levels reserved for tutorial beats and set-pieces where the generator is weakest. Connects to prior experience with this kind of architecture. Status: seed.

## Decisions
- Core scarcity is fleet bandwidth (a capped number of vehicles), not tube capacity.
- Second scarcity is finite surface land, contested by a growing, visible rubble pile.
- Rubble is buffered locally at the dig face; stalls happen only when the buffer fills.
- Selling rubble must always remain possible, even at a bad rate, to prevent hard-locks.
- Vehicle and route assignment is manual, Transport Fever style, not an automated rules engine.
- Processing and building happen inside the tunnels; the surface is for the pile, depot and recycling.
- Power is routed down the tube network and doubles as lighting / fog-of-war.
- Game structure is short puzzle levels with a zoomed-out logistics map as the meta outer loop.
- Levels are offline, client-side and deterministic; the meta map is online and server-authoritative.
- Rewards are binary solve-or-not, with free unlimited retries, to remove the incentive to cheat.
- The meta rewards which levels were cleared, not how well, to avoid creating an attack surface.
- MVP is four levels built on a headless sim with a crude UI.

## Open questions
- Whether the fleet cap generates enough decisions per minute, or leaves long stretches of watching vehicles.
- Whether the pile should physically block surface construction or remain a capacity number once the player is deep.
- What the fourth MVP level looks like: making the player feel the ceiling without explicitly teaching it.
- Exact tuning of the darkness/power mechanic so it's dramatic but not punishing.
- Multiplayer meta design, deliberately deferred.

## Next actions
- [ ] Prototype a vertical slice of four levels, each introducing exactly one thing: dig and haul, the buffer squeeze, fleet allocation, a taste of power and light.
- [ ] Build the prototype as a headless sim plus crude UI, so solver scaffolding comes free later.
- [ ] Design the fourth level so it conveys depth potential without tutorialising.
- [ ] Decide pile-blocks-building versus pile-as-number before the prototype UI work.

## Quotes
- "The factory slash automation game is a bit underrepresented there. So I see a market opportunity."
- "I have a mental picture of the old game Boulder Dash."
- "If we limit the number of vehicles that can flow on the tube network, then you can dig as many parallel routes as you want. But the number of vehicles is limited anyways."
- "I like the visible pile on the surface. To create tension."
- "I like the short interruptible session after a few rounds of, let's say, three minutes, two minutes. Puzzly type levels. I think it's more suitable for the mobile audience."
- "The more level you progress, the more capabilities on the zoomed out version. That's a nice meta outer loop."
- "Retries are basically free, you try the level as many times as you want."
- "This would be my MVP."
