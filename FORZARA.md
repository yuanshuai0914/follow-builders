# FORZARA: Follow Builders, Not Influencers

## What is this?

An AI skill that acts like your personal research assistant for the AI industry. It monitors the people who are actually *building* things in AI — founders, PMs, researchers, engineers — and delivers you a curated digest of what they're saying, so you don't have to scroll through Twitter or watch 2-hour podcasts.

Think of it like a daily newspaper, but instead of news reporters deciding what's important, you're getting it straight from the source: the builders themselves.

## The Philosophy

The AI space is noisy. There are thousands of people tweeting about AI, but most of them are just repackaging information from a few dozen people who actually know what's happening because they're *doing* the work. This skill cuts through the noise by going directly to those builders.

The name says it all: **follow builders, not influencers.**

## How the Technical Architecture Works

Imagine a three-person team:

1. **The Runner** (`scripts/fetch-content.js`) — A Node.js script that does the mechanical work of going out and grabbing content. It visits YouTube channels via the Supadata API to get podcast transcripts, and checks X/Twitter profiles via Rettiwt-API (in "guest mode" — more on that below) to get recent posts. It keeps a memory file (`state.json`) so it never fetches the same thing twice.

2. **The Brain** (`SKILL.md`) — This is the instruction manual for the AI agent. It tells the agent: "When you wake up for your cron job, run The Runner, take what it brings back, and remix it into a digestible digest using the prompts." It also handles onboarding new users and processing their settings changes.

3. **The Voice** (`prompts/`) — Four markdown files that control *how* the content gets summarized. They're written in plain English so anyone can tweak them. Want shorter summaries? Edit the file. Want a more casual tone? Edit the file. Or just tell your agent "make it more casual" and it edits the file for you.

### The Data Flow

```
Cron wakes up at 8am
  -> Agent reads your config (language, timezone, sources)
  -> Agent runs fetch-content.js
  -> Script calls Supadata API for YouTube transcripts
  -> Script calls Rettiwt-API for X/Twitter posts
  -> Script returns JSON blob of new content
  -> Agent reads the prompt files
  -> Agent summarizes each source using the prompts
  -> Agent translates if you want Chinese/bilingual
  -> Agent outputs the digest
  -> OpenClaw/Claude Code delivers it to your Telegram/Discord/etc.
```

## The Structure of the Codebase

```
follow-builders/
  SKILL.md                     <- The brain. Agent reads this.
  README.md                    <- User-facing docs.
  config/
    default-sources.json       <- The curated list of 32 X accounts + 5 podcasts
    config-schema.json         <- Documents what user config looks like
  prompts/
    summarize-podcast.md       <- How to summarize a podcast
    summarize-tweets.md        <- How to summarize tweets
    digest-intro.md            <- How to assemble the full digest
    translate.md               <- How to translate to Chinese
  scripts/
    fetch-content.js           <- The Runner. Does all the API calls.
    package.json               <- Node.js dependencies
  examples/
    sample-digest.md           <- Shows users what the output looks like
```

**On the user's machine** (created during setup):
```
~/.follow-builders/
  config.json                  <- Their preferences (language, timezone, etc.)
  state.json                   <- Memory of what's been fetched already
  .env                         <- Their Supadata API key
```

## Technologies Used and Why

### Rettiwt-API (for X/Twitter)
This was the most interesting technical decision. We needed to fetch tweets from 32 accounts daily. The options were:
- **Official X API** ($100/month) — too expensive for users
- **Apify** (cloud scraping service) — free tier turned out to be too limited (10 items, no API access)
- **Playwright** (browser automation) — X blocks automated browsers aggressively
- **Rettiwt-API** (Node.js library) — uses Twitter's *internal* API in "guest mode"

We went with Rettiwt-API because it works like a logged-out browser visiting Twitter. No login needed, no API key, no risk of account bans. The tradeoff is that it can break when X changes their internal API (every 2-4 weeks), but the library is actively maintained and fixes come within days.

### Supadata (for YouTube)
Supadata provides a simple REST API to get YouTube video transcripts. We use three endpoints:
- `/youtube/channel/videos` — get recent video IDs from a channel
- `/youtube/video` — get video metadata (title, date)
- `/youtube/transcript` — get the full transcript as text

