# From prototype HTML to production-ready, scalable interactives

This document describes the current **standalone HTML prototypes** in this repository and what is required to evolve them into **production-grade** learning interactives that can **scale across games** and **integrate into a larger application** later.

---

## 1. Current state

### What exists today

| File | Purpose | Implementation shape |
|------|---------|------------------------|
| `vowel-sound-sort.html` | Short/long vowel sorting (two rounds: **a**, then **i**) | Single file: HTML + large `<style>` + `<script>` |
| `letter-adventure.html` | Letter recognition across **multiple visual themes** (ice cream, flower, stars, ladybugs, truck) | Single file: HTML + CSS + JS, including inline SVG generation |

### Strengths of the prototypes

- Clear **pedagogical intent** and complete **user flows** (instructions → play → feedback → celebration).
- **Data-driven rounds** (e.g. `LEVELS` / `THEMES` + letter pool) — good mental model for a real app.
- Thoughtful **UX touches**: toasts, progress, confetti-style effects, wrong-answer scaffolding.

### Limits for production and scale

- **Monolithic files** — hard to review, test, and reuse; merge conflicts explode as you add games.
- **No separation of concerns** — UI, game rules, audio, and asset URLs are intertwined.
- **No type safety** — regressions are easy as content and logic grow.
- **Client-side secrets** — any third-party API key in HTML/JS is **not production-safe** (scraping, abuse, billing).
- **Audio** — browser `speechSynthesis` is free but **inconsistent**; cloud TTS from the browser without a backend is **insecure** and still **latency-sensitive**.
- **External image dependency** (e.g. AI image URLs) — reliability, moderation, and policy need explicit handling for school contexts.
- **Accessibility, i18n, analytics** — not systematically addressed in a single-file prototype.

---

## 2. Target definition: “production level”

For this product line, treat **production** as meeting at least:

1. **Maintainability** — multiple engineers can change games without duplicating entire pages.
2. **Correctness** — game rules and edge cases are testable; refactors don’t silently break pedagogy.
3. **Security** — no secrets in the client; safe embedding (CSP, iframe assumptions documented).
4. **Reliability** — predictable audio and assets; graceful degradation when network or TTS fails.
5. **Accessibility** — keyboard/switch access where applicable, focus management, ARIA for custom controls, sufficient contrast.
6. **Performance** — fast first load on school devices; lazy-load heavy assets per game.
7. **Portability** — a clear boundary so these games can be **dropped into another host app** (shell, auth, router) with minimal rewiring.

---

## 3. Recommended technical direction

### Stack (aligned with “simple but scalable”)

- **Vite + React + TypeScript** — industry default for static deploys and component reuse.
- **CSS**: start with **plain CSS modules** or a single design-token layer; avoid heavy UI frameworks until needed.
- **Optional later**: internal package (`game-kit`) via monorepo **when** a second deployable or shared library justifies it.

### Architecture: layers

| Layer | Responsibility |
|-------|----------------|
| **Design tokens / theme** | Colors, spacing, typography, radii — one source of truth. |
| **Game kit (shared)** | Shell layout, header, progress, buttons, modals, toast, celebration, common motion patterns. |
| **Game modules** | Per-game: level config, rules engine (pure functions where possible), screen composition. |
| **Speech service (pluggable)** | `speak()` / `preload()` / `cancel()` behind an interface; swap Web Speech → server TTS later. |
| **Assets** | Static audio (MP3) in `public/` or CDN; images either static, curated CDN, or generated behind your own API. |

### Data and content

- Move level/theme definitions into **typed modules or JSON** validated at build time (e.g. Zod) so bad content fails in CI, not in the classroom.
- Prefer **content IDs** for speech (“`instruction.round1`”) where phrases are stable, to support **pre-recorded** audio later.

---

## 4. Audio strategy (demo → paid)

### Phase A — Show app / $0

- **`window.speechSynthesis`** behind a `WebSpeechAdapter`.
- Optional: **short pre-recorded MP3s** for hero lines (intro, big win) for demo polish without paid APIs.

