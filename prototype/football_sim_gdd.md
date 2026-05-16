# Football Sim — Game Design Document (GDD)

> **Status:** Skeleton v0.13 — Draft
> **Owner:** Sandro
> **Last updated:** 2026-05-15
>
> **How to use this doc:** Canonical source of truth for design decisions. §11 is the single index of unresolved questions. When a question is resolved, the design itself goes in the relevant section, the §11 entry is crossed off with a one-line note, and the *reasoning* is preserved in §12 (Decision Log).

---

## 0. One-Liner & Pillars

**One-liner:** A multiplayer football (soccer) team-ownership simulator where the player issues open-ended natural-language instructions to AI officers, who execute them inside a shared, persistent regional league world.

**Design pillars** (the things that must be true for this game to be itself — cut anything else first):
1. **Open-ended input, bounded output.** The player can say *anything*; the simulator interprets it and produces consequences shaped by the team's real state (money, personnel, reputation, etc.).
2. **AI officers as the primary interface.** The player operates *through* characters, not through menus. Officers have stats, personalities, and they take time to do things.
3. **Shared persistent world.** Other real players are the league. Promotion/relegation, the team market, and reputation all derive from a shared regional reality.
4. **No on-pitch gameplay.** The interesting decisions are managerial. Matches are simulated; the player never controls a player on the pitch.

**Anti-pillars** (things this game is *not*, to keep scope honest):
- Not Football Manager. We will not have deeper tactical/scouting depth than FM.
- Not a chatbot toy. The AI is in service of game state, not free-form roleplay.
- Not a single-player sandbox. The shared world is core, not optional.

---

## 0.1 Scope ladder

The doc uses three stages to describe when something exists in the design. Most decisions are about v1; deferred work goes to v1+, with a size tag.

- **Prototype** — internal-only build. Its job is to validate the three load-bearing risks: §3 classifier reliability, §2 outcome math, §10.5 cost envelope. Small player count (single-digit, internal or invited). Known design holes are acceptable here — the prototype tolerates things v1 proper will close (e.g., no fail state, no bankruptcy mechanics, no damage cap, no transfer windows). Exit criterion: the three validation targets pass.

- **v1** — first publicly-shippable product. Prototype holes are closed. Includes everything in the doc labeled as v1 design intent (2 officers, 8 novel-intent domains, transfer market, club state attributes, the §6 launch model, monetization, etc.). The prototype is a partial implementation of v1; most v1 design decisions apply to both.

- **v1+** — deferred work. Not roadmapped, no commitment to sequencing. Individual items carry a size hint:
  - **v1+ (additive)** — small additive features that can be slotted in after v1 ships without restructuring existing systems (hiring counter-offers, support-stat-as-difficulty modifier, event chaining, loan-out availability flags).
  - **v1+ (structural)** — major new systems requiring real design work (cross-club transfers, youth academy, multi-region scheduling, stadium ownership, world-wide events, officer roles beyond DoF/CD, the team market).

The size tag is descriptive, not a release commitment. "v1+ (additive)" doesn't promise "soon"; it notes that the design surface is small. "v1+ (structural)" doesn't promise "far"; it notes the surface is large. A v1+ (structural) feature can ship before a v1+ (additive) one if priorities shift.

**Usage convention.** When something is a v1-design-intent decision that also holds in the prototype, say "v1." When something is a prototype-only behavior that v1 proper will close, say "v1 prototype" (e.g., "v1 prototype tolerates arbitrary negative cash"). The pattern names what the doc is committing to *and* what build it applies to.

"MVP" is retired in favor of "v1." Older Decision Log entries (per §12 append-only rule) may still use "MVP" and "v2"/"v2+" — see the §12 preamble for the mapping.

---

## 1. Core Loop

**World model:** Real-time. The world ticks whether the player is online or not. This follows directly from Pillar 2 (officers take time) and Pillar 3 (shared persistent world) — a paused-when-offline mode is incompatible with both.

**Session cadence (design target):**
- **Floor:** ~once a week. A player checking in weekly should not fall irrecoverably behind.
- **Target:** ~once a day. The core loop is designed around daily check-ins.
- **Engaged ceiling:** multiple check-ins per day is fine but not assumed.

**Session length:**
- **Floor:** ~5 minutes (quick check, resolve any blocking decisions, close).
- **Target:** ~15 minutes (the canonical session — review, decide, strategize).
- **Ceiling:** ~30 minutes (the game should have meaningful things to offer an engaged player for this long, but not require it).

