# Design Patterns — Profile Pages

A running record of profile-page design decisions. Specific rulings are captured verbatim; where a
ruling encodes a reusable principle, the general form is extracted so it can be applied elsewhere.

## Exit / destructive actions — placement & hue

**Ruling (Rodeo profile page, 2026-08-14):** Sign-out sits at the **very bottom of the profile page,
centered**, and its color is shifted toward a redder hue — **burgundy** in the current palette.

**Generalized principle (palette- and project-agnostic):**

Session-ending and destructive actions — *sign out, delete, leave, revoke, disconnect, remove* —
should be visually separated from primary/constructive actions along three axes:

1. **Hue.** Shift them toward the warmer/redder end of the *active* palette — a muted **burgundy /
   darkened red** that harmonizes with the design's neutrals, **not** a saturated alert-red (which
   reads as an error, not a deliberate exit). The exact value is palette-relative: derive it from the
   palette's own red/neutral ramp rather than hardcoding a hex, so the same rule produces a coherent
   result in any theme (light, dark, or a rebrand).
2. **Weight.** Give them *lower* visual weight than the primary action — ghost / outline / text, not a
   filled button. The hue signals "this ends or removes something"; the weight signals "this is not the
   thing you came here to do."
3. **Position.** Place them out of the primary action flow — e.g. page-bottom, centered — so they're
   reachable but not accidentally triggered next to constructive controls.

The intent is *legible finality without alarm*: a person should recognize an exit/destructive control
on sight and never fire one by reflex, in any palette the product wears.