### Phase B — Production

- **Server-mediated TTS** (serverless or small API) — keys never ship to the browser; optional **SSML** and caching by text hash.
- **Hybrid**: canned pedagogy as **audio files**; dynamic words via **TTS** only where needed.

All game code should depend only on **`SpeechPort`** (interface), not on a vendor SDK.

---

## 5. Security and compliance checklist

- [ ] Remove **all embedded API keys** from client bundles; rotate any keys that ever appeared in git history.
- [ ] Define **COPPA / student privacy** stance if minors use the app (logging, analytics, voice recording — usually **no recording** for TTS-only).
- [ ] **CSP** and **iframe** embedding rules for the eventual host LMS.
- [ ] **Content safety** for any AI-generated imagery (policy, allowlist hosts, fallbacks).

---

## 6. Accessibility and inclusion

- [ ] Keyboard path for all actions that are currently tap-only (or document “touch-primary” with assistive alternatives).
- [ ] Visible focus rings; logical tab order; `aria-live` for feedback where appropriate.
- [ ] Respect **`prefers-reduced-motion`** for non-essential animations.
- [ ] Test on **iPad + Chromebook** (common school devices).

---

## 7. Quality engineering work

| Area | Actions |
|------|---------|
| **Testing** | Unit tests for pure game logic (sorting rules, win conditions); a few **Playwright** smoke tests per game. |
| **Error boundaries** | React error boundary so one game failure doesn’t white-screen the shell. |
| **Telemetry** | Optional analytics interface (host app may supply implementation). |
| **i18n** | Externalize strings early if non-English is possible. |

---

## 8. Deployment and integration

- **Build output**: static `dist/` suitable for **S3/CloudFront**, **Netlify**, **Vercel**, or static hosting inside the parent app.
- **Embedding**: document expected **viewport**, **postMessage** API if the parent must pass user/session/context.
- **Versioning**: semver or build IDs surfaced in UI for support (“which build is this teacher on?”).

---

## 9. Migration map (prototype → components)

Roughly how the two HTML files decompose:

### `vowel-sound-sort.html`

- **Layout**: `GameShell`, `InstructionBar`, `ProgressStars`, `WordTray`, `SortBin`, `CelebrationModal`.
- **Logic**: level loader, selection state machine, scoring, round transitions.
- **Speech**: instruction + word speak + bin phrasing → `SpeechPort`.
- **Assets**: image strategy abstracted (`ImageSource` / static URLs / placeholder).

### `letter-adventure.html`

- **Themes**: each theme becomes **data + a small renderer** (SVG or components), not one mega-script.
- **Input**: unify **pointer drag** and **tap-to-select** behind a single interaction model for testing.
- **Speech**: intros and praise lines → phrase keys + optional TTS.

---

## 10. Phased roadmap (suggested)

| Phase | Outcome |
|-------|---------|
| **1** | Vite + React + TS repo; design tokens; `GameShell`; `SpeechPort` + Web Speech implementation. |
| **2** | Rebuild **one** game (e.g. vowel sort) using shared components; parity with prototype flow. |
| **3** | Second game (letter adventure); extract duplicated patterns into `game-kit`. |
| **4** | Hardening: a11y pass, E2E smoke tests, production audio path design (even if not enabled). |
| **5** | Host-app integration: auth/context, routing, deployment pipeline, documentation for embedders. |

---

## 11. Summary

The existing HTML files are **excellent prototypes**: they prove UX and pedagogy. To make them **production-ready and scalable**, the work is primarily **engineering structure** (components, TypeScript, pluggable speech, secure APIs), plus **hardening** (a11y, testing, deployment, privacy). The games themselves do not require a game engine—**React + a thin shared game kit + clear boundaries** is the right level of complexity for long-term reuse inside another application.

---

## References in this repo

- `vowel-sound-sort.html` — vowel sorting flows and level data.
- `letter-adventure.html` — themed letter recognition, drag/tap interactions, SVG item generation.

*(When this doc is committed, update paths if files move under `legacy/`.)*
