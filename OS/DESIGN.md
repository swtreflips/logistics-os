# Design

How the modules look, and why they look like one product.

This document exists to be **decided from**, not debated. If a question about colour, spacing or
motion comes up, the answer is here or it becomes here.

---

# The idea

You already know the feeling from Android and from NetSuite: every screen is obviously part of
one system, and yet you always know which part you are in. That is not achieved by making
everything identical. It is achieved by holding almost everything constant and letting exactly
one thing vary.

> **The invariants carry the family. One variable carries the module.**

Type, spacing, depth, motion, component shape and — most importantly — **what each colour
means** never change between modules. A single accent colour does.

This is why "make them all the same colour" is the wrong instinct. Four apps in matching
colours still feel unrelated if a dark surface means *navigation* in one and *selected* in
another. Consistency of MEANING is what reads as one system. Consistency of hue is not.

---

# The invariants

Identical in every module. Defined once, in `linen.css`, loaded by every app from the same file.

| | Value | Token |
|---|---|---|
| UI type | DM Sans | `--font-sans` |
| Headings | Fraunces | `--font-display` |
| Numbers, codes, labels | DM Mono | `--font-mono` |
| Neutral surfaces | warm greys | `--c-fog-*` |
| Ink | deep warm green-black `#112424` | `--c-harbor-*` |
| Positive / success | green | `--c-sea-*` |
| Corner radius | 10 / 14 / 20px | `--radius-*` |
| Depth | soft, diffuse | `--shadow-card` |
| Motion | 180 / 340 / 560ms | `--motion-*` |

**Numbers always use DM Mono with tabular figures.** In a logistics UI, numbers are read down a
column and compared. Proportional digits make columns ragged and comparison slower. This one is
not a style preference.

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

This rule matters more than any colour choice in this document, because dark is the strongest
signal on a screen and the eye reads it first. Today the same ink means three different things:

| Module | Dark is currently used for | Correct? |
|---|---|---|
| RatesApp | the sidebar | ✅ chrome |
| Planner | grid header, container-tray header | ❌ that is content |
| Schedules | selected chips, active buttons, primary button | ❌ that is a state |

A user moving between modules is being told something different by the loudest element each
time. Fixing that will do more for coherence than any repaint.

**Column headers are content furniture, not chrome.** RatesApp's grids already treat them that
way — light background, mono uppercase label. That is the pattern; the planner should match it.

**Selected state is the accent's job, not ink's.**

---

# The one variable: module accent

One accent per module. It is how you know where you are without reading a word.

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

Everything below is already true:

- All four modules load the same `linen.css`, with the same variable names
- No component in any module hardcodes a palette colour
- Grid, map and MUI themes read tokens rather than literals
- Every rendered contrast pair passes AA in RatesApp; the others improved on what they had

- Dark is global navigation chrome only. The planner's grid header and container tray band are
  light and now match RatesApp's grids; Schedules' ten ink fills — selected chips, active tab,
  chosen routing, checked filters, the primary button — use its accent
- Every module carries its own accent, per the table above

One thing is not yet aligned:

**RatesApp's sidebar is the only gradient in the estate** (`--bg-harbor-mesh`). Either every
module's chrome adopts it once they are in the hub, or it flattens to solid ink. Two treatments
of the same surface is exactly the inconsistency this document exists to remove — and it will
become visible the moment two modules sit behind one shell.

A note learned applying this: **semantic colours have to stay distinguishable from the module
accent, not merely from each other.** The planner's accent is green, and linen's success colour
is also green — side by side the "saved" flash stopped meaning anything. Success moved to a
true teal. Check this whenever a module accent is assigned.

**Schedules is the awkward one, and that is worth naming.** It is deliberately austere — square
corners, hairlines, no depth, fast transitions — while the platform base is soft, rounded and
unhurried. It has adopted the palette and the type but rejects the shape and motion. That is a
defensible local character, but it is a decision to make on purpose rather than to drift into:
either Schedules relaxes toward the base, or the base accepts "density" as a documented per-
module variation alongside accent.

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
- [ ] Add the module and its accent to the table above the day it is created

The cheapest moment to inherit the system is before the first screen exists. Inbound proved
that: it had no design of its own, and adopting the base cost four small stylesheets. Retro-
fitting the other three cost considerably more.
