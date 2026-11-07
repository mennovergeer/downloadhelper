---
name: downloadhelper
description: >
  Build, install, and iteratively tune a personal, 100% local learning Download
  Helper desktop app tailored to the user's OS, folders, apps, and mail client.
  Use when the user wants a smart assistant for downloaded files that suggests
  filing to client/project folders, opening in preferred apps, or creating email
  drafts with attachments — learning quietly from real actions while keeping all
  data strictly on-device.
---

# Build a learning Download Helper — instructions for a coding agent

You are going to build, install and tune a small desktop app for the user you are working with. Talk to the user
in their own language. Work in short iterations: build something that works, let the user try it, read the log,
improve. Do not write a 1,500-line design up front.

## 1. What the app does (the user-facing contract)

After every completed download, a small always-on-top window appears near a screen corner showing 2–4 suggested
actions for that file. One click executes the action. The suggestions come in two steps:

1. **category**: `move` (to a folder) · `open` (with an application) · `attach` (to a new email draft) · `ignore`
2. **action** within that category: *which folder* / *which application* / *which recipient*

The app quietly watches **what the user actually does first** with each download (within ~30 min) — whether they
clicked a suggestion or not — and stores that as a learning example. Future suggestions are based on those
examples. It gets better by use, without the user configuring anything.

Hard rules (never break these, whatever the user's setup):
- **Never send email.** "Attach" only opens a draft with the file attached; the user presses Send.
- **Everything stays local, always.** No cloud APIs, no telemetry: file names, folder structures, document
  text and email addresses never leave the machine — not even if asked to "make it smarter"; offer a better
  local model instead. Store only file name, extension, size, a coarse "kind", source host (domain only, no
  URL), timestamp, the chosen action, and — if document text is used — an extracted short label such as a
  customer name. Never store document contents or full URLs.
- The window is **only visible after a download**, disappears after a configurable time (default 30 s; the
  countdown pauses while the mouse is over it), never steals keyboard focus.
- Learn only from the **first** action per download. Later moves/opens of the same file are not examples.
- Never delete user files. A move into a folder that does not exist yet is only done after the user picks it.

## 2. Phase 0 — interview the user (short, before writing code)

Ask at most these, in one message, and give sensible defaults so the user can just say "ok":
1. OS and version; do they have a GPU (which); Python allowed? (Default: Python 3.11+, tkinter GUI.)
2. Where are downloads saved? Which browsers?
3. Where do files usually go: cloud-sync folders (OneDrive, Google Drive, Dropbox, iCloud, Nextcloud), project
   or client folders? Ask for the 1–3 most important root folders.
4. Which mail client (Outlook desktop, Apple Mail, Thunderbird, webmail only → then `attach` is disabled or
   falls back to opening a `mailto:` + revealing the file).
5. Is there an existing archive of already-filed files (e.g. `Clients/<name>/…`) that can be used to pre-train?
6. Do they want auto-execution later (with undo) once the app is confident? (Default: off until it has earned it.)

Then check the machine yourself (paths, installed apps, cloud folders, Python, GPU) instead of asking more.

## 3. Architecture

```
watcher ──download event──▶ classifier ──suggestions──▶ window ──click──▶ executor
   ▲                           │  ▲                                        │
   │                           ▼  │                                        ▼
observer (moves/opens/mail) ─▶ examples store (JSON, rolling, ~600) ◀── "first action" recorder
```

One process, a few threads, a single-instance lock, a plain-text log file you (the agent) will read during tuning.
Keep the decision backend behind one function so it can be swapped:

```python
def choose(state: dict, question: str, options: list[str]) -> dict[str, float]:  # probabilities per option
```

### 3.1 Decision backend — pick by hardware, in this order of preference
- **No model at all (always implement this first; it is also the fallback):** vote counting over similar
  examples (see 3.4). This alone gets most of the accuracy once there are ~10 examples.
- **Small local choice/classification model** if available (e.g. a GLiDE-style server that scores a fixed list of
  options, or a local LLM through llama.cpp / Ollama scoring each option by log-probability). A 0.4–3 B model is
  enough; on CPU keep the prompt short (similar examples only, ≤15 options) and aim for < 2 s. The model
  server must listen on localhost only. If the machine is too slow for any model, stay with vote counting.
Blend model and votes: `p = (1-w)·p_model + w·p_votes`, with `w = min(0.7, n/(n+3))` where n = number of similar
examples. Votes must dominate once there is a clear history — small models cannot count.

### 3.2 Detecting downloads (fast and reliable)
- Watch the downloads folder with native events (`watchdog` works on all three OSes; on Windows
  `ReadDirectoryChangesW` directly is fine). Treat **rename from a temp name** (`.crdownload`, `.part`,
  `.download`, `.tmp`, Safari `.download` bundles) to the final name as "download complete". This also catches
  downloads that **overwrite** an existing file — a pure "new file appeared" check misses those.
- Keep a slow polling scan as a backup. Wait until the size is stable. Ignore hidden/temp/zero-byte files.
- Source host: Windows `Zone.Identifier` ADS (`HostUrl`/`ReferrerUrl`), macOS xattr
  `com.apple.metadata:kMDItemWhereFroms`, Linux xattr `user.xdg.origin.url` (Chromium) — keep only the domain.
  Keep meaningful bucket/tenant names (e.g. `<bucket> (Amazon S3)`).

### 3.3 Observing what the user does (the learning signal)
Watch for ~30 min after each download; the first matching event wins.
- **Moves**: native file events on the user's home folder *and* every cloud-sync root (they may be on another
  drive or a junction/symlink — resolve them). Match by file name + size, also when renamed in the same step.
