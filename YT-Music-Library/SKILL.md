---
name: yt-music-library-adder
description: >
  Adds singles and song albums to YouTube Music library in batches. Use this skill whenever
  the user wants to bulk-add songs, singles, or albums to their YouTube Music library,
  or mentions searching YouTube Music and saving albums/songs to their library. Trigger
  any time the user provides a list of songs/artists to add to YT Music, mentions
  "yt_music_library_cache.json", or says things like "add these songs to my YT Music
  library", "save these albums to YouTube Music", "process my music list on YT Music",
  or "sync my songs to YouTube Music". Also trigger when the user wants to update or
  check their YT Music library cache.
---

# YouTube Music Library Adder

This skill automates adding singles and song albums to YouTube Music library in batches of 10, tracking what's been added in a local cache file.

## What this skill does

1. Reads a list of songs/artists from the user
2. Loads an existing `yt_music_library_cache.json` cache (or creates one)
3. **Checks the cache first** — skips any song whose album is already cached (no browser work needed)
4. Processes uncached songs in **batches of 10** via YouTube Music search
5. For each song: searches, extracts album URL via JS, checks year + library status, adds if needed
6. **Writes the cache to disk immediately after each song** — not just at the end of the batch

---

## Before you begin

**Gather these from the user if not already provided:**
- The list of songs/artists to process (paste in chat)

**Determine the cache file path:**

The cache is stored in a fixed folder that persists across all sessions:

```
<mnt-prefix>Claude/yt_music_library_cache.json
```

The `mnt/` prefix is injected at the top of every skill invocation in a line like:

```
Base directory for this skill: /sessions/<session-name>/mnt/.skills/skills/yt-music-library-adder
```

Extract everything up to and including `mnt/` from that path. For example, if the injected base directory is:
```
/sessions/nice-jolly-cannon/mnt/.skills/skills/yt-music-library-adder
```
Then the cache path is:
```
/sessions/nice-jolly-cannon/mnt/Claude/yt_music_library_cache.json
```

This resolves to `~/Documents/Claude/yt_music_library_cache.json` on the user's computer, regardless of session name.

**If the cache file does not exist**, create the `Claude/` directory if needed (e.g. `mkdir -p`), then create a new empty cache using the format defined in the Cache file format section below before proceeding.

The user may override this path by specifying a different location explicitly in chat. Otherwise, always use the `Claude/` path derived above.

**Get a browser tab ready:**
Use `tabs_context_mcp` to get or create a tab. You'll be navigating to YouTube Music for each search.

---

## Cache file format

The cache lives at the path determined above. Create it if it doesn't exist.

```json
{
  "_meta": {
    "description": "Local cache of YouTube Music albums/singles already in library. Used to skip re-checking YouTube Music during batch processing.",
    "format": "Keys are normalized album names (lowercase, trimmed). Values contain album details and confirmation date.",
    "last_updated": "2026-03-20",
    "total_entries": 55
  },
  "albums": {
    "some album title (from \"some movie\")": {
      "display_name": "Some Album Title (From \"Some Movie\")",
      "artists": "Artist One, Artist Two",
      "year": 2026,
      "type": "Single",
      "url": "https://music.youtube.com/playlist?list=OLAK5uy_...",
      "status": "in_library",
      "batch": "Batch label — added during processing",
      "confirmed_date": "2026-03-20"
    }
  }
}
```

**Key rules:**
- Keys are the album's display name, **lowercased and trimmed** (e.g. `"namo re (from \"nagabandham\") (telugu)"`)
- **Only `in_library` entries are written to the cache** — whether newly added or confirmed already saved
- Songs that are not found or have the wrong year are NOT cached — they will be retried fresh on the next run
- `url` is the YouTube Music playlist URL for the album/single page; set to `null` if not available
- `type` is `"Single"`, `"EP"`, or `"Album"`

---

## Cache check (before any browser work)

**Load the cache at the start.** Before processing each song, normalize the expected album name (lowercase, trimmed) and check if it exists as a key in `albums`. If it does and `status` is `"in_library"`, skip it entirely — no browser action needed.

The cache key is the **album name**, not the song title. Use the album title you'd expect to find on YouTube Music (e.g. `"rubaroo (from \"dacoit (telugu)\")"` not the artist name). When the album name isn't obvious from the user's input (e.g. a song title was given), do a quick mental match against existing cache keys before deciding to search.

---

