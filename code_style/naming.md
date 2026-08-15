# Code Style — Naming

Conventions for what we name things and, as importantly, what we never name by
hand. Each entry records the concrete ruling and, where it generalizes, the
principle behind it.

## Opaque / random-looking class names — never author or target them

**Ruling (qack, 2026-08-15):** A class like `.surface-item.s-HmxhlUSUYybo` was
found setting `background: #fafafa`. Names like `s-HmxhlUSUYybo` must never be
used intentionally — not written by hand, not depended on, not targeted in a
selector.

**Where these actually come from (so the rule is applied correctly):** in this
stack the `s-<hash>` / `svelte-<hash>` token is **compiler output** — Svelte
generates it to *scope* a component's `<style>` to that component. A developer
does not type it; the compiler stamps it onto the component's elements and
prefixes it to the component's own CSS selectors. It is not a bug in itself. The
anti-pattern is anything that **treats such a name as stable or meaningful**:

1. **Never hand-write an opaque/hash-like class name** (`s-x9f2`, `c-1`, `wrap2`).
   A class name is documentation for the next reader; a random string documents
   nothing. Use a semantic name that says what the element *is*
   (`.surface-item`, `.severity-chip`), never how it happens to be scoped.
2. **Never write a selector that targets a generated scope hash**
   (`.foo.s-HmxhlUSUYybo { … }`). The hash is derived from the component's style
   content, so it **changes the moment the styles change** — a rule keyed on it
   silently stops matching after the next edit, or keeps matching a stale build.
   Style the semantic class; let the compiler scope it.
3. **If the generated names leak into debugging and hurt**, that is a signal to
   configure the compiler's `cssHash` for deterministic, readable output — not a
   signal to start writing or matching the raw hashes yourself.

**Generalized principle (framework-agnostic):** A name a machine generates is an
*implementation detail of scoping/bundling*, never part of your public surface.
Author only names that carry meaning to a human; never bake a generated,
content-derived identifier (a scope hash, a content-hash filename, a minified
symbol) into hand-written code or selectors — it is unstable by construction, so
anything keyed on it breaks or goes stale on the next build. Semantic on the
inside, generated on the outside, and never cross the streams.
