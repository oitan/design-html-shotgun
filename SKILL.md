---
name: design-html-shotgun
description: Generate multiple full-fidelity HTML mockup variants for a screen, served through a tab-switching iframe board with per-variant scroll memory. Use when asked to "shotgun designs", "explore design variants", "show me 3-5 takes on this screen", "design board", or when comparing static UI directions side-by-side before committing to one. Each variant is a standalone full-page mockup styled with the project's existing CSS tokens; the framework (sticky tab bar + iframe slot) is reused across sessions and projects.
---

# design-html-shotgun

## What it does

Generates N standalone HTML mockup variants for a screen (or flow of screens), drops them into a tabbed iframe board, and opens it in the user's browser. Each tab swaps `iframe.src`; scroll position is remembered per variant across switches and reloads via `sessionStorage`.

The **framework** (tab bar + iframe + scroll memory) is static and ships once with the skill. Each **session** generates only fresh variant HTML files + a `manifest.js` describing them. The framework is copied (not symlinked) into each session dir so the board is self-contained and shareable.

## When to use

- User asks: "shotgun designs", "explore variants", "show me 3-5 takes", "design board", "compare directions".
- User has a screen or flow that needs visual exploration before implementation.
- User wants standalone HTML (not React/Svelte/etc) — pixel-level mockups to share, screenshot, or iterate on.

Do **not** use when:
- User wants production framework code → use `/design-html` (Pretext) or `/design-shotgun` (live components).
- User wants an interactive prototype with state → different tool.

## Workflow

### Phase 1 — Detect project tokens

Read whichever of these exists, in order:
- `app/globals.css`
- `globals.css`
- `styles/globals.css`
- `src/styles/globals.css`
- `tailwind.config.{js,ts}` (for theme.extend.colors)

Extract:
- `:root { --foo: ... }` custom properties
- Font families from `@import` / `font-family` rules
- Brand colors

If nothing found, ask the user via `AskUserQuestion`:
- Primary color (hex)?
- Font family?
- Background tone (light/dark)?

Save the extracted/asked tokens as a snippet to embed in each variant's `<style>` block.

### Phase 2 — Scope the shotgun

Ask via `AskUserQuestion` (use multiple questions in one call):

1. **What screen/flow?** Free-form description.
2. **How many variants?** 3 / 4 / 5.
3. **Language?** (so copy reads naturally — default to project's language if obvious from existing files, otherwise English.)

### Phase 3 — Brainstorm concepts

Generate N distinct design concepts. Each has:
- **ID**: short letter (A, B, C, D, E).
- **Name**: 1-3 words capturing the angle ("Minimal", "Card-heavy", "Inline chip", "Discovery panel").
- **One-line pitch**: what makes it different.

Present the list to the user via `AskUserQuestion` ("Approve these concepts? Edit any?"). If user wants edits, regenerate. Don't proceed without explicit OK.

### Phase 4 — Generate session dir

```bash
# Try gstack-slug, fall back to git basename, fall back to dir basename
if command -v gstack-slug >/dev/null 2>&1; then
  SLUG=$(gstack-slug)
elif git rev-parse --show-toplevel >/dev/null 2>&1; then
  SLUG=$(basename "$(git rev-parse --show-toplevel)")
else
  SLUG=$(basename "$PWD")
fi

DATE=$(date +%Y%m%d)
# NAME = kebab-cased description of the screen, ≤30 chars
SESSION_DIR="$HOME/.gstack/projects/$SLUG/designs/$NAME-$DATE"
mkdir -p "$SESSION_DIR"
```

If `$SESSION_DIR` already exists, append `-2`, `-3`, etc.

### Phase 5 — Write variants

For each concept, write `$SESSION_DIR/variant-{ID}.html` as a **complete standalone HTML page**:

- Full `<!doctype html>` + `<html>` + `<head>` + `<body>`.
- Embed project CSS tokens at the top of `<style>` (extracted in Phase 1).
- No iframe-aware code, no scroll-restoration script, no tabs. The framework handles that.
- Each variant should render multiple screens of the flow stacked vertically if the user requested a flow (use `<section>` with `<h2>` labels, or visual frames with `frame-label` rules).
- Use realistic copy (not Lorem Ipsum). If the user has a domain (e.g., marketplace, dashboard), use plausible domain data.
- Aim for **full fidelity**: real tables, real components, hover states inline if cheap. ≤2k lines per variant is fine.

### Phase 6 — Write manifest.js

```js
// $SESSION_DIR/manifest.js
window.DS_MANIFEST = {
  id: "campaigns-tracking-20260519",
  title: "Design Shotgun — /campaigns tracking UX",
  subtitle: "4 directions · scroll is remembered per variant",
  variants: [
    { id: "A", label: "A — Minimal",    file: "variant-A.html" },
    { id: "B", label: "B — Card",       file: "variant-B.html" },
    { id: "C", label: "C — Inline chip", file: "variant-C.html" },
    { id: "D", label: "D — Discovery",  file: "variant-D.html" }
  ],
  notes: "Optional free-form notes shown nowhere yet — reserved for future."
};
```

The `id` field namespaces the `sessionStorage` key so different boards don't collide.

### Phase 7 — Copy framework

```bash
SKILL_DIR="$HOME/.claude/skills/design-html-shotgun"
cp "$SKILL_DIR/framework/framework.html" "$SESSION_DIR/index.html"
cp "$SKILL_DIR/framework/framework.css"  "$SESSION_DIR/framework.css"
cp "$SKILL_DIR/framework/framework.js"   "$SESSION_DIR/framework.js"
```

Note: `framework.html` is copied as `index.html` so the dir opens cleanly in finder/file servers.

### Phase 8 — Open

```bash
open "$SESSION_DIR/index.html"
```

(macOS. On Linux: `xdg-open`. On Windows: `start`.)

Tell the user:
- Session dir path.
- "Switch variants via the tab bar. Scroll position is remembered per variant — refresh keeps state."
- Ask what they want next: rate, comment, iterate on one variant, generate more, or finalize.

### Phase 9 — Iterate

On user feedback ("variant B but tighter", "merge A's table with D's chips"):
- Write new `variant-{NEW_ID}.html` (use next letter, or `A2`, `A3` if revising one).
- Append to `manifest.js` `variants` array (read existing file, edit in place — don't overwrite earlier variants unless asked).
- Tell user to reload the open board tab. No need to re-copy framework files.

When user picks a winner and wants to ship it, suggest handing off to `/design-html` (Pretext-native) or directly implementing in the project framework.

## Constraints

- **Same-origin requirement**: scroll memory reads `iframe.contentWindow.scrollY`. This works on `file://` for files in the same directory in Chrome, Safari, Firefox. Don't put variants on different origins.
- **No external JS in variants**: variants should be self-contained. Inline `<style>`, inline `<script>` (if any), no `<script src="...">` to a CDN unless it's a font CSS link.
- **Fonts**: Google Fonts via `<link>` is fine. Keep it to one family per board for visual coherence unless the variants intentionally explore typography.
- **Don't re-implement scroll memory in variants** — the framework does it from outside the iframe.

## Files this skill manages

```
~/.claude/skills/design-html-shotgun/
├── SKILL.md              (this file)
├── README.md             (human-facing install)
├── LICENSE               (MIT)
└── framework/
    ├── framework.html
    ├── framework.css
    └── framework.js
```

Per-session output (one dir per shotgun run):

```
~/.gstack/projects/$SLUG/designs/$NAME-$YYYYMMDD/
├── index.html            (copy of framework.html)
├── framework.css
├── framework.js
├── manifest.js
├── variant-A.html
├── variant-B.html
├── ...
```
