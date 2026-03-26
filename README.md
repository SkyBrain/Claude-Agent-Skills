# Claude Agent Skills

A collection of reusable skills for [Claude](https://claude.ai) desktop (Cowork mode) and the Claude Agent SDK. Each skill is a self-contained module that teaches Claude how to perform a specific task — automating browser workflows, creating documents, processing files, and more.

---

## What is a Skill?

A skill is a folder containing a `SKILL.md` file that provides Claude with step-by-step instructions, context, and best practices for a particular task. When a skill is loaded, Claude follows its guidance to produce higher-quality, more consistent results than it would from a generic prompt alone.

Skills live under the `.skills/skills/` directory in your Claude workspace and are picked up automatically by Claude desktop.

---

## Skills in This Repo

| Skill | Description |
|---|---|
| `yt-music-library-adder` | Bulk-adds songs and albums to a YouTube Music library in batches of 10, with local caching to skip already-processed entries |

---

## Installation

### Option A — Install a `.skill` file (recommended)

Each skill in this repo can be packaged as a `.skill` file — a zip archive that Claude desktop can install directly. Download the `.skill` file for the skill you want, then open it in Claude desktop and click **"Copy to your skills"**.

### Option B — Clone the repo

```bash
git clone https://github.com/SkyBrain/Claude-Agent-Skills.git
```

Then copy the skill folder(s) you want into your Claude skills directory:

```bash
cp -r claude-agent-skills/yt-music-library-adder ~/Documents/Claude/.skills/skills/
```

Claude desktop picks up skills automatically from `~/Documents/Claude/.skills/skills/` at the start of each new session.

---

## Usage

Once a skill is installed, just describe what you want in Claude desktop — Claude reads the `description` field of each installed skill and decides automatically which one to use.

**Example:**

> "Add these songs to my YouTube Music library: …" → triggers `yt-music-library-adder`

No special syntax or commands are required. The skill's description is tuned to match the natural language you'd use to ask for the task.

---

## Contributing

Pull requests are welcome. When adding a skill, please:

- Write a clear `description` in the frontmatter (this is what Claude uses to decide whether to trigger the skill — make it specific and keyword-rich).
- Include an **Examples** or **What this skill does** section near the top of `SKILL.md`.
- Test the skill end-to-end at least once before submitting.

---

## License

MIT