The free tier gives 200 credits/month (1 credit per transcript), which is plenty for 5 podcast channels.

### Why Node.js (not Python)?
OpenClaw runs on Node.js, so every OpenClaw user already has it installed. No extra runtime to set up. Also, Rettiwt-API is a Node.js library.

### Why file-based state instead of a database?
For a personal tool that processes ~50 items/day, a JSON file is perfect. No database to install, no server to run, easy to inspect and debug. We use `proper-lockfile` to prevent corruption if two runs overlap.

## Key Design Decisions

### "The agent is the UI"
Non-technical users shouldn't have to edit JSON files. So *all* configuration happens through conversation. Want to add a source? Tell your agent. Want to change the language? Tell your agent. Want shorter summaries? Tell your agent. The agent edits the files behind the scenes.

### Editable prompts in plain English
The summarization prompts are stored as separate markdown files, not baked into the code. This means:
1. Users can customize how their digest reads by editing plain English (or asking the agent to)
2. The skill creator (you) can iterate on prompts without touching any code
3. The agent reads prompts fresh on each run, so changes take effect immediately

### Dual platform compatibility
The skill works on both OpenClaw and Claude Code. The SKILL.md detects which platform it's running on and adjusts the cron setup accordingly. The fetcher script is platform-agnostic — it just outputs JSON.

### Source merging (defaults + customization)
Instead of asking users to maintain a full source list, we ship with curated defaults and let users *add* or *remove* from them. The merge logic: start with defaults, add user additions, subtract user removals. This way updates to the default list automatically flow to users.

## Bugs We Ran Into and How We Fixed Them

### The Apify Detour
We originally planned to use Apify for X/Twitter scraping. The spec review caught that Apify's free tier only allows 10 items and no API access. We would have discovered this at runtime — a frustrating experience for users. The fix: switched to Rettiwt-API (free, no API key, guest mode). Lesson: **always verify free tier limits before committing to a paid service as a dependency.**

### Supadata Endpoint Confusion
The initial plan used wrong Supadata endpoints (`/youtube/channel` instead of `/youtube/channel/videos`). The API returns just an array of video ID strings, not full video objects — so we needed an extra API call per video to get metadata (title, date). The code review caught this before we wrote the code. Lesson: **read the actual API docs, not summaries. Verify response shapes.**

### State File Race Conditions
If a user triggers `/ai` manually while a cron job is running, two processes would read/write `state.json` simultaneously. Could corrupt the file. Fix: file locking via `proper-lockfile`. If a lock exists, the second process waits (3 retries) then tells the user to try again. Lesson: **any file that two processes might touch needs locking.**

### Git Init Ordering
The implementation plan had `git init` in Task 9, but individual commits started in Task 1. All those commits would have failed. Caught in plan review. Fix: moved `git init` to Task 0. Lesson: **plan reviews catch sequencing bugs that are obvious in retrospect but easy to miss when writing.**

## Lessons for How Good Engineers Think

### 1. Hybrid Architecture
Good engineers match the tool to the job. Fetching data from APIs is mechanical — a script does it faster and cheaper than an LLM. Summarizing content is creative — an LLM does it better than a script. So we use both: script for fetching, LLM for remixing. Don't use an LLM for everything just because you can.

### 2. Design for the user, not the developer
The biggest design insight was making the agent the configuration UI. A developer would happily edit a JSON file. A non-technical user would bounce. Same functionality, radically different UX.

### 3. Degrade gracefully
The fetcher never blocks the entire digest because one source failed. If @karpathy's profile is unreachable, you still get the other 31 builders and all the podcasts. Partial results beat no results.

### 4. Test your assumptions about APIs early
Both our original API choices (Apify free tier, Supadata endpoints) had issues that would have caused runtime failures. The spec review and research phase caught both. Spending an hour verifying APIs saves days of debugging.

### 5. Separate content from logic
Keeping the summarization prompts as editable files (not hardcoded strings) means you can iterate on the "voice" of the digest without touching code. This is a pattern worth reusing: if users might want to customize text, put it in a file.
