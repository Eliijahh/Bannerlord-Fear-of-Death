# Fear of Death — Design Document

**Version:** 0.1 (draft)
**Target:** Mount & Blade II Bannerlord, version TBD
**Premise:** Every soldier is a person who does not want to die. Everything else is a consequence.

---

## 1. Thesis

This is not a morale mod. A morale mod adds a bar to a formation and makes the bar go down. This mod starts at the
other end: each man on the field is running a small, continuous, personal calculation about whether the next step is
worth the risk, and the visible shape of a battle — the cautious approach, the halt, the skirmishing, the grind, the
collapse — is what those calculations look like from a distance.

A soldier is not brave because his formation has high morale. He stands because the man beside him is standing, because
his officer is watching, because the enemy looks as frightened as he is, and because there is nowhere to run. Take any
of those away and he behaves differently, without anyone issuing an order.

That is the whole design.

---

## 2. Axioms

These are architectural constraints, not flavor. Every later section must obey them.

**A1 — Fear primacy.**
The only root cause of behavior is the individual soldier's fear of death, modified by his body, his training, his
culture, and what he can see. There is no group-level fear, morale, cohesion, or momentum that acts as an independent
input.

**A2 — Local knowledge only.**
A soldier perceives what a man standing where he stands could perceive: the enemy in front, the men immediately around
him, his own wounds and breath, the officer's voice. He does not know the army's total strength, the overall kill
count, or whether the battle is "going well." Any global information entering the model is a bug.

**A3 — Bodies are real.**
Reach, fatigue, wounds, footing, and the press of the crowd constrain what a man can do. Psychology proposes an
action; the body decides whether it is physically available.

**A4 — Group behavior is emergent.**
Formation-level behavior is computed *from* the distribution of individual state. It is a projection, and the
projection is allowed to be lossy, but it must never invent intent that the men do not have.

**A5 — Fear never reaches zero.**
Even the bravest man is managing it. There is no "fearless" state, only a high threshold and a lot of practice.

**A6 — Dying is not a number.**
Casualties matter to the men who see them, not to a scoreboard. The death of the man beside you is the single most
important input in the model. The death of a man on the other side of the field is not an input at all.

---

## 3. The individual soldier

### 3.1 State variables

| Symbol | Name | Range | Timescale | Meaning |
|---|---|---|---|---|
| `C` | Courage | 0..1 | per battle | Baseline willingness to accept risk. Built from training, culture, role, experience, and context. Stable during a fight. |
| `F` | Fear | 0..1 | seconds | Acute, present-tense fear. Spikes on events, decays when safe. The main driver. |
| `D` | Dread | 0..1 | minutes | Accumulated, low-grade terror. Rises over a long fight, decays very slowly. Makes men brittle late in battles. |
| `E` | Exhaustion | 0..1 | minutes | Physical cost of running, swinging, carrying a shield. |
| `H` | Health | 0..1 | — | Current health fraction. Wounds raise fear sensitivity. |
| `X` | Exposure | 0..∞ | battle | Number of contacts survived. Feeds habituation. |
| `T` | ContactTime | seconds | — | Time spent in melee since last disengagement. |
| `Q` | EscapeViability | 0..1 | seconds | Can I physically get out of here, and where would I go? |
| `R` | Reach | metres | constant | Weapon reach. Sets the personal standoff distance. |
| `A` | Aura | 0..1 | seconds | Presence of a leader, standard, or hero nearby. |
| `S` | Shame | 0..1 | seconds | Social cost of running *right now*. |
| `B` | Boldness | 0..1 | per probe | Sampled inclination to be the one who steps forward first. |

Derived per tick:

| Symbol | Meaning |
|---|---|
| `Threat` | How dangerous this moment feels |
| `Resolve` | How much danger this man can currently absorb before fear climbs |
| `d*` | The distance he wants to keep from the nearest enemy |
| `F_flee` | The fear level at which he runs |
| `p_probe` | Probability he is willing to be the first to close |

### 3.2 Perception

Computed from an egocentric slice, radius roughly 15 m, wider in front than behind.

