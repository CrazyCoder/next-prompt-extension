# Editor delegation fix plan — next-prompt

## Goal

Make the decorating ghost editor forward the **full editor API that pi drives on
`this.editor`**, so coexistence with a prior editor owner (pi-powerline-footer)
preserves clipboard paste, input history, and the editor border color.

Single file (`next-prompt.ts`) + regression tests. No behavior change when no
prior editor exists.

## Evidence — the exact contract pi expects

pi 0.85.1 only ever addresses the top editor through `this.editor.<member>`:

```
addToHistory, borderColor, focused, getExpandedText, getText, handleInput,
insertTextAtCursor, onSubmit, setAutocompleteMaxVisible,
setAutocompleteProvider, setPaddingX, setText, setWorkingStatusIndicator
```

`DecoratingGhostEditor` currently forwards: `render`, `handleInput`, `getText`,
`getExpandedText`, `setText`, `getPaddingX`, `setPaddingX`,
`setAutocompleteMaxVisible`, `setAutocompleteProvider`, `invalidate`, and the
callbacks in `FORWARDED_CALLBACKS`.

**Not forwarded: `insertTextAtCursor`, `addToHistory`, `borderColor`.**

## Findings

### E-01 — Ctrl+V paste is dropped (reported bug)

pi's paste handler writes the clipboard bytes to a file, then inserts the path
through the top editor:

```js
const filePath = path.join(tmpdir(), `pi-clipboard-${uuid()}.${ext}`)
fs.writeFileSync(filePath, Buffer.from(image.bytes))
this.editor.insertTextAtCursor?.(filePath)      // ← text fallback uses this too
```

`DecoratingGhostEditor` inherits `Editor.insertTextAtCursor`, which writes the
**decorator's own buffer**. But `getText()` / `render()` / `setText()` all
delegate to `this.prior`, so the decorator's buffer is never read — the path is
silently lost.

**Live evidence:** pi's `wl-paste` read path works and the PNG is written
(`/tmp/pi-clipboard-*.png`, three of them from the reported attempts), yet
nothing appears in the input. The file exists; only the insert is dropped.

**Fix:** `insertTextAtCursor(text) { this.prior.insertTextAtCursor?.(text) }`.

### E-02 — input history entries are dropped

pi calls `this.editor.addToHistory?.(text)` before issuing bash commands,
extension commands, and streaming **steer** messages. The decorator inherits
`Editor.addToHistory`, so those entries land in the decorator's unused history;
the prior's history — the one that actually drives up/down, because key handling
is delegated to the prior — never receives them.

**Fix:** `addToHistory(text) { this.prior.addToHistory?.(text) }`.

### E-03 — editor border color is dropped

`updateEditorBorderColor()` assigns `this.editor.borderColor = <color>` for bash
mode and the thinking level. The decorator's own field is written, but rendering
is delegated to the prior, which keeps its own `borderColor` — so the border
never reflects bash mode / thinking level under the ghost.

**Fix:** forward the `borderColor` property to the prior (get/set accessor via
`Object.defineProperty`, matching the existing `FORWARDED_CALLBACKS` pattern),
with the decorator's original value as fallback.

## Non-finding (documented, out of scope)

`setWorkingStatusIndicator` / `embedWorkingStatus`: pi's `isWorkingStatusEditor`
guard requires `embedWorkingStatus === true`, which the decorator does not set,
so pi returns `false` and falls back to the status widget. No breakage.
Deliberately enabling the inline working status in the border would be an
enhancement, not part of this fix.

## Fix shape

- Extend `PriorEditorLike` with `insertTextAtCursor?`, `addToHistory?`,
  `borderColor?`.
- Add the three forwards in `DecoratingGhostEditor` — two plain methods (no
  `override`, to keep OMP's pinned `CustomEditor` types happy) and one
  `Object.defineProperty` accessor for `borderColor`.
- Single file, no new dependencies, no abstractions.

## Regression tests (failing first, in the style of C15)

Extend the `C15` `DistinctiveEditor` fake prior with recorders for
`insertTextAtCursor`, `addToHistory`, and `borderColor`, then assert:

1. `ed.insertTextAtCursor("/tmp/pi-clipboard-x.png")` reaches the prior and is
   observable through the prior's `getText()` once the fake stores it.
2. `ed.addToHistory("!ls")` reaches the prior.
3. `ed.borderColor = fn` sets `prior.borderColor === fn` (and reading
   `ed.borderColor` returns it).

## Verification

- `bun test` — currently 349/349; must stay green including the new tests.
- `npm run typecheck` and `npm run typecheck:omp` — both green.
- Live smoke in a pi session with pi-powerline-footer loaded:
  PrtScr → Ctrl+V inserts `/tmp/pi-clipboard-*.png`; up-arrow recalls an entry
  added through a bash command; `!` bash mode shifts the border color.
- Record the live smoke line in `TUI_SMOKE_TEST.md` if it gates the release.

## Execution order (each step gated on explicit approval)

1. Add the three failing assertions to C15.
2. Implement the forwards.
3. Run the full suite + both typechecks; live smoke; update the smoke record.

## Execution status (2026-09-12)

1. **EXECUTED** — three failing `C15` assertions added (`insertTextAtCursor`, `addToHistory`, `borderColor`); red before the fix (first assertion failed as predicted).
2. **EXECUTED** — forwards added in `next-prompt.ts` (`PriorEditorLike` + `FORWARDED_PROPERTIES` + two plain methods). `bun test` 350/350, `typecheck`, `typecheck:omp`, `verify:package` all green.
3. **VERIFIED (live)** — extension reloaded from the repo source; PrtScr → Ctrl+V inserted
   `/tmp/pi-clipboard-d0e9e836-…png` (802 KiB, 2095×1231 PNG). E-02/E-03 covered by unit tests;
   the `!`-mode border and up-arrow history checks remain optional eyeballs.

Not committed yet — this lands with the release series (Step 8 of the main plan).
