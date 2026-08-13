# `forge-ai-artifact-card` — proposal

> **This file is not intended to merge.** It rides along on the `feat/artifact-card`
> branch so a reviewer gets the component and the reasoning behind it from one clone.
> Before any PR it should be dropped, and its content moved into the PR description.
>
> Written by the Socrata / Data & Insights team, who need this component and are
> offering it upstream. Everything below is a proposal — Forge owns the public surface
> once it ships, and every decision here is yours to overrule.

A card representing an artifact an agent produced. Renders a leading icon, two lines
of truncating text, and a trailing action affordance, all inside a single button that
covers the whole card. Clicking it emits one event; the consumer decides what opens.

Presentational only — no data fetching, no agent or tool-call awareness, no knowledge
of what it opens.

**To see it:** `pnpm i && pnpm --filter @tylertech/forge-ai storybook`, then
**AI Components → Primitives → Artifact Card**. Six stories, plus an MDX docs page.

## Identity

|           |                                         |
| --------- | --------------------------------------- |
| Package   | `@tylertech/forge-ai` (`packages/ai`)   |
| Directory | `packages/ai/src/lib/ai-artifact-card/` |
| Tag       | `forge-ai-artifact-card`                |
| Class     | `AiArtifactCardComponent`               |
| Tag const | `AiArtifactCardComponentTagName`        |

## API

### Properties

| Property       | Attribute       | Type      | Default |
| -------------- | --------------- | --------- | ------- |
| `titleText`    | `title-text`    | `string`  | `''`    |
| `subtitleText` | `subtitle-text` | `string`  | `''`    |
| `assetId`      | `asset-id`      | `string`  | `''`    |
| `active`       | `active`        | `boolean` | `false` |
| `disabled`     | `disabled`      | `boolean` | `false` |

`titleText` rather than `title`, which is a global HTML attribute and would render a
native tooltip. `forge-ai-chatbot` sets this precedent.

`active` and `disabled` both reflect — the SCSS needs `:host([active])` and
`:host([disabled])`. Both default to `false` and are set by presence:
`<forge-ai-artifact-card active>`.

`assetId` is an opaque identifier the component never interprets. It exists to be
echoed back in the event detail, so consumers rendering a stream of cards can attach
one delegated listener instead of sniffing `event.target`.

### Slots

| Slot     | Purpose                                                  |
| -------- | -------------------------------------------------------- |
| `icon`   | Leading icon. Consumer supplies it.                      |
| `action` | Trailing affordance — label text, plus an optional icon. |

Slots, not properties, so no icon set or label vocabulary is baked into the library.
The consumer swaps `action` content itself when `active` changes; there is no
separate active-state slot.

### Events

| Event                         | Detail                | Options                           |
| ----------------------------- | --------------------- | --------------------------------- |
| `forge-ai-artifact-card-open` | `{ assetId: string }` | `bubbles: true`, `composed: true` |

`composed` is required or the event cannot cross the shadow boundary and no consumer
ever sees it. Fires on click and — because the internal root is a native `<button>` —
on Enter and Space. Does not fire when `disabled`.

Export a `ForgeAiArtifactCardOpenEventData` interface for the detail type and register
both the tag and the event in a `declare global` block.

### CSS custom properties

| Property                                 | Default                |
| ---------------------------------------- | ---------------------- |
| `--forge-ai-artifact-card-accent-color`  | theme `primary`        |
| `--forge-ai-artifact-card-background`    | theme `surface`        |
| `--forge-ai-artifact-card-border-radius` | shape `large` (8px)    |
| `--forge-ai-artifact-card-padding`       | spacing `small` (12px) |
| `--forge-ai-artifact-card-gap`           | spacing `small` (12px) |

One accent property, not two — the icon color and the action-label color are always
the same value.

`:host { display: block }`. The card fills its container. Note this differs from
`ai-attachment`, which is `inline-block` because attachments sit in a row.

## Styling

