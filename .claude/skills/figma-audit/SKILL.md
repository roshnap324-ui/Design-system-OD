---
name: figma-audit
description: >-
  Audit and fix a Figma frame/node against the Ooredoo design system. Use this
  whenever the user wants to check, QA, review, lint, or clean up a Figma design —
  alignment and spacing, design-system token compliance (colors/fonts/radii),
  whether the design is in sync with the source, or instance/component health —
  and whenever they say things like "check alignment", "find errors in this
  Figma", "is this design correct", "audit this frame", "fix the spacing",
  "why does this look off", or paste a Figma node URL asking what's wrong.
  Trigger it even when the user only describes symptoms ("this frame looks messy",
  "the colors seem inconsistent", "text is cut off") rather than naming an audit.
  The skill reports findings first, grouped by severity with node IDs, then
  applies fixes only after the user confirms.
---

# Figma Audit & Fix (Ooredoo DS)

Audit a Figma node against the Ooredoo design system, produce a findings report,
then fix on confirmation. Four lenses: **alignment/layout**, **DS token
compliance**, **content sync**, **instance/component health**.

The value of this skill is that it catches the things a screenshot glance
misses — a container that's secretly opaque white, a color that's a hardcoded hex
instead of a bound token, text that's clipped by two pixels, a font that quietly
became Inter. These are invisible until they aren't, so the audit reads the node
tree programmatically rather than trusting the eye.

## Prerequisites & rules

- **Load the `figma-use` skill before every `use_figma` call.** It carries API
  rules (color ranges, font loading, traversal gotchas) that every script here
  depends on. Pass `skillNames: "figma-use"` on each call.
- **Source of truth is code, not the Figma file.** The canonical tokens live in
  `index.html` (`:root` custom properties) and are
  mirrored in the Figma variable collection **"Ooredoo DS"** (`color/*`,
  `radius/*`). Per the project's `CLAUDE.md`, never treat a Figma source design's
  raw hex/font as correct — map it to the closest DS token.
- **Only touch the node the user names.** Audit and fix that node and its
  descendants. Never modify sibling frames or other screens.
- **Report before you fix.** Always deliver the findings report and get a "go"
  before mutating the canvas — this file is shared. The one exception is a
  read-only screenshot.
- **Scripts are atomic.** A failed `use_figma` script creates nothing, so on
  error, fix the script and re-run rather than patching half-applied state.
- **Trust the server render, not the plugin's.** The plugin's
  `node.screenshot()` (called from inside `use_figma`) can render a node with its
  *intended* variable colors even when the committed canvas differs — it has
  masked real bugs (black text that looked white). For any visual verdict, use
  the **`get_screenshot` MCP tool** (server-side render), which shows what the
  user actually sees. When they disagree, the server render wins.

## Workflow

### Step 1 — Identify the target

Get the node from the user. From a URL
`https://figma.com/design/:fileKey/:name?node-id=24-21755`, extract
`fileKey` and `nodeId` (`24-21755` → `24:21755`). If no node-id is present, ask
for a node-specific link or selection — auditing a whole page blind is rarely
what they want.

Take a `get_screenshot` of the node first, so you have a visual reference to
cross-check programmatic findings against (e.g., confirm a "clipped text"
finding is actually visible).

### Step 2 — Load the reference tokens

Read the DS tokens once so the audit can compare against them:

- Grep `index.html` for the `:root` block to get the
  color hexes, radii (`--radius-*`), and the two font families (`Rubik` for
  headings, `Noto Sans` for body; `Outfit` only appears in the doc's own spec
  labels, not product UI).