- **Enemies:** count, distance, type (infantry / archer / cavalry), facing, whether they are advancing or recoiling.
- **Allies:** count, distance, their current state (engaged, probing, backing off, running).
- **Deaths:** ally deaths within 10 m, with time and violence (a man cut down in one blow is more shocking than one
  who fought for twenty seconds).
- **Missiles:** incoming volleys, near misses, the man beside you taking an arrow.
- **Leadership:** officer, hero, standard within 12 m.
- **Self:** health, breath, footing, how tightly packed he is, whether there is open ground behind him.

No input from outside this slice. In particular, no knowledge of formation strength, army size, or the wider battle.

### 3.3 Appraisal

```
Threat = w_t · threatFromEnemies
       + w_c · casualtyShock
       + w_h · (1 - H)
       + w_m · missilePressure
```

```
Resolve = C · (1 - 0.5·E)
        + A
        + w_s · supportBonus        // friends close enough to matter
        + w_x · habituation(X)      // X / (X + k)
        + w_v · recentSuccess       // I or my neighbours just won an exchange
        - D
```

Fear is a leaky integrator with asymmetric rate:

```
rise = k_r · max(0, Threat - Resolve)
fall = k_f · (1 - Threat) · F
F   += (rise - fall) · dt
```

Dread accumulates only when fear stays elevated, and decays only with genuine rest (rotation to the rear, a lull,
a rout in your favour):

```
D += k_d · max(0, F - θ_d) · dt
D -= k_dd · rest · dt
```

**Habituation is the courage generator.** `habituation(X) = X / (X + k)` grows quickly at first and then flattens.
A man who has survived three contacts is dramatically calmer than one who has survived none; a man who has survived
thirty is only slightly calmer than one who has survived ten. This is the mechanical expression of "skirmish a bit and
find courage."

**Casualty shock** is an impulse, not a rate. Each qualifying death adds `shock_impulse` scaled by proximity and
violence, decaying over roughly 8 seconds. The death of the man to your left is worth many times the death of a stranger
ten metres away.

### 3.4 The three urges

At every moment a soldier is pulled three ways. Behavior is the resolution of that argument, and it is why men do not
simply run when frightened.

1. **Get away.** Fear. Pushes toward retreat, distance, safety.
2. **Do not be a coward.** Shame. Pushes toward staying, especially in front of officers and peers.
3. **Stay with the man next to me.** Social anchoring. Pushes toward matching the behavior of neighbours, in either
   direction.

Shame is what makes the *first* man to run rare, and panic contagion is what makes the *tenth* man to run cheap:

```
S       = shameSensitivity · holdingFraction · (A > 0 ? 1.2 : 1.0)
F_flee  = base_flee + shameGain · S - panicContagion · runningFraction
```

`holdingFraction` is the fraction of nearby allies still engaged. `runningFraction` is the fraction already fleeing.
The asymmetry is deliberate: seeing a neighbour run raises fear far more than seeing a neighbour stand lowers it.
Resolve is contagious slowly; panic is contagious fast.

### 3.5 Action selection

Evaluated in priority order. The first condition that holds wins.

| # | Condition | State | Behavior |
|---|---|---|---|
| 1 | `F ≥ F_flee` and `Q ≥ q_min` | **Flee** | Run for the nearest friendly mass or rear. Drop target. |
| 2 | `F ≥ F_flee` and `Q < q_min` | **Desperate** | Cornered. Shorten standoff, attack harder, do not attempt to leave. |
| 3 | shock impulse just applied and `F` jumped | **Shock** | 0.4–1.2 s freeze. No attack, no movement. The line wavers. |
| 4 | `T ≥ contactBudget` or `F ≥ F_rotate` | **Disengage** | Break off, step back, rotate to the rear. |
| 5 | `F ≥ F_probe` | **Probe** | Hold at weapon reach, feint, bark, do not commit. |
| 6 | enemy within reach and `F < F_engage` | **Engage** | Normal melee. |
| 7 | otherwise | **Approach** | Move toward `d*`, slowing as `F` rises. |

Two ordering details matter:

- **Flight is checked before rotation.** A man who can run does not politely rotate; he leaves. Rotation is what
  happens to men who are spent but not yet broken.
