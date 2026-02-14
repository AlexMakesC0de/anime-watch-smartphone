# Mobile App — Build Guide

This document describes how to build the **mobile version** of the AnimeWatch desktop app. It covers the architecture, UI/UX behavior, hard problems we solved in the desktop version, and pitfalls to avoid. The desktop source lives in this repo — reference it as the single source of truth for behavior.

---

## 1. Recommended Tech Stack

| Layer | Recommendation | Reason |
|---|---|---|
| Framework | React Native (Expo) or Flutter | Cross-platform (iOS + Android) |
| Video player | `react-native-video` or `flutter_video_player` with HLS support | Must handle HLS `.m3u8` streams natively |
| Database | SQLite (`expo-sqlite` / `sqflite`) | Same schema as desktop — easy data parity |
| HTTP | Built-in `fetch` or `dio` (Flutter) | For AniList GraphQL + provider scraping |
| Navigation | React Navigation (stack + bottom tabs) or `go_router` | See navigation structure below |

---

## 2. Architecture Overview

The desktop app has three processes (main/preload/renderer) because of Electron. On mobile this collapses to **two layers**:

```
┌──────────────────────────────────────────────────┐
│  UI Layer (React Native / Flutter)               │
│  - Screens, navigation, video player             │
│  - AniList GraphQL client (direct fetch)         │
│  - In-memory cache with TTL                      │
└──────────────────┬───────────────────────────────┘
                   │
┌──────────────────▼───────────────────────────────┐
│  Service Layer (runs in-process)                 │
│  - SQLite database (same schema as desktop)      │
│  - Streaming provider (gogoanime scraper)        │
│  - HLS proxy (if needed — see §6)               │
└──────────────────────────────────────────────────┘
```

