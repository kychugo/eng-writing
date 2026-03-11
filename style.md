# Style Guide — The Writer's Desk

A reference for all design and style decisions used across this project.  
Apply these conventions consistently to any new page or component.

---

## Colour Palette

| Token           | Value                        | Usage                              |
|-----------------|------------------------------|------------------------------------|
| `--ink`         | `#2c2416`                    | Primary text, headings             |
| `--ink-light`   | `#5a4e3a`                    | Secondary text                     |
| `--ink-faint`   | `#8c7f6a`                    | Muted labels, hints, placeholders  |
| `--parchment`   | `#f5f0e8`                    | Page background                    |
| `--parchment2`  | `#ede7d9`                    | Card background hover, tabs        |
| `--parchment3`  | `#e0d9cc`                    | Borders, dividers                  |
| `--gold`        | `#b8965a`                    | Accent, primary action, icons      |
| `--gold-light`  | `#d4b07a`                    | Gold hover state                   |
| `--rust`        | `#8b4a2f`                    | Danger actions, quiz errors        |
| `--moss`        | `#5a6b4a`                    | Success states                     |
| `--shadow`      | `rgba(44, 36, 22, 0.12)`     | Card drop shadow                   |
| `--shadow2`     | `rgba(44, 36, 22, 0.06)`     | Subtle shadow                      |

---

## Typography

- **Body font**: `Lato` (300, 400 weight), `sans-serif` fallback
- **Display / headings**: `Cormorant Garamond` (300, 400, 500, 600; italic variants)
- **Decorative**: `IM Fell English` (italic) — used sparingly for ornamental spans

### Type Scale

| Element           | Font                   | Size                        | Weight |
|-------------------|------------------------|-----------------------------|--------|
| Page title (h1)   | Cormorant Garamond     | `clamp(2rem, 5vw, 3rem)`    | 300    |
| Card title        | Cormorant Garamond     | `1.15rem`                   | 600    |
| Body text         | Lato                   | `0.9rem`                    | 300    |
| Labels / caps     | Cormorant Garamond     | `0.78rem`, uppercase        | 600    |
| Muted / hints     | Lato                   | `0.78–0.82rem`              | 300    |

---

## Icons

**Rule: No emoji anywhere in the UI. Use inline SVG only.**

- All icons are inline SVG (`viewBox="0 0 24 24"`) with `fill="none"`, `stroke="currentColor"`, `stroke-width="1.5"`, `stroke-linecap="round"`, `stroke-linejoin="round"`.
- Always add `aria-hidden="true"` to decorative SVGs.
- Sizing:
  - Tab icons: `14×14`
  - Button icons: `14–16×14–16`
  - Card header icons: `20×20`
  - Empty-state icons: `32–36×32–36` (with `opacity: 0.4`)
  - Score/result icons: `36×36`

### Common SVG patterns used

```html
<!-- Lightbulb (Generator) -->
<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor"
     stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
  <path d="M9 18h6M10 22h4M12 2a7 7 0 017 7c0 2.38-1.19 4.47-3 5.74V17H8v-2.26C6.19 13.47 5 11.38 5 9a7 7 0 017-7z"/>
</svg>

<!-- Pencil (Quiz) -->
<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor"
     stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
  <path d="M12 20h9M16.5 3.5a2.121 2.121 0 013 3L7 19l-4 1 1-4L16.5 3.5z"/>
</svg>

<!-- Card / Flipcard -->
<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor"
     stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
  <rect x="2" y="5" width="20" height="14" rx="2"/>
  <line x1="2" y1="10" x2="22" y2="10"/>
</svg>

<!-- Clock (History) -->
<svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor"
     stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
  <circle cx="12" cy="12" r="10"/><polyline points="12 6 12 12 16 14"/>
</svg>
```

---

## Layout

- **Max content width**: `1100px`, centered with `margin: 0 auto`
- **Container padding**: `2rem 1.5rem 4rem` (desktop), `1.5rem 1rem 3rem` (mobile)
- **Page background**: parchment grain texture via `body::before` SVG noise filter
- **Decorative ink splashes**: two fixed `radial-gradient` blobs at top-left and bottom-right corners, `opacity: 0.025`, `pointer-events: none`

---

## Header

- Sticky, blurred glass effect: `backdrop-filter: blur(8px)`
- Thin bottom border: `1px solid var(--parchment3)`
- Left: nothing (no back button on sub-pages; navigation happens within the SPA)
- Center: page/section title in `IM Fell English` or Cormorant Garamond, `letter-spacing: 0.18em`
- Right: contextual action (e.g., History button)
- **No "← Back to …" button** on sub-pages. Navigation handled via SPA or browser.

---

## Cards

