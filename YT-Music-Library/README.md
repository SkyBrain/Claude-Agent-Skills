# YouTube Music Library Adder — Skill for Claude

This skill lets Claude automatically add songs and albums to your YouTube Music library in bulk. Instead of searching and saving each song one by one, you give Claude a list and it handles everything — including remembering what it has already added so it never does the same work twice.

---

## What you'll need

Before getting started, make sure you have:

- **The Claude desktop app** (Cowork) installed on your Mac
- **Google Chrome** with the **Claude in Chrome** extension installed
- A **YouTube Music account**, and to be logged into it in Chrome

---

## How to install the skill

1. Locate the `yt-music-library-adder.skill` file you received.
2. Double-click it — or open it from the Claude desktop app.
3. You'll see a **"Copy to your skills"** button. Click it.
4. That's it! The skill is now installed and Claude will use it automatically whenever you ask to add songs to YouTube Music.

---

## How to use it

### Step 1 — Open Claude (Cowork)

Launch the Claude desktop app on your Mac.

### Step 2 — Give Claude your list of songs

Type something like:

> "Add these songs to my YouTube Music library:"

Then paste your list. It can be in any format — song titles, artist names, or both. For example:

```
Blinding Lights – The Weeknd
Levitating – Dua Lipa
Peaches – Justin Bieber
```

### Step 3 — Let Claude work

Claude will open YouTube Music in Chrome and start processing your list, 10 songs at a time. For each song it will:

- Search YouTube Music
- Find the album or single
- Check if it's already in your library
- Add it if it isn't

You'll see progress updates in the chat as each batch is completed.

### Step 4 — Review the summary

When all songs have been processed, Claude will give you a summary showing:

- ✅ Songs newly added to your library
- ✅ Songs that were already in your library
- ⏭️ Songs skipped (wrong release year — only recent releases are added)
- ❌ Songs not found on YouTube Music

---

## Things to know

**Only recent releases are added.** The skill is designed to add music released in the current year or the previous year. Older albums will be skipped and noted in the summary.

**It remembers what it's done.** A small file called `yt_music_library_cache.json` is saved to your `Documents/Claude/` folder. This is your personal record of every song Claude has added. On future runs, Claude checks this file first so it never re-processes songs it has already handled — making repeat runs much faster.

**YouTube Music must be open and logged in.** Claude uses Chrome to interact with YouTube Music on your behalf. If you're not logged in, Claude will stop and let you know.

**You can run it multiple times.** If you have a long list or want to add more songs later, just paste a new list and Claude will pick up right where it left off — skipping anything already in the cache.

---

## Troubleshooting

**Claude says it can't find YouTube Music or isn't logged in**
Open Chrome, go to [music.youtube.com](https://music.youtube.com), and make sure you're signed into your Google account. Then try again.

**A song wasn't added but it should have been**
It may have been released before last year, or the title on YouTube Music might be slightly different. You can try asking Claude to search for it directly by its exact YouTube Music title.

**The cache file — do I need to touch it?**
No. Claude manages it automatically. You never need to open or edit it.

---

## Uninstalling

To remove the skill, open the Claude desktop app, go to your Skills settings, and delete **yt-music-library-adder**. The cache file at `Documents/Claude/yt_music_library_cache.json` can be deleted manually if you want to clear the history.
