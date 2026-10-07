---
name: design-html-shotgun
description: Generate multiple full-fidelity HTML mockup variants for a screen, served through a tab-switching iframe board with per-variant scroll memory. Use when asked to "shotgun designs", "explore design variants", "show me 3-5 takes on this screen", "design board", or when comparing static UI directions side-by-side before committing to one. Each variant is a standalone full-page mockup styled with the project's existing CSS tokens; the framework (sticky tab bar + iframe slot) is reused across sessions and projects.
---

# design-html-shotgun

## What it does

Generates N standalone HTML mockup variants for a screen (or flow of screens), drops them into a tabbed iframe board, serves it on a local HTTP server and opens it in the user's browser. Each tab swaps `iframe.src`; scroll position is remembered per variant across switches and reloads via `sessionStorage`.

The **framework** (tab bar + iframe + scroll memory) is static and ships once with the skill, in `framework/` next to this file. Each **session** generates only fresh variant HTML files + a `manifest.js` describing them. The framework is copied (not symlinked) into each session dir so the board is self-contained and shareable.

## When to use

- User asks: "shotgun designs", "explore variants", "show me 3-5 takes", "design board", "compare directions".
- User has a screen or flow that needs visual exploration before implementation.
- User wants standalone HTML (not React/Svelte/etc) — pixel-level mockups to share, screenshot, or iterate on.

Other jobs belong to gstack skills (see **Handoff**): production framework code → `/design-html` (Pretext); AI-generated image variants → `/design-shotgun`. An interactive prototype with state is a different tool.

## Workflow

Read [`references/design-rules.md`](references/design-rules.md) before Phase 3: anti-convergence, the AI-slop blacklist, UX behavior rules, edge cases and the craft baseline all apply to every variant.

### Phase 1 — Read the project

Read whichever exist, before asking anything:

- `DESIGN.md`, `PRODUCT.md` (design system, audience, voice) — they decide what varies and what stays fixed;
- `app/globals.css`, `globals.css`, `styles/globals.css`, `src/styles/globals.css`;
- `tailwind.config.{js,ts}` (`theme.extend.colors`).

Extract `:root { --foo: ... }` custom properties, font families (`@import`, `font-family`), brand colors. Save them as a snippet to embed in each variant's `<style>`.

Nothing found → ask via `AskUserQuestion`: primary color (hex), font family, background tone (light/dark).

### Phase 2 — Scope

One `AskUserQuestion` call, skipping anything Phase 1 already answered:

1. **What screen/flow?** Free-form.
2. **Who and what job?** Who uses it, what they are trying to get done.
3. **What exists?** Current screen, constraints, must-keep elements.
4. **Edge cases that matter?** Long names, empty, error, mobile — see the rules file.
5. **How many variants?** 3 / 4 / 5.
6. **Language?** Default to the project's language if obvious from its files, else English.

At most two rounds of questions. After that, state your assumptions and proceed.

### Phase 3 — Concepts

Generate N distinct concepts, per the anti-convergence rule. Each has:

- **ID**: letter (A, B, C, D, E).
- **Name**: 1-3 words for the angle ("Minimal", "Card-heavy", "Inline chip").
- **One-line pitch**: what makes it different.

Confirm via `AskUserQuestion`: approve as is / edit a concept / drop one / add one. At most two rounds; after the second, go with the latest list and say which feedback you applied.

### Phase 4 — Session dir

Build the path from sanitized parts only — free text from the user never goes into a shell command as is:

- `SLUG` — project name: `basename` of `git rev-parse --show-toplevel`, else of `$PWD`;
- `NAME` — kebab-cased screen description in Latin letters, ≤30 chars (transliterate non-Latin names: `kabinet`, not `кабинет`).

Both lowercased and reduced to `[a-z0-9-]`; nothing left → `design`:

```bash
clean() { r=$(printf '%s' "$1" | tr '[:upper:]' '[:lower:]' | tr -c 'a-z0-9-' '-' | tr -s '-' | cut -c1-30 | sed 's/^-//; s/-$//'); printf '%s' "${r:-design}"; }
ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
SLUG=$(clean "$(basename "$ROOT")")
NAME=$(clean '<kebab-name you chose>')   # you write NAME; it is already [a-z0-9-]
BASE="$HOME/.design-shotgun/$SLUG/$NAME-$(date +%Y%m%d)"
SESSION_DIR="$BASE"; n=2
while [ -e "$SESSION_DIR" ]; do SESSION_DIR="$BASE-$n"; n=$((n+1)); done
mkdir -p "$SESSION_DIR" && echo "$SESSION_DIR"
```

### Phase 5 — Write variants

For each concept, write `$SESSION_DIR/variant-{ID}.html` as a **complete standalone HTML page**:

- Full `<!doctype html>` + `<html>` + `<head>` + `<body>`.
- Project tokens at the top of `<style>` (from Phase 1).
- No iframe-aware code, no scroll-restoration script, no tabs. The framework handles that.
- A flow → screens stacked vertically, each a `<section>` with an `<h2>` label.
- Realistic copy and plausible domain data, never Lorem Ipsum — including the edge cases from Phase 2.
- **Full fidelity**: real tables, real components, hover and focus states inline. ≤2k lines per variant is fine.
- Meets the craft baseline in the rules file: 375/768/1440, focus styles, dark scheme if the project has one, reduced motion.