- **Opens**: this is the hard one per OS.
  - Windows: process creation events (WMI `__InstanceCreationEvent WITHIN 1` + a fast poll), check command lines
    for the file path.
  - macOS: `NSWorkspace` notifications / `lsof` polling on the file, or `kMDItemLastUsedDate` changes.
  - Linux: inotify `IN_OPEN`/`IN_ACCESS` on the file plus `/proc/*/fd` or `/proc/*/cmdline` to name the process.
  - Ignore your own process, shells, script interpreters, file managers, antivirus/indexers, thumbnailers and the
    browser itself (browser "open" happens before the download is complete). Do **not** use "recent files"
    lists — they produce false positives from file pickers and previews.
- **Attach**: inspect the mail client's open drafts and recently sent items for an attachment with the same name
  (Outlook: COM `Inspectors`, `ActiveInlineResponse`, Sent Items — call `CoInitialize` in each thread; Apple Mail:
  AppleScript; Thunderbird: hard — skip unless the user needs it). Store the recipient address.
- When the user clicks a suggestion, record it as the example immediately ("clicked").
- An **undo** of an auto-action is stored as a negative marker that breaks the streak (see 3.7).

### 3.4 Examples and similarity (this is where the quality comes from)
Example record: `name, ext, kind, size, source_host, client?, category, target, how (clicked|seen|archive|undone), time`.

To build the prompt/votes for a new download, select the most similar examples, not the most recent ones:
score = 2·same ext + 1·same kind + 2·same source host + 1 per shared meaningful name token; keep score ≥ 2, top ~30.
**Do not restrict to the same extension** — a user who opens every PDF in tool X and every DOCX in the same tool
should get X for an ODT too. `kind` (document, image, calendar, archive, installer, spreadsheet, …) carries that.

Also give the backend a one-line vote summary ("similar files: 7× move to A, 2× open with B").

### 3.5 Candidate lists (two-step keeps them small)
Never show the model hundreds of options, and **never collapse to one option** after learning — always keep a
range so the user can correct it.
- Folders: learned target folders + their **siblings when the parent is a collection** (e.g. ≥ 8 subfolders like
  `Clients/*`: if one client folder was learned, all client folders are candidates) + the OS's recent/pinned
  folders + the configured roots. Exclude generic ancestors (don't offer `Documents` when `Documents/Clients/X`
  is the real pattern). Cache the list and rebuild it when examples change.
- Apps: learned apps + the OS default for the extension + the system's "open with" list for that extension
  (Windows `OpenWithList/OpenWithProgids` + Store app aliases; macOS LaunchServices; Linux `mimeapps.list`/
  `.desktop` files), ~15 max, with friendly names. Open the default app via the OS's own mechanism
  (`os.startfile`, `open`, `xdg-open`) so special verbs work (e.g. calendar files in Outlook need `/ical`).
- Recipients: learned recipients + a short configured list.
Show labels shortened to the last 2 path parts; the window grows to fit all options.

### 3.6 The window (looks matter — users judge the app by it)
Build it as a small **card**, not a default dialog: borderless, always on top, bottom-right above the taskbar,
rounded corners and a thin border (Windows 11: `DwmSetWindowAttribute` corner preference + border colour; macOS/Linux:
a borderless window with a rounded background). Use the platform's UI font and icon font (Windows: Segoe UI
Variable + Segoe Fluent Icons; macOS: SF Pro/SF Symbols; Linux: the desktop's font + a symbolic icon theme).
Draw shapes anti-aliased (in tkinter: render rounded rectangles, icons and rings with PIL at 4× and downscale;
let the toolkit draw the text so it stays crisp).
- **Header**: a coloured file-type badge (PDF red, Word blue, Excel green, PowerPoint orange, calendar teal,
  other kinds by category), the file name (middle-ellipsis if long), and a grey meta line "PDF · 182 KB · from
  <host>". A small × on the right = ignore.
- **Suggestion rows** (one per option, the best one first): a rounded row with a coloured icon tile per category
  (folder amber, open blue, email violet, new folder green), a small grey verb ("Move to", "Open with",
  "Email to (draft)", "Create new folder"), the target in semibold (paths left-ellipsised), and on the right the
  percentage with a thin bar. The best row is highlighted (accent outline + tinted background). Hover highlight
  on every row; the whole row is the click target.
- **Footer**: a **countdown ring** on the left that drains smoothly (update ~20×/s; grey with a pause symbol
  while the mouse is over the card) — no "disappears in N s" text; only show text for extra info such as
  "2 waiting". Pill buttons "Other…" and "Ignore" on the right.