- **Desperate is a distinct state, not a modifier.** When there is no escape route, fear does not produce flight, it
  produces a shorter standoff and more violence. This is why a pressed formation fights harder rather than dissolving.
  It is also the correct answer to "why doesn't everyone just run?"

### 3.6 The body

**Reach sets the standoff.** A man with a spear is willing to stand at 2.8 m. A man with a knife wants 1.2 m, but only
if he is brave. The line halts at weapon reach because that is where individual men stop wanting to go, not because
anyone ordered a halt.

```
d* = R + reactionMargin + fearBias·(F - 0.5) + threatBias·(cavalry ? 1 : 0)
```

**Fatigue** rises with running, swinging, and carrying a shield; it falls when rotated out or during a lull. It is
tracked by the mod and fed into the game's stat model (movement speed, swing speed). There is no usable public stamina
value, so this is our own number.

**Wounds** do not just lower health. Below 50% the fear sensitivity rises sharply; below 25% the man is looking for the
rear regardless of courage. A wounded soldier is a soldier who has already learned this fight can kill him.

**Crowding is a physical constraint that substitutes for courage.** If allies are packed within about 1.5 m behind him,
`Q` drops sharply. He cannot flee through his own friends, so fear converts into `Desperate` instead of `Flee`. This is
the single most important reason a line holds: the men behind you are a wall, and the wall is made of people who also
do not want to die but cannot go anywhere either.

### 3.7 Building courage

Courage is assembled once per battle from parts the player can reason about:

```
C = clamp( baseByTier + cultureMod + experienceMod + roleMod + contextMod )
```

**Base by tier (placeholder values):**

| Tier | `C` |
|---|---|
| Recruit / levy | 0.20 |
| Regular infantry | 0.32 |
| Veteran | 0.45 |
| Elite | 0.60 |
| Hero / lord | 0.70 |

**Culture (tuning placeholders, not lore claims):**

| Culture | Fear sensitivity | Shame sensitivity | Habituation rate | Leader dependence |
|---|---|---|---|---|
| Empire | 1.00 | 1.05 | 1.10 | 1.00 |
| Sturgia | 0.90 | 1.10 | 0.95 | 0.90 |
| Vlandia | 1.00 | 1.00 | 1.00 | 1.05 |
| Battania | 0.95 | 0.95 | 1.05 | 0.90 |
| Khuzait | 1.05 | 0.90 | 1.00 | 0.85 |
| Aserai | 1.00 | 1.00 | 1.00 | 0.95 |
| Mercenaries | 1.00 | 0.85 | 1.00 | 0.90 |
| Bandits | 1.15 | 0.70 | 0.90 | 0.80 |

**Role:** shield-and-spear line infantry get a small bonus (the shield is a psychological object as much as a physical
one); skirmishers get a penalty in melee; cavalry get a large bonus while mounted and lose most of it once dismounted.

**Context:** defending your own ground, recently victorious, well fed, and fighting alongside a lord all raise `C`.
Assaulting walls, being outnumbered, marching all day, and being unpaid all lower it.

**Battle type** modifies the environment rather than `C` directly:

| Battle type | `Q` | Dread rate | Notes |
|---|---|---|---|
| Open field | 0.80 | 1.00 | Escape looks easy, so men run sooner |
| Siege assault | 0.20 | 1.30 | Nowhere to go; more `Desperate` |
| Siege defence | 0.30 | 1.10 | Home ground bonus to `C` |
| River crossing | 0.30 | 1.20 | Bad footing, bad escape |
| Night | — | 1.20 | Perception radius reduced ~30% |
| Facing cavalry in the open | — | 1.10 | Threat bias up, standoff up |

---

## 4. From soldiers to lines

### 4.1 The front is a percentile, not a variable

Every soldier has his own `d*`. Sort them. The formation's front is not a commander's decision; it is where enough men
refuse to go further.

- `d_line = P75(d*)` — the line's natural halt distance.
- Men in the bottom quartile (boldest, or least informed by fear) push ahead of `d_line`. They are the skirmishing fringe.
- If `d_line` is greater than the current distance to the enemy, the line wants to open up.
- If `d_line` is smaller, the line wants to close, but only at the speed of its most reluctant members.