| Element               | Value                                                                           |
| --------------------- | ------------------------------------------------------------------------------- |
| Card radius           | `forge-shape.variable(large)` — 8px                                             |
| Card padding          | `forge-spacing.variable(small)` — 12px, all sides                               |
| Icon/text/action gap  | `forge-spacing.variable(small)` — 12px                                          |
| Action label→icon gap | `forge-spacing.variable(xxsmall)` — 4px                                         |
| Icon slot wrapper     | 24px square, `flex-shrink: 0`                                                   |
| Title                 | `@include forge-typography.style(body1)` (14px) + `font-weight: 500`            |
| Subtitle              | `@include forge-typography.style(label1)` (12px), `theme.variable(text-medium)` |
| Action label          | `@include forge-typography.style(label2)` (13px)                                |
| Truncation            | `@include forge-typography.ellipse` on title and subtitle                       |
| Title color           | `theme.variable(text-high)`                                                     |

Three notes on how this is built:

1. **The button _is_ the card surface** — one element, not a card wrapping a button.
   `:host` is a bare `display: block` and `.artifact-card` is the `<button>`, carrying
   the border, radius, padding, and background itself. This is why there is no
   corner-clipping problem: with a single element there is no inner background to paint
   square corners over a rounded parent.
2. **The title weight needs a literal.** `forge-typography.style(body1)` emits its own
   `font-weight`, so 500 must be overridden after the mixin. The `medium` weight
   function exists only at `sass/core/styles/tokens/typography/weight`, which the
   typography module does not re-forward. `ai-attachment.scss` overrides a font size
   the same way.
3. **Build the card surface from tokens.** Do not wrap `forge-card` — `forge-ai`
   components may not import from `@tylertech/forge` in TypeScript.

## States

| State          | Treatment                                                                                                   |
| -------------- | ----------------------------------------------------------------------------------------------------------- |
| Hover          | `@include forge-elevation.box-shadow(2)`, 1px lift, button fill `theme.variable(surface-container-minimum)` |
| Focus-visible  | 2px `accent-color` outline, `outline-offset: -2px`                                                          |
| Active         | `inset 0 0 0 1px` in `accent-color`; composes with the hover shadow                                         |
| Disabled       | Native `disabled` on the button, `opacity: 0.5`, `cursor: default`, no event, no hover treatment            |
| Reduced motion | No hover lift; the hover shadow still transitions                                                           |

**No mount animation.** The component must not animate on entry. A consumer that
wants one can animate the card from outside; a consumer that doesn't cannot remove it
from inside.

## Accessibility

The whole card is one `<button>`, so its accessible name comes from its text content.
The icon is decorative and `aria-hidden`.

**The `action` slot's text label is the only non-color type signal**, so it must
contain real text and never an icon alone — that text is what distinguishes one card
from another for screen reader and colorblind users. Document this on the slot and
assert it in tests.

Guidance to document for consumers: an active-state label that drops the type
("Viewing") loses that distinction exactly when users are comparing cards. Prefer
keeping the type in it.

## Conventions

- `@customElement(AiArtifactCardComponentTagName)`, `public static override styles = unsafeCSS(styles)`
- JSDoc header with `@tag`, `@summary`, `@description`, `@event`, `@cssproperty` — the manifest and docs generate from it
- `#` for private members; explicit `public`/`private`; return types on everything
- `readonly` for static templates, `get #x()` for anything that must re-render — a `readonly` template holding dynamic content silently stops updating
- Inline `<svg>` only; no `<forge-icon>` in the template; no `@tylertech/forge`, `forge-core`, or `tyler-icons` imports in TS
- SCSS: `@use '@tylertech/forge/sass/core/styles/{spacing,theme,typography,shape,elevation}'`, referenced as `#{forge-spacing.variable(small)}`
- No inline styles; logical properties for margins (`margin-inline-start`)
- A Storybook control for every property and slot; MDX docs updated alongside

## What shipped on this branch

```
packages/ai/src/lib/ai-artifact-card/
  ai-artifact-card.ts
  ai-artifact-card.scss
  ai-artifact-card.test.ts
  index.ts
packages/ai/src/stories/components/primitives/ai-artifact-card/
  AiArtifactCard.stories.ts
  AiArtifactCard.mdx
packages/ai/src/lib/index.ts        (export added)
```