- **Messages** (done, started, error, auto-done with an accent "Undo" button) reuse the same card with an
  icon tile instead of the file badge.
- **Right-click menu**: "Visible after download ›" (10–120 s), "Language ›" (at least the user's language and
  English), "Quit". Settings persist in the config file immediately; a visible card re-renders in the new
  language and restarts its countdown with the new time. All UI strings go through one translation table.
- **Preview/demo mode**: `--preview <lang> <state>` shows the card with made-up data (no file access, no
  learning) for screenshots, demos and design iterations. Use it to check every state (suggestions, paused,
  done, auto-done, error) visually before showing the user — take screenshots and look at them.

### 3.7 Optional, add when the basics work
- **Document text** (PDF first pages, DOCX/PPTX/XLSX, text): use it in the *action* step only (not for the
  category — that hurts latency and accuracy). Typical win: put a signed quote/invoice in the right client folder.
  1. Look at 10–20 already-filed documents in the user's archive and **derive regexes for where the customer
     name is printed** (e.g. the line after "To:"/"Aan:", or after a fixed sentence in their invoicing tool's
     template). Make them configurable (`client_patterns`). Measure on the archive and show the user the hit rate.
  2. Match the extracted name to folder names with a score: all words → 1.0, acronym/initials → 0.95,
     common abbreviations (e.g. "Hospital"→"H") and word permutations lower, otherwise token overlap.
     Strip legal forms (B.V., Ltd, GmbH, Inc., Foundation…). Reject the user's own name/company, salutations,
     anything with digits or `@`.
  3. Learn aliases: store the extracted `client` on each example so "UMC Utrecht" → folder "UMCU" is learned.
  4. If the name matches no folder, offer **"➕ New folder: <name>"** as an option (created only on click).
  5. Read encrypted/signed PDFs too (`pypdf` + `cryptography`). Scanned PDFs without text: skip (OCR is optional).
- **Pre-training from the archive**: a `--seed FOLDER PATTERN` command that turns already-filed files into
  `archive` examples (with extracted client). Archive examples count as votes but never trigger auto-mode.
- **"Other…" button**: folder picker / "open with…" submenu / other address — the result is learned like a click.
- **Auto mode with undo**: only after ≥ 3 consecutive identical *real* (non-archive) choices for similar files and
  high probability. Never for `attach`, never for new folders. Show a 15 s toast with "Undo" and "Stop doing this
  automatically". Undo restores the file and records a negative marker.
- Desktop shortcut / launcher, autostart at login, second launch offers to quit the running instance.

## 4. Quality bar and how to test (do this yourself before asking the user)
- **Latency** from download complete to window visible: target < 1 s (< 2 s with document text on CPU). Measure
  and log it. Tricks that worked: event-driven detection, similar-only history, ≤ 15 options, short paths,
  no document text in the category step, cached folder list, skip the second category question when the first
  is confident (≥ 0.6).
- **Replay tests** you can run without the user: create a fake download (temp name → rename), simulate a
  move/open, assert an example was stored with the right fields; check overwrite detection; check that a
  second action is not learned; check the window appears and disappears.
- **Leave-one-out on the archive** (if seeded): for each filed document, learn from all others and predict its
  folder. Report "N of M correct" to the user, and list the misses with the reason. Typical honest misses:
  inconsistent filing in the archive, unreadable scans.
- Keep a self-test command (`--selftest`) that checks backend reachability, watchers, mail client access.

## 5. Iteration protocol with the user
1. **MVP (day 1):** watcher + window + vote-only suggestions + executor + learning from clicks and moves.
   Install it, start it, ask the user to download 3–5 real files and act normally.
2. **Read the log yourself** after each session. For every download: what was suggested, what the user did,
   was it learned, latency. Report briefly: "5 downloads, 4 learned, 1 missed because …; fix: …".
3. Ask the user narrow questions only when the log can't answer them ("You opened X in Paint after the window
   closed — should that count?"). When the user says a suggestion was wrong, find out *why* from the data
   (wrong candidates? too few examples? a generic folder winning?) before changing weights.
4. Add the optional features from 3.7 one at a time, each with its own measurement.
5. Before changing stored data (deleting examples, reseeding), back it up and tell the user.

## 6. Pitfalls already learned the hard way
- Recent-files lists, browser previews and file pickers create fake "opened with" examples → process-based only.
- Your own test commands (python, shells) become examples unless ignored.
- Caching folder/app lists without invalidation makes newly learned targets invisible.
- Store app / sandboxed app launchers (Windows Store, Flatpak, macOS app bundles) need their own launch path.
- Long labels push options off-screen → shorten and auto-size.
- Default toolkit widgets (grey buttons, title bar, jagged canvas circles) look dated → custom-drawn card, see 3.6.
- When screenshotting your own window for checks or demos, crop exactly to the window rect: anything around
  it is the user's private screen.
- A stale quit flag or lock file from a crash can kill the next start → clean up on start.
- COM/AppleScript calls from the GUI thread may need per-thread initialisation.
- Cloud folders are often on other drives, symlinks or junctions → watch the resolved paths.
- Probabilities of a small model over a long history are not counts → always blend with real vote counts.
