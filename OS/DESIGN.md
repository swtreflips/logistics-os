# Design

How the modules look, and why they look like one product.

This document exists to be **decided from**, not debated. If a question about colour, spacing or
motion comes up, the answer is here or it becomes here.

---

# The idea

You already know the feeling from Android and from NetSuite: every screen is obviously part of
one system, and yet you always know which part you are in. That is not achieved by making
everything identical. It is achieved by holding the things that carry meaning constant, and
being deliberate about the few that are allowed to differ.

> **The invariants carry the family. The accent carries the module.**

Type, neutrals, ink and — most importantly — **what each colour means** never change between
modules. The accent does, and it is the only variation whose job is to say *which module you
are in*.

A module may also depart from the base on density, pace or a semantic colour, where its work
genuinely differs. Those are local character, not identity, and they are permitted only when
written down — see the table below. The distinction matters: identity is deliberate signalling,
character is a considered exception, and drift is neither.

This is why "make them all the same colour" is the wrong instinct. Four apps in matching
colours still feel unrelated if a dark surface means *navigation* in one and *selected* in
another. Consistency of MEANING is what reads as one system. Consistency of hue is not.

---

# What is fixed, and what a module may vary

Two lists. Confusing them is how a design system either strangles a module or stops meaning
anything, so the line between them is drawn explicitly.

## Always identical

Never varies. Defined once in `linen.css` and loaded by every module from the same file.

| | Value | Token |
|---|---|---|
| UI type | DM Sans | `--font-sans` |
| Headings | Fraunces | `--font-display` |
| Numbers, codes, labels | DM Mono | `--font-mono` |
| Neutral surfaces | warm greys | `--c-fog-*` |
| Ink | deep warm green-black `#112424` | `--c-harbor-*` |
| **What each surface MEANS** | see below | — |
| **What the accent is for** | see below | — |

The last two carry more weight than any value above them. A module may use a different green;
it may not decide that dark means something else.

**Numbers always use DM Mono with tabular figures.** In a logistics UI, numbers are read down a
column and compared. Proportional digits make columns ragged and comparison slower. Not a style
preference.

## Base defaults a module may vary — if it is recorded here

| | Base | Varied by |
|---|---|---|
| Accent | — | **every module**, one each (table below) |
| Corner radius | 10 / 14 / 20px | Schedules: square (`--radius-none`) |
| Motion | 180 / 340 / 560ms | Schedules: 120–150ms |
| Depth | soft, diffuse | Schedules: none — hairlines instead |
| Positive / success | `--c-sea-*` green | Planner: teal `#16809a` |

**Schedules varies density and pace on purpose.** It is a reading tool — square, hairlined,
immediate — and forcing rounded corners and 340ms easing on it would be overwriting a design
rather than skinning one. It takes the palette, the type and every rule about meaning; it keeps
its own shape and tempo. That is a decision, recorded, not a drift.

**The planner varies success because its accent is green.** Two greens side by side read as one
colour, and the "saved" flash stopped meaning anything. Which is the general rule: *semantic
colours must stay distinguishable from the MODULE ACCENT, not merely from each other.* Check it
whenever an accent is assigned.

Anything not in this table is not variable. Add a row before you vary something, or you are
drifting rather than deciding.

---

# Surfaces: the three levels

Every screen in every module is built from exactly three surfaces. This is the part that was
inconsistent, and it is the part that most affects whether the estate feels like one thing.

| Surface | What it is | Token |
|---|---|---|
| **page** | the canvas everything sits on | `fog-50` |
| **raised** | cards, panels, grids — content you act on | white |
| **inverse** | dark. **Global navigation chrome ONLY** | `harbor-950` |

## What dark means

**Dark = "this is the application frame, not your content."** The sidebar. The top bar. That is
the entire list.

This rule matters more than any colour choice here, because dark is the strongest signal on a
screen and the eye reads it first. It is worth recording what it cost to get wrong: for a while
the same ink meant the sidebar in RatesApp, the column header in the planner, and "this chip is
selected" in Schedules. A user moving between modules was told something different by the
loudest element on every screen — and no amount of palette alignment would have fixed it,
because the palette was already identical.

**Column headers are content furniture, not chrome.** Light background, mono uppercase label.

**Selected state is the accent's job, not ink's.**