## Processing loop

Process the song list in **batches of 10**. For each batch:

1. Tell the user which batch you're starting (e.g., "Processing batch 1/5: items 1–10")
2. For each item, **check the cache first** (see above) — skip if already cached
3. For uncached items, process using the browser steps below
4. **Write the cache to disk immediately after each song is confirmed** — do not wait until the end of the batch
5. Report batch results: how many were cache hits, newly added, wrong year, not found

### Per-song steps (for uncached items only)

#### Step 1: Navigate to search using the search bar

**Always use the search bar to navigate** — never use the `navigate` tool for mid-session searches. The `navigate` tool can trigger "Leave site?" dialogs if music is playing in the player. Using the search bar avoids this entirely.

Find the search bar with `find "search bar input"`, triple-click it (to select any existing text), type the search query, and press Enter:

```
Search query format: SONG TITLE ARTIST NAME
Example: Blinding Lights The Weeknd
```

Wait **1 second** for the search results to load. Do **not** take a screenshot — go straight to Step 2.

The first time in a session (before any music has played), using `navigate` once to reach `https://music.youtube.com` is fine. After that, use the search bar for all navigation.

#### Step 2: Extract the album URL via JavaScript (preferred — no menu clicks)

Run this JS on the search page. It tries two extraction methods in order — the fastest one that returns results wins:

```javascript
(() => {
  // Method 1: ytInitialData (fastest when it works)
  try {
    const raw = JSON.stringify(window.ytInitialData || {});
    const matches = [...raw.matchAll(/"playlistId":"(OLAK[^"]+)"/g)];
    const ids = [...new Set(matches.map(m => m[1]))];
    if (ids.length > 0) return ids.slice(0, 3).map(id => 'https://music.youtube.com/playlist?list=' + id);
  } catch(e) {}

  // Method 2: Scrape OLAK IDs from anchor hrefs in the DOM (reliable fallback)
  const links = [...document.querySelectorAll('a[href*="OLAK5uy_"]')];
  const ids = [...new Set(links.map(a => {
    const m = a.href.match(/(OLAK5uy_[^&"?]+)/);
    return m ? m[1] : null;
  }).filter(Boolean))];
  return ids.slice(0, 3).map(id => 'https://music.youtube.com/playlist?list=' + id);
})()
```

- **Result has URLs** → use the first one and go to Step 3
- **Result is still empty `[]`** → fall back to the menu method (see Fallback section below)

The first URL is almost always the correct album for the top search hit. For film/show queries, the soundtrack album will be first.

#### Step 3: Navigate to the album page

Use the **search bar** again to navigate — triple-click it, type/paste the full album URL (e.g. `https://music.youtube.com/playlist?list=OLAK5uy_...`), press Enter.

Wait **1 second**. Do **not** take a screenshot — go straight to Step 4.

#### Step 4: Check year AND library status in one JS call

```javascript
(() => {
  // Use the header element for reliable year extraction
  const header = document.querySelector(
    'ytmusic-detail-header-renderer, ytmusic-immersive-header-renderer, ytmusic-responsive-header-renderer'
  );
  const yearMatch = header?.innerText.match(/\b(20\d{2})\b/);

  const libraryBtn = [...document.querySelectorAll('button[aria-pressed]')]
    .find(b => b.getAttribute('aria-label')?.toLowerCase().includes('library'));

  return {
    year: yearMatch ? parseInt(yearMatch[1]) : null,
    headerSnippet: header ? header.innerText.substring(0, 200) : null,
    libraryLabel: libraryBtn?.getAttribute('aria-label') ?? null,
    libraryPressed: libraryBtn?.getAttribute('aria-pressed') ?? null
  };
})()
```

**Interpret the result:**
- `year` is `null` → take a screenshot to diagnose, then skip (treat as wrong year)
- `year` is not the current calendar year or the previous calendar year (e.g. if today is 2026, accept 2025–2026; if today is 2027, accept 2026–2027) → log as wrong year (no cache write), move on
- `libraryLabel` contains `"Remove from library"` and `libraryPressed` is `"true"` → already saved → **cache it and move on**
- `libraryLabel` contains `"Save to library"` and `libraryPressed` is `"false"` → not yet saved → **add it (Step 5)**
- Both `null` → take a screenshot to diagnose

#### Step 5: Add to library (if needed)

Click the Save/bookmark button via JS to avoid stale ref issues:

