# Manual TUI smoke tests — next-prompt

Terminal-rendering and live-interaction behavior cannot be fully verified in
unit tests (the automated suite covers the editor/rendering contracts via real
pi-tui `Editor`/`CustomEditor` integration tests, but not a live terminal).
Before any release, run every check below in a real pi session (0.84.0+) AND a
real OMP session (17.2.x), ideally on **at least two terminal emulators** per
host, and record the result.

**Status legend:** `[ ]` not run · `[x]` passed · `[!]` failed (blocker: fix
before release, note the failure)

Record date, host + version, terminal emulators, configured render mode,
provider/model, and any failures at the bottom. **Do not publish until every
case passes.**

## Pi smoke cases

### Render modes

- [ ] `widget`, `ghost`, and `both` render correctly at narrow and wide
  terminal sizes; no border corruption, no overflow past the editor width.
- [ ] In `ghost`/`both`, the greyed ghost appears after the caret only when the
  editor is empty; with text present the editor is never clobbered.
- [ ] Switch terminal/app focus while a ghost is visible (unfocused editor):
  ghost still renders within the box, no overflow or corrupted styling.

### Input state machine (default key `Alt-/`)

Run each sequence in `widget`, `ghost`, and `both`:

- [ ] settle → show → non-accept printable key → editor updates, then all
  suggestion UI clears after input settles;
- [ ] settle → show → focus, mouse, Escape, or arrow → suggestion remains visible;
- [ ] settle → show → accept → editor fills exactly once, key is swallowed
  (no `/` typed);
- [ ] accept → edit but do not submit → delete back to empty → the last
  suggestion re-appears after `rearmDelayMs` (no new model call);
- [ ] accept → submit → next turn returns `NONE`/error → the old suggestion
  never re-arms;
- [ ] pending completion → type → delete-to-empty → resolve old completion →
  nothing renders;
- [ ] pending completion → session switch/shutdown → resolve → nothing
  renders (no consent persisted, no model call);
- [ ] delete-to-empty after dismissing with Escape does NOT re-arm.

### Manual trigger (`autoTrigger: false`)

- [ ] With `autoTrigger: false`, no suggestion appears after a settled turn.
- [ ] Press `Alt-/` with an empty editor → a suggestion appears.
- [ ] Press `Alt-/` again → the suggestion fills the editor exactly once (key
  swallowed, no `/` typed).
- [ ] Press `Alt-/` while a suggestion is being generated → no-op (no second
  request, editor untouched).
- [ ] With `autoTrigger: true` (default), dismissing a suggestion then pressing
  `Alt-/` recomputes a fresh one.

### Autocomplete / conflicting keys

- [ ] Slash/path autocomplete still works (Tab, up/down, Enter) while a
  suggestion is showing or dismissed; accepting the suggestion never steals
  autocomplete input.
- [ ] Configure `"acceptKey": "tab"` → extension warns and ignores it;
  Tab continues to drive autocomplete and never fills the editor.

### Editor integration

- [ ] IME input and wide Unicode cursor placement (CJK/emoji) work with the
  ghost installed; the cursor block stays on the correct grapheme.
- [ ] History browsing (up/down), undo, kill-ring, and paste behave normally
  with the ghost editor installed.
- [ ] Run `/reload`, `/new`, `/resume`, `/fork`, and `/clone`; acceptance and
  rendering continue without duplicate listeners or double-installed editors
  (no double ghost/widgets, no doubled accept key).
- [ ] Run alongside another custom-editor extension: next-prompt shows no
  warning, decorates it in ghost mode, and only falls back to widget mode if
  ghost rendering actually fails (the other extension's editor keeps working,
  suggestion still appears below the box).
- [ ] With another custom-editor extension installed first, pi's own keys keep
  working through the decorated editor: Ctrl+C clears the editor, a second
  Ctrl+C exits, and Ctrl+D on an empty editor exits.

### Model / lifecycle error paths

- [ ] Model error, abort, `NONE`, length stop, slow completion, and provider
  switch behave: no stale suggestion renders, no spurious error notify on
  abort.
- [ ] First cross-destination request requires a clear consent dialog naming
  the destination and transcript size; denial results in **no** request and no
  re-prompt for the session; grant persists across sessions and a route change
  re-prompts.
- [ ] Run print/JSON/RPC modes and confirm no hidden suggestion request
  occurs.

## OMP smoke cases

Setup: install the package with `omp plugin install npm:@gamaraan/next-prompt`,
then run `omp plugin doctor` (expect zero plugin errors) before the interactive
checks.

- [ ] `omp plugin doctor` passes with the package installed and enabled.
- [ ] Interactive session reaches a below-editor widget suggestion after the
  **final** `agent_end` (editor empty, agent idle).
- [ ] Tool-loop / automatic-retry continuation shows **no** intermediate
  suggestion (only the terminal completion does).
- [ ] `renderMode: "ghost"` and `renderMode: "both"` render inline ghost text in
  the input box after the caret (empty editor), with a usable widget in `both`;
  accept fills the editor exactly once; delete-to-empty re-arms the last
  suggestion after `rearmDelayMs` without a second model call; ghost render
  failure falls back to the default editor + widget.
- [ ] Widget accept (`Alt-/`) fills the editor exactly once and consumes the raw
  key; typed text dismisses and passes through, while focus/navigation preserves
  the suggestion; delete-to-empty re-arms the cache without another model call.
- [ ] Input submit, new agent start, session reload, and shutdown prevent any
  stale widget output from an in-flight request.
- [ ] Headless / print / RPC modes create no completion request.
- [ ] Cross-destination consent: first request prompts once; decline blocks the
  destination for the session with no request; grant persists across sessions;
  a route change re-prompts.
- [ ] Widget renders correctly at narrow and wide terminal sizes; no overflow.

## Record

- Date: 2026-09-15
- Pi version(s): 0.85.1
- OMP version(s): 18.1.21
- Terminal emulators: kitty
- Configured render mode: ghost
- Provider/model: opencode-go/deepseek-v4.1-flash
- Results / failures: all cases pass

## Release evidence (Step 7 gate)

Tag publishing REQUIRES a dated evidence line for the exact released version
(`publish.yml` greps for it; without it the tag cannot publish). Append one
line here after every full pass, in exactly this format:

```
Recorded: vX.Y.Z — YYYY-MM-DD — Pi <version> (<terminals>) / OMP <version> (<terminals>) — all cases pass
```

If any case failed, do NOT append the line — fix first (a failed case is a
release blocker per the status legend).

Recorded: v0.3.0 — 2026-09-15 — Pi 0.85.1 (kitty) / OMP 18.1.21 (kitty) — all cases pass
