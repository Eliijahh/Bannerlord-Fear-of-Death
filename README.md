# Fear of Death

A Bannerlord mod that makes soldiers behave like people who want to live.

Vanilla melee AI issues one order to a formation and every man executes it identically. The result is a wall of men
walking into a wall of men with no hesitation, no flinching, no yielding ground, and no individual sense that the
next step might kill them.

This mod gives every soldier a private calculation: how dangerous is this, how tired am I, how many friends are
still standing next to me, and is running even possible? Group behavior — the slow approach, the halt at spear
length, the skirmishing fringe, the grinding rotation, the collapse — is not scripted. It is what happens when a
few hundred people each make that calculation at once.

## Status

Design phase. No code yet.

- [docs/DESIGN.md](docs/DESIGN.md) — the model, the state machine, tuning ranges, and the Bannerlord API mapping.

## Core rule

There is no army-level morale variable driving anything. Formation behavior is a *readout* of individual state,
never an input to it. If a group-level number ever becomes an independent cause of individual behavior, the design
has drifted and should be corrected.
