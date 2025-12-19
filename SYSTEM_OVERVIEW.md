# Stellaris System Overview

This document explains how the major files, functions, and data in Stellaris fit together, highlights the available features, and shows how the Inertia + React frontend is wired.

## Architecture at a Glance
- **Backend:** Laravel routes -> controllers -> services/models -> Inertia responses.
- **Frontend:** Inertia bootstraps React pages (`resources/js/Pages`) that compose UI components and call backend routes via Inertia helpers.
- **Data:** PostgreSQL tables for users, discovered stars, and per-user star notes.

## Core Data Model
- **users:** Standard Laravel auth users.
- **discovered_stars:** Saved stars tied to a `user_id` so each user keeps a personal list.
- **star_notes:** Per-user notes/favorites keyed by `star_name`; exposed via `User::notes()` relationship.

## Request Flow & Backend Responsibilities
- **Public landing** (`/` → `routes/web.php`): Renders the `Welcome` Inertia page with warp intro.
- **Main console** (`/starmap` → `StarController@index`):
  - Loads the signed-in user, their `DiscoveredStar` names (fallback: `['Sirius', 'Vega']`), and any `StarNote` records.
  - Returns `StellarisHome` with props: `sidebarList`, `userNotes`, `auth`, and `searchedStar` (null initially).
- **Search** (`/star/search` → `StarController@search`):
  - Looks for the star in the current user’s `discovered_stars`.
  - If missing, calls `StarApiService::fetchStars()` (API Ninjas) and saves the first result to `discovered_stars` with `user_id`.
  - Builds a `searchedStar` payload (adds color/visual size) and returns `StellarisHome` with updated sidebar, notes, and `isNewDiscovery` flag.
- **Captain’s log save** (`POST /chapter` → `StarController@store`): Validates `star_name`/`story_chapter`, then `updateOrCreate` in `star_notes` with `is_favorite=true`.
- **Delete log** (`DELETE /chapter/{star_name}` → `StarController@destroy`): Removes the user’s note for that star.
- **Observatory** (`/observatory` → `StarController@observatory`): Pulls all `star_notes` (favorites) ordered by `updated_at` and renders the `Observatory` page.
- **Profile** routes (Breeze defaults) allow editing/updating/deleting the user profile.
- **External service** (`app/Services/StarApiService.php`): Wraps HTTP calls to API Ninjas, handles params, timeouts, and logs failures.

## Frontend (Inertia + React) Map
- **Bootstrap** (`resources/js/app.jsx`): Sets up Inertia, resolves pages, and renders the root React tree. `resources/js/bootstrap.js` configures Axios defaults.
- **Layout** (`Layouts/CosmicLayout.jsx`): Provides the global scanline HUD look, custom cursor, and page transition wrapper used by all pages.
- **Pages**
  - **`Pages/Welcome.jsx`**: Warp-drive intro. Uses `@react-three/fiber` stars field and on “Engage Warp Drive” navigates to the login route via `router.visit`.
  - **`Pages/StellarisHome.jsx`**: Main console. Key behaviors:
    - `useForm({ query })` ties the search box to `GET /star/search`.
    - Shows sidebar (`sidebarList`) of discovered stars; clicking triggers a search for that star.
    - Tracks selected star state and opens `StarDetails` when a star is returned.
    - Displays captain’s greeting modal per user (sessionStorage flag) and commander profile modal.
    - Surfaces validation errors (`errors.search`) and the `isNewDiscovery` notice from the server.
  - **`Pages/Observatory.jsx`**: Lists saved favorites (`favorites` prop). “Purge” calls `DELETE /chapter/{star_name}` via Inertia `useForm().delete`. Link back to `/starmap`.
- **Components**
  - **`Components/SpaceScene.jsx`**: 3D backdrop. Renders galaxy particles, ambient stars, and—when provided—`FoundStar` centered with glow and click-to-select. Uses `@react-three/fiber`, `drei` helpers, and orbit controls.
  - **`Components/StarDetails.jsx`**: Slide-in details panel for the selected star. Shows basic stats, and contains the Captain’s Log form bound to `POST /chapter`; delete uses `DELETE /chapter/{star_name}`. Reactively updates form state when the selected star or note changes.
  - **Shared UI** (`Components/*`): Breeze-derived inputs/buttons plus nav links used across pages.

## Feature Checklist
- Warp-drive welcome screen that routes to authentication.
- Authenticated starmap console with 3D galaxy visualization.
- Personal “Pokedex” of discovered stars, scoped per user.
- Star search against local cache, falling back to API Ninjas; new discoveries auto-saved and flagged.
- Captain’s Log CRUD for each star (favorites list).
- Observatory archive for reviewing/purging saved logs.
- Commander profile modal and logout controls.
- Error handling for missing stars and visual notification for new discoveries.

## How Pieces Interact (Example Search Flow)
1. User enters a star name in `StellarisHome` search box (`useForm.get(route('star.search'))`).
2. `StarController@search` checks `discovered_stars` for the user, or calls `StarApiService` to fetch and persist the star.
3. Controller rebuilds the sidebar list and notes, then returns `StellarisHome` with `searchedStar` and `isNewDiscovery`.
4. `StellarisHome` receives props, sets `selectedStar`, and passes it to `SpaceScene` and `StarDetails`.
5. `SpaceScene` centers the star model; clicking it opens `StarDetails`.
6. `StarDetails` uses Inertia forms to save or delete notes, which hit `/chapter` routes and refresh props on success.

Use this map as a quick reference when navigating or extending the codebase.