```css
.card {
  background: #faf7f2;
  border: 1px solid var(--parchment3);
  border-radius: 10px;   /* --r-lg */
  padding: 1.75rem;
  box-shadow: 0 4px 28px var(--shadow), 0 1px 4px var(--shadow2);
}
/* Decorative gold top-bar */
.card::before {
  content: '';
  position: absolute; top: 0; left: 0; right: 0; height: 3px;
  background: linear-gradient(90deg, var(--gold), var(--gold-light), var(--gold));
}
```

Card headers use a left-aligned icon (`40×40px` rounded square) + title + optional hint.

---

## Buttons

| Class        | Background       | Text           | Use                        |
|--------------|-----------------|----------------|----------------------------|
| `.btn-primary` | `var(--ink)`   | `var(--parchment)` | Primary actions          |
| `.btn-yellow`  | `var(--gold)`  | `var(--parchment)` | Generator actions        |
| `.btn-red`     | `var(--rust)`  | `var(--parchment)` | Quiz start, try again    |
| `.btn-ghost`   | `var(--parchment2)` | `var(--ink-light)` | Secondary/cancel     |

- Font: `Cormorant Garamond`, `1rem`, `font-weight: 500`, `letter-spacing: 0.12em`
- Shape: `border-radius: 4px` (`--r`)
- Padding: `0.65rem 1.25rem`
- Hover: lift with `translateY(-1px)` + deeper shadow
- Always include an SVG icon before the button label when applicable.

---

## Forms

- Labels: `Cormorant Garamond`, `0.78rem`, uppercase, `letter-spacing: 0.14em`, `color: var(--ink-faint)`
- Inputs: `Lato`, `0.9rem`, `font-weight: 300`, parchment background
- Focus ring: `border-color: var(--gold)` + `box-shadow: 0 0 0 3px rgba(184,150,90,0.14)`
- Multi-column layout uses CSS Grid; collapses to single column on mobile (`< 640px`)

---

## Tab Bar

- Max width `480px`, centered
- Background: `#faf7f2`, bordered, rounded (`10px`)
- Active tab: `background: var(--ink)`, `color: var(--parchment)`
- Tab labels: short text (1 word preferred), preceded by a small SVG icon

---

## Vocabulary Cards

- Parchment background with gold left accent border (`3px solid var(--gold)`)
- Word: `Cormorant Garamond`, `1.2rem`, bold
- Part of speech: italic, muted, small
- Definition: `Lato`, `0.875rem`
- Example sentence: italic, left-border accent
- Chinese translation: displayed as a small inline badge — **no flag emoji**, just the text
- Add-to-Flipcard button: small, pill-style, muted. SVG `+` icon prefix

---

## Quiz

- Progress bar at top (gold gradient)
- One question visible at a time
- After answering, **the correct answer is highlighted** and an explanation appears
- A **"Next →" button** (with circle-arrow SVG) must be clicked to advance — no auto-advance
- Score card at the end uses SVG icons (trophy, thumbs-up, book) sized `36×36`

---

## Flipcard (SRS)

- Tab name: **Flipcard** (not "SRS Review")
- Card word displayed large; definition blurred until tapped
- Rating buttons text-only: `Forgot` / `Hard` / `OK` / `Easy` — no emoji faces
- "Add to Flipcard" CTA in vocabulary cards (replaces "Add to SRS")

---

## History Panel

- Slides in from the right as a drawer (`width: min(500px, 100vw)`)
- History entries: each row is **directly clickable** to restore the session immediately
- No expand/collapse preview — clicking an entry loads it directly
- Delete button: small, rust-coloured, SVG `×` icon
- No emoji anywhere in the panel — use SVG icons

---

## Responsiveness

- Breakpoints: `640px` (tablets), `900px` (desktop)
- On mobile (`< 640px`):
  - Container padding reduces
  - All action rows stack vertically
  - Vocab grid becomes single column
  - Matching quiz grid becomes single column
  - Tab bar text may truncate — keep labels short (1 word)
- Font sizes use `clamp()` for fluid scaling

---

## Animations & Motion

- Standard transition: `0.3s ease` (`--t`)
- Slide panel: `0.55s cubic-bezier(0.22, 1, 0.36, 1)` (`--t-slide`)
- Card hover: `translateY(-2px)` lift
- `@media (prefers-reduced-motion: reduce)` disables all transitions/animations

---

## Accessibility

- All interactive elements have `:focus-visible` outline using `--gold`
- All SVG icons are `aria-hidden="true"` (decorative)
- Tab roles use `role="tab"`, `aria-selected`, `aria-controls`
- Buttons have descriptive `aria-label` when label alone is insufficient
- Dialog/panel uses `role="dialog"`, `aria-modal="true"`, `aria-label`

---

## Don'ts

- **No emoji** anywhere in the UI (not in buttons, labels, cards, icons, or JS-rendered HTML)
- **No "← Back to …" navigation** in sub-pages
- **No auto-advancing** after quiz answers — always require a user action
- **No flag emoji** before translated text (e.g., no 🇨🇳 before Chinese)
- **No unnecessary descriptive text** — keep UI minimal and clean
- Do not show preview panels before loading — load directly on click