- Build the Figma variable map (`name → {id, hex}`) from the "Ooredoo DS"
  collection with the snippet in
  [references/audit-checks.md](references/audit-checks.md#loading-the-token-map).

This gives you both directions: "what hex should this token be" and "which token
does this hex belong to".

### Step 3 — Run the four audits

Run each audit as its own `use_figma` traversal (they return structured
findings; nothing is mutated). The full, ready-to-run scripts and the exact
rules for each live in
[references/audit-checks.md](references/audit-checks.md) — read that file and use
its scripts rather than improvising traversals, because the gotchas (getter
properties that throw, opacity that doesn't read back, image fills) are subtle.

The four lenses, and what each is really looking for:

- **A. Alignment & layout** — off-pixel geometry (fractional x/y/w/h), siblings
  in a stack whose leading edges don't line up, inconsistent padding/gap inside
  one container, overlapping siblings, and text that's clipped (fixed-height text
  whose content needs more room). Misalignment usually means a value was typed by
  hand instead of flowing from auto-layout.
- **B. DS token compliance** — `SOLID` fills/strokes that are **not** bound to a
  variable, and especially **unbound opaque white** on container frames (Figma's
  `createAutoLayout`/`createFrame` seed a white fill, so transparent-looking
  wrappers are often secretly white — invisible on light sections, glaring on
  dark ones). Also: colors whose hex matches a DS token but aren't bound (should
  be bound), colors that match no token at all (new value — decide per CLAUDE.md),
  fonts outside {Rubik, Noto Sans}, and corner radii outside {4, 8, 12, 16, 100}.
- **C. Content sync** — placeholder/default text still present ("Title",
  "Heading", "Lorem", "Button", empty strings), and — if the user gives a
  reference node (e.g. the desktop source) — a text diff of the two so outdated
  or missing copy surfaces.
- **D. Instance/component health** — instances whose main component is missing
  (`mainComponent === null`, i.e. detached/deleted), empty text layers, and
  layers still carrying default names that hint at unfinished work.

### Step 4 — Deliver the findings report

Consolidate every finding into one report. Assign severity by user impact, not
by category:

- 🔴 **High** — visibly broken or wrong: clipped/overlapping content, stray
  opaque fill on the wrong background, detached instance, wrong font family,
  empty text where copy is expected.
- 🟡 **Medium** — DS non-compliance that isn't yet visibly broken: unbound color
  that matches a token, off-token radius/spacing, hand-set alignment that drifts
  from the grid.
- 🔵 **Low** — polish: default layer names, sub-pixel geometry, minor spacing
  variance, redundant wrappers.

Use this structure exactly:

```
# Figma Audit — <node name> (`<nodeId>`)
Scanned <N> nodes.  🔴 <h>  🟡 <m>  🔵 <l>

## 🔴 High
1. **<short title>** — `<nodeId>`  ·  <path e.g. Footer › Follow row>
   - Problem: <what's wrong, with the concrete value>
   - Fix: <the specific change, e.g. "clear fill (currently #FFFFFF @1, unbound)">

## 🟡 Medium
...

## 🔵 Low
...

## Suggested fix batch
<one line: which groups you'd fix automatically vs. which need a decision>
```

Every finding must carry a real node ID so the user can jump to it and so the fix
step can target it. If a lens produced nothing, say so briefly rather than
omitting it — "no content-sync issues" is useful information.

Then ask which findings to fix (all High, everything, a specific list, etc.).
Do not fix yet.

### Step 5 — Fix on confirmation

Apply only the fixes the user approved, grouped by type, each as an atomic
`use_figma` call. Fix patterns (with the correct API for each, including the ones
that bite — binding a color variable, clearing stray fills, nudging to integer
geometry, equalizing padding) are in
[references/fix-patterns.md](references/fix-patterns.md).

After fixing, `get_screenshot` the node again and confirm the change looks right
and nothing else moved. Report what changed with node IDs (so the user can undo
selectively), and re-run the relevant audit lens to confirm the findings cleared.

## Notes on judgment

- A finding is a hypothesis. Before reporting "text clipped" or "colors
  inconsistent", sanity-check against the screenshot — a translucent overlay is
  not a bug, and centered text with different widths is not misalignment.
- When a color matches no DS token, don't silently invent one. Follow CLAUDE.md:
  either map to the nearest existing token or, if it's genuinely new, propose
  adding it to `:root` (named after the Figma leaf) and to the "Ooredoo DS"
  collection — and surface that as a decision, not an auto-fix.
- Prefer fixes that remove hand-set values in favor of auto-layout/tokens over
  fixes that hand-set a "corrected" number. The goal is a design that stays
  correct as it changes, not one that's correct in this one screenshot.
