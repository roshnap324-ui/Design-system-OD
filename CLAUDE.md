# Project Rules — Ooredoo Design System

## Source of truth
`index.html` is the canonical design system. Its `:root`
CSS custom properties (colors, radii, shadows, spacing) and its component CSS
(`.btn-*`, `.proto-*`, `.track-*`, swatches, etc.) define every allowed style value.

## Core rule — design-system-first (applies to BOTH code generation and Figma push)
When generating solution code **or** pushing/recreating a design in Figma:

1. **Strictly follow the existing design system properties.** Use the DS tokens and
   components defined in `index.html`. Reuse existing
   components/classes rather than re-building primitives.
2. **Do NOT follow colors or font styles from the Figma source design.** Map every
   Figma color/typography value to the closest existing DS token. Never hardcode a
   hex/font pulled from Figma when a DS token exists.
3. **Adding new values is allowed only when missing** from the DS. If a Figma color/
   style has no DS equivalent:
   - Add it to `:root` in `index.html` using the **proper
     name used in Figma** (mirror the Figma variable path's leaf name, e.g.
     `colors/statuses/success` → `--color-notification-success-500`), and
   - Document it as a swatch in the Colors section.
4. **No repeated color tokens.** If a value already exists, alias the new token to the
   existing one via `var(--existing)` (keep the name working, single source of truth)
   instead of repeating the literal hex.

## Token naming conventions
- Colors: `--color-<figma-leaf-name>` (e.g. `--color-ooredoo-blue-500`,
  `--color-notification-warning-100`, `--color-mono-gray-100`).
- Aliases point to the canonical token: `--toggle-active-bg: var(--color-ooredoo-blue);`

## Reading Figma variables
To pull variables from a Figma node, read them via the plugin API (`use_figma`):
walk the node + descendants, collect `boundVariables`, resolve each variable
(following `VARIABLE_ALIAS` chains) and convert `{r,g,b}` 0–1 to hex. This gives exact
Figma name + value without relying on a selection or screenshots.

## Pushing designs to Figma (code-to-design)
- Build natively with auto-layout (`figma.createAutoLayout`), set `FILL`/`HUG` only
  after `appendChild`.
- Use DS values (mapped as above), not the Figma source's raw colors/fonts.
- Only touch the screen/node the user names — never modify other screens.
