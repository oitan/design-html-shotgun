# Design rules for variants

Read before writing any variant (Phase 5) and again before the verification pass
(Phase 9). Adapted from gstack's `design-shotgun` and `design-html` skills
(MIT, Copyright (c) 2026 Garry Tan, https://github.com/garrytan/gstack),
condensed for static HTML mockups.

## Anti-convergence

Each concept is a distinct direction, as if separate design teams made them.

- **Project has tokens or a DESIGN.md:** keep its fonts and palette, vary layout,
  composition, density and information hierarchy.
- **No tokens:** vary font family, palette and layout approach.
- **Swap test:** if the headline text could move between two variants without
  anyone noticing, they are siblings. Rework the weaker one in a deliberately
  different direction.

## AI-slop blacklist

None of these by default. A DESIGN.md that blesses one, or an explicit user ask,
overrides it; say the tradeoff once.

- purple/blue gradients as default; cream-and-serif default palette; gradient text
- generic 3-column feature grids; identical card grids; nested cards
- center-everything layouts with no hierarchy
- kickers or icon tiles above headings; hero metric rows ("10k+ users")
- decorative blobs, waves, geometric patterns; glowing edges; pulsing status dots
- stock-photo placeholder divs; generic testimonial sections
- "Get Started" / "Learn More" generic CTAs
- rounded cards with drop shadows as the default component
- emoji as visual elements
- left-text right-image hero template

## How users behave

- **Don't make me think.** Every screen self-evident. A pause to wonder "what do
  I click?" is a failure.
- **Clicks don't matter, thinking does.** Three obvious clicks beat one puzzling one.
- **Omit, then omit again.** Halve the words, then halve again. No happy talk,
  no instructions that need reading.
- **Users scan, satisfice, muddle through.** Prominence = importance. Make the
  right choice the most visible one.
- **Conventions over cleverness.** Logo top-left, nav top/left, magnifier = search.
  Innovate only where you know you have a better idea.
- **Clickable looks clickable** without hover; mobile has no hover.
- **Eliminate noise** by removal: shouting, disorganization, clutter.
- **Clarity beats consistency** when the two conflict.
- **Wayfinding:** every screen answers what site, what page, what sections,
  where am I. Current section visibly marked.
- **Goodwill:** never hide what users want (price, contact), never punish their
  input format, no splash screens or forced tours. Make errors easy to recover.
- **Mobile:** same rules, higher stakes. Touch targets ≥ 44px. Things needed in a
  hurry close at hand.

## Edge cases every variant shows

Realistic data, not just the happy path. Per screen, include the ones that apply:

- long names and long numbers (truncation, wrapping)
- empty state (first use, no results)
- error state (failed load, invalid input)
- many items (pagination or scroll behavior)
- narrow screen (375px)

## Craft baseline

- responsive at 375 / 768 / 1440; no horizontal scroll at any of them
- visible `:focus-visible` styles; real `<button>`/`<a>`; labels on inputs;
  ARIA only where native semantics fall short
- `@media (prefers-color-scheme: dark)` when the project has a dark theme
- `@media (prefers-reduced-motion: reduce)` disables non-essential motion