This is why no separate "formation standoff" variable exists. It falls out of the distribution.

### 4.2 The projection layer

Bannerlord moves men efficiently through formation orders, so individual intent must be projected into commands. The
projection runs every 0.25 s per formation and is deliberately dumb:

1. Compute distributions: `P75(d*)`, median `F`, `routFraction`, `engagedFraction`, mean `E`, cohesion.
2. Choose **one** order from a small set: `StandGround`, `Move(forward)`, `Move(back)`, `FallBack`, `Retreat`,
   `Advance`, `Pursue`.
3. Apply hysteresis: a new order must persist for at least 1.2 s and represent a change of more than 1.5 m before it
   is issued.
4. **Never issue `Charge`.** Charge exists only inside the Pursuit protocol (§6), where it is gated.

The projection may be wrong about individuals. That is acceptable and even desirable: a formation order that most men
disagree with produces stragglers, drift, and local disorder, which is exactly what real formations look like.

### 4.3 What we should see if this is working

- The last 20 m of an approach slow down visibly.
- Lines stop at weapon reach and probe, rather than colliding at full speed.
- A fringe of a few bold men operates ahead of the line; the line creeps up behind them.
- Front-rank men filter back over time and rear-rank men press forward.
- The line steps back when it takes casualties, then creeps forward again in a lull.
- A rout starts with one or two men, hesitates, then accelerates.
- A confident army runs, halts short, reforms, and only then walks in.

If any of these are missing, the model is not doing its job.

---

## 5. The contact protocol

The full lifecycle of a melee, as experienced by one man and by the line he is in.

### 5.1 Approach

Men advance toward `d*`, not toward the enemy's position. As `F` rises with proximity, `d*` grows and the approach
slows. The formation's speed is the speed of its most reluctant member, so a nervous line visibly drags.

Entry: formation is ordered forward or the enemy is closing.
Exit: every man is at or beyond his `d*`, or the enemy is within reach.

### 5.2 Halt

The line stops where the distribution says it stops. This is not an order; it is the aggregate of men refusing to take
another step. Expect shuffling, dressing of ranks, shield adjustments, barks.

At this point the standoff can persist. That is correct and intended. Real melee engagements often had long periods of
shouting at spear length before anyone committed.

**Stalemate needs an exit.** Add a slow `CommitmentPressure` that rises while the line stands in contact range without
fighting: officers shouting, shame, the enemy's missiles, the simple fact that standing still is also dangerous. Without
this, two frightened formations will stare at each other forever.

### 5.3 Probe

The courage generator. A small number of men — sampled by `Boldness`, shaped by `C` and `F` — step forward to weapon
reach, exchange one to three blows or throws, and step back unless the enemy gives ground.

- A probe lasts 2–6 s.
- Surviving a probe lowers `F` for the prober and mildly for neighbours, and raises `recentSuccess`.
- Dying on a probe spikes `F` for everyone who saw it.
- A successful probe is what pulls the line forward. Nobody orders the advance; the line follows the men who did not die.

This is the mechanical heart of "skirmish a bit and find courage," and it should be tuned so that probes are common,
brief, and occasionally fatal.

### 5.4 Engage and rotate

Once committed, each man has a **contact budget**:

```
contactBudget = base · C · (1 - 0.5·E) · (1 + 0.3·A)
```

Roughly 8–20 s. When it runs out, or when `F` crosses `F_rotate`, the man disengages and heads for the rear. Fresh men
from the depth move into the gap.

Implementation limits, stated honestly:

- Native melee locks agents once engaged; backing out looks like stumbling. Short scripted retreat pulses are the best
  available approach and will need iteration.
- There is no rank API. Front and rear are inferred by projecting positions onto the formation's forward axis.
- Rotation is the riskiest part of the mod. Rotate a small batch (10–15% of the formation per cycle), route them
  through existing gaps, and accept that it will look imperfect. Fallback: represent rotation statistically (fresh men
  fight better, spent men fight worse) and visually rotate only a token few.