The AniList API is called directly from the UI layer (it's a public GraphQL API, no auth needed). The streaming provider and database are local services.

---

## 3. Navigation & Screen Structure

### Bottom Tab Bar (always visible except on Watch screen)
| Tab | Icon | Screen |
|---|---|---|
| Home | `Home` | HomePage |
| Discover | `Search` | SearchPage |
| Library | `Library` | LibraryPage |

### Stack screens (pushed on top of tabs)
- **AnimeDetailScreen** — pushed when tapping any anime card
- **WatchScreen** — fullscreen, hides tab bar and status bar

### Routes (mirroring desktop)
| Desktop route | Mobile equivalent |
|---|---|
| `/` | Home tab |
| `/search` | Discover tab |
| `/search?q=...` | Discover tab with pre-filled query |
| `/library` | Library tab (all) |
| `/library/:status` | Library tab with filter chip selected |
| `/anime/:id` | AnimeDetailScreen (push) |
| `/watch/:id/:episode` | WatchScreen (fullscreen push) |

---

## 4. UI/UX Specification

### 4.1 Color Scheme & Theming

**Dark mode only.** The app has no light theme.

```
Background levels:
  dark-950: #111214  (deepest background — screen bg)
  dark-900: #1a1b1e  (cards, list items, sidebar)
  dark-800: #2a2b2f  (hover states, input fields, skeleton placeholders)
  dark-700: #3a3b3f  (borders, dividers)
  dark-500: #6b6c70  (secondary text, placeholders)
  dark-400: #8b8c90  (muted text)
  dark-300: #a0a1a5  (body text)
  dark-200: #c0c1c5  (emphasized text)

Accent:
  accent: #6c5ce7    (primary purple — buttons, active states, progress bars)
  accent-hover: #7d6ef0

Status colors:
  Watching:      blue-400    (#60a5fa)
  Plan to Watch: yellow-400  (#facc15)
  Completed:     green-400   (#4ade80)
  On Hold:       orange-400  (#fb923c)
  Dropped:       red-400     (#f87171)

Score:           yellow-400  (#facc15) — star icon filled
```

Font: **Inter** (or system font as fallback).

### 4.2 Home Screen

Three scrollable sections stacked vertically, each with a horizontal or grid layout:

1. **Continue Watching** (only if items exist)
   - Horizontal scroll of cards
   - Each card: cover thumbnail (left, 64×80), title (truncated, 1 line), "Episode X · M:SS / M:SS", accent-colored progress bar
   - Tapping resumes directly into `WatchScreen` at the saved timestamp
   - Data source: `getContinueWatching()` query — joins `anime` + `watch_progress` where `status = 'WATCHING'`, `completed = 0`, ordered by most recent `watched_at`

2. **Trending Now** — grid of `AnimeCard` components (2 columns on phone, 3 on tablet)
3. **[Current Season] [Year]** — same grid layout  
4. **All-Time Popular** — same grid layout

**Loading state:** Skeleton placeholders — `animate-pulse` rectangular blocks matching card proportions (3:4 aspect ratio for cover, short bars for text). Show 6-12 skeleton cards per section.

**Error handling:** Use `Promise.allSettled` for all four parallel fetches. If some fail but others succeed, show the successful sections + a subtle error banner at the top: "Some sections failed to load" with a Retry button. Never show a blank screen if you already had data — preserve stale content on refresh failure.

### 4.3 AnimeCard Component

The foundational card used everywhere anime is displayed in a grid.

- **3:4 aspect ratio** cover image (lazy loaded)
- **Top-left badge:** format (TV, MOVIE, OVA, etc.) — small pill with semi-transparent black bg
- **Top-right badge:** score as "★ 8.5" — yellow text, semi-transparent black bg
- **Below image:** Title (2-line clamp), year, episode count, up to 3 genre chips
- Tapping navigates to `AnimeDetailScreen`

### 4.4 Anime Detail Screen

This is the richest screen. Layout from top to bottom:

1. **Banner area** (h: ~250px)
   - Banner image fills width, `object-cover`
   - If no banner: solid color fill using `coverImage.color` from AniList
   - Gradient overlay: transparent at top → `dark-950` at bottom
   - Back button (top-left, floating, semi-transparent black bg)

2. **Cover + Info** (overlaps the banner by ~130px)
   - Cover image: ~150w × 225h, rounded corners, drop shadow, positioned left
   - To the right of cover:
     - **Title** (English preferred, romaji fallback) — bold, large
     - Romaji subtitle if different from English — muted text
     - **Meta row:** ★ score, format, episode count, season/year, studio — inline with icons
     - **Genre chips:** horizontal wrap, small rounded pills, bg-dark-800
     - **Action buttons row:**
       - "Add to Library" / status dropdown (see StatusSelector below)
       - "Start Watching" / "Continue Watching" primary button

3. **Description** — collapsed to 4 lines with "Read more" toggle. Strip HTML tags from AniList's description.

4. **Episodes grid** — numbered buttons in a tight grid
   - **Unwatched:** `bg-dark-900`, muted text
   - **In-progress:** `bg-dark-800`, white text, thin accent progress bar at bottom
   - **Completed:** `bg-accent/20`, accent text, small checkmark icon top-right
   - Header: "Episodes (X / Y watched)"
   - Tapping an episode → `WatchScreen`
   - **Long-press to toggle watched/unwatched** (this replaces the desktop right-click)

5. **Related anime** — horizontal scroll of small cover images with relation type label (SEQUEL, PREQUEL, SIDE_STORY). Tapping navigates to that anime's detail screen.

### 4.5 StatusSelector (Library Status Dropdown)

- Dropdown/bottom sheet with 5 options: Watching, Plan to Watch, Completed, On Hold, Dropped
- Each option has a colored icon (matching status colors above)
- Currently selected option is highlighted with accent
- "Remove from Library" option at bottom (red, separated by divider)
- **Important behavior:** Setting status to "Completed" should automatically **mark all episodes as watched** in the database

### 4.6 Search / Discover Screen

- Large search input at top, auto-focused when tab is selected
- **Before searching:** show trending anime as default content
- **After typing + submit:** show results in a grid
- Debounce search by ~500ms
- Results grid: same `AnimeCard` layout as home
- Empty state: "No anime found. Try a different search term."
- AniList search query: `media(search: $search, type: ANIME, sort: POPULARITY_DESC)`

### 4.7 Library Screen

- **Filter chips** at top: All, Watching, Plan to Watch, Completed, On Hold, Dropped — each with its colored icon. Horizontally scrollable.
- **List layout** (not grid — library entries are denser):
  - Each row: small cover thumbnail (48×64), title (truncated), format badge, status badge (colored pill), progress "X / Y" with progress bar
  - Swipe-to-delete (or long-press → action sheet with "Remove")
  - Tapping → AnimeDetailScreen
- **Empty state:** Library icon (large, muted), "Your library is empty", "Discover Anime" button
- Sort: most recently updated first (`ORDER BY updated_at DESC`)

### 4.8 Watch Screen (THE HARD ONE)

This is the most complex screen. It should be **fullscreen** — hide status bar, navigation bar, and tab bar.

#### Layout
- **Video player** fills the screen
- **Overlay controls** that auto-hide after 3 seconds of inactivity, reappear on tap
- **Top bar overlay** (when visible):
  - Back button (returns to AnimeDetailScreen, NOT back in nav stack history)
  - Title + "EP X" label
  - SUB/DUB toggle (segmented control — two small buttons: "SUB" | "DUB")
  - Prev/Next episode buttons with "X / Y" counter
  - Episode list button → opens a slide-in panel or bottom sheet

#### Video Player Controls (overlay, auto-hiding)
- Play/pause (center tap or button)
- Seek bar with thumb indicator + time display (MM:SS / MM:SS)
- Skip back 10s button
- Previous/Next episode skip buttons
- Volume slider (or use system volume)
- Fullscreen / orientation lock toggle

#### Episode List Panel
- Slides in from the right (or bottom sheet on phone)
- Lists all episodes with number, watched status (checkmark vs number), and progress percentage
- Active episode highlighted in accent
- **Long-press to toggle watched/unwatched**
- Tapping an episode switches to it (in-place, no navigation)

#### Progress Saving
- Report progress every **5 seconds** via a callback → writes to SQLite
- Mark episode as **completed when `currentTime / duration > 0.85`** (85% threshold)
- On video end: save as completed, then **auto-advance to next episode** after 1.5s delay
- Save the video source URL in `watch_progress.video_source` so it can be reused if the user returns

#### Auto-Add to Library
- When the user starts watching ANY episode, if the anime isn't in the library yet, **automatically add it with status "WATCHING"**

#### Audio Type (Sub/Dub) Persistence
- Store per-anime audio preference in local storage: key = `audio-pref-{anilistId}`, value = `'sub'` or `'dub'`
- Changing audio type triggers a full re-fetch from the provider with the new type

---

## 5. AniList GraphQL API

### Key Details
- Endpoint: `https://graphql.anilist.co` (POST)
- No authentication needed for public queries
- **Rate limiting:** Returns 429 with `Retry-After` header. Implement retry with exponential backoff, **cap wait at 5 seconds** per retry (max 3 retries). Don't freeze the UI.
- Returns HTML in `description` field — you must **strip HTML tags** before displaying

### Caching Strategy (Important for Performance)
Implement an **in-memory TTL cache** to avoid redundant API calls when users navigate back and forth:

| Query type | TTL |
|---|---|
| Trending, Popular, Seasonal | 5 minutes |
| Search results | 2 minutes |
| Anime details | 10 minutes |

Cache key format: `${queryType}:${JSON.stringify(variables)}`

### GraphQL Fragments
The desktop app uses two fragments — `AnimeFields` (for list views) and `AnimeDetailFields` (for detail view with relations and trailer). Reuse the same structure:

**List query fields:** `id, title { romaji english native }, coverImage { large extraLarge color }, bannerImage, description, episodes, format, status, season, seasonYear, genres, averageScore, popularity, studios(isMain: true) { nodes { name } }, nextAiringEpisode { airingAt episode }`

**Detail query adds:** `trailer { id site }, relations { edges { relationType(version: 2) node { id title { romaji english } coverImage { large } format type } } }`

### Season Calculation
```
Month 1-3  → WINTER
Month 4-6  → SPRING
Month 7-9  → SUMMER
Month 10-12 → FALL
```

---

## 6. Streaming Provider — The Hardest Part

### How It Works on Desktop
1. Take the AniList anime title → search gogoanime by keyword
2. Fuzzy-match the results to find the correct show (sub or dub variant)
3. Cache the mapping (AniList ID → gogoanime slug) in SQLite `provider_cache` table
4. Fetch the episode page → extract embed URLs from `data-video` attributes via regex
5. Load the embed URL in a **hidden BrowserWindow** (headless Chromium) to intercept the actual `.m3u8` video URL from network requests
6. Proxy the HLS stream through a local HTTP server that uses the Chromium session (to carry Cloudflare cookies)

### Why This Is Hard on Mobile
Steps 5 and 6 are **Electron-specific** — you don't have a hidden Chromium window or session-based proxy on mobile. You need an alternative approach:

#### Option A: Backend Server (Recommended)
Build a small backend (Node.js/Deno/Python) that:
- Accepts `(title, episode, audioType)` requests
- Does the gogoanime search + slug matching (reuse `findBestMatch` logic from `src/main/providers/gogoanime.ts`)
- Uses puppeteer/playwright headless browser to extract m3u8 URLs
- Returns the direct HLS URL to the mobile client
- The mobile app plays the HLS URL directly (native HLS support on iOS, ExoPlayer on Android)

#### Option B: WebView-Based Extraction on Device
Use an invisible WebView to load embed pages and intercept network requests (similar to the desktop approach). This is fragile but keeps the app self-contained. On React Native you'd use `react-native-webview` with `onShouldStartLoadWithRequest` to intercept URLs.

#### Option C: Direct API Reverse Engineering
Some streaming providers have internal APIs that return video URLs directly. This avoids the headless browser entirely but breaks whenever they change their obfuscation. The desktop app explicitly chose NOT to do this because it's too brittle.

### Title Matching Algorithm (Reuse This Exactly)
The fuzzy matching in `findBestMatch()` was hard to get right. Key rules:
1. **Normalize:** lowercase, strip non-alphanumeric, collapse whitespace
2. **Exact match** → score 100
3. **Contains match** (either direction) → score 75-80
4. **Word overlap** → score proportional to % of words shared (×60)
5. **Length penalty:** If the result title is significantly longer than the search title, penalize by `min(lengthDiff × 2, 20)` — this prevents matching "Tokyo Ghoul:re" when searching for "Tokyo Ghoul"
6. **Minimum threshold:** Reject matches scoring below 30
7. **Tie-break:** Prefer shorter titles (closer to exact match)
8. **Sub vs Dub filtering:** Dub entries have `(Dub)` in the title or `-dub` suffix in the slug. Filter by audio type first, fall back to all results if no matches for the preferred type.

### Domain Fallback
Gogoanime changes domains frequently. The desktop app maintains a list of 6 mirror domains and tries them in order:
```
gogoanimes.fi, anitaku.pe, anitaku.bz, anitaku.so, gogoanime3.co
```
It validates each by checking if the response HTML contains known keywords (`gogoanime`, `Recent Release`, `anime_name`, `last_episodes`). Cache the working domain, but **reset and re-probe on errors** (call `resetDomain()`).

### Search Fallback Strategy
If the initial title search returns no usable match:
1. Try the English title if different from romaji
2. Try a "simplified" title: strip season/part numbers, remove special characters
3. If all fail, throw a user-visible error: "Could not find [title] on the streaming provider"

### Provider Cache
Cache AniList→gogoanime slug mappings in SQLite:
- Table: `provider_cache (anilist_id, provider, provider_id, cached_at)`
- Separate cache entries for sub vs dub: provider = `'gogoanime'` vs `'gogoanime-dub'`
- "Re-search" button: lets the user clear the cache for an anime and forces a new search (useful when wrong anime was matched)

---

## 7. Database Schema

Use the exact same SQLite schema as the desktop app for data compatibility:

```sql
CREATE TABLE anime (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  anilist_id INTEGER UNIQUE NOT NULL,
  title TEXT NOT NULL,
  title_english TEXT,
  cover_image TEXT,
  banner_image TEXT,
  description TEXT,
  episodes_total INTEGER,
  status TEXT NOT NULL DEFAULT 'PLAN_TO_WATCH',
  format TEXT,
  genres TEXT,          -- JSON string: '["Action","Drama"]'
  season TEXT,
  season_year INTEGER,
  score REAL,
  added_at DATETIME DEFAULT (datetime('now')),
  updated_at DATETIME DEFAULT (datetime('now'))
);

CREATE TABLE watch_progress (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  anime_id INTEGER NOT NULL,
  episode_number INTEGER NOT NULL,
  watched_seconds REAL DEFAULT 0,
  total_seconds REAL DEFAULT 0,
  completed INTEGER DEFAULT 0,     -- 0 or 1
  video_source TEXT,               -- cached URL for quick resume
  watched_at DATETIME DEFAULT (datetime('now')),
  FOREIGN KEY (anime_id) REFERENCES anime(id) ON DELETE CASCADE,
  UNIQUE(anime_id, episode_number)
);

CREATE TABLE provider_cache (
  anilist_id INTEGER NOT NULL,
  provider TEXT NOT NULL,
  provider_id TEXT NOT NULL,
  cached_at DATETIME DEFAULT (datetime('now')),
  PRIMARY KEY (anilist_id, provider)
);
```

### Important Database Behaviors
- **Never un-complete an episode:** If `completed = 1` in an existing row, a `saveProgress` call should NOT overwrite it back to `0`. The desktop app explicitly checks for this.
- **Persist on every write** (not just on interval) — mobile apps can be killed at any time.
- **Genres are stored as JSON strings**, not a separate table.
- `status` column uses union type: `'WATCHING' | 'COMPLETED' | 'PLAN_TO_WATCH' | 'ON_HOLD' | 'DROPPED'`

---

## 8. Critical Behaviors & Edge Cases

### 8.1 Progress Saving
- Fire every **5 seconds** (use a ref/timer to debounce, not every state update)
- 85% threshold for completion — `currentTime / duration > 0.85`
- When video ends naturally: mark completed, then auto-advance after 1.5s
- When user navigates away mid-episode: save whatever progress exists immediately
- Store `video_source` URL so returning to an episode can skip the provider fetch

### 8.2 Episode Completion Logic
- `toggleEpisodeCompleted`: If episode has no progress row, create one with `completed = 1, watched_seconds = 0, total_seconds = 0`
- `markAllEpisodesCompleted(anilistId, totalEpisodes)`: Loop 1..N, create or update each episode row. Used when status changes to COMPLETED.

### 8.3 Continue Watching Query
The "Continue Watching" section on Home shows:
- Only anime with status `WATCHING`
- Only the **most recent episode** per anime (by `watched_at`)
- Only episodes where `completed = 0`
- Sorted by `watched_at DESC`, limit 20

### 8.4 Smart Episode Resume
On the AnimeDetailScreen, the "Continue Watching" button should navigate to the **next logical episode**:
```
nextEp = max(episode_numbers with progress) + (all_have_progress_completed ? 1 : 0)
ep = min(nextEp, totalEpisodes)
```
If no progress exists at all, start at episode 1.

### 8.5 Error Handling Patterns
- **AniList errors:** Retry with backoff, cap at 5s. Show section-level errors, not full-screen errors. Use `Promise.allSettled` on the Home screen so one section's failure doesn't break others.
- **Provider errors:** Show centered error with message + "Retry" button + "Re-search" button (clears provider cache). The Re-search is critical for when the wrong anime gets matched.
- **Network offline:** Preserve any cached/existing data on screen. Show a non-blocking banner.

### 8.6 Audio Type (Sub/Dub)
- Default to `'sub'`
- Persisted **per anime** (not globally) — different anime may have different preferences
- Changing it triggers a full refetch from the provider (different gogoanime slug for sub vs dub)
- Provider cache stores separate entries: `gogoanime` vs `gogoanime-dub`

---

## 9. Mobile-Specific Adaptations

### Things That Change from Desktop
| Desktop | Mobile |
|---|---|
| Custom TitleBar with window controls | Use native status bar, hide it on WatchScreen |
| Sidebar navigation (left rail) | Bottom tab bar |
| Right-click to toggle watched | **Long-press** to toggle watched |
| Hover effects on cards | Remove hover states, add press/tap feedback (opacity or scale) |
| Mouse-based auto-hide controls | Tap to show/hide video overlay controls |
| Hidden BrowserWindow extraction | WebView-based or backend server (see §6) |
| Local HLS proxy via `http.createServer` | May not be needed if playing HLS directly |
| Keyboard shortcuts (space, arrows, f, m, k) | Touch gestures: double-tap sides to seek ±10s, swipe up/down for volume |
| Window fullscreen | Landscape orientation lock on WatchScreen |
| Drag regions for window dragging | Not applicable |
| `webSecurity: false` for loading images | Use a standard image component (React Native `Image` handles cross-origin fine) |
| `localStorage` for preferences | Use `AsyncStorage` or `SharedPreferences` |

### Touch Gestures for Video Player
- **Tap** center → toggle play/pause
- **Double-tap left** → seek back 10s
- **Double-tap right** → seek forward 10s
- **Swipe up/down (right side)** → volume up/down  
- **Swipe up/down (left side)** → brightness up/down
- **Pinch** → zoom/fit toggle
- **Swipe down from top** → close WatchScreen

### Orientation Handling
- Home, Search, Library, AnimeDetail: **portrait locked**
- WatchScreen: **landscape preferred**, allow rotation. Lock to landscape on enter, unlock on exit.

### Image Loading
- Use progressive/lazy loading for all anime cover images
- Cache images aggressively (the AniList CDN URLs are stable)
- Placeholder: solid `dark-800` rectangle matching the aspect ratio

### Offline Support (Nice to Have)
- Library + watch progress should work fully offline (SQLite)
- AniList data can be stale-displayed from a previous cache
- Provider fetching obviously requires internet — show clear error

---

## 10. Files to Reference in the Desktop Codebase

| What | File | Key sections |
|---|---|---|
| AniList API client | `src/renderer/src/services/anilist.ts` | GraphQL queries, fragments, caching, rate limit handling |
| Type definitions | `src/renderer/src/types/index.ts` | `AniListAnime`, `LocalAnime`, `EpisodeProgress`, `WatchStatus`, `ContinueWatchingItem` |
| Provider types | `src/main/providers/types.ts` | `VideoSource`, `StreamingInfo`, `AudioType` |
| Provider orchestration | `src/main/providers/index.ts` | Search → match → cache → fetch flow |
| Gogoanime scraper | `src/main/providers/gogoanime.ts` | Domain fallback, search, `findBestMatch`, hidden window extraction |
| Database layer | `src/main/database.ts` | Schema, all queries, progress save logic, "never un-complete" guard |
| HLS proxy | `src/main/proxy.ts` | Playlist rewriting, session-based proxying |
| Home screen | `src/renderer/src/pages/HomePage.tsx` | Continue watching UI, `Promise.allSettled` pattern, error banner |
| Anime detail | `src/renderer/src/pages/AnimePage.tsx` | Episode grid, status selector, "next episode" logic |
| Watch screen | `src/renderer/src/pages/WatchPage.tsx` | Auto-hide controls, sub/dub toggle, progress saving, auto-advance |
| Video player | `src/renderer/src/components/VideoPlayer.tsx` | HLS.js setup, controls overlay, keyboard shortcuts, progress reporting |
| Embed player | `src/renderer/src/components/EmbedPlayer.tsx` | WebView extraction, CSS/JS injection, autoplay workarounds |
| Library | `src/renderer/src/pages/LibraryPage.tsx` | Filter by status, progress display, empty state |
| Status selector | `src/renderer/src/components/StatusSelector.tsx` | Dropdown behavior, remove option |
| Anime card | `src/renderer/src/components/AnimeCard.tsx` | Card layout, badges, genre chips |
| Global styles | `src/renderer/src/styles/globals.css` | Button variants, progress bars, scrollbar styling |
| Tailwind config | `tailwind.config.js` | Color palette definition |

---

## 11. Summary of What Was Hardest to Get Right

1. **Hidden BrowserWindow video extraction** — The streaming providers use obfuscated JS players. We couldn't reverse-engineer them reliably. Loading the page in a hidden Chromium window and intercepting network requests for `.m3u8` URLs was the only stable approach. On mobile, you'll need to solve this differently (see §6).

2. **Cloudflare bypass** — The CDN is behind Cloudflare. On desktop we reuse the hidden window's session (which has passed the CF challenge) to proxy all HLS requests. On mobile, you may need a server-side component or a WebView-based approach.

3. **HLS playlist rewriting** — m3u8 playlists contain relative URLs to `.ts` segments and encryption key files. All these URLs must be rewritten to go through the proxy so Cloudflare cookies are included. The `rewritePlaylist()` function handles this. On mobile with direct HLS playback, you might not need this if you can handle auth headers natively.

4. **Title matching / fuzzy search** — Getting the right anime from a keyword search is surprisingly hard. Season numbers, special characters, punctuation, sequels with similar names — the scoring algorithm in `findBestMatch()` was iterated many times. Copy it exactly.

5. **Domain rotation** — Gogoanime frequently changes domains. The fallback system with validation needs to be robust. Don't hardcode a single domain.

6. **Progress save timing** — Saving too frequently hammers the database; too infrequently risks losing progress if the app is killed. 5-second intervals with the "never un-complete" guard is the sweet spot.

7. **Autoplay in embed players** — Web players aggressively block autoplay. The desktop app uses a multi-attempt strategy: inject JS to click play buttons, dispatch synthetic mouse/keyboard events, force `video.muted = true` then play, and unmute after playback starts. It retries up to 8 times with delays. Even then it's not 100%. On mobile native players, this is less of an issue.

8. **Episode auto-advance** — After an episode ends, auto-navigating to the next one needs a delay (1.5s) to let the completion save finish. Navigate with `replace: true` so back button goes to the detail screen, not to the previous episode.

9. **Error resilience on Home screen** — Using `Promise.allSettled` instead of `Promise.all` was critical. One failing API section shouldn't blank the entire screen. Show what you can, offer retry for the rest.
