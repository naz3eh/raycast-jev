# Jev for Raycast

Type a plain-English request into Raycast's root search and have
[Jev](https://docs.typesafe.ai/introduction) (TypeSafe's System One model)
turn it into a concrete action: opening your most recent download, launching
an app, finding a file, or opening a site.

```
⌥Space → "open the pdf i last downloaded" → Enter
```

(That's one Option+Space, since it's bound directly to the command — see
Setup step 4 — then one Enter to confirm the preview.)

## How it works

`Ask Jev` is a normal Raycast command. You reach it one of two ways:

- **Direct hotkey (recommended)** — bind a global hotkey straight to the
  command (Setup step 4). Pressing it opens Jev's search box immediately;
  there's nothing to pick from a list first.
- **Fallback command** — Raycast also lets any eligible command opt into the
  root-search fallback list (shown when a normal search has no results), but
  that's a one-time opt-in *you* grant per command in Raycast's own Settings
  — an extension can't silently register itself there, by design, so it
  doesn't save you a step over the direct hotkey above. Both are covered in
  Setup below; pick whichever fits how you want to invoke it.

Either way, once you're in the command, it receives your query as
`searchText` (seeded from `LaunchProps.fallbackText` when launched via the
fallback route). From there:

1. Jev classifies the request (`open_download` / `open_app` / `open_file` /
   `open_url` / `unsupported`) and, in the same request, speculatively
   answers a few follow-up questions — which file type, which installed app,
   which recent file, which known site — all as TypeSafe `Choice` questions
   over real candidates: installed apps, files actually found under
   `~/Downloads` / `~/Desktop` / `~/Documents`, and a curated site table.
   Jev only ever *selects* from real options; it never invents a file path
   or app name. Those same candidate lists are also handed to the
   classifier itself as `state` (not just the raw sentence) so it can
   recognize e.g. "cursor" as a real installed app rather than guessing
   blind.
2. Plain TypeScript resolves the winning branch into one concrete target
   (exact file, app, or URL) — deterministic lookups (like finding the
   newest download of a given type, spotting an explicit URL in your text,
   or guessing `<word>.com` for an unambiguous brand name not in the site
   table) stay in code, not in the model.
3. A single preview item shows what will happen. Press Enter again to run
   it via Raycast's `open()`.

See `src/lib/typesafe.ts` for the question definitions and `src/lib/run.ts`
for how an answer becomes an action.

## Setup

1. Install dependencies:

   ```sh
   npm install
   ```

2. Get a TypeSafe API key (see <https://docs.typesafe.ai/introduction> if you
   don't have one yet).

3. Run the extension locally:

   ```sh
   npm run dev
   ```

   This opens Raycast and imports the extension. The first time you run the
   `Ask Jev` command, Raycast will prompt for its **TypeSafe API Key**
   preference — paste your key there. (You can also set/change it later via
   Raycast → `Ask Jev` → `⌘,`.)

4. Bind Option+Space directly to the command (one-time, in Raycast Settings
   → **Extensions** → **Jev** → `Ask Jev`): click its hotkey field in the
   list and press Option+Space. This is the step that makes it "just work" —
   after this, Option+Space always opens straight into Jev's search box, no
   root search or fallback list involved. (If Option+Space is already your
   general Raycast hotkey, pick a different combo here, e.g. `⌥J`, so the two
   don't collide.)

   — *or*, if you'd rather it show up as a suggestion in ordinary root
   search instead of its own hotkey: Raycast Settings → **Advanced** →
   **Fallback Commands** → add `Ask Jev`. This is the same kind of one-time,
   user-granted opt-in Raycast requires for any extension's fallback command
   — there's no way for the extension to enable this for itself.

5. Try it: press your hotkey from step 4, type something like `open the pdf
   i last downloaded`, and press Enter on the preview to run it.

## Design notes from debugging real queries

The first version worked for the exact demo query ("open the pdf i last
downloaded") but failed on most other in-scope requests. Three real bugs,
fixed by changing how questions are asked rather than by special-casing
inputs:

- **The classifier judged the sentence in isolation.** `open cursor` came
  back "not sure what you mean" even though Cursor was installed, because
  the `action` question never saw the list of installed apps — it was
  guessing from the words alone. Fix: `installedApps`, `recentFiles`, and
  `knownSites` are now part of the shared `state` for every question in the
  request, and `action`'s instructions explicitly say to cross-check the
  request against them.
- **A hard confidence cutoff rejected valid-but-unfamiliar requests before
  they were even resolved.** Anything under a 0.35 confidence score was
  thrown out regardless of what it resolved to. Fix: removed — the only
  automatic bail-out now is an explicit `unsupported` classification;
  everything else attempts real resolution, and a *specific* failure (no
  matching app/file/site) is what produces the "couldn't tell…" message.
- **File search was recency-only.** Candidates were capped to the 40 most
  recently modified files, so `open my cv` failed whenever the résumé
  hadn't been touched recently — it was never even offered to Jev as an
  option. Fix: `listFileCandidates` in `src/lib/candidates.ts` now scans
  Downloads/Desktop/Documents plus one level of subfolders and ranks by a
  blend of recency *and* filename/query keyword overlap, so an
  older-but-matching file still makes the candidate list.
- **The site table was too small**, so brand names like `facebook` (not
  installed as an app, not in the table) had nowhere to resolve. Fix:
  expanded the curated table, and added a deterministic last-resort guess
  (`<single unambiguous word>.com`) in code for anything still unmatched —
  plain string handling, not model generation.

## Note on `npm run lint`

`ray lint` checks `package.json`'s `author` field against a registered
Raycast Store username (it's currently a placeholder, `"nazeeh"`). This only
matters if you plan to `ray publish` — running the extension locally via
`npm run dev` doesn't need it. If you do want to publish, change `author` in
`package.json` to your real Raycast account username first.

## Extending it

The action set intentionally stays inside TypeSafe's closed-set `Choice`
primitive — Jev classifies and selects, code executes. To add a new action:

- Add a branch to the `action` Choice in `src/lib/typesafe.ts`.
- Build whatever candidate list it needs (or none, if it's a fixed action)
  in `src/lib/candidates.ts`.
- Resolve it to a `ResolvedAction` in `src/lib/run.ts`.

Chained/multi-step actions, arbitrary shell execution, and free-text
generation (e.g. "search the web for…") are deliberately out of scope for
this version.