### 5.5 Give ground, hold, or break

Per man, the outcome of contact is one of three trajectories:

- `F` falls as he survives and the enemy recoils → he holds or presses.
- `F` rises slowly with fatigue and casualties → he rotates out, and the line gradually steps back.
- `F` crosses `F_flee` → he runs, and the men who see him run become more likely to run.

---

## 6. The pursuit protocol

Men do not chase a formed enemy. They chase a broken one, and even then they stop before contact and start again slowly.

`Charge` and `Pursue` are permitted only when at least one holds:

- enemy rout fraction > 0.5, or
- enemy cohesion has collapsed and local force ratio > 3:1, or
- an explicit player order, with the men's `C` high enough to accept it.

Even when permitted, the sequence is fixed:

```
Run → Halt line 10–15 m short → Reform (1–3 s) → Approach
```

The only exception is pursuit of men who are already **routing**: a fleeing man is not a formation, and running him down
is both realistic and lethal. The halt-and-reform rule applies to any enemy that still has cohesion, no matter how
outmatched.

This is what produces the requested behavior: even a certain victory is a run, a stop, a breath, and then a slow walk
into contact.

**Gameplay tradeoff:** halting short lets broken-but-orderly enemies disengage and reform. That is the point, but it
will make some victories less total. It should be exposed as a tuning option.

---

## 7. Collapse

A rout is not a decision. It is an avalanche with a slow start.

1. **Casualty shock** raises `F` locally.
2. **Wounded men** disengage first, thinning the front.
3. **Shame collapses** as `runningFraction` rises, lowering `F_flee` for everyone who sees it.
4. **Dread** has been accumulating for minutes, so thresholds are already lower than they were at the start.
5. The first runner is expensive and rare. The tenth is nearly free.

Rallying is possible only near a standard or leader, after distance has been gained, and only if `D` is not saturated.
A routed unit that rallies should feel like a genuine recovery, not a reset.

**Leader death** applies a local shock impulse and removes `A` from everyone nearby. In a leader-dependent culture this
can be the difference between a grinding line and a collapse.

---

## 8. Tuning constants

All values are starting points, expected to move during playtesting.

| Constant | Default | Range | Effect |
|---|---|---|---|
| `k_r` fear rise | 0.90 /s | 0.5–1.5 | How fast danger converts to fear |
| `k_f` fear decay | 0.25 /s | 0.1–0.5 | How fast calm returns |
| `θ_d` dread threshold | 0.45 | 0.3–0.6 | Fear level at which dread accumulates |
| `k_d` dread gain | 0.05 /s | 0.02–0.10 | How fast a long fight wears men down |
| `k_dd` dread rest decay | 0.02 /s | 0.01–0.05 | How much a lull actually helps |
| `w_t` threat weight | 1.00 | 1.0–2.0 | Enemy proximity and numbers |
| `w_c` casualty shock | 0.60 | 0.3–1.0 | Weight of seeing friends die |
| `w_h` wound weight | 0.50 | 0.3–0.8 | How much injury feeds fear |
| `base_flee` | 0.55 | 0.4–0.7 | Baseline flight threshold |
| `shameGain` | 0.35 | 0.2–0.5 | How much social pressure suppresses flight |
| `panicContagion` | 0.40 | 0.2–0.6 | How fast a rout accelerates |
| `F_probe` | 0.35 | 0.25–0.45 | Threshold for holding at reach |
| `F_rotate` | 0.60 | 0.5–0.7 | Threshold for rotating out |
| `q_min` | 0.25 | 0.15–0.35 | Below this, flee becomes desperate stand |
| `contactBudget` base | 12 s | 8–20 | How long a man can stay in melee |
| `probeDuration` | 2–6 s | — | Length of a skirmish excursion |
| `rotationBatch` | 12% | 5–20% | Fraction rotated per cycle |
| `contagionRatio` | 1.8:1 | 1.2–3.0 | Panic spreads faster than resolve |
| `dwell` | 1.2 s | 0.8–2.0 | Anti-twitch for formation orders |
| `deadband` | 1.5 m | 1.0–3.0 | Movement below this is ignored |

---

