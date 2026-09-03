# IRON LOG

A personal gym tracker built as an installable PWA. One HTML file, no build step, no backend, no accounts — open it, tap through your session, and everything is saved locally on your device.

**Live:** https://iron-log-tau-two.vercel.app

---

## What it does

- **3-day Push / Pull / Legs split**, 9 entries per day (warm-up → compounds → core → cardio)
- **Per-set toggles** — tap `S1`, `S2`, `S3`… as you finish each set; the exercise auto-completes when all sets are ticked
- **Weight logging with automatic PR detection** — enter a working weight, and a 🏆 appears when you beat your best. PRs can be cleared per exercise (handy when you fat-finger `450` instead of `45`)
- **Per-exercise notes** — "felt heavy, switch grip next time"
- **A demo GIF for each exercise**, shown when you expand it
- **Progress bar** per day, plus a "SESSION DONE" screen when you clear everything
- **Reset Day** — clears completion ticks but *keeps* your weights and notes
- **Installable** — Add to Home Screen gives you a real standalone app with its own icon, no browser chrome

## Design constraints

The whole app is one `index.html` (~23KB). That's deliberate:

- **No build step.** React 18 and Babel are loaded from a CDN and JSX is compiled in the browser. Clone it, open it, done. The tradeoff is a slower first paint and an in-browser Babel warning in the console — fine for a personal tool, not what you'd ship at scale.
- **No backend.** All state lives in `localStorage`, so there's nothing to host, secure, or pay for.
- **No accounts.** Nothing to sign into.

## Running locally

No install, no dependencies. Any static file server works:

```bash
npx serve .
```

Then open the printed URL. Opening `index.html` directly via `file://` also mostly works, but a server is closer to production.

## Deploying

It's a static site — the whole repo is the deployable artifact, and `index.html` at the root is the entry point. No config needed.

```bash
npx vercel --prod
```

This repo is wired to Vercel, so pushes to `main` redeploy automatically.

## Customising the workout

Everything lives in the `DAYS` object at the top of `index.html`. Each exercise looks like:

```js
{ id: "p1", name: "Flat Bench Press", type: "STR", sets: 4, reps: "8–12" }
```

| Field | Purpose |
| --- | --- |
| `id` | **Unique, permanent key** for saved data — see the warning below |
| `name` | Display name |
| `type` | `WU` · `STR` · `CORE` · `CARDIO` — drives the coloured badge |
| `sets` | Number of set toggles, or `null` for untimed/cardio entries |
| `reps` | Free text (`"8–12"`, `"12/side"`, `"30–45s"`), or `null` to hide |
| `setLabels` | *Optional.* Replaces `S1/S2/S3` with named buttons, e.g. the warm-up's `["Warmup", "Push-Ups", "Pull-Ups"]` |
| `weighted` | *Optional.* Forces a weight field on a non-`STR` exercise (the cable woodchopper uses this) |

To add an exercise image, drop a file into `Images/` and map it in `EXERCISE_MEDIA` by exercise id. Exercises with no mapping simply render without an image — nothing breaks.

> ⚠️ **Exercise `id`s are the primary key for all saved data.** Renaming an exercise is safe; reusing an existing `id` for a *different* movement is not — your old bench press weights would silently reattach to whatever now holds that id. When you swap a movement out, give it a new id and let the old data fall away.

## Data & storage

Everything is stored under a single `localStorage` key, `gymlog_v5`:

```json
{
  "sessionState": {
    "A": { "p1": { "sets": [true, true, false, false], "weight": "60", "notes": "", "done": false } }
  },
  "prs": { "p1": 60 },
  "bannerDismissed": true,
  "activeTab": "A"
}
```

On load, saved state is **merged onto** a freshly built default rather than replacing it, so adding or removing exercises won't crash the app or wipe unrelated entries.

Worth knowing, since it's a deliberate tradeoff rather than an oversight:

- Data is **per-device and per-browser**. No sync between your phone and laptop.
- **Clearing your browser data wipes your logs.** There's no export yet.
- On iOS, Safari can evict storage for sites you haven't opened in a while. **Installing to the Home Screen** makes this much less likely.

## Project structure

```
index.html           # the entire app — markup, styles, and React components
manifest.json        # PWA manifest (name, standalone display, theme colour)
apple-touch-icon.png # 180×180 home-screen icon
Images/              # exercise demo GIFs/JPGs (~6MB, 21 files)
```

## Image credits

The exercise animations in `Images/` are **not my work** — they're from [fitnessprogramer.com](https://fitnessprogramer.com) and carry that site's watermark. They're included here for a personal training log, not licensed for redistribution. If you fork this, swap in your own assets rather than reusing these.

Two files (`Calfraises.jpg`, `Tricep-pushdown.jpg`) are actually WebP images with a `.jpg` extension. Browsers sniff the real format, so they display fine.

## Licence

No licence yet, which by default means **all rights reserved** — you can view the code, but not reuse it. If you'd like to make it reusable, add an OSI licence (MIT is the usual pick).
