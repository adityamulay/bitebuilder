# BiteBuilder — Architecture

## Overview

BiteBuilder is a personalized meal-suggestion app. A user picks what's in their kitchen and sets a few preferences (diet, spice level, healthiness, cuisine, age group, meal type), and the app returns a ranked list of real recipes that fit, plus a few "just a few ingredients away" grocery-run ideas. It runs entirely in the browser — there is no backend server, no database, and no API key to manage.

## Why a single static file

The whole app lives in one file, `index.html`. There's no build step (no Vite, no webpack, no npm install) — the page loads React, ReactDOM, and a tiny JSX-like templating library (`htm`) directly from a CDN (`esm.sh`) as native ES modules, and everything else is plain JavaScript. This was a deliberate choice for a few reasons:

- It keeps hosting free and zero-maintenance — a static file can be served by GitHub Pages with no server to patch, scale, or pay for.
- It avoids any build pipeline that could break or need updating over time.
- It makes the whole app easy to read top to bottom in one place.

The tradeoff is that the file is long (~1,200 lines), but it's organized into clearly labeled sections (see "Code layout" below).

## Code layout

`index.html` is structured top to bottom as:

1. **`<head>`** — fonts, Tailwind CDN (used for a handful of utility classes), the PWA manifest link and Apple home-screen meta tags, and a `<style>` block with the design tokens that can't be done with inline styles (scrollbars, the range-slider thumb, the custom select chevron, hover states for the primary button).
2. **Constants** — the preset ingredient list, cuisine options, age groups, meal types.
3. **Recipe data layer** — everything related to fetching and interpreting recipes (see "Data sources" below).
4. **Recommendation engine** — `buildCandidatePool`, `scoreMeal`, `generateSuggestions` (see "How suggestions are generated" below).
5. **Shared style tokens** — the `C` (colors) and `F` (font sizes) objects every component pulls from, so the look stays consistent.
6. **UI components** — small reusable pieces (`Pill`, `Tag`, `Slider`) followed by the larger ones (`FilterPanel`, `IngredientPicker`, `RecipeCard`, `GroceryCard`, `RecipeModal`), and finally `App`, which holds all top-level state and wires everything together.

## Data sources

Two free, keyless sources feed the recipe pool:

- **TheMealDB** (`themealdb.com`) — a free public recipe API. It's queried by cuisine, by meal-type category, by ingredient, and by a few random letters, and the results are merged into a candidate pool. TheMealDB has broad international coverage but thin Indian-specific listings.
- **A curated Indian recipe set, built into the app** (`INDIAN_RECIPES`) — 18 hand-written vegetarian/vegan Indian dishes (dosa, chana masala, paneer dishes, biryani, etc.), each in the same shape as a TheMealDB recipe so they merge into the same scoring pipeline with no special-casing. These exist because there's no reliable free, keyless Indian-specific recipe API — building them in directly avoids that dependency entirely and guarantees they always load instantly.

Both sources are merged into one in-memory pool before scoring runs, so a single ranked list can mix dishes from either source.

## Diet model

The app is vegetarian/vegan only by design. `classifyDiet` looks at an ingredient list and returns `Non-Vegetarian` if it contains any meat, fish, or egg keyword (`EXCLUDE_WORDS`), `Vegetarian` if it contains a dairy keyword (`DAIRY_WORDS`: milk, butter, cheese, cream, yogurt, ghee, paneer, honey), or `Vegan` otherwise. `generateSuggestions` always filters out anything classified `Non-Vegetarian` before ranking, regardless of the user's selected filter, and additionally restricts to `Vegan` only if that's what the user picked.

## How suggestions are generated

There is no AI model involved — an earlier version called the Claude API for this, but it was replaced with a transparent local scoring formula so the app needs no API key and no per-request cost:

1. **`buildCandidatePool`** gathers a pool of candidate recipes from TheMealDB (filtered loosely by cuisine, meal type, and up to three selected ingredients, plus a few random letters for variety) and always merges in the curated Indian set.
2. **`scoreMeal`** scores each candidate: ingredient overlap with what the user has on hand (up to 40 points), a cuisine match bonus, a meal-type match bonus, closeness to the requested spice level and healthiness, and a personalization bonus/penalty based on the user's own like/dislike history for that dish name.
3. **`generateSuggestions`** hard-filters by diet, sorts by score, takes the top 5 as the main recipe suggestions, and separately picks a few lower-ranked but "almost there" recipes (missing 1–4 ingredients) as the grocery-run suggestions.

## Persistence

Everything is stored in `localStorage`, scoped to the browser/device it runs on — there's no account system and no shared backend:

- `bb_ing` — selected ingredients
- `bb_filters` — diet/spice/health/cuisine/age/meal-type preferences
- `bb_bookmarks` — saved recipes
- `bb_feedback` — per-recipe like/dislike
- `bb_history` — a rolling log of past sessions' likes/dislikes, used for personalization

Because this is per-device, each family member who installs the app gets their own private bookmarks and history — nothing syncs between phones, which keeps the architecture simple and avoids needing any backend or login.

## Hosting and distribution

The app is hosted as a static site on **GitHub Pages**, served directly from the `main` branch of this repo at `https://adityamulay.github.io/bitebuilder/`. There's no server to run or pay for — GitHub serves the static files from its own CDN.

For a native-app-like feel on iPhone without going through the App Store, the app includes a `manifest.json` and Apple home-screen meta tags (`apple-mobile-web-app-*`, `apple-touch-icon`). Opening the URL in Safari and using Share → Add to Home Screen installs an icon that launches the app full-screen, with no browser chrome.

## Known limitations

- No cross-device sync — bookmarks/history are local to whichever phone/browser they were created on.
- Recipe images for the curated Indian dishes link to Wikimedia Commons by guessed filename; if a filename doesn't exist, the app gracefully falls back to a plate emoji rather than showing a broken image.
- TheMealDB occasionally rate-limits or is briefly unreachable; the app falls back to its own random-recipe calls and the always-available curated Indian set so a useful result still appears.