## 9. Bannerlord mapping

Everything in this table is a planned mechanism. Exact patch targets must be confirmed against the installed
assemblies once the game is available; nothing here should be assumed correct until inspected.

| Need | Mechanism | Difficulty |
|---|---|---|
| Per-agent state and simulation | `MissionBehavior.OnMissionTick(float)` with staggered update tiers | Low |
| Perceive hits, deaths, removals | `OnAgentHit`, `OnAgentDie`, `OnAgentRemoved`, `OnAgentBuild` | Low |
| Nearby enemies and allies | `Mission.GetNearbyEnemies`, `GetNearbyAllyAgents`, cached ~0.25 s | Low |
| Personal standoff and probe movement | `Agent.SetScriptedPosition`, `SetScriptedPositionAndDirection`, `DisableScriptedMovement` | Medium |
| Line-level movement | `Formation.SetMovementOrder` with `MovementOrder.Move / StandGround / Advance / FallBack / Retreat` | Medium |
| Prevent the native charge from winning | Harmony patch at formation behavior selection, or persistent order reassertion | High |
| Fatigue effects | Custom `AgentStatCalculateModel` feeding speed and swing multipliers | Medium |
| Barks, shouting, panic audio | `Agent.MakeVoice` | Low |
| Debug telemetry | Overlay showing `F`, `D`, `E`, state per agent | Low |
| Configuration | MCM settings for every value in §8 | Low |

**Performance budget:** target under 1.5 ms per frame at 1,500 agents.

| Tier | Population | Update rate |
|---|---|---|
| In contact or within 15 m of an enemy | ~200 | 0.15 s |
| Within 40 m | ~500 | 0.5 s |
| Rest of field | ~800 | 2.0 s |

Neighbour queries are the expensive part. Cache them, share them across agents in the same formation, and never call a
radius query per agent per frame.

---

## 10. Failure modes

| Failure | Symptom | Mitigation |
|---|---|---|
| Oscillation | Formations twitch forward and back | Hysteresis, dwell time, deadband |
| Stalemate | Two lines stand at spear length forever | Commitment pressure, missiles, officers, flanking |
| Passivity | Nobody ever closes, battles never resolve | Floor on `p_probe`, tune `F_probe` down, cap standoff |
| Robotic movement | Scripted men look like puppets | Short pulses, prefer formation orders, never script continuously |
| Rotation chaos | Men mill around inside the formation | Batch caps, route through gaps, statistical fallback |
| Total escape | Pursuit halts let routers reform endlessly | Tune halt distance, allow direct chase of routing men |
| Performance collapse | Stutter in large battles | Update tiers, cached queries, profile early |
| Compatibility | Conflicts with RBM or other AI overhauls | Detect and disable overlapping subsystems, document load order |

---

## 11. Implementation phases

**P0 — Harness and telemetry.** Module skeleton, mission behavior, debug overlay. Nothing visible in battle yet; the
goal is to see `F`, `D`, `E`, and state for any agent. Tuning is impossible without this.

**P1 — Approach, halt, probe.** The highest-value phase and the one that changes the feel of battles most. No rotation,
no rout changes. If this phase alone does not make battles feel different, the model needs revision before anything
else is built on it.

**P2 — Contact, disengage, rotation.** The experimental phase. Expect to rewrite it at least once.

**P3 — Collapse and pursuit.** Contagion, shame collapse, rallying, the pursuit protocol.

**P4 — Culture, training, battle type, MCM.** The full parameter surface.

**P5 — Compatibility and performance.** RBM coexistence, large-battle profiling, load-order documentation.

---

## 12. Open questions

1. How hard does native melee actually lock an agent? This determines whether disengage is feasible or whether the mod
   must rely entirely on pre-contact behavior.
2. Is there a clean patch point for formation behavior selection, or must orders be reasserted every tick?
3. Can rotation be made to look acceptable, or is the statistical fallback the real design?
4. How much stalemate is tolerable before it reads as a bug rather than tension?
5. Does this compose with RBM, or do the two systems need to be mutually exclusive?
6. Should the player be able to see individual fear, or does that break the illusion that soldiers are people?
