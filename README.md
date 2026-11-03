# WA-web-clean

> A Tampermonkey userscript that makes WhatsApp Web look like *MY* WhatsApp Web.
> Built by someone who learned CSS by right-clicking "Inspect" and whispering "what the hell is `x1c4vz4f`."

---

## What is this

A single userscript that:

- **Hides the left icon rail** so the app doesn't look like a control panel from 2011.
- **Replaces the intro screen** with an auto-rotating wallpaper slideshow pulled from this repo.
- **Hides in-chat call bubbles** (`Missed voice call`, `Missed video call`, "Click to call back") so your chat isn't a graveyard of missed calls.
- **Removes the "This business won't see if you read their messages" banner** (Block / Stay) — gone, permanently, via CSS.
- **Cleans up the right-click context menu** (bye, "Add to contacts", "Pin chat", "Add to list", "Add to Favorites", "Block", "Clear chat").
- **Fixes the giant left gap** in the chat panel because WhatsApp decided padding is a personality.
- **Sizes the intro wallpaper properly**: never upscales (no blur), never crops, never distorts. Small images sit at native resolution, large images scale down to fit, and the blurred layer behind fills the rest. Works for portrait, landscape, ultrawide, and whatever cursed aspect ratio you throw at it.

Everything is client-side. No servers, no analytics, no "telemetry" with a wink. The only network calls are to GitHub and jsDelivr to fetch the wallpaper list and image bytes.

---
## Before & After

| Before | After | Notes |
|:------:|:-----:|:-----:|
| <img width="1234" height="684" alt="side bar original" src="https://github.com/user-attachments/assets/4e660476-d9df-4bf9-aaf3-f2a2f9612c16"/> | <img width="1280" height="720" alt="side bar gon" src="https://github.com/user-attachments/assets/c5912651-9764-4490-a677-7ab15b6a1668" /> | Sidebar hidden, intro panel removed |
| <img width="1280" height="720" alt="original call in chat" src="https://github.com/user-attachments/assets/ffc15df9-36c4-4329-9ba4-a8827f7aea51" /> | <img width="1280" height="720" alt="call cleaned" src="https://github.com/user-attachments/assets/761b8d3f-7d22-4bd5-8a3c-b503d2dbcf0d" /> | in-chat Call bubbles gone |
| <img width="1280" height="720" alt="context orginal" src="https://github.com/user-attachments/assets/9288d11d-187c-4c39-bf33-3f9824e1374d" /> | <img width="1280" height="720" alt="context cleaned" src="https://github.com/user-attachments/assets/21a595ca-b054-427d-b5a5-0e75cc3899f5" /> | context menu cleaned |
| no screenshot | too lazy to get them | block/add banner removed |

---
## How it works (the 30-second version)

1. **Listing** — asks the GitHub API for every image in this repo (`?recursive=1`). Falls back to jsDelivr if GitHub rate-limits you. Caches the result in `GM_setValue`.
2. **Cache** — downloads image bytes once, stores them in **IndexedDB** as blobs. Every subsequent show is a zero-network read.
3. **Deck** — shuffles all image paths into a deck and draws one per rotation. Every image shows exactly once per cycle, then reshuffles. No repeats, no "random" that keeps showing the same three wallpapers.
4. **Sizing** — measures each image's natural dimensions once (cached alongside the blob), reads the intro panel's `clientWidth`/`clientHeight`, computes `scale = min(1, panelW/natW, panelH/natH)`, and injects the exact pixel size into the `::after` background. `::before` stays `cover` + `blur(28px)` so the empty space around a small image looks intentional, not broken.
5. **Rotation triggers** — tab becomes visible again, or the intro panel remounts (closing a chat). 10-second cooldown so it doesn't strobe.
6. **Self-healing** — 10 seconds after boot, it re-fetches the file list, prunes the deck of deleted paths, and deletes IndexedDB records for images that were removed or changed upstream. LRU eviction keeps the cache at ≤40 entries / ≤30 MB.

---

## Does this eat my CPU / RAM?

No. Here's what the script actually does to your machine, and why it's cheap:

