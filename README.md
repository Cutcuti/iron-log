# A gym tracker that survives its own schema changes

A push/pull/legs workout log, built as an installable PWA. One 430-line `index.html`, no build
step, no backend, no accounts — open it, tap through your session, and everything is saved on
your device.

**Live:** https://iron-log-tau-two.vercel.app

```bash
npx serve .          # any static server; there is nothing to compile
```

---

## Why it's worth reading

Up front: this is a small personal app. There's no algorithm here, no test suite, and the React
is unremarkable. **If you're looking for something clever, this isn't it.**

What it does have is a handful of places where the obvious implementation was quietly wrong —
mostly around *data that has to outlive the code that wrote it*, and platform behaviour that
fails silently rather than loudly. Those are the parts worth the read.

### Exercise IDs are a primary key, not a slot — [`index.html:47`](index.html#L47)

The workout roster was replaced wholesale partway through: a generic split became a specific
push/pull/legs program. Every exercise changed. The obvious move is to keep the existing IDs
(`a1…a7`, `b1…b8`, `c1…c7`) and just rewrite the names around them — they're only array slots.

That would have been **silently, unrecoverably wrong.** Saved weights and PRs are keyed by ID.
Under the old roster `a2` was *Barbell Bench Press*; under the new one, position 2 of day one is
*Incline Dumbbell Press*. Reusing the ID would have reattached an 80kg bench PR to a movement you
press maybe 30kg on — not a crash, not an error, just a number that looks plausible and is
nonsense. You'd never catch it.

So the new roster got a **fresh ID namespace** (`p*` / `u*` / `l*`). Nothing collides, so nothing
misattributes. The cost is real and worth stating: **all previous logs were orphaned.** That's the
right trade — losing history beats silently corrupting it — but it is a loss, not a free win.

The same reasoning is now a documented constraint: renaming an exercise is safe, reusing an ID for
a different movement is not.

### Restoring saved state can't be an assignment — [`index.html:149`](index.html#L149)

The original load path was one line:

```js
if (saved.sessionState) setSessionState(saved.sessionState);   // initial commit
```

Fine until the roster changes. Add an exercise and its ID is absent from every previously-saved
blob, so the first render reaches into `undefined` for `.sets` and the app **white-screens on
launch** — for existing users only. A fresh install works perfectly, which is the worst possible
failure mode to debug.

Now the saved blob is merged *onto* a freshly-built default rather than replacing it: known IDs
take the saved value, unknown ones fall back to a clean slate, and IDs that no longer exist are
dropped. Adding, removing and renaming exercises are all non-events. This was verified by seeding
a pre-revamp blob into `localStorage` and reloading, not by assuming.

### The home-screen icon can't be an SVG — [`index.html:12`](index.html#L12)

The entire reason this app is self-hosted is to get a *real* home-screen icon instead of a
generic browser shortcut. The original manifest and touch icon were inline SVG data URIs — compact,
no extra files, obviously correct.

iOS Safari doesn't reliably support SVG for `apple-touch-icon`, and when it fails it doesn't error:
it substitutes a screenshot of the page, which defeats the one thing self-hosting was for. Worse,
rasterising that SVG to check confirmed it **didn't even render correctly** — the background rect
only covered part of the canvas, leaving a white corner.

It's a real 180×180 PNG now, and the manifest is a real file. Both are boring. Boring is the point.

### "It loaded" is not "it's correct" — [`index.html:352`](index.html#L352)

The exercise demo images are 360×360 squares. They were first dropped into a full-width, 170px-tall
box with `object-fit: cover`, which is the reflexive choice for thumbnails.

`cover` fills the box and crops the overflow. A square scaled to fill a wide, short box gets scaled
up enormously and cropped to a horizontal sliver — the app showed an extreme close-up of a torso
with the barbell out of frame. On an *instructional* image, that's not a cosmetic issue; it removes
the only reason the image is there.