**Files are never overwritten.** A name that is taken gets a new one — a revision of A is `variant-A2.html`, not an edit of `variant-A.html`.

### Phase 6 — manifest.js

```js
// $SESSION_DIR/manifest.js
window.DS_MANIFEST = {
  id: "campaigns-tracking-20260519",
  title: "Design Shotgun — /campaigns tracking UX",
  subtitle: "4 directions · scroll is remembered per variant",
  variants: [
    { id: "A", label: "A — Minimal",     file: "variant-A.html" },
    { id: "B", label: "B — Card",        file: "variant-B.html" },
    { id: "C", label: "C — Inline chip", file: "variant-C.html" },
    { id: "D", label: "D — Discovery",   file: "variant-D.html" }
  ]
};
```

`id` namespaces the `sessionStorage` key so different boards don't collide. Use the session dir's basename.

### Phase 7 — Copy framework

`SKILL_DIR` is the directory this `SKILL.md` was loaded from (the "Base directory for this skill" line). Don't assume `~/.claude/skills/...`: installs via `npx skills` live in `~/.agents/skills/` behind a symlink.

```bash
cp "$SKILL_DIR/framework/framework.html" "$SESSION_DIR/index.html"
cp "$SKILL_DIR/framework/framework.css"  "$SESSION_DIR/framework.css"
cp "$SKILL_DIR/framework/framework.js"   "$SESSION_DIR/framework.js"
```

`framework.html` becomes `index.html` so the dir opens cleanly in file servers.

### Phase 8 — Serve and open

Serve over HTTP. Scroll memory reads `iframe.contentWindow`, which needs same origin; Chrome and Firefox give every `file://` page its own origin, so a board opened as a file has no scroll memory there.

Run in the background (Bash `run_in_background`), port 0 = any free port:

```bash
python3 -u -m http.server 0 --bind 127.0.0.1 --directory "$SESSION_DIR"
```

Read the port from its first output line (`Serving HTTP on 127.0.0.1 port 54321`), then:

```bash
open "http://127.0.0.1:<port>/"     # Linux: xdg-open
```

No `python3` → open `index.html` directly and say scroll memory works only in Safari that way.

Tell the user: the session dir, the URL, "switch variants via the tab bar; scroll position is remembered per variant". Stop the server when the session ends.

### Phase 9 — Verify

If a browser automation tool is available (Playwright or Chrome DevTools MCP), load each variant URL directly at 375, 768 and 1440 wide:

- screenshot each and look at it;
- `document.documentElement.scrollWidth > clientWidth` → horizontal overflow, fix it;
- check against the AI-slop blacklist once.

One fix pass, not a loop. Fixes go into a new variant file (see Phase 5). No tool → say the variants were checked by reading only.

### Phase 10 — Iterate

Feedback ("B but tighter", "merge A's table with D's chips"):

- Copy the source variant to the next id (`A` → `A2`), then make **targeted `Edit`s** — never regenerate a whole page for a local change.
- Append it to `manifest.js` `variants` (edit in place; earlier variants stay).
- Tell the user to reload the board tab. Framework files need no re-copy.
- Cap: 10 rounds. Past that, ask whether to finalize or rethink the concepts.

### Phase 11 — Pick a winner

Before finalizing, restate the choice in one message: **preferred variant, notes, direction** — and ask "right?". On yes, write `$SESSION_DIR/approved.json`:

```json
{ "variant": "B2", "file": "variant-B2.html", "notes": "…", "direction": "…", "date": "2026-10-08" }
```

Later runs on the same screen read it first.

## Handoff

The next step after a winner is a **gstack** skill:

- `/design-html` — finalize the chosen variant as production HTML/CSS (Pretext);
- `/design-shotgun` — AI-image variants and taste memory, if the user wants broader exploration.

Check they exist before offering: `ls ~/.claude/skills/design-html/SKILL.md ~/.claude/skills/design-shotgun/SKILL.md`. Missing → say gstack is not installed and suggest:

```bash
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup
```

then restart Claude Code. Without gstack, offer to implement the winner directly in the project's framework.

## Constraints

- **Same origin**: serve the board (Phase 8). Don't put variants on different origins.
- **No external JS in variants**: inline `<style>`, inline `<script>` if any, no `<script src>` to a CDN. Font CSS `<link>` is fine.
- **Fonts**: Google Fonts via `<link>` is fine. One family per board unless the variants intentionally explore typography.
- **Don't re-implement scroll memory in variants** — the framework does it from outside the iframe.

## Files this skill manages

```
design-html-shotgun/
├── SKILL.md                  (this file)
├── README.md                 (human-facing install and update)
├── LICENSE                   (MIT)
├── references/
│   └── design-rules.md       (rules every variant follows)
└── framework/
    ├── framework.html
    ├── framework.css
    └── framework.js
```

Per-session output (one dir per shotgun run):

```
~/.design-shotgun/$SLUG/$NAME-$YYYYMMDD[-n]/
├── index.html            (copy of framework.html)
├── framework.css
├── framework.js
├── manifest.js
├── variant-A.html, variant-B.html, …, variant-A2.html
└── approved.json         (after Phase 11)
```