Both are now true everywhere. The rule earns its place at the top of this document because it
is the one that was violated silently and mattered most.

---

# The accent

One per module, and the only variation that exists to answer *where am I*. It is how you know
without reading a word.

| Module | Accent | Fill `500` | Text / border `600` |
|---|---|---|---|
| **Rates** | terracotta | `#cd7146` | `#ad552a` |
| **Schedules** | harbour blue | `#099acf` | `#107ca6` |
| **Planner** | container green | `#45a465` | `#25864a` |
| **Inbound** | transit violet | `#907dd6` | `#7461b5` |

These are siblings by construction, not four colours picked to sit near each other: identical
lightness, identical chroma, four hues. They carry the same visual weight, so no module shouts
louder than another.

All four clear WCAG AA in all three roles they are used in — as text on white, as a fill under
white text, and as a fill under ink. Verified, not assumed: `tools/skins/contrast.py`.

## What the accent is for

**Only two things:**

1. The **primary action** on a screen — one per screen, the thing you most likely came to do.
2. The **current selection or active state** — the row you are on, the tab you are in.

## What the accent is never for

Decoration. Section headings. Icons that are not interactive. Chart series. Borders on things
you cannot click. Every extra use makes the two real uses quieter, and an accent that is
everywhere points at nothing.

---

# Rules with teeth

**1. Anything styled outside CSS opts out of the theme, silently.**

Learned three times in one day — MUI `sx` blocks in RatesApp, `themeQuartz` params in the
planner, MapLibre paint in Inbound. In each case the surrounding page changed and that one
element kept the old palette, with no error anywhere. When you style through JavaScript, read
the token: `getComputedStyle(document.documentElement).getPropertyValue('--c-…')`.

**2. Never write a hex value in a component.** If a colour is needed that the skin does not
define, the skin gains it. A hex in a component is a value that cannot be re-themed and will be
missed.

**3. A second copy of a style rule is a second thing to remember.** `NewRateRequest` carried a
near-duplicate of the shared grid styling and kept maritime colours after everything else moved.
Import the shared object; override only genuine differences.

**4. Muted text is `fog-500`, never `fog-400`.** 4.97:1 versus 3.1:1. `fog-400` is for borders,
icons and disabled states — things that are not read.

**5. Fills and text need different shades of the same accent.** `500` is tuned to sit *under*
near-black text as a button fill, which is exactly what makes it fail *as* text. Text and
borders take `600`. This caught every single module.

**6. Run the contrast audit before shipping a palette change.** `python tools/skins/contrast.py`.
It is fifteen pairs and it takes a second. The design shipped for months with a focus ring at
2.04:1 against a 3:1 requirement, and nobody noticed by looking.

---

# Where the estate stands

True today, across all four modules:

- All four load the same `linen.css`, with the same variable names
- No component anywhere hardcodes a palette colour
- Grid, map and MUI themes read tokens rather than literals
- Dark is global navigation chrome only — the planner's grid header and container tray are
  light and match RatesApp's grids; Schedules' ten ink fills use its accent instead
- Every module carries its own accent, and all four clear AA in every role they are used in
- Every variation from the base is recorded in the table above

**One thing is not aligned: RatesApp's sidebar is the only gradient in the estate**
(`--bg-harbor-mesh`). Either every module's chrome adopts it once they are behind one shell, or
it flattens to solid ink. Two treatments of the same surface is exactly the inconsistency this
document exists to remove — and it stays invisible until two modules sit side by side, which is
precisely when it will be most annoying to fix.

---

# Adding a module

Before writing any UI:

- [ ] Load `linen.css` from `public/skins/`, via `<link id="skin">` — the same file, not a copy
- [ ] Pick an accent hue **at the same lightness and chroma** as the four above, and run the
      contrast audit
- [ ] Dark is chrome only
- [ ] Accent is primary action and selection only
- [ ] Numbers are DM Mono, tabular
- [ ] Nothing styled through JavaScript uses a literal colour
- [ ] Add the module and its accent to the accent table the day it is created
- [ ] Record any departure from the base — density, pace, a semantic colour — in the
      variations table, with the reason. An unrecorded variation is drift

The cheapest moment to inherit the system is before the first screen exists. Inbound proved
that: it had no design of its own, and adopting the base cost four small stylesheets. Retro-
fitting the other three cost considerably more.