**Session shape (target case):**
1. Player opens the game → sees notifications/events that occurred since last login.
2. Player reviews state (officers' status, finances, upcoming match, recent events).
3. Player resolves any open decisions surfaced by officers or events.
4. Player issues new instructions or responds to events.
5. Player closes the app; world continues.

Session orientation for v1 relies on notifications, the club log, the officer panel (showing active and frozen tasks), and surfaced event response windows. A structured "goals list" as an additional orientation anchor is deferred to v1+ (see §11 §2) pending prototype data on whether session orientation needs more than these surfaces.

**Season-level loop:** Match calendar plays out, finances accumulate, a final standing determines promotion/relegation/staying.

**How the player should feel after a 15-minute session:** *Open question — likely some mix of "caught up," "in control of next moves," and "curious about how the next 24 hours will play out." To be sharpened during prototype.*

---

## 2. AI Officer System ⭐ *Most novel; deepest design work needed*

**Concept:** The player hires officers (Director of Football, Commercial Director, possibly more in later versions). Each is an AI character with stats and distinct character. The player delegates by speaking/typing to them. Officers take *time* to complete tasks and are blocked while doing so.

**v1 officer roster: 2 officers.**
- **Director of Football (DoF)** — squad, coaching, tactics, transfers, youth.
- **Commercial Director (CD)** — sponsorships, brand, fan engagement, merchandise, media relations, crisis management, public statements.

**Reasoning:** depth over breadth at v1. Two sharply-differentiated officers force good characterization and a clean mental model for the player ("on-pitch goes to the Director of Football, everything else goes to the Commercial Director"). Real-football language (rather than generic C-suite titles) teaches the player something true about how clubs are structured and correctly signals that both officers report to the player as owner/principal.

The Commercial Director's title is slightly stretched — a real-world Commercial Director doesn't usually handle crisis PR — but the mismatch is small and the player will learn the convention within one session. The alternatives (a v1+ crisis officer leaving a v1 gameplay gap, or a vaguer title like "Managing Director") were worse.

**Deferred to v1+ (structural):** dedicated Communications Director (split crisis/media off the CD), Chief Financial Officer, separate Head Coach (split from DoF), Scout, Academy Director, Stadium Operations, Legal Counsel. Order of reintroduction TBD based on which absences feel most acute in playtesting.

**Known facts so far:**
- Officers have stats. Higher stats = better outcomes.
- Player can choose to do an officer's job themselves (be hands-on).
- Officers contribute to a shared **goals list** with the player. The goals list is the primary "what am I working on?" anchor for a returning session — officers propose goals from their domain (e.g., Commercial Director: "land a kit sponsor by end of month"), the player approves/edits/rejects, and the list persists across sessions. Mechanic detail to be designed.
- Tasks are created from player input via the §3 interaction model. An officer is assigned a task with a defined relevant role stat, difficulty (1–20), and time cost; the §2 outcome math runs at task completion.

**Financial information in v1:** With no CFO in the v1 roster, financial state is surfaced as UI/dashboard rather than through a character. Financial constraints surface through whichever officer is relevant ("boss, we don't have the wage budget for him" from the Director of Football; "the sponsor's offer barely covers our operating shortfall" from the Commercial Director). Finance is texture *across* officer interactions rather than its own siloed character.


### Task Slot Model

Officers handle one active task at a time and can hold one frozen (paused) task. The chat *interface* is always available — the player can always talk to an officer — but issuing a new strategic instruction while the officer is mid-task triggers a recall-or-keep prompt.

**Slot mechanics:**

- **Active slot.** The task the officer is currently executing. One per officer.
- **Frozen slot.** A paused task, retained at its current progress, waiting to be resumed. One per officer.
- **Issuing a new strategic instruction with the active slot occupied** prompts the player: keep current task, or recall and freeze it to start the new one?
- **Issuing a new strategic instruction with both slots occupied** prompts the player: replace the active task (freezing it forfeits the existing frozen task — confirmation required).
- **When the active task completes**, any frozen task auto-resumes. Before resuming, code performs a relevance check: does the task's target condition still exist / is the goal still achievable? If not, the task is cancelled and the player is notified.
- **Special events are strategic tasks if the player acts on them.** Acting auto-freezes the current task; any previously frozen task is dropped (with confirmation).

**Instant vs. strategic:**

- **Instant actions** do *not* consume a slot. These are menu actions and chat instructions the classifier identifies as routine/recognized intents (per §3). Officer handles them immediately — fiction is "a quick phone call the officer can sign off on."
- **Strategic tasks** consume the active slot. These are novel chat instructions and substantive event responses, processed through the full LLM pipeline (per §3).


**Task duration:**

Strategic tasks have durations measured in **days**, ranging from 1 day (short-term) to ~50 days (roughly half a season — the long tail). The duration is computed by code as part of the strategic-task pipeline, based on intent and game state (per §10.5 principle 5). The officer communicates the duration in human-friendly terms aligned with the daily batch schedule ("I'll have an answer for you tomorrow morning," "give me 4 days," "this is going to take a couple of weeks"). All task resolution happens at the daily state-change batch (midnight PT per §10), so tasks complete at predictable, batch-aligned times.

**At-capacity dialogue:** templated response (no LLM call), explains the consequences of replacing the active and/or frozen task. The fiction is the same as the rate-limit "officer is busy" surface (per §10.5 principle 8) — the player cannot tell the difference, and shouldn't need to.

**Outcome delivery:** completed tasks apply tangible state changes (signed player, new sponsor, finances updated) and surface a summary in the club log on next login.


### Stat Model

Officers have two stat pillars, each with 5 stats: **Character stats** (universal, shared across all officers) and **Role stats** (specialized per officer, with limited overlap).

**Why this split:** Character stats describe *the person* and let the player compare candidates across roles when hiring (a low-Loyalty DoF and a low-Loyalty CD are both retention risks, even though they do different work). Role stats describe *the work* and are the primary driver of how well an officer executes their domain.

#### Character stats (universal, all officers)

- **Loyalty** — likelihood to stay with the club when poached, leak information, or undermine ownership decisions.
- **Ambition** — drive to push for bigger deals, bigger signings, and a bigger profile. High ambition with a low-status club creates retention risk.
- **Integrity** — willingness (or refusal) to engage in grey-area or illegal activity: match-fixing approaches, doping shortcuts, bribes, intentional leaks.
- **Charisma** — force of personality and persuasiveness. Distinct from raw negotiation skill — this is how well the officer *sells* whatever they're working on.
- **Composure** — performance under pressure. Affects variance in high-stakes situations (crisis press conferences, deadline-day deals, big-match preparation).

#### Role stats — Director of Football

- **Scouting** — finding talent and accurately evaluating players, including hidden gems and youth prospects.
- **Player Development** — getting the most out of the existing squad through training, role assignment, and individual man-management.
- **Negotiation** — transfer fees, player wages, agent dynamics. *(Also a Commercial Director stat, but the work is genuinely different — transfer markets and sponsorship deals don't share much.)*
- **Football Network** — relationships across the football world (other DoFs, scouts, agents, managers). Drives access to hidden-market opportunities and favorable agent treatment.
- **Strategic Vision** — long-term sporting direction: playing identity, recruitment philosophy, squad construction, succession planning at key positions. *Squad-building skill (right mix of ages, positions, depth, contract lengths) is part of this.*

#### Role stats — Commercial Director

- **Marketing** — brand-building, campaign effectiveness, fan acquisition and growth.
- **Crisis Management** — handling scandals and damage control: the skill at *minimizing* damage from negative events, distinct from the Composure character stat that governs poise during them.
- **Negotiation** — sponsorship terms, vendor deals, commercial contracts. *(Also a DoF stat — see DoF Negotiation note.)*
- **Media Relations** — relationships with press and the ability to shape narrative: planting stories, managing leak risk, getting favorable coverage.
- **Commercial Network** — relationships in the commercial world (sponsors, agencies, media owners, vendors). Drives access to deal opportunities others can't reach.

#### Notes on the model

- **Negotiation appears in both officer kits.** Deliberate: it's a domain skill that genuinely applies to both, but the work is different enough that one stat shouldn't cover both. A DoF and CD can have very different Negotiation values and that's correct.
- **Network stats are parallel across officers** (Football Network / Commercial Network). Same shape, different domain. Useful for hiring comparison.
- **Crisis Management vs. Composure** is the most likely confusion point. Composure is *how well you hold up* under pressure (a character trait); Crisis Management is *the technical skill* of damage control (a learned domain skill). A high-Composure, low-Crisis-Management CD is calm but ineffective during a scandal. A low-Composure, high-Crisis-Management CD knows what to do but might crack while doing it. Both axes matter.
- **No general "Competence" stat.** Role stats *are* competence at specific things; a meta-Competence stat would either be redundant or mask design vagueness about which role stat governs an outcome.
- **Stat scale and outcome impact deferred.** Open questions: numerical scale (1–10? 1–20? hidden values?), how stats roll into outcomes, whether stats can develop over time, whether some stats are revealed and others hidden at hiring time.


### Outcome Math

Outcomes for officer-executed tasks are resolved by a single roll: **2d6 + relevant role stat vs. task difficulty (1–20).** The margin between the rolled total and the difficulty determines the outcome tier.

#### Resolution

1. **Task difficulty** is set by the simulation: 1–20, hidden from the player as a number.
2. **Roll:** 2d6 + officer's relevant role stat.
3. **Margin** = rolled total − difficulty.
4. **Tier mapping** (placeholder thresholds, tune in prototype):

| Margin | Tier |
| --- | --- |
| +6 or more | Triumph |
| +3 to +5 | Good |
| 0 to +2 | Okay |
| −1 to −3 | Suboptimal |
| −4 to −6 | Bad |
| −7 or worse | Catastrophic |

The 1–20 difficulty range is chosen to match the rolled-total range (3–22 with 2d6 + stat), so margins are symmetric and all six tiers are reachable. A 1–10 difficulty range would skew margins systematically positive and break the symmetric tier mapping.

#### Difficulty bands and officer dialogue

The player does not see the numerical difficulty before the task resolves. Instead, the officer communicates a **difficulty band** in dialogue when the task is assigned:

| Band | Difficulty | Officer dialogue tone |
|---|---|---|
| 1 | 1–4 | "Easy enough." |
| 2 | 5–8 | "Should be fine." |
| 3 | 9–12 | "This'll take work." |
| 4 | 13–16 | "Tough ask, boss." |
| 5 | 17–20 | "Honestly? Long shot." |

The officer's *stated* band is not always the *true* band. It is filtered through two character stats:

- **Integrity** determines accuracy. High Integrity → stated band matches reality. Low Integrity → stated band is biased toward overclaiming capability (the officer says "easy" when it's hard, to win the assignment or keep the player happy).
- **Charisma** determines how confidently the stated band is delivered. High Charisma → smooth, assured delivery. Low Charisma → hedging, visible doubt.

This produces four character profiles that the player learns to read over time:

| Integrity | Charisma | Officer behavior |
|---|---|---|
| High | High | Accurate read, delivered with confidence. Trustworthy and reassuring. |
| High | Low | Accurate read, delivered with hesitation. Trustworthy but reads as unsure. |
| Low | High | Inflated read, delivered with confidence. **Most dangerous — confidently wrong.** |
| Low | Low | Inflated read, delivered with hedging. Read between the lines and you can spot it. |

Per §10.5 principle 5, the *true* difficulty is computed by code; the LLM is only told the officer's *stated* band when generating dialogue. Code computes truth; LLM narrates a possibly-distorted version.

#### Post-resolution reveal

After the task resolves, the player sees both the outcome tier *and* the true difficulty band. This lets the player learn their officers over time — a Triumph on a "Long shot" task with a low-Integrity DoF teaches the player that the officer underclaimed; a Catastrophic on an "Easy enough" task teaches them the officer overclaimed. The reveal is informational, not punitive — the data the player needs to evaluate which officers they can actually trust.

#### Character stat interactions with the math

Most character stats do *not* enter the per-task roll. They live on a separate event/disposition layer:

- **Loyalty** is checked when poaching offers arrive and on leak/betrayal events.
- **Ambition** drives officer satisfaction, retention, and willingness-to-take-risks events.
- **Integrity** drives corruption events, scandal probability, *and* the difficulty-band accuracy mechanic above.
- **Charisma** governs persuasion-flavored discrete events *and* delivery confidence on stated difficulty.

The exception is **Composure**, which modifies the die *only on special events* (crisis press conferences, deadline-day deals, scandal moments, big-match pre-match work). The dice change but the average stays the same:

| Composure | Die used on special events |
|---|---|
| 8–10 (high) | 3d4 (range 3–12, tighter spread) |
| 4–7 (average) | 2d6 (range 2–12, normal) |
| 1–3 (low) | 1d12 (range 1–12, flat distribution — extreme outcomes equally likely) |

A high-Composure officer is *predictable* in a crisis (rolls cluster near average); a low-Composure officer is *swingy* (every outcome from disaster to triumph is on the table). Crisis Management is what they know how to do; Composure is whether they stay on the rails while doing it.

Routine tasks are not "special events" and Composure does not apply.

#### Why no general "Competence" stat in this math

Role stats *are* competence at specific things. Adding a meta-Competence multiplier on top would either double-count the role stat or mask design vagueness about which role stat governs an outcome. The roll uses the relevant role stat directly.


### Character Archetypes

Each officer has a **character archetype** assigned at generation — a voice/personality template that shapes how the officer speaks and behaves in dialogue. Archetypes have **no direct mechanical effect** on the §2 outcome math (no roll modifiers, no special triggers). Their role is purely characterization — ensuring two hired DoFs across two playthroughs feel like different people, not interchangeable function calls.

**Three archetypes per officer role, six total for v1:**

**Director of Football archetypes:**

1. **Old-school football lifer** — career inside football, knows everyone, traditional values, instincts over data, loyal to the way things have always been done. Skeptical of fads.
2. **Modern analytics-driven operator** — younger, often from a non-football background, talks in xG and contract optimization, sees the game as a market. Trusts data over feel.
3. **Wheeler-dealer with connections** — career networker, every signing has a backstory, loves a complicated deal, comfortable in grey areas, charisma over rigor.

**Commercial Director archetypes:**

1. **Slick deal-maker from outside football** — career in tech, advertising, or finance; sees the club as a brand; high polish, big-picture thinker, unsentimental about the football side.
2. **Grizzled PR veteran** — career in sports communications, seen every scandal, defensive instincts, knows the press personally, conservative about brand risk. Pessimistic by default.
3. **Marketing visionary / brand-builder** — career in lifestyle marketing, fashion, entertainment; ambitious about cultural relevance, willing to take risks for upside, comfortable making the club a *thing* beyond sport.

**Archetype-biased stat generation.** Archetype is chosen first; stats are then rolled within ranges that fit the archetype, with random variation. A "modern analytics-driven" DoF leans toward high Scouting and Strategic Vision; a "wheeler-dealer with connections" leans toward high Football Network and Negotiation. Stats can still surprise — a low-Integrity wheeler-dealer is the dangerous version of the archetype, a high-Integrity wheeler-dealer is a charming-but-honest networker — but the *baseline* is shaped by archetype. Specific stat-range mappings per archetype tuned in prototype.

**Archetype visibility.** The player does *not* see the archetype label directly. The officer's character is conveyed through dialogue, mannerisms, and the texture of how they respond to instructions. Players learn an officer's archetype by working with them. Can revisit (surfacing archetype at hiring time, e.g.) once we have playtesting data on whether implicit discovery feels right or feels punishingly opaque.

**LLM implementation.** Archetype is stored on the officer record and included in the system prompt for any LLM call generating that officer's dialogue. Cached as stable context (per §10.5 principle 3) since archetype doesn't change after generation.

### Hiring Market

Officers are hired from a regional pool of NPC-generated candidates. The mechanic mirrors the §5 transfer market in spirit — shared regional scarcity, NPC-only for v1 — but the differences in volume, character depth, and gating make it a distinct system.

**Architecture:**
- Single shared regional NPC pool of officer candidates.
- Reputation-filtered visibility (per §4.5): each club sees a subset of the pool based on its reputation tier. A Disreputable club sees only lower-quality candidates; an Elite club sees the full pool. Top-tier candidates simply do not appear in a low-reputation club's view.
- Pool size: small. Working number ~6-12 candidates per officer role visible to a typical club. Tune in prototype.
- Pool refresh: once per season (every 26 weeks per §6), aligned with the offseason transition. Pool is stable within a season.
- v1 candidates are NPC-generated only. Officers leaving rival human clubs to join yours is a v1+ (structural) feature (parallel to cross-club player transfers).

**Candidate record:**
- Identity: name, age, background blurb.
- Archetype: one of three per role (per §2 Character Archetypes), assigned at generation. Archetype shapes the candidate's interview dialogue and biases their stat ranges.
- Stats: rolled within archetype-biased ranges.
- Asking terms: wage and contract length, generated based on the candidate's stat strength and reputation tier.

**Stat visibility at hiring time:**
- **Role stats revealed.** The player sees what the candidate is good at — their CV is on the table.
- **Character stats hidden.** Loyalty, Ambition, Integrity, Charisma, Composure are not visible at hiring. The player discovers them through working with the officer.
- This is the inverse of the player transfer market, where Scouting partially reveals hidden player attributes. Officer hiring has no equivalent scouting layer in v1 — character is discovered, not scouted.

**Hiring interaction:**
- A new "Hiring Market" view (per §3) lists visible candidates. Player can browse the pool.
- Tapping a candidate generates an *interview*: LLM-driven dialogue revealing the candidate's archetype-flavored personality. This is the player's primary signal about character before working with them.
- "Hire" is a menu action (per §3): accepts the candidate's asking terms. Single confirmation, instant.
- v1 has no counter-offer mechanic. Terms are what they are. Walk away or accept.

**Interview cost:**
- Each interview is an LLM call (mid-tier per §10.5). With ~6-12 candidates per role and 2 roles, browsing the full pool exhaustively is ~12-24 LLM calls. This is bounded — once per season, with most players not interviewing every candidate.
- Interview content can be cached (per §10.5 principle 3) — if a player re-opens the same candidate's interview, the cached generation returns. Re-interviewing a candidate after working with them once (e.g., re-hiring a fired officer) is a v1+ (additive) question.

**What's deferred:**
- Headhunting (actively recruiting a specific officer not in your visible pool). v1+ (structural).
- Officer retention battles (your CD gets a poaching offer). §2 Loyalty mechanic and §7 event design partially cover this; the *poaching offer event* is part of §7 event library work.
- Agent dynamics for officers.
- Counter-offers and hiring negotiation. Deferred to v1+ (additive). Small enough scope to add post-v1, but not needed for prototype validation.

---

## 3. Open-Ended Input → Bounded Output ⭐ *Most novel; deepest design work needed*

**Concept:** The player's input is unconstrained natural language. The output (effect on the world) must be constrained to plausible, game-state-respecting consequences.

This is **the single most important design problem in the project.** Get it right and nothing else like this exists. Get it wrong and the game feels arbitrary or empty.

### Interaction model

The player has three available interaction surfaces, all available at all times. What differs is which system processes the input and whether it consumes an officer's task slot (per §2):

1. **Standard menu actions (no LLM, instant).** Defined surface of UI views and structured state mutations available without going through chat. Friction-free, no rate limit, no task slot consumed. Full enumeration in the "Menu surface" subsection below.

2. **Open-ended chat with officers (LLM).** The player tells an officer to do something in natural language. Always available. Routed by classifier:
   - **Recognized intent → instant action.** "Sign player X for €5m" is a structured action even when typed as a sentence. Cheap-tier LLM classifies, code executes. No task slot consumed.
   - **Novel/strategic intent → strategic task.** "Build a youth-development reputation in South America." Mid-tier LLM, full pipeline. Produces a strategic task with difficulty, time cost, and assigned officer. Consumes the officer's active task slot (per §2).

3. **Event responses (LLM).** Responses to surfaced events go through the full LLM pipeline as a strategic task and consume a slot, same as a novel chat instruction.

The cost-and-slot model is binary: **instant actions** (no LLM cost or cheap-tier classification only, no slot consumed) and **strategic tasks** (full LLM pipeline, active slot consumed). The three surfaces above describe how the player *initiates* an interaction; the instant-vs-strategic distinction describes what the system does with it.

### Menu surface

The menu surface splits into **views** (read-only UI panels that observe game state) and **menu actions** (structured state mutations the player can trigger directly without chat). Both are no-LLM, no-slot, always available.

#### Views (no state change)

Read-only panels. Friction-free observation surfaces.

- **Club dashboard** — overview: finances summary, current standing, upcoming match, active and frozen tasks, recent events.
- **Finance panel** — detailed financial state. Depth depends on §4 economy abstraction (open question).
- **Squad list** — players, positions, contract length, wage, core match stats per §5. Hidden personality stats (Ambition, Loyalty, Temperament) are not surfaced here — discovered through DoF reports and events.
- **Officer panel** — for each officer: name, age, current active task and frozen task (if any), with the officer's estimated remaining time on the active task. Role stats visible (per §2 hiring market resolution); character stats hidden post-hire and discovered through working with the officer.
- **Match calendar** — upcoming matches, recent results, league standing.
- **Club log** — chronological history of completed task outcomes, events, and match results.
- **Transfer market** — the regional NPC pool of available players (per §5). Lists pool players with basic stats, asking prices, and wage demands. Hidden attributes (true potential, personality stats) are revealed based on DoF Scouting per §5.
- **Rival squads** — squad lists of other human clubs in the region (per §5). Shows players, positions, ages, and basic stats. Contract details (wage, length) are not visible.
- **Hiring market** — for each officer role, the currently-visible pool of candidates (reputation-filtered per §4.5 and §2). Tapping a candidate opens an interview view.


The Goals list (referenced in §1) will become a view once the goals list mechanic is designed (§11 §2). Deferred for v1.

#### Menu actions (state mutations, instant)

Structured state changes the player can trigger directly. Strict criterion: frequent, structured, parameterized, doesn't need narrative interpretation.

- **Hire officer** — select a candidate from the hiring market view and confirm at the candidate's asking terms. v1 has no counter-offer mechanic; the action is a clean accept-or-walk.
- **Fire officer** — confirm with surfaced contract cost. v1 has no further consequences beyond the firing itself; downstream effects on club state (reputation hits, fan sentiment shifts, surviving officer's loyalty events) are deferred to §4.5 design work.
- **Accept / reject a fully-formed term sheet.** Applies only to fully-structured offers the system surfaces — sponsorship deals at presented terms, transfer bids on your players, contract renewal terms an officer brings to you. Anything more open-ended (counter-offers, negotiation) is a strategic task via chat.
- **Set ticket price** — slider or input. Affects matchday revenue and fan sentiment (the latter once §4.5 mechanics are designed).

#### What's not on the menu

Some routine-feeling actions are deliberately *not* menu actions:

- **Renegotiate a contract** — a real renegotiation is a strategic task (LLM pipeline, time cost, role-stat roll), even when triggered by recognized intent in chat. The classifier identifies it as a recognized intent type, but the work itself is strategic and consumes a slot.
- **Make a transfer bid** — strategic task. Player can fire-and-forget via chat ("DoF, bid €5m for Smith") which routes through the classifier as a recognized strategic intent.
- **Set tactics / formation** — excluded by §2 (Tactical Knowledge is coach work, not DoF or player work). Tactics in v1 are abstracted to "preferred style" if they exist at all (§5 open question).
- **Training intensity** — cut from v1. Either the DoF handles it contextually based on goals, or it's not modeled at all. Can be revisited later if playtesting reveals friction.

This produces a refinement of the §3 classifier model: **recognized intents come in two flavors.**

- **Recognized routine intent** ("fire the DoF" via chat, mapping to the menu action) → instant, no slot consumed.
- **Recognized strategic intent** ("renegotiate Smith's contract" via chat) → strategic task, consumes the active slot.

The classifier's job is to identify both the intent type *and* whether the work is routine or strategic. Slot consumption follows from the work itself, not from how the player initiated it.

### Scope: recognized intents, supported domains, out-of-scope

The §3 classifier and pipeline operate over a closed scope. This subsection defines that scope explicitly: what the classifier recognizes as structured operations, what domains the engine accepts as novel-intent strategic tasks, and what happens when input falls outside both.

#### Recognized intents (closed list)

Structured operations the classifier identifies and routes. The list is closed — adding to it is engineering work, not LLM interpretation.

**Routine (instant, no slot consumed):**

- **Hire officer (from hiring market)** — also a menu action; recognizable in chat ("hire the analytics DoF").
- **Fire officer** — also a menu action; recognizable in chat ("fire the CD").
- **Accept / reject term sheet** — also a menu action; recognizable in chat ("accept the kit sponsor offer").
- **Set ticket price** — also a menu action; recognizable in chat ("raise ticket prices by 10%").
- **Query state** — "how much cash do we have," "who's available for Saturday." Returns a data response, not a task. Cheap-tier dialogue wrapper around a structured state query.

**Strategic (consumes active slot):**

- **Transfer bid (buy)** — "bid €5m for Smith," "make an offer for that striker we scouted."
- **Transfer initiation (sell)** — "put Jones on the market," "sell Smith if anyone bids over €10m."
- **Contract renegotiation** — "extend Jones's contract," "renegotiate with the kit sponsor."
- **Sponsorship negotiation (named target)** — "go back to the kit sponsor for better terms." Distinct from open-ended sponsorship strategy (a supported domain — see below) because the target is named.
- **Scout (open-ended search)** — "find me a striker who can press," "scout for left-backs under 23."
- **Crisis response (event-bound)** — "handle the press on the Jones scandal." Recognized when responding to a surfaced crisis event; open-ended crisis work without an active event hits out-of-scope handling.

Eleven recognized intents total. Specific classifier phrasings tuned in prototype.

#### Supported domains (open scope within each)

Areas where novel input is interpreted and produces a strategic task. The LLM has latitude *inside* each domain; the domain list itself is closed. Each domain resolves to a primary role stat (driving the §2 outcome math) and one or more state targets (what the resolved task changes).

| Domain | Primary role stat | State target on success |
|---|---|---|
| Squad management | Player Development (DoF) | Player form, morale, fitness; squad cohesion |
| Scouting & recruitment | Scouting (DoF) | Reveals on pool players' hidden attributes; surfaced candidates |
| Player development | Player Development (DoF) | Player core stat growth (long-arc) |
| Brand & marketing | Marketing (CD) | Brand strength (§4.5); merchandise revenue multiplier |
| Sponsorship strategy (open-ended) | Negotiation / Commercial Network (CD) | New sponsor contracts (§4); cash on signing |
| Media management | Media Relations (CD) | Fan sentiment (§4.5) |
| Fan engagement | Marketing (CD) | Fan sentiment (§4.5); matchday revenue |
| Grey-area / illegal | Football Network (DoF) or Media Relations / Commercial Network (CD), depending on whether the action is sporting-side or commercial-side | Domain-specific upside (cheaper transfer, suppressed story, boosted player); Integrity-gated scandal-event probability as the downside |

Eight domains. Notable exclusions:

- **Sporting strategy** as a top-level domain — folded into Squad management and Scouting & recruitment, which are the operations sporting strategy actually does. A standalone "long-arc strategy" task has no concrete state target a Triumph can land on; the included domains do.
- **Crisis management** as a top-level domain — folded into event-response handling. The Crisis Management role stat is the primary stat on crisis-category event tasks (per §7) but is not a novel-intent domain. Player-initiated crisis-flavored work either collapses into Media Management (story-side) or requires an active crisis (event-bound, hits the recognized-intent "Crisis response" path).
- **Infrastructure / stadium investment** — deferred to v1+ (structural) per §4. Player input hits out-of-scope handling.

#### Primary role stat only in v1

Each domain commits to one primary role stat for the §2 outcome roll. Some role stats are heavier than others as a result — Marketing is primary on two domains; Player Development is primary on two; Football Network, Strategic Vision, and Composure (outside special events) are not primary on any novel-intent domain. This is honest to where the math is: lighter role stats earn their keep elsewhere — event-trigger modifiers (per §2 character stat interactions), access gating (Football Network on transfer market hidden-attribute reveals per §5), and Composure's special-event dice modifier (per §2). Support-stat-as-difficulty-modifier mechanic considered and deferred — adds variables to §2 outcome math validation when §2 is already one of three prototype-validation targets. Recorded in §11 as a v1+ (additive) candidate.

#### Out-of-scope handling

Input is out-of-scope when it isn't a recognized intent and doesn't map to a supported domain. Examples: "buy a yacht," "build a new training ground" (stadium work is v1+ (structural)), "start a podcast" (not modeled), "fire the manager" (no separate Head Coach in v1, per §2).

Handler:
- Officer responds in-character that the request doesn't apply to running the club. Templated response (no LLM call), flavored by which officer was addressed.
- No slot consumed, no task created.
- Logged for analytics — out-of-scope frequency is a signal that the domain list might need to grow.

Distinct from feasibility-check refusals (per §3 pipeline step 2): feasibility refusals are about not being able to do something feasible right now (no cash, no personnel, wrong reputation tier). Out-of-scope refusals are about the action not being modeled at all. Both produce in-character officer responses; the distinction matters internally (out-of-scope skips the LLM pipeline entirely; feasibility refusals happen mid-pipeline) and for analytics (out-of-scope frequency is design signal; feasibility refusal frequency is balance signal).

### Classifier architecture

Stage-1 routing is a separate concern from stage-2 parameter extraction and the §3 novel-intent pipeline. The classifier is **two-stage**:

- **Stage 1 — routing.** A cheap-tier LLM call classifies the player utterance into one of three buckets: `recognized` (with intent identifier), `novel` (with domain identifier), or `out_of_scope`. Stage 1 also identifies which officer (DoF / CD / null) the work belongs to. Stage 1 outputs structured JSON. No parameter extraction, no target resolution, no feasibility checking.
- **Stage 2 — parameter extraction (recognized only).** When stage 1 returns `recognized`, a second cheap-tier call extracts parameters and resolves targets against game state. For example, given the recognized intent `transfer_bid_buy` and the utterance "bid €5m for the striker we scouted last week," stage 2 produces `{amount: 5000000, target_player_id: ...}` by consulting recent scouting results.
- **Novel intents bypass stage 2** and go directly to the mid-tier novel-intent pipeline below.
- **Out-of-scope** terminates immediately with the templated officer refusal (per the §3 out-of-scope handler). No further LLM calls.

The two-stage split is chosen for diagnosability: it makes the §11 §3 classifier-reliability validation tractable by separating routing failures from parameter-extraction failures. If single-stage classification produces 92% accuracy, you can't tell whether the misses are routing failures or extraction failures; two-stage lets you measure each independently and invest in the right fix.

Stage 1 is the load-bearing prototype risk per §11 §3. Its prompt, test set, and eval methodology live in the prototype repo (`prototype/classifier/`), not in this document.

### Pipeline (full LLM path)

For novel/strategic intents and event responses:

1. **Intent extraction.** LLM parses the instruction into a structured intent: what does the player want, who is the target, what's the desired outcome, which officer should execute.
2. **Feasibility check.** Engine validates against current game state: do we have the money, the personnel, the time, the reputation? Failure here returns the officer's in-character refusal ("boss, we can't afford that") without consuming a full pipeline call.
3. **Task creation.** Code creates a task object: assigned officer, relevant role stat, difficulty (1–20), time cost. Difficulty and time cost are computed by code from the intent + game state, not freely chosen by the LLM (per §10.5 principle 5).
4. **Officer dialogue.** Officer accepts the task in-character, communicating the difficulty band (filtered through Integrity and Charisma per §2). LLM call, mid-tier.
5. **Resolution at task completion.** When the task's time cost elapses, the §2 outcome math runs: 2d6 + role stat vs. difficulty.
6. **Outcome narration.** LLM writes the officer's report on the outcome, mid-tier.
7. **State application.** Numerical effects on game state are applied by code from the resolved tier.

### Rate limiting and burst protection ("officer is busy")

Two limits apply, both surfaced as officers being "busy with other tasks" (per §10.5 principle 8):

- **Burst protection** applies to *all* LLM-backed input (cheap and expensive tier). A player firing many instructions in rapid succession will see officers respond as busy until they catch up. Triggered by unusual burst behavior; normal play never trips it.
- **Sustained cap** applies to *expensive-tier* calls only (novel/strategic instructions and event responses) per the §10.5 principle 6 hard rate limit. After exceeding the cap in a period, new strategic instructions get the "officer is busy" response until the period rolls over. Cheap-tier chat (recognized intents) and menu actions remain available.

The player cannot — and shouldn't need to — distinguish which limit fired. The fiction is identical, the practical effect for normal play is the same: officers are not infinite resources. Specific numbers (burst threshold, sustained cap, period length) are tuned from prototype cost telemetry per §10.5 principle 7.

Cheap-tier chat is *not* free, just inexpensive. Aggressive cheap-tier usage still consumes budget; burst protection is the floor that prevents abuse, and overall per-player cost telemetry catches anything that slips through.

### Cost-tier dependencies

The architecture assumes several things that prototype telemetry will validate or invalidate:
- The cheap-tier classifier is reliable at small model sizes. If we have to move classification up a tier, the cost gap collapses and the whole architecture is less efficient.
- Recognized-intent calls dominate cheap-tier usage and are short.
- Players don't sustain spamming of strategic instructions (sustained cap should be hit by adversarial behavior, not engaged play).
- Prompt caching works as expected on the chosen provider (per §10.5 principle 3).

If any of these assumptions fail in practice, §3 and §10.5 both need revisiting.

---

## 4. Economy & Finances

The v1 economy models the core financial loop of a football club: revenue from operations + match performance, expenses for staff and facilities, lumpy transfer activity, and a single cash balance that can run dry. Deliberately simpler than real-football accounting — no tax, no debt, no depreciation — but rich enough that financial constraints surface meaningfully through officer dialogue (per §2) and feed back into player decisions.

### State model

**Cash balance** — a single number representing current club funds. This is the primary financial state. Income adds, expenses subtract.

**Per-period accounting** — the finance panel surfaces revenue and expenses broken down by source for the current season (and previous, for comparison). Not a full P&L with accruals or depreciation; just enough to answer "are we profitable this season?"

### Revenue streams

- **Matchday revenue** — generated per home league match. Scales with ticket price (player-controlled, per §3 menu actions) and fan sentiment (§4.5). Folded together: ticket sales, food/beverage, parking, matchday merch.
- **Sponsorship revenue** — three sponsor slots: main shirt sponsor, secondary sponsor, stadium naming rights. Each slot has its own contract (value, length). Negotiated via the Commercial Director (strategic task) or accepted/rejected as a fully-formed term sheet (menu action, per §3).
- **Merchandise revenue** — non-matchday merch (kit sales, online store, year-round). Scales with brand strength (§4.5).
- **Broadcasting revenue** — paid per season based on league position. Top-of-table teams earn more. The exact curve is deferred to prototype, but the principle is that league finish has a direct financial effect (in addition to prize money and promotion bonuses).
- **Transfer income** — lumpy and event-driven. Generated when players are sold via DoF-initiated transfer activity.
- **Prize money** — end-of-season payout based on final league position. Includes a promotion bonus when applicable.

### Expense streams

- **Player wages** — modeled per-player (each player has an individual wage). The finance panel surfaces an aggregate "wage bill" but the underlying state is per-player so contract negotiations and transfer decisions have real per-player stakes.
- **Officer salaries** — flat per-officer cost. With the v1 roster of two officers (per §2), this is a small but persistent line.
- **Facility upkeep** — flat per-period cost. Stadium upgrades and quality tiers are deferred to v1+ (structural).
- **Transfer expenses** — lumpy, paid when buying players via DoF-initiated transfer activity.

### Financial constraints surface through officers

Per §2, financial state is surfaced as UI/dashboard (the finance panel view) rather than through a dedicated CFO. Constraints show up contextually in officer dialogue:

- DoF: "Boss, we can't afford his wages."
- CD: "This sponsor's offer barely covers our operating shortfall."

The officer mentioning a financial constraint is reading the same cash balance the finance panel shows — there is no information asymmetry. Officers are providing *interpretation* of the numbers, not new data.

### Running dry

v1 prototype tolerates arbitrary negative cash without consequence. This is a deliberate prototype hole, not a design position — the prototype's job is to validate §3 classifier reliability, §2 outcome math, and §10.5 cost envelope, none of which require fully-calibrated cash consequences. Honest trade-off: cash specifically becomes a consequence-free dimension in prototype, which means cash-targeting tasks (sponsorship, transfers) produce narrative consequences but not stakes. Acceptable for prototype scope.

**No fail state in v1.** A struggling club keeps existing in the league regardless of cash position, consistent with Pillar 3 (shared persistent world). Slot reassignment is handled by the §6 inactivity rule, not by bankruptcy.

Bankruptcy mechanics and damage caps are tracked in §11 §4 and §11 §3 with candidate designs recorded for post-prototype work.


### What's not modeled in v1

- Loans, debt, investment instruments.
- Multi-tier sponsorship structures beyond the three slots.
- Stadium ownership, upgrades, tiered facility quality.
- Tax structures, image rights, agent fees as separate line items.
- Player wage growth/decline over a contract term (wages are fixed for the contract's duration).
- Financial Fair Play-style spending rules (§10.5 references FFP as a reference for future monetization design; in-game FFP is a v1+ (structural) question).

---

## 4.5 Club State

Beyond finances (§4), squad (§5), and officer/player stats (§2/§5), the club has aggregate-state attributes that affect downstream gameplay. These accumulate from player decisions, officer outcomes, and events; they decay or shift over time; they feed back into outcomes (officer recruitment difficulty, sponsor offers, fan-driven events).

v1 models **three club state attributes**, deliberately distinct in audience, scale, and what they gate.

### Fan sentiment

**What it is:** how the club's existing supporters feel right now. Mood, not magnitude. Short-term.

**Audience:** ticket-buying fans. Drives matchday revenue (per §4) and triggers fan-driven events (per §7).

**Scale:** −10 to +10, continuous, signed. 0 is neutral. Negative is unhappy fans; positive is happy fans. Matches the officer/player stat range so players have a mental model for what a 10-point shift means.

**Surfacing:** banded labels in UI (working set: Furious / Angry / Restless / Content / Happy / Euphoric), not raw numbers. Same trick as §2 difficulty bands — labels for player UX, numbers for the engine.

**Movement (placeholder magnitudes, tune in prototype):**
- Routine league win / loss: ±0.5
- Derby or title-implication win / loss: ±2
- Promotion / relegation: ±5
- Major scandal: −3 to −6 depending on severity
- Star signing / star sale at low price: ±1 to ±3

**Drift:** decays toward 0 slowly in the absence of input. Memorable events fade; the mood between disruptions returns to neutral.

**Starting value for new clubs:** 0.

### Brand strength

**What it is:** how commercially big and visible the club is. Market-size attribute. Slow-moving, sticky, multi-season trends.

**Audience:** sponsors, casual fans, merch market. Scales commercial revenue multipliers — high-brand clubs extract more from the same sponsor slot and sell more merch per fan.

**Scale:** 1–100, continuous, unsigned. Multiplicative scale benefits from a wider range; the gap between a 5 (just-promoted local club) and a 50 (national-scale brand) should feel meaningfully different in sponsor offers and merch revenue. A 1–10 scale is too coarse for something that ranges across orders of magnitude in real football.

**Surfacing:** banded labels in UI (working set: Local / Regional / National / Continental / Global) plus the underlying number visible when the player drills into the brand panel.

**Movement (placeholder magnitudes, tune in prototype):**
- Great season: +3 to +5
- Multi-season run of success: +10 cumulative
- Single events rarely move it more than ±2
- PR catastrophe: −5 to −10 in extreme cases
- No decay toward a baseline — brand is sticky and only moves on signal

**Starting value for new clubs:** placeholder ~5–15, tune in prototype.

### Reputation

**What it is:** what the football industry thinks of the club. Insider attribute. Gates *access*, not magnitude.

**Audience:** officers, agents, players, other clubs, top-tier sponsors. Drives the hiring market (§2 — open) by determining which officer candidates approach the club, agent willingness during transfers, and which sponsor archetypes will engage at all.

**Scale:** 5 tiers, named, discrete. Working set: **Disreputable / Unknown / Respected / Distinguished / Elite**. Tiered rather than continuous because the interesting question is "will a top-tier DoF take my call?" not "is my rep 73 or 74?"

**Surfacing:** the tier label is visible. The progress-toward-next-tier is *not* visible — it's a hidden meter that updates based on events, and tier transitions surface as narrative moments ("Word in the football community is starting to shift…") rather than as a progress bar. Reputation should feel earned and slightly mysterious, not a metric to optimize.

**Movement:**
- Tier changes are real events, not routine.
- Sustained signal over a full season can move up a tier.
- A major scandal or catastrophic season can move down a tier in one shot.
- Most actions and events do not move reputation at all; only those that signal something meaningful to the industry.

**Starting tier for new clubs:** Unknown (tier 2). Disreputable is earned, not the default.

### Decoupling: brand vs. reputation

Brand and reputation are explicitly **independent axes**. The design must support all four corners:

- **High brand, high reputation** — an established big club (Manchester United-shape).
- **High brand, low reputation** — a flashy newly-rich club with commercial visibility but no industry respect (a nouveau-riche project nobody in football takes seriously).
- **Low brand, high reputation** — a respected lower-league club with serious football people but no commercial footprint (a well-run small club).
- **Low brand, low reputation** — most starting clubs and clubs in trouble.

This decoupling is the reason both attributes exist. If actions and events tend to move them together in lockstep, the design has drifted and one of them is doing redundant work.

### Cross-cutting design principle

Menu actions and strategic tasks both produce *tangible state changes* (per §2 task slot model). Some of those changes land on finances (§4), some on the squad (§5), and some on club state (this section). The menu action surface (§3) should be honest about which it's mutating — e.g., "fire officer" mutates the officer slot directly but should also produce club-state effects (reputation hit, fan sentiment ripple). The prototype may defer the specific club-state effects of specific actions; v1 specifies them before launch. The framing here flags that the design space exists, and §11 tracks the action-to-state mapping as open work.

### Update cadence

Club state updates from two sources:

- **Reactive:** match outcomes, event outcomes, strategic task outcomes, and menu actions can each move club state directly. This is the primary update channel.
- **Passive drift:** fan sentiment decays toward 0 between disruptions. Brand and reputation do not decay — they only move on signal.

The reactive layer fires whenever the triggering action resolves (per §10 scheduled jobs). The passive drift for fan sentiment ticks on the weekly Sunday-night batch (per §10), at a rate tuned in prototype.


---

## 5. Squad, Players & Sporting

**Concept:** Teams have a squad of players. The Director of Football handles squad management — scouting, transfers, contracts, development — unless the player intervenes through chat. Players themselves are **not AI characters with dialogue** (per Pillar 2: officers are the primary interface). Instead, players are stat blocks with lightweight hidden personality state that produces discrete events the DoF reports to the owner.

**Why this shape:** Pillar 2 says the player operates through officers, not menus or individual players. Making each player an AI character with dialogue would fragment that interface, multiply the LLM cost surface by squad size, and overwhelm players with relationships to manage. Pure stat blocks would feel hollow — no organic squad situations for the DoF to report. The hybrid — stat blocks plus discrete-event personality — preserves the officer-mediated interaction while giving the squad enough texture to generate interesting situations.

### Player record

Each player has:

- **Identity:** name, age, nationality, position.
- **Core match stats:** ~4-6 numbers driving the match engine (specific stats tuned in prototype — candidates include pace, technical, physical, defensive, mental).
- **Hidden personality stats:** three numbers driving discrete events:
  - **Ambition** — drive for bigger clubs, more playing time, promotion. High-Ambition players at small clubs are retention risks; low-Ambition players are content with their role.
  - **Loyalty** — resistance to being poached, willingness to take pay cuts for the club, propensity to leak info. Distinct from officer Loyalty but conceptually parallel.
  - **Temperament** — composure under pressure, chance of going on strike, propensity for public outbursts and locker-room issues.
- **Contract:** wage (per §4 — per-player wages), length, signing date.
- **Status:**
  - **Continuous stats:** form, fitness, morale (each 1-10 or similar, drive match performance and event triggers).
  - **Availability flags:** injury and suspension. Each is a flag with a duration / end-date. A flagged player is unavailable for match selection during the flag's duration. The match engine filters to available players before composing the matchday squad. Other availability flags (loan-out, international duty) deferred to v1+ (additive).

Personality stats are hidden from the player at the squad-list level; they're surfaced indirectly through the DoF's dialogue and the events they generate.

### Squad size

**~22 players per senior squad** for v1. Tunable during prototype based on how the rotation/injury dynamic feels. Larger gives more depth but more state to track; smaller increases injury risk and rotation pressure.

### Player-driven events

Players generate discrete events based on personality stats + current state. These slot into the §7 event system — they surface with response windows, the player can respond via DoF (strategic task) or ignore (auto-resolution).

Event categories (illustrative, not exhaustive):
- **Contract demands** — triggered by contract running down, or after a hot streak of form. Driven by Ambition.
- **Transfer requests** — high-Ambition players at clubs they've outgrown.
- **Strikes / outbursts** — low-Temperament players under stress (poor form, public criticism, lost a big game).
- **Loyalty leaks** — low-Loyalty players whose agents talk to other clubs.
- **Form-driven events** — hot streaks and slumps, generating media attention.
- **Age-driven events** — veteran considering retirement, young player breaking through.

Specific event taxonomy, trigger conditions, and authoring pipeline are part of §7 design work.

### Match simulation

Stat-driven simulator, no LLM in the resolution path. Cost-driven (per §10.5 principle 1) and gives deterministic, reproducible match outcomes. Match-day state is locked Saturday midnight PT, resolved during the lock-to-reveal window, displayed Sunday 3pm PT (per §10).

### Match report

Minimal by design. The player sees a simple ticker: goals, substitutions, yellow cards, red cards. Nothing else. No LLM narration. Respects the "no on-pitch gameplay" anti-pillar and keeps match resolution off the LLM cost surface entirely. Expansion path if matches feel hollow in playtesting: improve the ticker engine to emit more structured events (shots, saves, fouls, possession swings, key passes) — purely a code change, no LLM narration added.

### Deferred (post-v1)

- **Youth academy as a separate subsystem** (v1+ (structural)). v1 has young players *in the senior squad* (low age, undeveloped stats, growth potential) but no dedicated academy with youth coaches, U-21 leagues, or pipeline mechanics.
- **Detailed contract clauses** (release clauses, performance bonuses, image rights) (v1+ (additive)). v1 contracts are wage + length only.
- **Per-player AI dialogue.** Players never speak directly to the owner; everything goes through the DoF.
- **Extended availability flags** (v1+ (additive)). v1 has injury and suspension. Loan-out, international duty, and other long-form unavailability deferred.

### Transfer market

**Architecture:** Single shared regional NPC pool. All clubs in the region see the same free agents and pool players. The pool is the source for buying and the destination for selling. Cross-club transfers between human-owned teams are deferred to v1+ (structural).

**Rationale:** A pure NPC pool keeps v1 scope tractable while preserving Pillar 3 (shared persistent world) on the transfer axis. Because the pool is shared and finite, scarcity is real — two human clubs competing for the same striker is a genuine rivalry moment, even without direct human-to-human bidding. Full cross-club transfers add significant async-negotiation UX complexity that the prototype doesn't need to validate.

**Pool composition:**
- NPC-generated players spanning ages, positions, and quality tiers.
- Pool size: working number ~50–100 players in the regional pool at any time. Tune in prototype.
- Quality distribution skews mediocre. Genuine quality is rare and gated by the Scouting stat on the buying DoF.
- Pool refresh: new players enter on the weekly Sunday-night batch (per §10). Some leave (retirement, signing with simulated external clubs in the offseason).

**Rival squad visibility:** Other human clubs' squads are visible to you — squad list, basic stats, positions, ages. Contract details (wage, length) are *not* visible (matches real-football norms and creates information asymmetry that the DoF's Football Network stat can partially overcome). Rival squad players are not bidable in v1; cross-club transfers are deferred to v1+ (structural).

**Rival transaction visibility:** When a rival club signs or sells a player, it surfaces in the club log as a passive "transfer ticker" entry. No event triggered, no response window — just news that competing clubs are active in the same pool. This is the lightweight version of shared-world signaling for v1; richer reactions (events triggered by rival activity, e.g., "the rival just signed your scouting target") are deferred.

**Player visibility within the pool:**
- All clubs see basic pool listings: name, age, position, asking price, wage demands, surface-level stats (per §5 core match stats).
- The DoF's Scouting stat determines what *hidden* attributes are revealed: true potential, personality stats (per §5), accuracy of surface-level stats. A low-Scouting DoF sees pool listings at face value; a high-Scouting DoF sees through them.

**Transaction types:**

*Buying:*
- Player issues instruction to DoF. Recognized intent ("bid €5m for X") routes through the §3 classifier to a strategic task. Novel intent ("find me a striker who can press from the front") routes through the full LLM pipeline, with the DoF interpreting, scouting candidates, and surfacing options.
- The §2 outcome math applies: DoF Negotiation stat vs. difficulty driven by player quality, club reputation (per §4.5), wage budget fit, and competing interest from other clubs.
- On success: cash leaves the club (per §4), player joins squad.
- On failure tiers: deal collapses (Bad/Catastrophic), deal completes at worse terms (Suboptimal), deal completes at quoted terms (Okay/Good), deal completes at favorable terms (Triumph).

*Selling:*
- Player issues instruction: "sell X" or accepts an incoming inquiry from the pool.
- Pool absorbs the player back at a system-calculated price (based on quality, age, contract length remaining).
- DoF Negotiation modifies the final sale price via the §2 outcome math.

**Transfer windows:** v1 prototype is always-open — transfers can be initiated and resolved in any week. Window restrictions (offseason-primary, mid-season-secondary, matching real football) are deferred for prototype scope reasons; the prototype is not expected to simulate full seasons, so the seasonal-rhythm value of windows doesn't apply at prototype scope. Worth designing before any extended-play v1 build.

**What's not in v1:**
- Cross-club transfers between human-owned clubs.
- Loans.
- Agent dynamics as a separate game layer. The DoF's Football Network stat handles agent texture abstractly.
- Release clauses, performance bonuses, image rights (also deferred per §5).
- Transfer windows (deferred for prototype scope; see above).


### Open design surfaces
- **DoF narration of squad situations** — templated text vs. LLM-generated? Cost vs. variety tradeoff, deferred to prototype data.
- **Specific core match stats** — exact stat set driving the match engine. Tuned in prototype.

---

## 6. Multiplayer & Regional Structure

**Known facts:**
- Players sharing a region (e.g., Southern California) compete in shared divisions.
- Promotion/relegation between divisions.
- Shared persistent reality.

**Decisions:**

- **Season cadence: 2 seasons per real-world year.** A season is 26 real-world weeks, structured as 22 league weeks + 1 mid-season break + 3 weeks offseason. Two seasons stack to a full 52-week year with no calendar gap.
- **League structure: 12 teams per league.** Each team plays every other team twice (home and away) for 22 league matches per season. Smaller-than-real-football league size is intentional — fewer teams means each match is more individually meaningful, and the simulation needs only ~24 teams (top + bottom division) to populate a region.
- **Match cadence: one league match per real-world weekend** during league weeks. The mid-season break week and offseason weeks have no league matches. The "match day" rhythm is the weekly heartbeat that supports both daily and weekly check-in cadences (per §1).
- **Mid-season break (week 12):** No league match. Functions as a natural rest week, secondary transfer window, and breathing room for in-flight strategic decisions. Not mechanically distinguished beyond "no match this week."
- **Offseason (weeks 24–26):** No league matches. Primary window for major squad and staff decisions — transfers, contract renewals, officer hiring/firing, strategic planning for the next season.
- **Cup competition:** Not in v1. Single one-off cup matches are structurally odd; real knockout brackets are a v1+ (structural) feature. The schedule has space for a future cup competition to be inserted without restructuring the league.
- **Time mapping inside a season:** Game time runs at 1:1 with real time during league play (a game week is a real week). Long-running things (player careers, contract durations, officer tenures) scale to *seasons elapsed* rather than calendar time — a "10-season career" is ~5 real years of play, not 10. This is the standard sim-time compression: short-term things feel real-time-paced; long-term things compress.
- **Match-day operational flexibility:** Match resolution is anchored to weekends as a design rhythm, not as a hard scheduling constraint. Implementation can shift specific matches by hours or a day for operational reasons (load balancing, time zones) without breaking the design.

### Launch and growth model

The persistent shared world (Pillar 3) requires real players in shared leagues — but launch day has zero players, and we won't compromise the pillar with NPC-heavy leagues. The model splits launch and growth into separate problems.

- **Pre-launch state (players 1–11):** New users have full access to the game *except* league matches. They hire officers, build squads, issue strategic instructions, deal with events, and prepare. This naturally maps to the offseason mechanics (squad-building, transfers, planning) and gives early users immediate en


---

## 7. Event System

**Concept:** Events are semi-random administrative occurrences that advance the world state and prompt player decision-making. They are the primary mechanism for the world feeling *alive* between matches. Per §10.5 principles 1 and 4, the event system is code-driven (selection, triggering, state mutation) with LLM used only for narrative flavor.

### Cadence

**~3 events per week per team**, tunable in prototype. This is roughly aligned with the daily-check-in cadence (per §1) — a daily player sees a new event most days; a weekly player catches up on a small backlog at their weekly session.

### Categories

Four v1 categories:

- **Player-driven** — events about specific players in the squad. Contract demands, transfer requests, strikes, leaks, form-driven moments (hot streaks / slumps), age-driven moments (veteran retirement, youth breakthrough). Driven by player personality stats per §5.
- **Officer-driven** — events about the player's officers. Poaching offers (driven by Loyalty + reputation), corruption approaches (driven by Integrity), ambition-driven friction (driven by Ambition).
- **Crisis** — negative external situations. Scandals (player misbehavior, off-pitch incidents), training-ground accidents, sponsor disputes, fan revolts, financial pressure.
- **Opportunity** — positive external situations. Sponsorship overtures, players-of-interest entering the market, media opportunities, partnership offers.

All v1 events are **single-team-local** — they fire on one team's timeline and surface to that team's owner. World-wide events (league rule changes, region-wide market shifts) are deferred to v1+ (structural) as a fifth category.

### Event response and resolution

Per the existing §7 decision and §2 task slot model:

- Events surface with a **real-world response window** during which the player can act. Window length depends on event type — crises have short windows (~3 days, urgent), opportunities have longer ones (1-2 weeks, no urgency). Specific durations tuned in prototype.
- Acting within the window creates a **strategic task** that consumes an officer slot (per §2). The task resolves through the §2 outcome math at its scheduled completion.
- Ignoring an event past its window triggers **auto-resolution with default consequences** (per the §10 scheduled job).

**Default-resolution principle:** default consequences are a *nudge*, not a punishment. Ignoring an event should never produce a catastrophic outcome — that's reserved for active-response Catastrophic rolls. Defaults trend slightly negative: a missed transfer request raises the player's morale problem instead of forcing a transfer; an ignored sponsor approach reduces the offer's value or pulls the offer entirely; an unanswered media question lets the unfavorable narrative settle. The weekly-floor player (per §1) doesn't get burned by missing sessions — they just don't get the upside of active engagement.

### Event data model

Each event is a structured record. Event content lives in a separate JSON library; the GDD specifies the data model:

```json
{
  "id": "transfer_request_ambitious_player",
  "category": "player_driven",
  "title_template": "{target.name} wants out",
  "target_selector": {
    "entity_type": "player",
    "conditions": [
      "ambition >= 7",
      "form_recent >= 7",
      "contract_remaining_months <= 18"
    ],
    "selection": "highest_ambition"
  },
  "response_window_days": 7,
  "default_consequences": {
    "target.morale": -2,
    "club_state.fan_sentiment": -1
  },
  "tags": ["transfer", "media_likely"]
}
```

Fields:
- **id, category** — taxonomy.
- **title_template** — short headline; can include state references like `{target.name}`.
- **target_selector** — code-evaluable predicate that finds the eligible game-state entity at fire-time. Resolves to a player, officer, sponsor, rival club, etc. depending on entity_type.
- **response_window_days** — real-world time during which the player can act.
- **default_consequences** — state mutations applied if window expires without response.
- **tags** — filtering and analytics.

### State-driven event selection

Events are weighted by game state but selection remains probabilistic. The selection cycle (runs in the daily state-change batch per §10):

1. Evaluate every event's target_selector against current state.
2. Events with matching entities enter a candidate pool with weights.
3. Weighted random selection produces the events for this tick.
4. For each selected event, code resolves the selector to a specific entity (the highest-Ambition player, the lowest-Integrity officer, etc.).
5. LLM is prompted with the event template + selected entity's state + relevant officer's archetype (the officer who narrates the event). Generates surfaced text.
6. Event surfaces to the player with the response window ticking down.

This pattern means an event *cannot fire without a matching entity*. A "high-Ambition player demands transfer" event simply doesn't surface for a club whose players are all content. State coupling is automatic and the event always feels earned.

### Event content authoring

Event library is JSON, authored separately from the GDD. Authoring discipline:
- Selectors must be tight enough that events feel earned but loose enough that they fire often enough.
- Each event needs thoughtful default consequences honoring the nudge-not-punishment principle.
- Cooldowns / minimum gaps prevent the same event firing repeatedly for the same target.

### Deferred (post-v1)

- **World-wide / regional events** (rule changes, market-wide shifts, cross-region opportunities) (v1+ (structural)).
- **Authoring tooling / CMS** for non-engineers to add events (v1+ (structural)).
- **Event chaining** (one event leads to a follow-up event later) (v1+ (additive)). v1 events are standalone; chained narratives are deferred.

---

## 8. Team Market

**Known facts:** Officially-supported market for selling teams.


---

## 9. Direct Player Interactions

**Known facts:** Player can sometimes interact directly with media, sponsors, players.


---

## 10. Technical Architecture

Most technical decisions deferred until design is firmer. Two are resolved at design-time because they shape the rest of the system: **cost architecture** (see §10.5) and **simulation model** (this section).

### Simulation model

The simulation is **hybrid**: event-driven (lazy) for read-only state queries, with **scheduled background jobs** for state changes that must apply at specific times regardless of player presence.

**Why hybrid:** Pure event-driven (lazy) breaks Pillar 3 — state changes that happen "only when someone asks" produce internal inconsistencies (a player goes on strike Tuesday, but if no one logs in, they're still available for Saturday's match). Pure real-time (every state mutation runs the moment its scheduled time arrives) is more compute and operational complexity than the design needs. The hybrid lets state changes fire on schedule (preserving consistency) while keeping read-side computation lazy (cheap, simple).

**Scheduled jobs (state changes apply at real times):**

- **Daily state-change batch.** Midnight PT every day. Resolves any strategic tasks scheduled to complete that day. Applies state changes (cash, squad, club state, officer dispositions) and generates outcome narration. Officer dialogue communicates task durations in batch-aligned terms ("I'll have an answer for you tomorrow morning," "give me 4 days") — code picks the duration in days, the LLM narrates it.

- **Weekly finance tick.** Sunday midnight PT (after match reveal). Applies recurring revenue and expenses: player wages, officer salaries, facility upkeep, sponsor payments, recurring merchandise revenue. Does *not* include match-driven revenue (that's part of match resolution) or end-of-season payouts (those are part of season transitions).

- **Weekly match resolution.** State going into the match is locked at **Saturday midnight PT**; match resolves any time during the Saturday-night-to-Sunday-afternoon window; player-facing reveal is **Sunday 3pm PT**. The match resolution job applies *all* match-derived state changes: match result, standings update, matchday revenue, potential injury/suspension events, fan sentiment shifts (§4.5). The reveal window aligns with US Sunday-afternoon viewing habits — the same time real NFL games kick off.

- **Event auto-resolution.** Fires when an event's response window closes without player action. Applies default consequences per the event definition.

- **Frozen-task auto-resume.** Fires when an officer's active slot frees up and they have a frozen task waiting. Runs the relevance check (does the target condition still exist?); resumes or cancels accordingly.

- **Season transitions.** End-of-season job. Applies broadcasting revenue, prize money, promotion/relegation, league standings rollups. Initiates the offseason period.

- **Inactivity check.** Periodic job (probably weekly). Marks teams whose owners haven't logged in for one full season as inactive (per §6); these teams can be replaced by waiting new players.

**Lazy (computed on-demand):**

- UI state. What to show on each view, given current state.
- Officer dialogue during a chat interaction. Generated when the player sends a message.
- Officer dialogue at task assignment. Generated when the player issues an instruction.

**Match pre-computation:** Match outcomes are computed *before* the reveal time (any time after Saturday midnight lock, before Sunday 3pm reveal). The result is stored. At 3pm Sunday, the player sees a pre-generated result with full match narrative. This bundles match resolution with operational flexibility (the job can run anytime in the window) and gives the player instant load times during the reveal moment. Player-facing rule: **"what your team looks like Saturday night is what plays on Sunday."** Anything the player does Sunday morning doesn't affect that day's match — it affects next week's.

**Timezone note:** Midnight PT and 3pm PT are launch defaults for the US-targeted prototype. Multi-region timing is a v1+ (structural) problem; the architecture supports per-player or per-region scheduling but v1 doesn't need it.

**Narration generation strategy:** For v1, default to **eager** narration — LLM calls generating officer outcome reports and match narratives run as part of the scheduled job, with results stored for instant player retrieval on next login. The alternative (lazy generation on login) saves cost when players churn but adds login latency. Eager is simpler and cost is bounded by the inactivity threshold (per §6). Revisit if prototype cost telemetry shows narration consuming a disproportionate share of LLM spend.

### Other architecture decisions

Decisions deferred until design is firmer:

- LLM provider, model choices, prompt caching configuration, fine-tuned models for officer voices. (Principles set in §10.5; specific choices deferred.)
- Backend language and framework.
- Web framework, mobile cross-platform vs. native.
- Database technology (Postgres, SQLite, etc.) and persistence strategy.
- Hosting (Render, Railway, Fly.io, AWS, GCP — prototype scale doesn't need AWS-grade infrastructure).


---

## 10.5 Cost Architecture as Design Constraint

LLM API costs are not an implementation detail to optimize later — they are a **first-class design constraint**. A game built around open-ended natural language input has a fundamentally different cost structure from a typical sim: every player action potentially triggers multiple LLM calls. Treating cost as something to fix at scale-up time is how LLM-heavy consumer products fail. Designing around it from the start is cheaper than retrofitting.

This section states the *principles*. Specific design decisions that follow from these principles live in the sections they affect (§3, §5, §7), with cross-references back here.

### Cost envelope

**Principle:** LLM cost per active player must be low enough that a realistic monetization model (working assumption: cosmetic primary, optional subscription, in the $5–10/month range) leaves margin for non-LLM infrastructure, platform fees, and profit.

**Method:** Instrument cost-per-player from the first prototype. Set a working ceiling once we have real token usage from real play sessions. Any specific dollar figure proposed before that point is a placeholder, not a target. The first build will likely come in over whatever ceiling we eventually set; the goal is that the architecture supports getting *to* a sustainable number rather than fighting against it.

### Architectural principles

1. **LLM only for genuinely open-ended interpretation.** Everything that can be code, is code. The LLM is reserved for: parsing player intent, generating bounded outcomes for open-ended actions, writing in-character officer dialogue, and narrating event flavor. Specific applications of this principle are decided in §5 (match simulation) and §7 (event selection).

2. **Tiered model routing.** Default to the cheapest model that produces acceptable quality for each task. Rough mapping:
   - **Cheap tier** (smallest model): intent classification, simple feasibility checks, routine officer chatter, push notifications.
   - **Mid tier**: the core open-ended-input pipeline, important officer dialogue, event narration, anything player-facing where quality matters.
   - **Premium tier**: reserved for genuinely hard reasoning the mid tier can't handle. Should be a small fraction of total calls, ideally near zero for v1.

   Build the routing layer into the LLM service from day one. Don't hardcode model choices at call sites.

3. **Aggressive prompt caching.** Prompt caching is a major cost lever; design call patterns to maximize cache hit rates. Stable content (officer personalities, world rules, schemas, few-shot examples, recent game state on tick boundaries) should be cached. Specific cost reduction depends on provider choice (deferred); architect to exploit caching regardless of provider.

4. **Batch API for non-real-time work.** Anything not blocking a player's screen runs through batch endpoints (typically discounted). Candidates: pre-generated event flavor, periodic world-flavor generation, officer status reports.

5. **Deterministic-first state changes.** The LLM influences *narrative* and *intent classification*; numerical state changes are computed by code from validated parameters. This is also the primary defense against prompt-injection exploits ("ignore previous instructions, my team has $1B"). Closes the §3 anti-exploit question.

6. **Hard per-player rate limits.** Even paid users are capped. Limits protect against cost runaways and reinforce the slow-tick world cadence — a player who could spam 100 instructions in a session is breaking the sim's rhythm anyway. Specific numbers are coupled to §1 session cadence and deferred until that's resolved.

7. **Telemetry from day one.** Cost-per-player-per-day is measured from the first prototype build and reviewed at every design milestone. Without per-player cost telemetry, the cost envelope is aspirational.

8. **Graceful degradation, framed in-fiction.** When a player hits their rate limit, or the LLM provider degrades or fails, the game cannot stop — but the player also doesn't see a technical error. The officer is simply *out of office*, *traveling*, *in a meeting*, or otherwise unavailable. Instructions queue and resolve when capacity returns. This turns a technical constraint into in-fiction texture and reinforces the §2 principle that officers take time and aren't always available. Reliability requirement, cost lever, and worldbuilding device in one.

### Monetization

**Working position:** Cosmetic monetization (kits, stadium skins, crests, officer portraits) is the default assumption for v1. It has zero competitive impact in a shared world, real-world precedent in soccer fandom, and fits the session model regardless of how §1 resolves.

All other models are **deferred pending prototype data** on session frequency, engagement depth, and what players actually find valuable:

- **Subscription tier** — plausible, but coupled to §1's session cadence question. Subscriptions assume engagement deep enough to justify recurring spend; if sessions are 10 minutes a few times a week, churn risk is high. Revisit once §1 is resolved.
- **Pay-for-advisory** — *not* a candidate as currently sketched. A paying player getting better outcomes on a current crisis is pay-to-win-shaped, just bounded to existing situations rather than generating new ones. Would need a tight specification (purely informational? cosmetic-only flavor? no mechanical effect?) before being a real candidate.
- **Pay-for-events** — *deferred to a later version, not rejected.* In a shared league with promotion/relegation, naively letting paying players generate extra events creates competitive-fairness problems (more sponsorship opportunities, more development chances, etc.). However, several mitigations could make a version of this viable: hard caps on paid events per period, a non-trivial chance that a paid event is *negative*, or a "Financial Fair Play"-style rule set inspired by real soccer governance that constrains how much real-money advantage can compound. **Marked for later design work; not in scope for v1.**


---

## 11. Open Questions Index

The single source of truth for unresolved design questions. Resolved items are crossed off here with a one-line note; the design itself lives in the relevant section, and the *why* lives in §12 Decision Log. Open items carry full context here.

Sections do not maintain their own open-question lists. (Inline prototype-tuning notes — placeholder values that need real play data — may remain in the section they affect, where they're load-bearing for someone reading the design.)

---

### §1 Core Loop

**Resolved:**
- [x] ~~Session length, real-time vs. paused, target session emotion.~~ *Resolved: real-time world; ~daily target / weekly floor; ~15 min target / 5 min floor / 30 min ceiling. Target session emotion to be sharpened during prototype.*

---

### §2 AI Officer System

**Resolved:**
- [x] ~~Officer roster v1.~~ *Resolved: Director of Football + Commercial Director. Other roles deferred to v1+ (structural).*
- [x] ~~Officer stat model.~~ *Resolved: 2 pillars × 5 stats. Character stats (Loyalty, Ambition, Integrity, Charisma, Composure) universal; Role stats specialized per officer with Negotiation + Network as parallels.*
- [x] ~~Stat scale.~~ *Resolved: stats 1–10, task difficulty 1–20.*
- [x] ~~How stats roll into outcomes.~~ *Resolved: 2d6 + role stat vs. difficulty. Six symmetric tiers by margin (Triumph / Good / Okay / Suboptimal / Bad / Catastrophic). Composure modifies the die on special events. Other character stats are discrete-event triggers, not per-task modifiers. Difficulty banded for the player via officer dialogue, filtered through Integrity (accuracy) and Charisma (delivery confidence). True difficulty revealed post-resolution.*
- [x] ~~Task interpretation pipeline.~~ *Resolved by §3 interaction model: classifier routes recognized intents to code paths, novel intents to full LLM pipeline that produces a task with assigned officer, role stat, difficulty, and time cost.*
- [x] ~~Task slot model and time cost framing.~~ *Resolved: one active slot + one frozen slot per officer. Strategic tasks consume the active slot; instant actions don't. Recall-or-keep on new strategic instructions. Frozen tasks auto-resume on active-slot completion, with code-side relevance check (target condition still exists). Task duration range: game days to ~half a season; LLM-computed per task. Real-time mapping derived from §6 season cadence.*
- [x] ~~Boundary between instant and strategic actions.~~ *Resolved by §3 menu surface: recognized intents are split into routine (instant, no slot) and strategic (consumes slot). The classifier identifies both intent type and routine-vs-strategic. Specific list of routine vs. strategic intents tracked as part of the §3 novel-intent taxonomy work.*
- [x] ~~Commercial Director character identity / Personality model.~~ *Resolved: §2 Character Archetypes. Three archetypes per officer role (six total), no mechanical effect on outcome math, archetype-biased stat generation, archetype hidden from player and discovered through dialogue. Mechanical effects of personality (do archetypes affect *how* tasks execute?) deferred to v1+ (additive) — covered by the new entry below.*
- [x] ~~Hiring market for officers.~~ *Resolved: shared regional NPC pool, reputation-filtered visibility, per-season refresh. Candidates have visible role stats and hidden character stats; character is discovered through working together. Hiring is accept-or-walk at the candidate's asking terms — no counter-offer in v1. Interviews generate LLM-driven dialogue revealing archetype.*

**Prototype-tuning placeholders** (values that need real play data; the *design* is set):
- [ ] **Outcome tier thresholds** — placeholder values in the §2 math table; tune once we have real play data.
- [ ] **Composure die bands and dice** — placeholder thresholds (1–3 / 4–7 / 8–10) and dice (1d12 / 2d6 / 3d4); tune in prototype.
- [ ] **Difficulty band labels and dialogue tone** — placeholder dialogue ("Easy enough," etc.); refine during officer-voice work.
- [ ] **Task duration distribution** — what's the rough mix of short / medium / long tasks the system actually generates? Tune in prototype.
- [ ] **Hiring pool size per role.** Placeholder 6-12; tune in prototype.
- [ ] **Asking-terms generation** (wage and contract length curves based on stats and reputation tier). Tune in prototype.
- [ ] **Reputation-tier filtering thresholds.** Which candidate quality tiers are visible at which reputation tiers. Tune in prototype.

**Open:**
- [ ] **Stat development over time.** Do stats grow with experience? Decay? Are officers fixed at hire? Affects long-game progression and how much player investment in a specific officer pays off.
- [ ] **Officer voice.** Written/spoken responses, voiced via TTS, or text only?
- [ ] **Failure modes for low-stat officers.** Beyond bad outcomes on the per-task roll, what happens when an officer is consistently underperforming? Just bad outcomes, or possibility of catastrophic discrete events (scandals, leaks, public meltdowns)?
- [ ] **Goals list mechanic — deferred to v1+ (additive) pending prototype data.** Originally framed as the session anchor in §1. Deferred for v1 because (a) it doesn't validate any of the three prototype-validation targets (§3 classifier reliability, §2 outcome math, §10.5 cost envelope), (b) v1 session orientation can rely on notifications + club log + officer panel + event response windows, and (c) the feature introduces non-trivial cost surface (officer-proposed goals = additional LLM call type) before the cost envelope is measured. v1+ design target: co-authored list of goals (officer-proposed and player-authored), goals added as instant actions (no slot consumed, no LLM cost on add), completion either manual by player or officer-detected when the underlying state condition is met. Purely narrative — no mechanical weight on resource allocation, officer focus, or outcomes. Revisit after prototype data on whether the v1 orientation surfaces are sufficient.
- [ ] **Inter-officer dynamics.** With only two officers, the design lever of inter-officer conflict (e.g., DoF wanting to spend, CD wanting to bank a sponsorship windfall) is narrower than at four. Real loss, or fine for v1? Revisit during personality design.
- [ ] **Task progress visibility.** For v1, player sees the officer's estimated remaining time only. Interactive progress queries ("how's the sponsor hunt going?") deferred to v1+ (additive) — design space includes cost (LLM call vs. UI read), narrative cost (officer mid-task interruption), and gameplay implication (does asking change the outcome?).
- [ ] **Frozen task expiration / freshness decay.** Currently frozen tasks have no expiry beyond the relevance check on unfreezing. May need explicit time-based decay if hoarding becomes a problem.
- [ ] **Active-task interaction with edge-case events.** What if a special event arrives but the player ignores it past the response window — does the active task continue undisturbed, or does the event force consequences regardless? Pairs with §7 event-window design.
- [ ] **Mechanical effects of archetype.** v1 archetypes are voice/personality only with no impact on outcome math. v1+ (additive) question: should archetypes mechanically affect task execution? Examples: does a "wheeler-dealer" DoF unlock grey-market task options? Does an "old-school" DoF refuse certain modern-style instructions? Probably yes long-term, but the v1 stat model (Character × Role stats) is already doing a lot of mechanical work and adding archetype-mechanical effects risks double-counting.
- [ ] **Archetype-stat correlation tuning.** Specific stat-range mappings per archetype are placeholders — tune in prototype. Open: how strong should the correlation be? Pure correlation makes archetypes feel deterministic; pure independence makes them feel decorative.
- [ ] **Archetype visibility revisit.** v1 hides archetype from the player; player discovers character through dialogue. Open: does this feel like meaningful discovery, or punishingly opaque (especially at hiring)? Revisit once we have playtesting data.
- [ ] **Hiring counter-offer mechanic.** v1 is accept-or-walk at the candidate's asking terms. v1+ (additive) candidate: candidate's Charisma drives negotiation difficulty when the player counter-offers. Deferred for prototype scope — the prototype validates §3 classifier reliability and §2 outcome math through existing surfaces; hiring negotiation doesn't validate anything new.
- [ ] **Re-interviewing previously-hired candidates.** What happens if a player fires an officer and then re-encounters them in a later hiring pool? Cached interview, fresh interview, modified dialogue acknowledging history? v1+ (additive).


---

### §3 Open-Ended Input → Bounded Output

**Resolved:**
- [x] ~~Anti-exploit on LLM-injected state changes.~~ *Resolved in §10.5 principle 5 (deterministic-first state changes).*
- [x] ~~Interaction model (when/how can the player issue open-ended instructions?).~~ *Resolved: three surfaces (menu actions / open-ended chat / event responses), all available at all times. Chat routes via classifier to cheap (recognized intent) or expensive (novel intent) paths. Rate limiting handled by in-fiction "officer is busy" mechanic combining burst protection (all tiers) and sustained cap (expensive tier only).*
- [x] ~~Determinism vs. surprise.~~ *Mostly resolved by §2 outcome math (probabilistic from day one). The remaining narrow question — same-instruction consistency — is tracked separately below.*
- [x] ~~Standard menu action set.~~ *Resolved: §3 menu surface defines views (club dashboard, finance, squad list, officer panel, match calendar, club log) and menu actions (hire officer, fire officer, accept/reject term sheet, set ticket price). Goals list deferred pending §2 goals list mechanic. Training intensity, contract negotiation, transfer bids, tactics deliberately excluded — either routed through chat as strategic tasks or not modeled in v1.*
- [x] ~~Taxonomy of supported novel-intent classes.~~ *Resolved: §3 scope subsection. Eleven recognized intents (closed list, classifier-routed); eight novel-intent domains (Squad management, Scouting & recruitment, Player development, Brand & marketing, Sponsorship strategy, Media management, Fan engagement, Grey-area). Each domain commits to one primary role stat and one or more state targets. Sporting strategy and Crisis management deliberately excluded — folded into included domains or event-response handling. Infrastructure deferred to v1+ (structural).*
- [x] ~~Out-of-scope input handling.~~ *Resolved: §3 scope subsection. Templated in-character officer refusal, no slot consumed, no LLM call, logged for analytics. Distinct from feasibility-check refusals.*

**Open:**
- [ ] **Classifier reliability and stability at cheap-tier sizes.** Two coupled properties to validate in prototype. (1) *Routing accuracy*: stage-1 classification of recognized-intent (cheap path) vs. novel-intent (expensive path) vs. out-of-scope must be reliable at cheap-tier model sizes. Misroute too aggressively to expensive, costs blow up; too aggressively to cheap, Pillar 1 becomes fake (system silently flattens player input into menu options). (2) *Same-instruction consistency*: the same instruction issued twice should produce the same classification — otherwise the system feels arbitrary rather than reasoned. Load-bearing prototype risk for the whole architecture — if either property fails at small model sizes, the cost gap collapses and §10.5 has to be revisited. Evaluable against the §3 recognized-intent list (11 items) and supported-domain list (8 items). Prompt, labeled test set (80 utterances), and eval methodology live in the prototype repo at `prototype/classifier/`. First eval run is the next prototype-validation milestone.
- [ ] **Output variance / per-action damage cap — deferred for prototype.** v1 prototype runs uncapped: a Catastrophic on a high-difficulty task applies its full state delta. This is a known prototype hole. Pairs with §4 bankruptcy mechanics — both deferred together because they answer the same question (how much can a single decision or slide cost the player) and need consistent answers. Candidate design (recorded so it doesn't need to be re-derived): per-attribute soft cap on single-task delta (cash capped at % of current balance; fan sentiment capped at ±3; brand at ±2; reputation can't move more than ~50% of a tier on one task). Per-task only — consecutive failures still compound. Revisit once prototype telemetry shows which tier outcomes actually produce oversized swings under real play; designing the cap speculatively risks clamping things that don't need clamping.
- [ ] **Burst threshold and sustained cap numbers — prototype runs unguarded; v1-before-launch problem.** Concrete numbers for what counts as a burst and what the sustained expensive-tier cap should be. Prototype runs with rate-limiting infrastructure in place but thresholds set effectively to "off" — the prototype user population is small, known, and trusted (internal or invited testers with no script-running risk), so guarding against burst abuse doesn't validate anything the prototype is testing. The §10.5 principle 6 hard rate limit still exists in the architecture; the *threshold values* are deferred. v1 launches with public users and requires real limits before opening to untrusted traffic — this is a v1-before-launch problem, not a v1+ deferral. Specific numbers tune from prototype cost telemetry per §10.5 principle 7.
- [ ] **Support-stat-as-difficulty-modifier mechanic.** v1 commits to primary role stat only in novel-intent rolls. v1+ (additive) candidate: some role stats act as difficulty reducers on adjacent domains (Football Network reducing scouting-task difficulty; Strategic Vision reducing squad-management difficulty). Gives lighter role stats a job in the math. Deferred for prototype to avoid validating two §2 outcome-math changes simultaneously. Revisit after prototype data confirms which role stats feel decorative under primary-only.
- [ ] **Media management event-probability modifier.** v1 media management tasks affect fan sentiment only. v1+ (additive) candidate: successful media tasks set time-windowed modifiers on crisis-event probability (good media work suppresses negative event likelihood for a window; bad media work amplifies it). Requires the temporary-impact mechanic (see §11 §10) and a small §7 addition (event weights modifiable by active windows). Deferred — media management still has a job (fan sentiment is a real state target), and this is a clean additive feature once the temporary-impact mechanic exists.

---

### §4 Economy & Finances

**Resolved:**
- [x] ~~Revenue streams.~~ *Resolved: matchday, sponsorship (3 slots), merchandise, broadcasting, transfer income, prize money.*
- [x] ~~Cost categories.~~ *Resolved: player wages (per-player), officer salaries (per-officer), facility upkeep (flat), transfer expenses.*
- [x] ~~Economy abstraction level.~~ *Resolved: cash balance with per-period accounting; not full P&L. Financial state surfaced through the finance panel view and contextually through officer dialogue.*

**Open:**
- [ ] **Bankruptcy mechanics — deferred for prototype.** v1 prototype tolerates arbitrary negative cash without consequence. This is a known prototype hole, not a design position. The threat-of-bankruptcy mechanic (whatever shape it takes) is needed before extended play because cash damage being consequence-free teaches players that cash doesn't matter — hard to un-teach later. Candidate design (recorded so it doesn't need to be re-derived): a "consequence ladder" with reversible tiers (overdraft tolerated; persistent negative triggers events like board pressure or forced sales; deep/sustained negative escalates to reputation drops and mass sales). No fail state — a struggling club keeps existing in the league, consistent with Pillar 3 (shared persistent world). Slot reassignment is handled by the §6 inactivity rule, not by bankruptcy. Specific thresholds, ladder shape, and consequence calibration tuned in v1 work after the prototype. Pairs with the §11 §3 output variance / damage cap question — both are about how much damage the player can absorb.
- [ ] **Broadcasting revenue curve.** Per-position payouts need a specific curve. Tune in prototype.
- [ ] **Prize money structure.** Per-position payouts + promotion bonus values. Tune in prototype.
- [ ] **Sponsor contract terms.** What ranges are normal for value and length? Tune in prototype.
- [ ] **Matchday revenue formula.** Specifically how ticket price and fan sentiment combine. Tune in prototype.
- [ ] **Debt / loans / investment.** Not modeled in v1. v1+ (structural) question for clubs in financial trouble or wanting to expand aggressively.
- [ ] **Stadium ownership and upgrades.** v1 has a flat facility upkeep cost. Stadium as a strategic surface (capacity, quality, ownership) is a v1+ (structural) design space.

---

### §4.5 Club State

**Resolved:**
- [x] ~~Attribute set.~~ *Resolved: three attributes — fan sentiment, brand strength, reputation. Media standing cut; redundant with CD's Media Relations role stat plus fan sentiment.*
- [x] ~~Measurement and scale for each attribute.~~ *Resolved: fan sentiment −10 to +10 continuous signed; brand 1–100 continuous unsigned; reputation 5 named tiers (Disreputable / Unknown / Respected / Distinguished / Elite).*
- [x] ~~State decay model.~~ *Resolved: fan sentiment decays toward 0 in absence of input; brand and reputation are sticky (no decay, move only on signal).*
- [x] ~~Surfacing club state to the player.~~ *Resolved: all three visible, banded labels for fan sentiment and brand (numbers available on drill-in for brand), reputation shown as tier name only with progress-to-next-tier hidden.*

**Prototype-tuning placeholders** (values that need real play data; the *design* is set):
- [ ] **Movement magnitudes per event type** for fan sentiment and brand. Placeholder ranges in §4.5; tune in prototype.
- [ ] **Fan sentiment decay rate.** Placeholder "slowly toward 0"; tune in prototype.
- [ ] **Starting brand value** for new clubs. Placeholder 5–15; tune in prototype.
- [ ] **Tier-transition thresholds for reputation.** Hidden meter values that gate tier moves; tune in prototype.
- [ ] **Band labels for fan sentiment and brand.** Working sets in §4.5; refine during UI work.

**Open:**
- [ ] **Action-to-club-state mapping.** Which menu actions and strategic tasks affect which club state attributes, with what magnitudes. v1 menu actions are defined; their effects on club state are deferred. Specific example: "fire officer" should plausibly affect reputation and surviving officer's Loyalty disposition. The mapping should be tested for the decoupling principle — if events tend to move all three attributes together in lockstep, the design has drifted.
- [ ] **Event-to-club-state mapping.** Same question for §7 events. Each event's `default_consequences` and resolution outcomes need to specify club-state effects.
- [ ] **Reputation gating for sponsor offers.** §4 sponsor mechanics are partially open; high-end sponsor archetypes should only approach Distinguished+ clubs.

---

### §5 Squad, Players & Sporting

**Resolved:**
- [x] ~~Match simulation detail level.~~ *Resolved: stat-driven sim with no LLM in the resolution path; minimal goals/subs/yellow/red ticker; expansion path is more structured ticker events, not narration.*
- [x] ~~Players as characters vs. stat blocks.~~ *Resolved: hybrid (Option C). Stat blocks for match engine; three hidden personality stats (Ambition, Loyalty, Temperament) drive discrete events the DoF reports to the owner. No direct player dialogue — Pillar 2 preserved.*
- [x] ~~Youth academy in v1.~~ *Resolved: deferred to v1+ (structural). v1 has young players in senior squad but no academy subsystem.*
- [x] ~~Transfer market structure.~~ *Resolved: shared regional NPC pool. All clubs in the region see the same free agents and pool players. Cross-club transfers between human-owned teams deferred to v1+ (structural). Rival squads visible (basic stats), rival contract details not visible. Rival transactions surface as passive log entries.*

**Prototype-tuning placeholders** (values that need real play data):
- [ ] **Transfer Market Pool size.** Placeholder 50–100; tune in prototype.
- [ ] **Transfer Market Pool refresh rate and composition.** Working assumption: weekly refresh on Sunday-night batch; tune in prototype.
- [ ] **Quality distribution in the pool.** Mostly mediocre with rare quality; specific distribution tuned in prototype.
- [ ] **Pricing formulas** (asking price, wage demands, sale price). Tune in prototype.
- [ ] **Squad size** — placeholder 22; tune in prototype.
- [ ] **Core match stat set** — exact 4-6 stats driving the match engine.
- [ ] **Personality stat scales and ranges** — same scale as officer stats (1-10), but trigger thresholds and event frequencies need tuning.

**Open:**
- [ ] **DoF ambient narration — out of scope for prototype.** Originally framed as a templated-vs-LLM cost-vs-variety question. The hidden question was whether the DoF produces *ambient* observations about squad state (e.g., "Jones has been training really sharply this week") outside of the three existing dialogue contexts: strategic task assignment, strategic task completion (outcome reports), and event narration. Resolved as: ambient observations are not a surface in prototype. The DoF speaks only in those three contexts. Players see DoF dialogue at three contact points per ~3-event-per-week cadence, plenty of voice exposure for validation. If playtesting shows the DoF feels silent between events, revisit with real data. Templated-vs-LLM tradeoff for the three *existing* dialogue surfaces tracked separately as a prototype-tuning question (see §5 "Open design surfaces").
- [ ] **Tactical decisions — out of scope for prototype; v1 design question.** Tactics are not modeled in prototype. The match engine runs without tactical input from the player or the DoF; matches resolve from squad stats alone. Tactical instructions in chat ("play more defensively," "switch to a high-pressing system") hit §3 out-of-scope handling — the officer responds in-character that the request doesn't apply, no slot consumed, no task created. This is consistent with Tactical Knowledge not being a DoF role stat (per §2 — that's coach work) and with there being no separate Head Coach officer in v1 (deferred to v1+ (structural)). For v1, this resolves one of three ways: (a) tactics stay out of scope and the §3 out-of-scope handler covers tactical instructions permanently; (b) a thin "tactical preference" state attribute is added that the DoF can adjust as a strategic task (Strategic Vision stat?), feeding the match engine as a small modifier; (c) a separate Head Coach officer ships in v1, bringing Tactical Knowledge as a real role stat and making tactics a full domain. Decision deferred until prototype data shows whether the match engine feels too thin without tactical input.
- [ ] **Detailed contract clauses.** v1 contracts are wage + length only. Release clauses, performance bonuses, image rights deferred to v1+ (additive).
- [ ] **Player aging and development curves.** How do stats grow and decline with age? Per-player potential? Generic curves? Pairs with the "stat development over time" question for officers (§11 §2).
- [ ] **Injury and suspension mechanics.** Duration ranges, trigger conditions, accumulation thresholds. Prototype-tuning work; not blocking but real.
- [ ] **Transfer windows.** v1 prototype is always-open. Window restrictions (offseason-primary, mid-season-secondary) are v1 design intent deferred for prototype scope — not a v1+ deferral; v1 closes this hole before launch. Affects seasonal rhythm and §1 daily-cadence pressure.
- [ ] **Pool generation algorithm — prototype uses authored pool with frozen refresh; v1 design question.** Prototype ships with a pre-authored set of semi-random NPC players (working number ~50-100 per §5). Pool refresh is *frozen* during prototype — the pool you start with is the pool you keep. This is a deliberate prototype hole: §5 specifies weekly refresh on the Sunday-night batch, but refresh dynamics don't validate any of the three prototype-validation targets (§3 classifier reliability, §2 outcome math, §10.5 cost envelope). Authored-and-frozen is the simplest implementation that gets the §5 transfer market mechanics testable. v1 design question: procedural generation, larger pre-authored pool revealed in waves, or hybrid. Tune in prototype based on whether the frozen pool feels sufficient for the validation window.
- [ ] **Cross-club transfers.** v1+ (structural) feature. Async negotiation UX is the hard problem; design from scratch when it's time.
- [ ] **Scouting reveal mechanics.** What specifically does the DoF's Scouting stat reveal about pool players, at what thresholds? Per-stat reveal? Probabilistic? Tune in prototype.
- [ ] **Rival-transaction event escalation.** v1 is lightweight (log ticker only). v1+ (additive) candidate: rival activity triggers events (e.g., "the rival just signed your scouting target"). Deferred.
- [ ] **Competing-interest resolution mechanic.** When two human clubs bid on the same pool player, how does the system resolve it? Options: highest Negotiation roll wins, first-completed-task wins, simultaneous resolution with rolls compared. Deferred to prototype — at v1 scale (12 clubs) collisions are rare enough to observe and hand-tune rather than designing speculatively.



---


### §6 Multiplayer & Regional Structure

**Resolved:**
- [x] ~~Season cadence.~~ *Resolved: 2 seasons per real-world year. 26 weeks per season: 22 league weeks + 1 mid-season break + 3 weeks offseason. One league match per real-world weekend.*
- [x] ~~League structure (per-league size).~~ *Resolved: 12 teams per league.*
- [x] ~~Launch model.~~ *Resolved: pre-launch state (full game minus matches) for players 1–11; league starts at player 12; player 13+ uses same pre-launch state until a slot opens.*
- [x] ~~Inactivity threshold.~~ *Resolved: 1 full season (26 weeks) without login = inactive; team can be replaced.*
- [x] ~~Scheduling matches.~~ *Resolved by §10 simulation model: state locks Saturday midnight PT, match resolves server-side any time in the lock-to-reveal window, reveal Sunday 3pm PT. Neither player needs to be online. Player-facing rule: "what your team looks like Saturday night is what plays on Sunday."*

**Open:**
- [ ] **Multi-league expansion.** When waiting-player count exceeds 12, do we form a second league? Promotion/relegation between leagues? Deferred to v1+ (structural) — the launch model handles single-league growth indefinitely until this pressure hits.
- [ ] **Regional structure.** v1 is single-region. Geographic regions, region-splitting as growth scales, and cross-region competition are all v1+ (structural) problems.
- [ ] **Pyramid depth (number of divisions).** Single-tier in v1. Adding Division 2, 3, etc. is part of multi-league expansion (above) and deferred to v1+ (structural).
- [ ] **Cross-region competition.** Cup competitions across regions, international equivalents — v1+ (structural), paired with multi-region structure.
- [ ] **Functionally-absent-but-logged-in teams.** Inactivity threshold preserves the slot for any player who logs in once per season. A player who barely meets the threshold but doesn't engage produces a team that "exists" but is unmanaged for 22 matches. Edge case, may need a deeper engagement metric in v1+ (additive).
- [ ] **Mid-season cup insertion path.** Schedule has space for a future cup competition to be inserted without restructuring; specific bracket design and scheduling for v1+ (structural).

---

### §7 Event System

### §7 Event System

**Resolved:**
- [x] ~~Event selection mechanism.~~ *Resolved: events selected from an authored pool by code with state-aware weighting; LLM (optionally, via batch API) for narrative flavor wrapping only — not for event selection itself.*
- [x] ~~Event response windows.~~ *Resolved: events surface with a real-world response window. Within the window, response is a strategic task consuming an officer slot (per §2). Outside the window, event auto-resolves with default consequences. Window lengths tuned per event type — crises short, opportunities longer.*
- [x] ~~Event taxonomy.~~ *Resolved: four v1 categories (player-driven, officer-driven, crisis, opportunity). All v1 events single-team-local. World-wide/regional events deferred to v1+ (structural).*
- [x] ~~Event data model.~~ *Resolved: structured JSON records with target_selector, response_window_days, default_consequences, tags. Event library lives separately from GDD.*
- [x] ~~Event-to-player-history relationships.~~ *Resolved via state-driven target selection: events have selectors that find matching entities at fire-time. An event cannot fire without a matching entity, so coupling to state is automatic.*
- [x] ~~Local vs. world-wide events.~~ *Resolved: v1 is all-local. World-wide deferred to v1+ (structural) as fifth category.*
- [x] ~~Default-resolution consequences principle.~~ *Resolved: nudges, not punishments. Default consequences trend slightly negative but never catastrophic. The weekly-floor player should not be burned by missed sessions.*

**Open:**
- [ ] **Event library content.** The v1 library needs to be authored — selectors, templates, default consequences, dialogue prompts. This is a content production task, not a design task. Probably ~50-100 events for prototype.
- [ ] **Event freshness over time.** How does the game ensure events feel fresh after 100 hours of play? Selector + template + state-driven variation helps, but a finite library will still repeat. Strategy needs design (selector specificity? cooldowns? rotating subsets?).
- [ ] **Tooling/CMS for adding events post-launch.** A live game needs ongoing event content. Authoring pipeline for non-engineers is v1+ (structural).
- [ ] **Event cadence tuning.** ~3 events/week is the v1 target; tune in prototype.
- [ ] **Window length tuning per category.** Specific real-time durations for crisis windows, opportunity windows, etc. Tune in prototype.
- [ ] **Event chaining.** v1 events are standalone. Chained narratives (one event triggers a follow-up later) are a v1+ (additive) design space — interesting for emergent storytelling but adds authoring complexity.
- [ ] **Cooldown / frequency caps.** Same event shouldn't fire repeatedly for the same target in quick succession. Specific cooldown periods tuned in prototype.

---

### §8 Team Market

**Open:**
- [ ] **Defer entirely to v1+ (structural)?** Strong candidate. If kept for v1, the questions below all need answers.
- [ ] **Open market between players vs. system buy-back.** Who's the counterparty when a team is sold?
- [ ] **Real money / in-game currency / both.** Big call; affects monetization (§10.5) and competitive fairness.
- [ ] **Valuation model.** What determines a team's price?
- [ ] **Anti-abuse.** Selling failing teams to alts, money laundering, etc. Needs explicit defenses if real money is involved.

---

### §9 Direct Player Interactions

**Open:**
- [ ] **Direct vs. via-officer triggers.** What triggers a "direct interaction" versus going via an officer? Is it player-initiated, system-initiated, or both?
- [ ] **Risk profile.** Is direct interaction higher reward / higher risk than going via officer? Needs to be a real trade-off, not a free upgrade.
- [ ] **Voice or text only.** Production decision; defer until §2 officer voice resolves.

---

### §10 Technical Architecture

**Resolved:**
- [x] ~~Real-time vs. tick-based simulation.~~ *Resolved: hybrid. Event-driven (lazy) for reads; scheduled background jobs for state changes that must apply on schedule. Daily state-change batch at midnight PT; weekly finance tick at Sunday midnight PT; weekly match resolution with Saturday midnight lock and Sunday 3pm reveal; season transitions; event auto-resolution; inactivity check.*

**Open:**
- [ ] **LLM strategy specifics.** Provider, model choices, prompt caching configuration, fine-tuned models for officer voices. *(Principles set in §10.5; specific choices deferred until prototype.)*
- [ ] **Backend language and framework.** Defer until prototype scope is firm.
- [ ] **Database technology and persistence strategy.** State schema is the bigger design question than the choice of database engine.
- [ ] **Hosting platform.** Prototype scale (tens to low hundreds of users) doesn't need AWS-grade infrastructure; managed platforms (Render, Railway, Fly.io) are simpler and adequate. Decision deferable until prototype scope is firm.
- [ ] **Multi-region scheduling.** v1 uses fixed PT timing; multi-region timing (per-player or per-region scheduled jobs) is a v1+ (structural) problem.
- [ ] **Narration generation strategy revisit.** Default for prototype is eager (generate during scheduled job). Revisit if cost telemetry shows narration consuming a disproportionate share of LLM spend.
- [ ] **Job orchestration mechanics.** What runs the scheduled jobs (cron, queue worker, etc.)? Implementation detail; defer until language/framework decided.
- [ ] **Temporary impact / impact-window mechanic.** Several state effects in the design are durational rather than permanent — sponsor contracts (2–3 year terms), player contracts, media windows (per §11 §3) suppressing event probability, doping boosts (player stat lifts for ~1 month), training-camp effects, scandal cooldowns, garden-leave on a fired officer. Currently the doc handles contract durations ad hoc inside §4 and §5, fan sentiment as continuous decay in §4.5, and other cases as "deferred." v1 needs a unified pattern: a state modifier with a start time, duration, and reversal effect, applied by code and unwound by a scheduled job at expiration. This is a §10 scheduled-job addition (generic "scheduled state reversal" job) and a §4.5 / §5 state-model addition (state effects can carry durations, not just values). Deferred — not all v1 surfaces need impact windows, and the ones that do (sponsor and player contracts) already have bespoke handling that works. But the pattern is in the doc's future and several v1+ features assume it. Design before any feature that needs durational impact lands.

---

### §10.5 Cost Architecture

**Open:**
- [ ] **Cost ceiling number.** Derive from prototype telemetry, not assertion.
- [ ] **Provider choice.** Anthropic, OpenAI, Google, multi-provider? Defer until prototype.
- [ ] **Cache invalidation strategy** for game-state-dependent caches.
- [ ] **Rate limit numbers.** Coupled to §1 session cadence and §3 burst/sustained limits. Tune from prototype telemetry.
- [ ] **Subscription tier viability.** Coupled to §1 session cadence; revisit when prototype data is available.
- [ ] **Pay-for-advisory.** Define tightly (purely informational? cosmetic-only flavor? no mechanical effect?) or kill. Currently not a candidate without a non-pay-to-win specification.
- [ ] **Pay-for-events.** Design caps / negative-event chance / Financial Fair Play-style ruleset for a future version. Marked for future design, not v1.

---

## 12. Decision Log

> Append-only. When a decision is made, add it here with date and rationale.
>
> **Terminology note:** entries dated before 2026-05-12 use the pre-ladder terminology — "MVP" for what is now called "v1," and "v2" / "v2+" for what is now called "v1+" (with size tag determined by the feature's design surface). See §0.1 Scope ladder for current terms. Older entries are not rewritten — the Decision Log is append-only and preserves the language used at the time.

- *2026-05-15 — §3 classifier architecture committed to two-stage: stage-1 routing + stage-2 parameter extraction, with novel intents bypassing stage 2 to the mid-tier pipeline.* The §3 classifier-reliability open question (§11 §3) needed an architecture commitment before it could be evaluated. Three options considered: (A) single-stage cheap-tier classifier producing routing + parameters + officer assignment in one call; (B) two-stage with stage 1 doing routing only and stage 2 doing parameter extraction for recognized intents; (C) cascade with stage-1 confidence scores escalating to mid-tier on low-confidence. (A) was rejected because it bundles routing failures and parameter-extraction failures into one accuracy number — the prototype eval can't tell which part is broken when overall accuracy is 92%, which means the prototype produces less learning per dollar. (C) was rejected for prototype because confidence calibration is a separate validation problem layered on top of routing — too many variables to validate at once. (B) won because it separates concerns cleanly, makes failure modes diagnosable per stage, and keeps each stage at cheap-tier (stage 1 is binary-ish routing; stage 2 is parameter extraction against a closed recognized-intent list, which cheap-tier models handle well). The decision is for the prototype; if stage 1 misses thresholds at cheap-tier, the architecture may need to evolve (e.g., move stage 1 to mid-tier, or adopt (C)) — but that finding is itself valuable. Out-of-scope terminates immediately with templated refusal (no further LLM calls). Officer-asks-clarification is *not* a stage-1 output — stage 1 always commits to a classification; clarification, if needed, is a stage-2 or pipeline behavior, deferred for prototype. Stage-1 context is officer roster + active event titles only; broader game state (cash, phase, log) belongs to stage 2 or later if needed. Stage-1 prompt, test set (80 labeled utterances), and eval methodology live in the prototype repo at `prototype/classifier/`, not in the GDD — the GDD commits to the architecture; the implementation is iterative.

- *2026-05-12 — Four §11 questions resolved: §3 burst threshold, §5 tactical decisions, §5 pool generation algorithm, §5 DoF ambient narration. All four are deferrals with explicit "what the prototype does" recorded, not just "open."* The pattern across all four: prototype runs with a known hole, the design question lives at v1-before-launch (§3 burst threshold, §5 tactical decisions, §5 pool generation) or post-prototype with-real-data (§5 ambient narration). Recording what the prototype actually does — rather than leaving the questions open without a fallback — means the prototype is buildable today without each question being answered. §3 burst threshold: prototype is unguarded because trusted-actor population means no script-burst risk; §10.5 rate-limit architecture exists but thresholds are off; v1 needs real limits before public launch. §5 tactical decisions: not modeled in prototype, tactical chat input hits out-of-scope handling, consistent with Tactical Knowledge not being a DoF role stat and no separate Head Coach in v1; v1 chooses between staying out-of-scope, adding a thin tactical-preference state attribute, or introducing a Head Coach officer. §5 pool generation: pre-authored ~50-100 player pool with frozen refresh (no Sunday-night additions during prototype), simplest implementation that exercises §5 transfer market mechanics; v1 chooses procedural, larger pre-authored, or hybrid. §5 DoF ambient narration: not a surface in prototype — the DoF speaks only during task assignment, task completion, and event narration; ambient squad observations are not modeled; revisit only if playtest data shows the DoF feels silent between events. The original §11 entry conflated two distinct questions (ambient narration surface existence vs. templated-vs-LLM implementation of existing narration); separated them — the second question remains open as prototype-tuning work in §5 "Open design surfaces." This finishes §3's open question list as far as design-time work is concerned; the remaining §3 item ("classifier reliability and stability") is the prototype-validation target itself, evaluated by running the prototype rather than designed in advance.

- *2026-05-12 — Scope ladder defined in §0.1; "MVP" retired, "v1.1" / "v2" / "v2+" replaced with "v1+ (additive)" and "v1+ (structural)".* The doc had been using four overlapping terms — "v1," "prototype," "MVP," and "v1.1/v2/v2+" — without explicit definitions, and the ambiguity had started to bite. Two specific examples: "v1 prototype tolerates arbitrary negative cash" only parses if "v1" and "prototype" are distinct things; "cosmetic monetization is the default assumption for v1" only parses if v1 is a launched product, not the prototype. Both readings had textual support; the doc was internally inconsistent. The ladder resolves this by naming three stages: prototype (internal build, validates the three load-bearing risks), v1 (first publicly-shippable product, prototype holes closed), and v1+ (deferred work, no roadmap commitment). v1+ items carry a size tag — (additive) for small features that can be slotted in without restructuring, (structural) for major new systems. The tag is descriptive, not a release commitment; either size can ship in either order based on priorities. Pre-ladder terminology preserved in older Decision Log entries per the append-only rule; a preamble points readers to the new terms. Considered alternatives: (A) collapse to two stages (prototype and v1, with everything else "deferred") — cleaner but loses scannability of §11 by removing the size distinction; (B) keep v1.1 / v2 / v2+ as three post-v1 stages — too much commitment to a sequencing we can't defend. The three-stage ladder with size tags is the minimum that keeps §11 scannable without overclaiming roadmap structure. Active-prose audit of remaining v1.1/v2/v2+/MVP uses across the doc is the next edit set.

- *2026-05-12 — §3 output variance and §4 bankruptcy mechanics jointly deferred for prototype, both as known prototype holes with candidate designs recorded.* No fail state in v1 — a struggling club keeps existing in the league, slot reassignment handled by §6 inactivity rule rather than bankruptcy. Consistent with Pillar 3 (shared persistent world) and avoids designing a separate code path for ownership churn. Damage cap and bankruptcy ladder both deferred because (a) prototype's job is validating §3 classifier reliability, §2 outcome math (the roll, tier distribution, dialogue layer), and §10.5 cost envelope — none require fully-calibrated state consequences; (b) designing consequence ladders and damage caps speculatively without play data invents numbers we can't validate; (c) the two questions answer the same underlying problem (how much damage can the player absorb) and need consistent answers, which is easier to design from telemetry than from first principles. Honest about the trade-off: cash specifically becomes a consequence-free dimension in prototype, which means cash-targeting tasks (sponsorship, transfers) produce narrative consequences but not stakes. Acceptable for prototype scope — the math validation is about whether outcomes feel calibrated, which the narrative resolution exercises regardless of whether the state delta has teeth. Candidate designs for both questions recorded in §11 entries so they don't need to be re-derived: bankruptcy as reversible consequence ladder (Pressure → Crisis → Collapse with event + state effects at each tier); damage cap as per-attribute soft cap on single-task delta. Both revisited from prototype telemetry rather than designed speculatively.

- *2026-05-12 — §3 scope resolved: 11 recognized intents (closed list, classifier-routed) and 8 novel-intent domains, each domain mapped to a primary role stat and concrete state targets.* The original §11 framing ("taxonomy of supported novel-intent classes") conflated two different things: the classifier's whitelist of structured operations (recognized intents) and the engine's scope for open-ended interpretation (supported domains). Separating them clarified both. The recognized-intent list is deliberately small (11 items) — everything else routes through novel-intent handling. The domain list was filtered against a "does a Triumph in this domain produce a concrete state change?" test, which dropped two candidates: Sporting strategy (long-arc, no concrete per-task state target — folded into Squad management and Scouting & recruitment, which are what sporting strategy actually does) and Crisis management (player-initiated crisis work either collapses into Media management or requires an active crisis, in which case it's an event response and the Crisis Management role stat fires there — not a novel-intent domain). Each remaining domain commits to one primary role stat for the §2 outcome roll. The list is honest about role-stat load: Marketing and Player Development are primary on two domains each; Football Network, Strategic Vision, and Composure (outside special events) are not primary on any domain. These lighter stats earn their keep through event-trigger modifiers (per §2 character stat interactions), access gating (Football Network on transfer market hidden-attribute reveals per §5), and the Composure special-event dice modifier. Out-of-scope handling resolved as templated in-character officer refusal — no slot, no LLM call, logged for analytics; distinct from feasibility-check refusals which happen mid-pipeline.

- *2026-05-12 — Support-stat-as-difficulty-modifier mechanic deferred to v1.1.* Considered as part of the §3 scope work. The idea: some role stats (Football Network, Strategic Vision) act as difficulty reducers on adjacent domains, giving every role stat a job in the novel-intent math. Rejected for v1 because it adds a new mechanic to §2 outcome math when §2 outcome math is already one of three prototype-validation targets. Validating two outcome-math changes simultaneously is bad prototype design. Recorded as a v1.1 candidate — small additive change once primary-only data confirms which role stats feel decorative.

- *2026-05-12 — Temporary-impact mechanic flagged in §11 §10.* Surfaced during §3 scope work — media management's "event probability shifts for a window" target would need a unified durational-impact pattern that doesn't yet exist. Several other v1 features have similar shapes (sponsor contracts, player contracts, doping boosts, training camps, scandal cooldowns, officer garden-leave). v1 handles the cases it needs (sponsor and player contracts) with bespoke per-feature duration logic; this works for v1 scope. The architecture pattern — state modifier with start time, duration, and scheduled reversal — is recorded in §11 §10 as deferred work to design before any feature needing durational impact lands. Pairs with the §11 §3 media-window v1.1 entry, which assumes the pattern exists.

- *2026-05-12 — §2 goals list deferred to v2 pending prototype data.* The goals list was the last named v1 design hole, originally framed in §1 as the session anchor. Deferred rather than designed because it fails the prototype-ROI test: it doesn't validate any of the three things the prototype exists to validate (§3 classifier reliability, §2 outcome math feeling right, §10.5 cost envelope). It also has hidden cost surface — officer-proposed goals = an additional LLM call type that would compete for the cost envelope before the envelope is measured — and non-trivial design surface even in its lightest form (where do goals come from, what's the lifecycle, when do they refresh). v1 session orientation falls back on notifications, club log, officer panel (with active/frozen tasks visible), and surfaced event response windows. §1 session shape updated to remove the goals-list step and to name the orientation surfaces explicitly so the doc is internally consistent. v2 design target recorded so it doesn't need to be re-derived: co-authored list (officer-proposed and player-authored), instant-action adds, manual or officer-detected completion, purely narrative with no mechanical weight. The deferral is honest about ROI; revisit after prototype data tells us whether the v1 orientation surfaces are sufficient or whether players need a structured anchor.

- *2026-05-12 — §11 audit cleanup.* Crossed off two residual open questions made redundant by later resolutions: §6 "Scheduling matches" (resolved by §10 simulation model — server-side resolution in the Saturday-midnight to Sunday-3pm window means neither player needs to be online) and §10 "Match-state lock window" (covered by §6 match-day operational flexibility and §10 timezone note; "may change if requirements change" describes every value in the doc and isn't worth tracking as a distinct open question). §3 "Same-instruction consistency" folded into "Classifier reliability" since both properties test the same pipeline (classifier + difficulty computation) and would be validated together by the same prototype evals; surfacing them as two separate items implied two distinct prototype risks when it's one.

- *2026-05-12 — §2 hiring market resolved: shared regional NPC pool, reputation-filtered visibility, per-season refresh, accept-or-walk hiring with no counter-offers in v1.* Architecture leans on §5 transfer market principles (shared regional scarcity, NPC-only for v1) but diverges in four real ways. (1) Volume: 2 officers vs. 22 players means hiring is a rare event, not routine activity — pool smaller (~6-12 per role), refresh slower (per-season, not weekly). (2) Character depth: officers are full AI characters with archetypes and dialogue, not stat blocks — each candidate generates an interview (mid-tier LLM call) revealing archetype-flavored personality. Bounded LLM cost because the pool is small and refresh is seasonal. (3) Reputation gating: §4.5 already specifies reputation gates which candidates approach the club, so visibility is reputation-filtered. A Disreputable club doesn't see Elite candidates at all. (4) No officer mediates the negotiation — the owner hires directly. This produces a question without a clean parallel in the transfer market: whose stat drives negotiation? Two options considered: A (no roll, accept-or-walk) and B (candidate's Charisma drives difficulty when player counter-offers). B is mechanically almost free but introduces non-trivial failure-mode design (what happens when a counter-offer fails — candidate withdraws? holds firm?). A chosen for v1 because the prototype's job is validating §3 classifier reliability and §2 outcome math, both of which are tested by existing surfaces (transfer market, event responses); hiring negotiation is a third surface for the same mechanics, not a new validation target. B explicitly marked as a v1.1 candidate, not v2 — small enough scope to add when the prototype graduates. Stat visibility at hiring: role stats revealed (CV is on the table), character stats hidden (discovered through working together) — the inverse of player transfer market's scouting-driven hidden-attribute reveal. Per-season refresh aligns the hiring market naturally with offseason rhythm without imposing a mechanical window restriction. Headhunting, retention-battle events, agent dynamics, and re-interviewing fired officers all deferred to v2.*

- *2026-05-12 — §5 transfer market resolved: shared regional NPC pool, no cross-club transfers in v1, always-open windows for prototype.* Four architectures considered (pure NPC, pure shared real-player market, hybrid with cross-club transfers, NPC market with simulated rival activity). Pure NPC chosen for v1 prototype because (a) it preserves Pillar 3 via shared-pool scarcity without requiring async human-to-human negotiation UX, (b) the prototype's validation goals are §3 classifier reliability and §2 outcome math, not market complexity, and (c) cross-club transfers are a real v2 feature with significant design surface (async coordination, valuation negotiation, multi-day deal flow) that doesn't need to ship to test the core. Shared pool over private per-club pools because private pools collapse Pillar 3 on the transfer axis entirely — every club playing solo against its own generator isn't a shared market. Rival squads visible but contract details hidden, matching real-football information asymmetry and giving the DoF's Football Network stat a job (partial reveal). Rival transactions surface as lightweight log entries rather than triggering events, to keep §7 event system uncoupled from market activity for v1; richer reactions are a clear v2 design space. Transfer windows deferred for prototype scope — the prototype is not expected to simulate full seasons, so the seasonal-rhythm and daily-cadence arguments for window restrictions don't apply at prototype scale. Marked in §11 as design-before-extended-play, not v2, because the design intent is to add windows when the prototype graduates. Scouting reveal mechanics, pool generation, and pricing formulas all deferred to prototype tuning — designing them speculatively without real play data invents numbers we can't validate.*

- *2026-05-12 — §4.5 club state resolved: three attributes (fan sentiment, brand strength, reputation), decoupling principle named explicitly.* Media standing cut because its claimed work (how stories land, frequency of media events) is downstream of the CD's Media Relations role stat plus current fan sentiment — a standalone attribute would have solved the same problem twice. Three remaining attributes sharpened by audience and what they gate: fan sentiment is supporter mood (continuous, signed, ±10), brand strength is commercial scale (continuous, unsigned, 1–100, multiplicative), reputation is industry access (5 named tiers, gates what's available rather than scaling magnitudes). Different scales chosen deliberately: fan sentiment matches the officer/player stat range for player intuition; brand uses a wider range because it's a revenue multiplier where order-of-magnitude differences need to feel different; reputation is tiered because the interesting question is "will a top-tier DoF take my call?" not "is my rep 73 or 74?" — and tiers force discrete state changes with discrete consequences, which is what an access-gating attribute should do. Decoupling between brand and reputation named explicitly with all four corners enumerated (high/low × brand/reputation) so future design work can stress-test against the principle. Decay model split: fan sentiment decays toward 0 (mood is short-term), brand and reputation are sticky (commercial scale and industry standing don't drift). Action-to-state and event-to-state mappings deferred to §11 — designing them speculatively without prototype data would invent numbers we can't validate, and the mappings are the natural place for the decoupling principle to be tested in practice.*

- *2026-05-11 — §5 player Status refactored into continuous stats + availability flags.* The original "current form, fitness, injury state, morale" bullet conflated two different things: performance levels (how good is this player right now) and availability (can this player play this week). Splitting them: form/fitness/morale are continuous performance stats; injury/suspension are availability flags with durations. The match engine filters to available players before composing the matchday squad. Other availability flags (loan-out, international duty) deferred to v2. Refactor surfaced when discussing event content — suspensions are real football state that events need to be able to reference and reason about.

- *2026-05-11 — §7 event system substantively resolved: four categories, state-driven target selection, JSON event library, nudge-not-punishment defaults.* The category set (player-driven, officer-driven, crisis, opportunity) was cut to four for v1 by deferring world-wide/regional events to a v2 fifth category — once committed to single-team-local scope, the data model simplifies (no scope field) and scheduling simplifies (no broadcast mechanism). The key design move was state-driven target selection: events have selectors that find eligible entities at fire-time, rather than pre-authoring entity-specific events. This makes the authoring model scale (one "high-Ambition transfer request" event covers every transfer-request situation in the game), guarantees state coupling (events cannot fire without matching entities), and bounds LLM cost (one mid-tier call per event surfacing). Default-resolution principle locked as nudges-not-punishments — preserves the §1 weekly-floor commitment (don't burn players who miss sessions) without making active engagement feel pointless. Event content lives in a separate JSON library; the GDD specifies the data model. Most v1 §7 questions resolve cleanly with this structure; remaining opens (library content, freshness strategy, authoring tooling) are content-production or v2 work.

- *2026-05-11 — §5 players as characters vs. stat blocks resolved: hybrid model (stat blocks + hidden personality stats driving discrete events).* Three options were considered: (A) pure stat blocks, (B) full AI characters, (C) stat blocks with discrete-event personality. (A) was too thin — the DoF's interesting dialogue depends on having interesting situations to report, and pure stat blocks don't generate those. (B) was rejected on three grounds: it breaks Pillar 2 (officers as primary interface — adding per-player AI characters fragments that), it multiplies the LLM cost surface by squad size (linear in players-per-team-times-teams-in-league), and it overwhelms players with too many relationships to manage. (C) preserves the officer-mediated relationship while giving the squad enough texture to generate organic situations. Three personality stats (Ambition, Loyalty, Temperament) chosen as the floor that produces distinct event categories (contract demands, transfer requests, strikes, leaks) without bloating per-player state. Squad size set to 22 as a tunable starting point. Youth academy as a subsystem deferred to v2 (v1 has young players in squad without separate academy). Transfer market deferred to its own design session — it's a significant design space coupling §5, §6, and §8.

- *2026-05-11 — §10 simulation model resolved: hybrid (event-driven reads + scheduled state-change jobs).* The architectural question forced itself once a concrete consistency case was walked through: an officer task producing a player strike must apply at its scheduled completion time, not at next-login time, otherwise the strike could happen "in the database" while the player plays a match using the striking player. Pure lazy evaluation breaks Pillar 3 (shared persistent world) for any state change that affects other systems (match engine, other players, scheduled events). The resolution: state changes fire on schedule; only narration and UI computation stay lazy. Scheduled jobs cover daily state-change batch (midnight PT), weekly finance tick (Sunday midnight PT), weekly match resolution (Saturday midnight lock, Sunday 3pm reveal), event auto-resolution at window close, frozen-task auto-resume, season transitions, and inactivity check. Match results pre-computed during the lock-to-reveal window; player-facing rule "what your team looks like Saturday night is what plays on Sunday" is named explicitly so player intuition matches system behavior. Task durations measured in days (not hours), aligning with the daily batch — officer dialogue communicates batch-aligned timeframes ("tomorrow morning," "in 4 days") so the player experience is honest. Narration defaults to eager generation as part of scheduled jobs (revisit if cost telemetry surprises). All scheduled times use PT for the US-targeted prototype; multi-region timing deferred to v2.

- *2026-05-11 — §4 finance split into match-driven (per-match) and recurring (weekly Sunday-night batch).* Match-driven revenue (tickets, matchday merch) lands as part of match resolution because it's caused by the match. Recurring revenue and expenses (wages, salaries, facility upkeep, sponsor payments, recurring merch) tick weekly because they accrue regardless of whether there's a match that week — salaries shouldn't only get paid during league weeks. End-of-season payouts (broadcasting, prize money) ride the season-transition job. Transfer income/expense is event-driven. Sunday-night recurring tick (after the 3pm match reveal) bundles "weekly reset" cleanly: match result, financial settlement, new week begins.

- *2026-05-11 — §4 economy resolved: cash balance + per-period accounting, six revenue lines, four expense lines.* The minimal version (sponsors / tickets / salaries / facility upkeep) was tempting but excluded several lines that distort real football economics if left out — particularly broadcasting revenue (often the largest line in real clubs) and transfer activity (lumpy but financially decisive). Six revenue and four expense lines is the minimum that captures the core financial tensions without overcommitting to complexity v1 doesn't need (no debt, no tax, no depreciation, no multi-tier sponsorship beyond the three slots). Cash balance over full P&L because the question the player needs answered is "am I in trouble or fine?" — multi-period accruals would be accuracy theater. Per-player wages over aggregate wage budget because transfer negotiations and contract decisions need per-player financial stakes; the aggregate is derived for UI. Three sponsor slots (main shirt, secondary, stadium naming rights) chosen as the floor that gives the CD meaningful negotiation work without requiring the player to manage a dozen partner relationships. Bankruptcy mechanics deferred to §11; v1 allows brief overdraft, persistent shortfall triggers consequences whose specifics are designed later. The "no CFO in v1" §2 decision is reinforced by this section: finance is texture across officer dialogue and a clear UI surface, not its own gameplay system.

- *2026-05-11 — §2 character archetypes resolved: 3 archetypes per officer role, 6 total for v1, voice/personality only with no impact on outcome math.* Two questions were tangled in §11 — "Commercial Director character identity" (a voice/dialogue question) and "Personality model" (a mechanical question about whether traits affect task execution). Separating them: v1 ships voice archetypes for *both* officer roles (symmetric — no "default" identity for either role), with no mechanical effect on the §2 outcome math. Mechanical effects of personality are deferred to v2, where they can be designed against playtesting data on whether stats alone are doing enough character work. Archetype-biased stat generation gives texture (an "analytics-driven" DoF feels different from a "wheeler-dealer" DoF *because* their stats land in different ranges) without adding new mechanical surfaces. Archetype hidden from player at hiring; discovered through dialogue. Three archetypes per role is the v1 floor — fewer feels samey, more dilutes authoring/voice attention per archetype. Specific stat-range biases per archetype deferred to prototype tuning.

- *2026-05-10 — §3 menu surface resolved: split into views (read-only panels) and menu actions (structured state mutations).* The original "menu actions" framing conflated two different things — observation surfaces (which take no input and don't fit the instant-vs-strategic axis at all) and structured mutations (which do). Splitting them clarifies what we're committing to design. Six views and four menu actions for v1, deliberately small. The four menu actions (hire officer, fire officer, accept/reject term sheet, set ticket price) meet a strict criterion: frequent, structured, parameterized, no narrative interpretation needed. Training intensity cut from v1 as DoF-flavored work that doesn't earn a player-direct lever; can return if playtesting shows friction. Contract renegotiation and transfer bids are deliberately *not* menu actions even though they're routine-feeling — the work itself is strategic and consumes a slot, regardless of how it was initiated. This produces a refinement of the §3 classifier model: recognized intents come in routine and strategic flavors, and slot consumption follows from the work, not the initiation surface.

- *2026-05-10 — Fire officer scoped narrowly for v1.* The action is one-click with a confirmation showing contract cost. Downstream consequences on club state (reputation, surviving officer Loyalty, fan sentiment) are real and worth modeling, but deferred to §4.5 design work. The narrow scoping for v1 keeps the menu action implementable without committing to club-state mechanics that haven't been designed.

- *2026-05-10 — §4.5 Club State introduced.* The doc had a missing layer: aggregate state (reputation, brand, fan sentiment, media standing) that isn't reducible to officers, players, or money but affects all three. Surfaced when discussing menu action consequences — firing an officer should plausibly hit reputation, but the doc had no place to put that. Created as §4.5 (between Economy and Squad) rather than as a §4 subsection because reputation isn't really economy, and rather than distributed across other sections because the mechanics are aggregate-state with their own rules and would drift if scattered. Deliberately thin for now: enumerates the state attributes and flags mechanics as deferred. Pairs with the menu action work — actions and tasks produce tangible state changes, some of which land on club state. v1 may defer the club-state effects of specific actions; the framing exists so they have a home when they're designed.

- *2026-05-09 — §6 season cadence and league structure resolved.* Two seasons per real-world year, 26 weeks each (22 league weeks + 1 mid-season break + 3 weeks offseason), one league match per real-world weekend, 12 teams per league. Two seasons stack to a full 52-week year cleanly. The match-per-weekend rhythm gives the game a real-world heartbeat that supports both daily and weekly check-in cadences (per §1) — daily players see ~3 events between matches per week, weekly players see one match-result per session. 12-team leagues chosen over 18-20 because each individual match matters more in a 22-game season than a 38-game one, and the simulation needs only ~24 teams per region to populate two-tier promotion/relegation. Smaller-than-real-football is a deliberate choice for engagement density. Cup competitions cut from v1 (single isolated cup match is structurally odd; real brackets are v2). Time mapping inside a season runs 1:1 with real time for short-term things (matches, events, task durations); long-term things (careers, contract lengths) scale to seasons-elapsed for sim-time compression.

- *2026-05-09 — §6 launch and growth model resolved.* Pre-launch state for players 1–11 (full game minus league matches; functions as preseason prep), league starts at player 12, player 13+ uses the same pre-launch state until a slot opens. Inactivity threshold: 1 full season without login = team can be replaced. The alternative — NPC-filled leagues at launch — was considered and rejected because it weakens Pillar 3 (shared persistent world) at exactly the moment players are forming first impressions; NPC teams are a real design surface (stat blocks, schedules, possibly AI behavior) that v1 doesn't need to take on. The pre-launch alternative trades NPC-heavy leagues for "early users have things to do but not matches" — players are managing clubs from minute one, just without league play, which maps cleanly to the §6 offseason mechanics. First-mover advantage (early signups have more preseason prep when league starts) is fair and supports community-building. Multi-league expansion (what happens when 12 more players are waiting) and regional structure (geographic sharding) are both deferred to v2 — the single-league launch model holds indefinitely until those pressures hit, at which point we'll have real population data to design against.

- *2026-05-09 — §6 season cadence and league structure resolved.* Two seasons per real-world year, 26 weeks each (22 league weeks + 1 mid-season break + 3 weeks offseason), one league match per real-world weekend, 12 teams per league. Two seasons stack to a full 52-week year cleanly. The match-per-weekend rhythm gives the game a real-world heartbeat that supports both daily and weekly check-in cadences (per §1) — daily players see ~3 events between matches per week, weekly players see one match-result per session. 12-team leagues chosen over 18-20 because each individual match matters more in a 22-game season than a 38-game one, and the simulation needs only ~24 teams per region to populate two-tier promotion/relegation. Smaller-than-real-football is a deliberate choice for engagement density. Cup competitions cut from v1 (single isolated cup match is structurally odd; real brackets are v2). Time mapping inside a season runs 1:1 with real time for short-term things (matches, events, task durations); long-term things (careers, contract lengths) scale to seasons-elapsed for sim-time compression. Pyramid depth (number of divisions) and region definition (how to handle density variation) deferred to §11.

- *2026-05-02 — §2 task slot model resolved: one active + one frozen slot per officer. Strategic tasks consume the active slot; instant actions (menu + recognized-intent chat) don't.* Resolves the "officer blocked while executing" tension in the original §2 framing without breaking §3's "always available" principle. The chat interface is always available; what's bounded is *concurrent strategic initiatives*, which is the right constraint — it makes time cost a real strategic decision (recall vs. wait) rather than just a duration. Considered alternatives: (a) full blocking — too punishing, makes both-officers-busy sessions hollow; (b) unlimited parallel tasks — destroys time-cost-as-constraint and makes officers feel like function calls; (c) numeric concurrent-task cap — solves the same problem but adds an arbitrary number the player has to learn. The recall-or-keep pattern with one frozen slot threads the needle: officer feels real and finite, player retains agency, time cost matters. Frozen-task auto-resume with code-side relevance check (target condition still exists) keeps the mechanic clean without LLM-driven subjective relevance judgments. Confirmation prompt on second freeze prevents silent loss of player work.

- *2026-05-02 — §2 action types restructured: instant vs. strategic, orthogonal to the three §3 interaction surfaces.* The previous "menu / chat / event response" framing mixed *initiation* (how the player kicks off an action) with *cost* (whether it consumes resources). Separating them: surfaces describe initiation, instant-vs-strategic describes cost and slot consumption. Player doesn't have to know which surface routed their input — the system makes that call invisibly. Cleaner model that maps to gameplay constraints rather than UI affordances.

- *2026-04-25 — §7 event response windows: events have real-world action windows; default-resolve if ignored.* Falls out of the §2 task slot model — events that the player chooses to act on are strategic tasks, so they need a way to *not* sit in a queue forever. Window mechanic gives time pressure as a real game lever, supports daily-check-in cadence (don't miss the window), and creates a clean fiction layer (the world doesn't wait). Crisis events get short windows (urgent), opportunities longer (no immediate urgency). Default-resolution consequences are typically worse than an active response but not necessarily catastrophic — preserves stakes without punishing missed sessions in the §1 weekly-floor case.

- *2026-04-25 — Open-question tracking consolidated to §11 as the single source of truth.* Section-level "Open questions" subsections are removed; §11 is canonical and carries full context for open items. Inline prototype-tuning notes (placeholder values that affect specific design tables) remain in sections where they're load-bearing for the reader. Rationale: the two-list structure was creating drift — section-level questions and §11 disagreed in small ways and would compound over time. Single source of truth is cleaner; rich context in §11 ensures sections aren't thinned out for active design problems. The §11 list is now organized by section with resolved-vs-open separation rather than as a flat checklist; for a doc this size, structured beats flat.

- *2026-04-25 — §3 interaction model resolved: three surfaces (menu actions / open-ended chat / event responses), all available at all times. Open-ended chat routes via classifier to cheap (recognized intent) or expensive (novel intent) paths; rate limiting handled by in-fiction burst protection rather than a hard cap on normal chat.* The alternative — gating open-ended input to event responses only — was considered and rejected because it inverts Pillar 1 (the player would be reacting to system-initiated events rather than initiating instructions). The cost concern that motivated that alternative is better addressed by classifier-routed processing: most chat-style interactions are recognized intents that go through the cheap-tier model, with only genuinely novel/strategic instructions consuming the expensive pipeline. Burst protection ("officer is busy") covers both per-player rate spikes and system-wide cost protection, with identical player-facing fiction (per §10.5 principle 8). Reframes the original question: "when can the player issue open-ended instructions?" and "how does the system process them?" are different questions, conflated in the original framing. Classifier reliability becomes the new high-leverage open question.

- *2026-04-25 — §2 outcome math resolved: 2d6 + role stat vs. difficulty (1–20), six symmetric tiers by margin (Triumph / Good / Okay / Suboptimal / Bad / Catastrophic).* Probabilistic from day one rather than deterministic-now-randomness-later: variance is core to the §2 character pillar (officers have good days and bad days), and once players learn a deterministic mapping, retrofitting randomness feels like a bug rather than a feature. 2d6 chosen over d6 (too modest) and d10 (too swingy) because the bell curve makes extreme outcomes feel earned and routine outcomes feel routine. Difficulty range 1–20 chosen over 1–10 because it matches the rolled-total range (3–22), preserving symmetric margins and reachability of all six tiers. Tier thresholds and exact dice are placeholders for prototype tuning.

- *2026-04-25 — §2 difficulty bands communicated through officer dialogue, filtered through Integrity (accuracy) and Charisma (delivery confidence).* Post-resolution reveal of true difficulty so the player can learn each officer's reliability over time. The four-quadrant Integrity × Charisma profile (especially "Low Integrity + High Charisma = confidently wrong") is the texture the §2 character pillar needs and falls out of existing stats without requiring a dedicated Self-Awareness stat. A Self-Awareness stat was considered and rejected: it would either duplicate Integrity or, if it replaced Integrity, lose the broader corruption/scandal coverage Integrity is doing across §2 and §3. Broad stats doing work across multiple systems is a feature, not a bug.

- *2026-04-25 — §2 character-stat math: most are discrete-event triggers, not per-task modifiers. Composure is the exception, modifying the die on special events only.* Cleaner mechanical separation between "how good is the work output?" (role stats, every task) and "what's this person's situation and disposition?" (character stats, separate timescales and systems). Composure earns its place in the per-task math because variance-under-pressure is a fundamentally different mechanical shape than expected-value modifiers, and special events are exactly where that shape matters. This also gives a clean answer to the most-likely-confused stat pair (Composure vs. Crisis Management): Crisis Management goes into the roll; Composure controls the spread of the roll.

- *2026-04-25 — §2 officer stat model resolved: 2 pillars × 5 stats per officer.* Character stats (Loyalty, Ambition, Integrity, Charisma, Composure) are universal across all officers and describe the person — they enable cross-role comparison at hiring time and drive personality-flavored failure modes. Role stats are mostly specialized but include two parallel axes (Negotiation, Network) to enable cross-role comparison on shared dimensions. DoF role stats: Scouting, Player Development, Negotiation, Football Network, Strategic Vision (which absorbs squad-building). CD role stats: Marketing, Crisis Management, Negotiation, Media Relations, Commercial Network. Tactical Knowledge and Squad Harmony intentionally excluded from the DoF as coach work, not DoF work — preserved as candidates for v2 if a separate Head Coach is introduced. No general Competence stat: role stats *are* competence at specific things, and a meta-Competence layer either double-counts or masks vagueness about which role stat should govern an outcome. Downstream questions (numerical scale, outcome math, development over time, visibility at hiring) deferred.

- *2026-04-25 — §2 MVP officer roster resolved: 2 officers (Director of Football, Commercial Director).* Real-football titles chosen over generic C-suite labels because they teach the player something true about how clubs are structured and correctly signal both officers as peers reporting to the player as owner/principal. Roster size of 2 chosen over 4 because depth-over-breadth forces sharp characterization and yields a clean player mental model ("on-pitch vs. everything else"). The Commercial Director absorbs what would have been CMO + CPRO scope; the title is slightly stretched (a real Commercial Director doesn't usually run crisis PR) but the mismatch is small and learnable. CFO dropped from MVP because most CFO work is consequence-of-other-decisions rather than open-ended interpretation, and §10.5 principle 5 already routes numerical state changes through code rather than the LLM — finance is better as UI surfaced through the other officers than as its own character. Additional roles (Communications Director, CFO, separate Head Coach, Scout, Academy Director, Stadium Ops, Legal Counsel) deferred to v2+, ordered by which absences feel most acute in playtesting.

- *2026-04-25 — §1 resolved: real-time world; target session cadence ~daily with weekly floor; target session length ~15 min with 5-min floor and 30-min ceiling.* Real-time follows from Pillars 2 and 3 — a paused-when-offline mode is incompatible with officers-take-time and shared-persistent-world. Daily-as-target chosen over weekly because (a) the shared-world pillar needs ambient life between sessions, (b) the §10.5 graceful-degradation-as-fiction move requires the player to notice officer unavailability within a reasonable window, (c) subscription viability is far more plausible at daily cadence, and (d) the player's own description of a session ("check in, resolve decisions, strategize") is a daily-check-in description, not a weekly catch-up. Weekly is preserved as a floor so missing days carries no penalty.

- *2026-04-25 — Goals list introduced as the session anchor.* A co-authored officer/player goals list gives a returning player a clear "what am I working on?" view and a natural place for officers to surface domain-specific proposals. Mechanic detail deferred to §2 design work.

- *2026-04-25 — Project framed as serious side project / portfolio piece.* Implication: aggressive scoping is allowed; we prioritize the novel mechanic over breadth.

- *2026-04-25 — Cost architecture promoted to a design-time concern.* LLM-heavy consumer products fail at the cost line when cost is treated as an implementation detail. Principles adopted now: LLM-only-for-open-ended, tiered routing, prompt caching, batch API, deterministic-first state changes, hard rate limits, telemetry from day one, graceful degradation framed in-fiction. Specific cost ceiling deferred until measured against a working prototype rather than guessed at the GDD stage. Rationale: a number invented now will either be ignored or will distort decisions; a method for arriving at one is durable.

- *2026-04-25 — Graceful degradation framed in-fiction.* When rate limits or provider outages prevent immediate LLM responses, officers are presented as "out of office," "traveling," or otherwise unavailable rather than surfacing technical errors. Reinforces §2's principle that officers take time and aren't always available; turns a technical constraint into worldbuilding texture.

- *2026-04-25 — Match simulation: stat-driven, no LLM in resolution path.* Follows from §10.5 principle 1. Yields deterministic, reproducible outcomes.

- *2026-04-25 — Match report: minimal goals/subs/yellow/red ticker.* No LLM narration. Intentionally thin; the interesting decisions are pre- and post-match. Expansion path if needed is more structured ticker events, not narration — purely a code change to the ticker engine.

- *2026-04-25 — Event selection: code-driven from authored pool with state-aware weighting; LLM for flavor only.* Follows from §10.5 principles 1 and 4.

- *2026-04-25 — §3 anti-exploit question resolved by §10.5 principle 5 (deterministic-first state changes).* The LLM never writes numerical state directly; it produces validated parameters that code applies.

- *2026-04-25 — Monetization: cosmetic is the default working assumption for v1; all other models deferred.* Subscription is coupled to §1 session cadence and revisited when that resolves. Pay-for-advisory is not a candidate without a tight non-pay-to-win specification. Pay-for-events is deferred to a later version with mitigation ideas (caps, negative-event chance, Financial Fair Play-style ruleset) marked for future design work — not rejected. Rationale: the GDD has no data yet on what players will pay for in this specific game; locking in a model now over-commits.

---

## 13. Glossary

- **DoF** — Director of Football. The v1 officer responsible for squad, coaching, tactics, transfers, and youth.
- **CD** — Commercial Director. The v1 officer responsible for sponsorships, brand, fan engagement, merchandise, media relations, crisis management, and public statements.
- **CFO / Communications Director / Head Coach / Scout / Academy Director** — deferred officer roles, v1+ (structural). See §2.
- **Region** — A geographically-bounded shared league reality (e.g., Southern California). All teams founded in a region compete in that region's pyramid.
- **Event** — A semi-random administrative occurrence that advances the world state and prompts player decision-making.
- **Officer** — An AI character employed by the team to manage a domain on the player's behalf.
- **Financial Fair Play (FFP)** — A real-soccer governance concept (UEFA's rule set constraining how much teams can spend relative to revenue) being held as a reference point for future design work on bounding real-money advantages in the shared league. Not yet defined for this game.