**CPU — basically idle.**
- The only work that happens "often" is the rotation itself: at most **once every 10 seconds** (there's a hard cooldown), and only when a trigger fires (tab becomes visible, or the intro panel re-appears). If you sit on one tab and don't touch anything, it does **nothing**.
- Each rotation = read one image blob from cache (or fetch it once), set a CSS background, fade it in. That's a handful of milliseconds *per swap*, not per frame. There is no animation loop, no `requestAnimationFrame`, no polling, no timers ticking in the background except the sparse rotation cooldown and a 5-second "keep my CSS alive" check that does nothing but re-attach a stylesheet object.
- No continuous processing. When you're not looking at the intro screen, the script is asleep.

**RAM — a few megabytes, bounded on purpose.**
- The image cache lives in **IndexedDB (disk), not RAM**. Images are only held in memory while the one currently on screen is decoded for display — one image at a time.
- The cache itself is hard-capped: **max 40 images or 30 MB total**, least-recently-used evicted first. So it can't grow forever, and it can't bloat your browser's memory over a long session.
- Old blob URLs are revoked shortly after the fade completes, so they don't pile up.

**Network — only on a cache miss.**
- Steady state: **zero requests.** The list is cached in Tampermonkey storage; image bytes are cached in IndexedDB. Nothing hits the network unless a brand-new image is drawn.
- On a new image: one request to GitHub raw (or jsDelivr) for the bytes, once, then cached.
- A single background check ~10 seconds after boot refreshes the file list and prunes anything deleted — that's it. No polling, no heartbeat, no analytics.

**Disk — one small cache, capped.**
- IndexedDB holds the cached images, ≤30 MB, auto-pruned. Deleting the script or clearing site data wipes it entirely.

**The real cost is your wallpaper files, not the script.**
- The script's own overhead is rounding-error small. If the slideshow feels heavy, it's because you're feeding it 40 MB PNGs, not because the code is doing heavy lifting. JPEG/WebP keeps everything instant.

---

## Installation

1. Install [Tampermonkey](https://www.tampermonkey.net/) (or Violentmonkey).
2. Create a new script, paste the contents of the `.user.js` file from this repo.
3. Open [web.whatsapp.com](https://web.whatsapp.com).
4. Enjoy a WhatsApp Web that doesn't look like it was designed by committee.

If you want your own wallpapers: fork this repo, drop images into a folder, edit the three constants at the top (`GH_USER`, `GH_REPO`, `BRANCH`), done. Subfolders are fully supported — the listing and URL builders handle nested paths and URL-encoded folder names.

---

## Adding images

Supported: `.png`, `.jpg`, `.jpeg`, `.webp`, `.avif`, `.bmp`. **No GIFs** (the deck and cache assume static blobs; GIFs would work but the slideshow would look like a rave).

Drop them anywhere in the repo — root or subfolder, both work. The script walks the tree recursively.

---

## A note on the wallpapers (and the DMCA-shaped elephant)

The images in this repo were pulled from the internet — wallpapers, screenshots, things that looked nice at 2 a.m. **This repo is not monetized.** No ads, no donations, no Patreon, no "buy me a coffee," no sketchy redirect links. It's a personal customization script and a folder of wallpapers that make it look less like a corporate chat app.

That said, **if you own an image in here and you'd rather it not be**, open an issue or email, and it'll *probably* be removed — no drama, no lawyers, no "please contact our legal department" autoresponder. Just note that I'm not glued to GitHub notifications, so give it a minute.

If you're reusing this script with your own images, same courtesy applies to you — credit or replace what isn't yours, and don't monetize a wallpaper folder and pretend it's a product.

The **code** is yours to do whatever with. The **images** are "best effort, remove on request, whenever I see the message."

---

## Known quirks

- **First boot after moving files** — the old cached image may paint briefly (IndexedDB doesn't know the paths changed until the 10 s revalidation prunes it). Then it snaps to the new deck. Harmless.
- **Window resize** — the intro panel isn't user-resizable in practice, and WhatsApp re-lays-out on resize anyway. If a stale fit ever appears, closing and reopening a chat triggers a re-fit. (A resize listener could be added; it wasn't, because it wasn't needed.)
- **`:has()` support** — the CSS relies on it. All current browsers have it. If you're on a browser from 2022, this script is not for you, and neither is most of the modern web.
- **WhatsApp updates its hashed class names constantly.** The selectors here lean on stable hooks (`data-testid`, `aria-label`, SVG `<title>`) for exactly that reason. If WhatsApp renames `not-spam-btn`, the banner hide breaks until someone updates the selector. That's the deal with userscripts.

---

## Disclaimer (the boring but necessary part)

This script modifies the DOM of WhatsApp Web in your own browser. It does not:
- send your messages anywhere,
- read your chats,
- exfiltrate your contacts,
- phone home,
- collect analytics,
- do anything a normal userscript wouldn't.

It **does** talk to GitHub and jsDelivr to fetch wallpapers. That's it. If that sentence sounds suspicious, read the source — it's one file, ~500 lines, and the network calls are all `gmFetch` to `api.github.com`, `raw.githubusercontent.com`, `data.jsdelivr.com`, and `cdn.jsdelivr.net`.

Use at your own risk. WhatsApp can change their DOM tomorrow and break everything. That's not a bug, that's userscripting.

---

## Credits

- **Code**: assembled by someone whose entire development process is copy, paste, refresh, and pray. The AI did the heavy lifting; I just said "bruh" until it compiled.
- **Wallpapers**: the internet, in all its uncredited glory. See the DMCA note above.
- **Inspiration**: free AI labor and the fact that WhatsApp Web's default look offends the eyes.