The fix is `contain` inside a fixed 1:1 card. Worth noting *how this got shipped broken*: the
verification step was "do all 21 images return 200?", which they did. Loading and being legible are
different properties, and only one of them was being checked.

### A magic number that would have drifted — [`index.html:406`](index.html#L406)

The day switcher is a fixed bottom dock, and the "Add to Home Screen" banner has to sit above it.
First attempt offset the banner by a constant: `calc(64px + env(safe-area-inset-bottom))`.

Measuring it in the browser showed a 2px overlap — the dock is 66px, not 64. Nudging the constant
would have "fixed" it, but the dock's height depends on `env(safe-area-inset-bottom)`, which differs
between notched and non-notched iPhones, so the correct constant is **device-dependent** and would
silently drift on hardware I can't test.

Both now live in a **single fixed container, stacked vertically**. Overlap isn't tuned, it's
structurally impossible.

---

## What this isn't

- **Not multi-device.** State is `localStorage`, scoped per browser, per device. No sync, no export.
  Clearing site data wipes your logs. Acceptable for a personal tracker; not a product.
- **Not tested.** No test suite. Verification was manual, in-browser, against seeded storage blobs.
- **Not optimised for cold start.** React and Babel come from a CDN and JSX compiles in the browser
  on every load. That's a deliberate trade for having no build step, and it's the wrong trade for
  anything with real users.
- **Not offline-capable**, despite being installable. There's no service worker, so a reload with no
  connection fails — the CDN scripts and fonts won't resolve.

## Customising the workout

Everything is in the `DAYS` object at the top of [`index.html`](index.html#L43).

```js
{ id: "p1", name: "Flat Bench Press", type: "STR", sets: 4, reps: "8–12" }
```

| Field | Purpose |
| --- | --- |
| `id` | Permanent key for saved data — **never reuse one for a different movement** |
| `name` | Display name |
| `type` | `WU` · `STR` · `CORE` · `CARDIO` — drives the coloured badge |
| `sets` | Number of set toggles, or `null` for cardio |
| `reps` | Free text (`"8–12"`, `"12/side"`, `"30–45s"`), or `null` to hide |
| `setLabels` | *Optional.* Replaces `S1/S2/S3` with named buttons — the warm-up uses `["Warmup", "Push-Ups", "Pull-Ups"]` |
| `weighted` | *Optional.* Adds a weight field to a non-`STR` exercise (the cable woodchopper is bodyweight-typed but loaded) |

`setLabels` and `weighted` both exist because the alternative was special-casing two exercises in
the render path. Adding a field the data can opt into is cheaper than a branch that has to be
maintained.

Images map by exercise ID in `EXERCISE_MEDIA`. Unmapped exercises render without one and nothing
breaks — [`index.html:350`](index.html#L350) guards on presence rather than relying on an `onError`
handler firing.

## Storage

One `localStorage` key, `gymlog_v5`:

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

PRs are write-high-water-mark, which means a typo (`450` instead of `45`) permanently poisons the
record — so each PR has a `CLEAR` action. On iOS, Safari can evict storage for sites left unopened;
installing to the Home Screen makes that far less likely.

## Structure

```
index.html           # the whole app — markup, styles, React components
manifest.json        # PWA manifest
apple-touch-icon.png # 180×180 home-screen icon
Images/              # 21 exercise demos, 6.1MB total, largest 666KB
```

The 6.1MB of images never loads up front: the expanded panel is conditionally rendered, so an
image is only fetched when you open that specific exercise.

## Image credits

The animations in `Images/` are **not mine** — they're from
[fitnessprogramer.com](https://fitnessprogramer.com) and carry that site's watermark. They're here
for a personal training log and are not licensed for redistribution. Fork this and you should swap
in your own.

(`Calfraises.jpg` and `Tricep-pushdown.jpg` are WebP files with `.jpg` extensions. Browsers sniff
the real format, so they display fine.)

## Licence

None yet, which means **all rights reserved** by default — readable, not reusable. MIT would be the
sensible fix.