```javascript
(() => {
  const btn = [...document.querySelectorAll('button[aria-pressed]')]
    .find(b => b.getAttribute('aria-label')?.toLowerCase().includes('save to library'));
  if (btn) { btn.click(); return 'clicked'; }
  return 'not found';
})()
```

Wait **1 second**, then re-run the JS from Step 4 to confirm `libraryPressed` changed to `"true"`. No screenshot needed unless the re-check fails.

#### Step 6: Write to cache immediately

**Right after confirming library status**, write the entry to the cache file:
- Normalize the album name (from `headerSnippet` or the page title) to lowercase + trimmed for the key
- Set `status: "in_library"` for both newly added and already-saved
- Note `"— added during processing"` in the batch label only for newly added entries
- Set `confirmed_date` to today's date (YYYY-MM-DD)
- Update `_meta.last_updated` and `_meta.total_entries`
- **Save the file to disk now** — don't accumulate and write at the end

This per-song write ensures that if the session is interrupted, all completed songs are already cached and won't be re-processed.

**Never write `not_found` or `wrong_year` entries to the cache.**

---

## Fallback: ⋮ → "Go to album" (when JS extraction returns empty)

Use this only when both JS methods in Step 2 return an empty array `[]`.

1. Take a screenshot to confirm the search result is visible and correct
2. Use `find` to locate the ⋮ action menu button for the top result banner, then click it
3. **Take a screenshot** to confirm the menu is open — you need to see it visually
4. **Click "Go to album" by coordinates** (not by `find` ref) — refs to menu items frequently go stale after navigation and silently redirect to the wrong album. Use the screenshot to determine the pixel coordinates of "Go to album" and click there directly.
5. Wait 1 second, confirm the tab title matches the expected album
6. Continue from Step 4 above

**If the tab title shows a different album than expected:** the ref was stale. Use the search bar to navigate back, repeat the search, and try again.

**If "Go to album" is not in the menu:** try to save directly from the search result. If not possible, log as not found (no cache write) and move on.

---

## Screenshot discipline

Screenshots are slow — each one adds meaningful latency across a batch of 10. Only take one when:
- JS returns `null`, an empty array, or something unexpected
- The fallback path requires visual confirmation of the open menu (always required in the fallback)
- A click doesn't register after a re-check
- The tab navigated to the wrong album

Do **not** take screenshots as routine confirmations between steps.

---

## Error handling

- **Page didn't load** → Wait 2 seconds and retry once. If still failing, log as not found (no cache write) and continue.
- **JS returns empty array on search page** → Fall back to ⋮ → "Go to album" method above.
- **Year or library status JS returns null** → Take a screenshot to diagnose; if still unclear, skip (no cache write).
- **"Go to album" navigated to wrong album** → Search bar back, retry. Use coordinate click — never use `find` refs for menu items.
- **"Go to album" not in menu** → Attempt to save from search result directly. If not possible, log as not found (no cache write).
- **Bookmark click didn't register** → Retry once. If it still shows `aria-pressed="false"`, log as not found (no cache write) and note the issue.
- **YouTube Music shows a login prompt** → Stop and tell the user they need to be logged into YouTube Music in the browser.

---

## After all batches complete

1. Report a full summary to the user:
   - Total processed
   - Cache hits (skipped — already cached ✅)
   - Newly added to library ✅
   - Already in library — confirmed via browser ✅
   - Wrong year — will retry next run (not cached)
   - Not found — will retry next run (not cached)
2. Show the list of songs that were newly added

---

## Important rules

- **Check the cache before every browser action** — normalize the album name to lowercase and look it up in `albums`; skip if found
- **Batch size is always 10** — don't process more than 10 in a group
- **Write cache after every single song** — immediately after confirming status; do not batch up writes
- **Never add to playlists** — only save to library via the bookmark/save button
- **Never click "Like"** — the heart/thumbs-up adds to Liked Music, which is different from library
- **Year filter is strict** — only the current calendar year and the previous calendar year are accepted; skip everything else (compute this dynamically from today's date at the start of each run)
- **Cache only stores `in_library` entries** — never write `not_found` or `wrong_year` to the cache
- **Cache is your source of truth** — a cache hit means no browser work at all
- **Minimize screenshots** — only when JS returns unexpected results or clicks don't register
- **Always use coordinates for fallback menu clicks** — never use `find` refs for "Go to album" after the menu opens
