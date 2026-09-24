# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-page React 18 + TypeScript "treasure hunt" game built with Vite, originally exported from Figma Make. The repo is used as a Claude Code training exercise: per the README, the `initial` branch is the starting point, and other remote branches (`complete`, `plugin-initial`, `plugin-complete`, `loop-goal-initial`, `gh-pages`) hold alternate/finished states of the exercise.

## Commands

- `npm install` — install dependencies
- `npm run dev` — Vite dev server on port 3000 (auto-opens browser)
- `npm run build` — production build to `build/` (not `dist/`)

There is no test runner, linter, or `tsconfig.json` configured; Vite (SWC) transpiles TS without type-checking.

## Architecture

- **All game logic lives in [src/App.tsx](src/App.tsx)**: a `boxes` state array of 3 `{id, isOpen, hasTreasure}` items, one randomly holding treasure. Opening a box scores +100 (treasure) / −50 (skeleton); the game ends when the treasure is found or all boxes are open. Animations use `motion/react` (Framer Motion).
- **Vite aliases ([vite.config.ts](vite.config.ts)) are load-bearing**:
  - `figma:asset/<name>.png` imports resolve to files in `src/assets/`. Adding a new `figma:asset/...` import requires adding a matching alias.
  - Version-suffixed imports (e.g. `@radix-ui/react-dialog@1.1.6`) used by the Figma-generated shadcn components are aliased to the plain package names.
  - `@` → `src/`.
- **Styling — important gotcha**: [src/index.css](src/index.css) is a *precompiled* Tailwind v4 stylesheet; Tailwind itself is not a dependency and there is no PostCSS/Tailwind build step. Only utility classes already present in `index.css` will render. Using a new Tailwind class requires either adding the CSS rule manually to `index.css` or setting up Tailwind. [src/styles/globals.css](src/styles/globals.css) holds the shadcn theme tokens (CSS variables) but is not imported by `main.tsx`.
- **[src/components/ui/](src/components/ui/)** is the stock shadcn/ui library (Radix + `cva` + `cn()` from `utils.ts`); only `Button` is currently used. `components/figma/ImageWithFallback.tsx` is a Figma Make helper.
- [src/guidelines/Guidelines.md](src/guidelines/Guidelines.md) is an empty Figma Make template for design rules — check it in case the user has filled it in.