Written by hand rather than via `pnpm plop:ai component`, because the generator is
interactive and the authoring session could not answer its prompts. The output follows
the templates in `packages/ai/templates` and the conventions in the surrounding
components; the `src/lib/index.ts` export was added manually, in alphabetical position.

Not yet done, deliberately: `pnpm generate-proxies` for the React and Angular wrappers,
and `pnpm changeset`. Both are cheap, and both are worth deferring until the API below
is agreed — the proxies are generated from the component's public surface, so they would
only need regenerating.

**Verified before commit,** against `@tylertech/forge` 3.15.2:

| Check                          | Result                                                              |
| ------------------------------ | ------------------------------------------------------------------- |
| `wtr --group ai-artifact-card` | 9 passed, 0 failed                                                  |
| `lit-analyzer`                 | 0 problems in 216 files                                             |
| ESLint / Stylelint / Prettier  | clean                                                               |
| Rendered tokens                | radius 8px, padding 12px, gap 12px, title 14px/500, subtitle 12px   |
| Event                          | fires once, `assetId` in detail, `composed` and `bubbles` true      |
| `active` / `disabled`          | ring computes to `1px inset` primary; disabled suppresses the event |

## Tests

`@open-wc/testing` with Chai under Web Test Runner. Run a single component's tests with
`pnpm exec wtr --group ai-artifact-card` — groups are derived automatically from the
directory names under `src/lib`.

**What CI actually gates.** The `Test` step in `ci.yml` is commented out, so no tests
run in CI at all. The real gates are `pnpm run format:check` and `pnpm run build`,
which runs lint (lit-analyzer, ESLint, Stylelint) as its first step. Prettier
formatting is therefore the easiest way to fail CI and the easiest to fix.

`web-test-runner.config.mjs` does declare coverage thresholds — statements 98.5,
branches 95.5, functions 96.5, lines 98.5 — but nothing enforces them: `test:ci` is
`wtr --group lib` with no `--coverage` flag, and CI does not invoke it. Treat the
thresholds as aspirational.

Worth knowing before proposing: `packages/ai` currently contains **no component
tests** — this is the only test file in the package. That is offered as a benefit, not
a reproach: it also meant there was no existing test to copy patterns from, so the shape
of this one is itself up for review.

The nine tests cover:

- Clicking the icon, the text, or empty card space each fires `forge-ai-artifact-card-open` exactly once
- The detail carries `assetId`, and the event crosses a shadow boundary
- Enter and Space fire the event; `disabled` suppresses all three paths
- `active` renders the ring; absent, it does not
- Long `titleText` and long `subtitleText` each truncate to one line
- `focus-visible` shows the outline on keyboard focus, not on mouse click
- The hover fill and the active ring stay inside the radius at every corner
- No animation on mount under any motion setting
- Under `prefers-reduced-motion: reduce` there is no hover lift, but the hover shadow still transitions
- Slotted `icon` and `action` content renders, and the accessible name includes the action label

## Open to revision

Everything above is a concrete proposal rather than a fixed contract, and it is written
that way on purpose: a reviewer should be able to disagree with a decision without
having to reverse-engineer why it was made.

The three most likely to be wrong, in order:

1. **The tag name.** `forge-ai-artifact-card` sits beside the existing
   `forge-ai-artifact`, and a compound name in this library sometimes reads as a
   sub-part of its prefix. This is a sibling that _opens_ an artifact, not a piece of
   one, and it renders with no `forge-ai-artifact` on the page at all.
2. **`assetId` as a property.** It exists so a consumer with one delegated listener over
   a stream of cards does not have to sniff `event.target`. If that does not justify a
   property, it is the first thing to drop.
3. **Text as properties rather than slots.** `title-text` and `subtitle-text` are the
   simplest thing that truncates, but slots would give consumers richer content and
   easier i18n.

This component originates in an internal Socrata prototype, where it currently renders
query and report results inside an agent chat transcript. That prototype is not public;
happy to walk through it, or share the original React implementation, on request.
