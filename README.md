# design-patterns

A design-pattern library for Rodeo-Org products — reusable UI/UX and operational conventions distilled
from specific product rulings. Each doc captures the concrete ruling verbatim and, where the ruling
encodes a reusable principle, extracts the general (palette-, project-, and stack-agnostic) form so it
can be applied elsewhere.

## Contents

- [`profile_pages.md`](profile_pages.md) — profile-page conventions (exit/destructive-action placement
  & hue).
- [`code_style/naming.md`](code_style/naming.md) — naming conventions; incl. never authoring or
  targeting opaque/compiler-generated class names (e.g. Svelte scope hashes like `s-HmxhlUSUYybo`).
- [`deploy_scripts.md`](deploy_scripts.md) — deploy-script conventions (branch-parameterized
  pull→build→run; always rebuild from source, never serve a stale prebuilt artifact).

## Convention

When a product decision is worth generalizing, record it here: **the specific ruling** (attributed +
dated) followed by **the generalized principle**. Keep principles palette-relative and framework-neutral
so the same rule holds across themes and rebrands.
