# next-prompt — next-prompt suggestions for pi and Oh My Pi

A [pi](https://github.com/earendil-works/pi-coding-agent) / [Oh My Pi](https://github.com/oh-my-pi) (OMP)
coding-agent extension that, after an agent turn fully settles and the input editor is empty, computes the single
most logical next instruction you'd type and shows it. Three render modes:

- **`widget`** (default) — a colored below-editor line:
  `↳ next: <suggestion>  (Alt-/ to accept)`
- **`ghost`** — inline greyed ghost text **in the input box** (renders even when the
  editor is unfocused, e.g. after switching tabs/apps)
- **`both`** — inline ghost AND the below-editor widget simultaneously

> **OMP editor-coexistence note:** `ghost`/`both` work on OMP too. OMP has no
> public editor-owner getter, so next-prompt cannot detect another custom-editor
> extension that installed first in the same session — the last installer wins
> there (Pi captures and can restore the prior owner). If the ghost install
> fails, the default editor is restored and the mode falls back to widget.

Accept with **`Alt-/`** (default; configurable) to fill the input box. The suggestion
persists across terminal focus, mouse, and navigation input, and dismisses when editor
text is entered; backspace down to empty re-arms the last suggestion after a short delay
(no new model call). No suggestion while streaming; the suggestion is cleared and any
in-flight model call aborted the instant you submit, start a turn, or the agent starts.

## About this fork

This is a fork of [`@gamaraan/next-prompt`](https://www.npmjs.com/package/@gamaraan/next-prompt)
by gamaraan, tracking upstream and adding one fix: a suggestion over an empty
editor survives terminal focus, mouse and navigation input, instead of being
dismissed when you switch back to the terminal.

Install the upstream package instead if you do not need that.

## What a suggestion is — and is not

A suggestion is **extension output only**: one line next-prompt computes with its
own model call and renders in the input area (`ghost`/`widget`). It is never part of
the coding agent's reply, and the extension never injects it into the conversation —
only accepting it (`Alt-/`) puts the text in the editor, and only the user submits it.

The required shape — the bare imperative the user would type next, not narration or a
status report — describes **this extension's rendered line only**. It is not a style
instruction for a coding agent's own messages, and reading it must never lead an agent
to append prompt-shaped "suggestion" lines to its replies.

## Manual trigger (`autoTrigger: false`)

By default suggestions are computed automatically after every settled turn. Set
`autoTrigger` to `false` to switch to **manual-only** mode: nothing is computed
automatically, and the accept key doubles as the trigger —

1. press **`Alt-/`** once → compute the next-prompt suggestion;
2. press **`Alt-/`** again (once it is shown) → accept it into the input box;
3. press **`Alt-/`** while a suggestion is being generated → ignored (no-op; never
   two concurrent requests).

```json
{ "autoTrigger": false }
```

When `autoTrigger` is `true` (the default), the accept key still triggers a fresh
computation whenever no suggestion is currently showing.

## Install

Pi and OMP auto-discover extensions from standard locations.

### From npm / the pi package gallery (recommended)

The published package is `@jetserge/next-prompt`, published under the npm account `jetserge`:

```bash
pi install npm:@jetserge/next-prompt
```

A specific release can be pinned with:

```bash
pi install npm:@jetserge/next-prompt@0.2.2
```

### OMP

Install through the OMP plugin manager (observed from `omp plugin --help` /
`omp plugin install --dry-run`):

```bash
omp plugin install npm:@jetserge/next-prompt
```

Pin a specific release the same way:

```bash
omp plugin install npm:@jetserge/next-prompt@0.2.2
```

After installing, run `omp plugin doctor` and confirm zero plugin errors. The
package manifest uses the `pi.extensions` form, which OMP accepts directly and
loads with its legacy `@earendil-works/pi-*` import remapping — there is no
separate OMP package.

### From GitHub

Installing from source tracks `main` instead of a release, and clones the
repository on every machine. Prefer npm above unless you want unreleased
changes. The source repository is `CrazyCoder/next-prompt-extension`:

```bash
pi install git:github.com/CrazyCoder/next-prompt-extension
```

To pin a GitHub release or commit, append the tag or commit reference:

```bash
pi install git:github.com/CrazyCoder/next-prompt-extension@v0.2.2
```

### Manual — copy the file

Copy `next-prompt.ts` into your global pi extensions directory:

```bash
cp next-prompt.ts ~/.pi/agent/extensions/next-prompt.ts
```

Or, for a single project only, place it in the project-local extensions directory
(loads after the project is trusted):

```bash
cp next-prompt.ts .pi/extensions/next-prompt.ts
```

### Manual — reference from settings

Add the path to the `extensions` array in `~/.pi/agent/settings.json`:

```json
{
  "extensions": [
    "/absolute/or/relative/path/to/next-prompt.ts"
  ]
}
```

Restart `pi` (or start a new session) after installing.

## Configure

### Interactive: `/next-prompt-config`

Run the `/next-prompt-config` slash command for a guided walkthrough of
**every configurable option except `systemPrompt`** (that one is config-file-only) —
a model picker over all available models (type to search; the list scrolls inside
a window sized to the terminal), render mode, thinking level, accept
key, re-arm delay, transcript/recent-turn/suggestion caps, cross-provider
disclosure, the diagnostic log, and auto-trigger. Every choice opens on its
saved value and an empty answer keeps it, so pressing Enter through the whole
walkthrough changes nothing (the recent-turn cap is removed by typing `all`).
Changes are saved to the host agent dir
(`~/.pi/agent/next-prompt.json` on Pi, `~/.omp/agent/next-prompt.json` on OMP)
and the host reloads so they take effect immediately.

### Config file

All fields optional. Merged global + project (project overrides global **per top-level
key**; the nested `model` block is replaced wholesale, not merged). The config root
comes from the host's `CONFIG_DIR_NAME`:

| Host | Global | Project |
| --- | --- | --- |
| Pi | `~/.pi/agent/next-prompt.json` | `<cwd>/.pi/next-prompt.json` |
| OMP | `~/.omp/agent/next-prompt.json` | `<cwd>/.omp/next-prompt.json` |

```json
{
  "model": { "provider": "ollama", "model": "deepseek-v4-flash:0731-cloud" },
  "thinking": "low",
  "acceptKey": "alt+/",
  "autoTrigger": false,
  "renderMode": "both",
  "rearmDelayMs": 2000,
  "maxTranscriptChars": 12000,
  "maxRecentTurns": 10,
  "maxSuggestionChars": 240,
  "allowCrossProvider": false
}
```

| Field | Default | Notes |
| --- | --- | --- |
| `model` | current model (`ctx.model`) | `{ provider, model, sessionId? }`. If the configured model isn't found, pi notifies once (`warning`) and falls back to the current model. For `opencode`/`opencode-go` the wizard also stores a `sessionId`, sent as the `x-opencode-session` header that gateway requires (it answers `400 MissingSessionID` without one); it is minted once per model and reused, so suggestions keep a stable route across sessions. Other providers ignore it. |
| `thinking` | unset | Reasoning level for the suggestion model: `"minimal"`/`"low"`/`"medium"`/`"high"`/`"xhigh"`/`"max"`. Set `"low"` for faster suggestions. Passed as `reasoning` to the model call. |
| `acceptKey` | `"alt+/"` | Any pi-tui `KeyId` (e.g. `"alt+/"`, `"ctrl+space"`, `"shift+enter"`). Intercepted **before** the base editor, so keys like `ctrl+space` (`\x00`) won't pollute the box. Accept only fires when a suggestion is showing and the autocomplete dropdown is closed. |
| `autoTrigger` | `true` | When `true` (default), suggestions are computed automatically after every settled turn. When `false`, manual-only: the accept key doubles as the manual trigger (first press generates, second press accepts, in-flight press is a no-op). |
| `renderMode` | `"widget"` | `"widget"` (below-editor line), `"ghost"` (inline greyed text in the box), or `"both"` (inline ghost + below-editor widget). On OMP, `ghost`/`both` work too (see the editor-coexistence note above). |
| `rearmDelayMs` | `2000` | Delay (ms) before re-arming the last suggestion after the user deletes back to empty. No new model call. |
| `systemPrompt` | built-in extractor | Config-file only (not prompted by `/next-prompt-config`). See `SYSTEM_PROMPT` in `next-prompt.ts`. |
| `maxTranscriptChars` | `12000` | Tail-truncation of the conversation transcript sent to the model. |
| `maxRecentTurns` | all | Disclosure minimization: only the last N user/assistant turns are sent (tool results are never sent regardless). Invalid values fail closed — suggestions are disabled. |
| `maxSuggestionChars` | `240` | Cap on the returned suggestion length (visible width; a hard code-point bound of 4× this value also applies, so zero-width payloads cannot bypass the cap). |
| `allowCrossProvider` | `false` | When `true`, a configured suggestion model on a **different destination** (provider + endpoint + model route) than the active model may be used — but only after explicit per-project consent (see Security). When `false`, fall back to the active model silently. Project config can never loosen a global `false`. |
| `allowCrossProviderPairs` | `[]` | Directional provider pairs that skip the consent dialog: `[["activeProvider", "suggestionProvider"]]` (e.g. `[["opencode-go", "openai"]]`). Set via the dialog's "Always allow for this provider pair" option (saved to the global config) or by hand. Case-insensitive; the reverse direction is NOT implied. Invalid entries fail closed — suggestions are disabled. |
| `debug` | `false` (absent) | When `true`, appends one JSON line per decision to `<agent dir>/next-prompt-debug.log`: event name, model, transcript/response **sizes**, stop reason, token counts — never transcript or suggestion text. Absent or `false` means no file is written at all. Toggled by the last step of `/next-prompt-config`. |

### Why `alt+/` is the default accept key

- **`tab`** — conflicts with pi's path-autocomplete and `/template` dropdown.
- **`ctrl+tab`** — many terminals send it as plain `tab`/`\t` or swallow it (window/tab switcher), so it's unreliable.
- **`ctrl+space`** — works (sends `\x00`, which pi-tui maps to `ctrl+space`); the extension intercepts it before the base editor so it no longer pollutes, but some terminals remap Ctrl-Space to IME toggle.
- **`alt+/`** — sends an unambiguous `\x1b/` sequence, not bound by pi or most terminals, and is memorable ("accept the suggested next command"). Recommended.

Override with any `KeyId`, e.g. `"acceptKey": "ctrl+space"`.

## Security: cross-provider transcript disclosure

The extension sends the conversation transcript (user + assistant text only — tool
results, thinking blocks, and tool-call arguments are skipped) to the suggestion
model. This is **no more** than what the active model already saw **if** the
suggestion model is on the **same destination** as the active model. A destination
is the provider label **plus** the endpoint origin **plus** the resolved model's
routing id: two different downstream models behind one gateway (e.g. `openai/gpt`
and `openai/claude` at the same `https://gateway.example/v1`) are **different**
destinations. If you configure a suggestion model on a **different** destination,
the transcript is sent to that second destination, which may have different
data-handling terms.

Cross-destination disclosure is **opt-in and fail-closed**:

- `allowCrossProvider` defaults to `false`. With `false`, a configured model on a
different destination is never used; the extension silently falls back to the
active model (and, when there is no active model, computes nothing).
- With `true`, the first time a different destination would receive the transcript,
the host shows a dialog naming the destination and the transcript size, with three
choices: **Allow once (this project)**, **Always allow for this provider pair**, and
**Decline**. "Allow once" persists consent per project + destination (provider +
endpoint + model route) in `<agent dir>/next-prompt-consent.json` (a 0600 file
outside the repository; `~/.pi/agent` on Pi, `~/.omp/agent` on OMP); declining
blocks that destination for the rest of the session without re-prompting. "Always
allow" additionally saves the directional provider pair (`[active provider,
suggestion provider]`) to the global config (via the same atomic 0600 write), so
that exact direction never prompts again in any project — the per-destination
consent record is kept too, so the dialog also stays silent when the config write
is refused.
- Consent is keyed by the full destination identity: changing the endpoint **or**
the model route invalidates a stored grant and prompts again. Records written by
older versions (without a model route) never match and also re-prompt — fail closed.
- Project config (`<cwd>/.pi/next-prompt.json` on Pi, `<cwd>/.omp/next-prompt.json`
  on OMP) is only honored for **trusted** projects, and can never loosen a global
  `allowCrossProvider: false` or increase a global `maxTranscriptChars` cap —
  repository content cannot silently redirect your transcript. Pi gates project
  config on its `isProjectTrusted()` API. **OMP exposes no project-trust API**, so
  OMP follows the configuration loader's default (trusted) and enforces the same
  global privacy floors and consent flow; there is simply no host project-trust
  signal to consult. An existing-but-unreadable or syntactically invalid
  global/project config disables suggestions entirely rather than falling back to
  defaults (both hosts).

Mitigations:

- `buildTranscript` redacts obvious high-entropy secrets (AWS `AKIA…`, OpenAI
  `sk-…`/`sk-proj-…`/`sk-ant-…`, GitHub `ghp_…`/`github_pat_…`, GitLab `glpat-…`,
  Google `AIza…`, Slack `xoxb-…`, JWTs, PEM private key blocks, and `key=value`
  assignment forms) from both user and assistant text before sending. This is
  defense-in-depth, **not** comprehensive secret detection — destination consent
  and least disclosure are the primary controls.
- `maxRecentTurns` minimizes what is sent by limiting the transcript to the last
  N turns.
- Suggestion output is sanitized before rendering: terminal control sequences
  (OSC/CSI/DCS/APC/PM/SOS — both ESC-prefixed and 8-bit forms), C0/C1 controls,
  DEL, carriage returns, bidi overrides, unpaired surrogates, and oversized
  zero-width payloads are stripped or bounded, so model text can never execute
  terminal commands (e.g. OSC 52 clipboard writes).

## How it works

1. On `session_start` (interactive mode only — Pi TUI `ctx.mode === "tui"`, OMP
`ctx.hasUI === true`; headless/RPC/JSON sessions never compute), a global
`ctx.ui.onTerminalInput` listener is registered to detect the accept key
**editor-independently**. `ghost`/`both` install a render-only `GhostEditor` via
`setEditorComponent`. If another extension owns the editor on Pi, the ghost
**decorates** it without a warning and keeps its behavior. An extension that
installs its editor later either wraps the ghost editor, which keeps it live, or
discards it; on Pi only a discarded ghost is installed again, when the next
suggestion is shown. If the ghost fails, the mode falls back to widget. A failed
install restores the owner that was current just before it, or the default
editor if that owner's own editor cannot be built; on OMP, which has no
editor-owner getter, it is always the **default** editor. A ghost that fails
later, while rendering, leaves the editor slot alone, so extensions installed
after it stay in place: the ghost editor stops drawing the ghost and passes
everything else through. An error from the other extension's own render is
passed through unchanged. OMP also has no host-side extension-editor teardown,
so next-prompt resets its editor to default at the next `session_start`. The
host clears extension listeners when the UI is reset; each fresh
`session_start` (reload/new/resume/fork) re-registers exactly one listener and
re-installs the editor once.
2. On completion:
   - **Pi:** `agent_settled` (its fully-settled contract) — if the editor is
     empty, the controller calls
     `ctx.modelRegistry.complete(model, { systemPrompt, messages }, { signal, reasoning })`
     with the resolved model, the configured thinking level, and the (redacted,
     tail-truncated) transcript.
   - **OMP:** only a **terminal** `agent_end` (never when `event.willContinue`
     is true — continuations, automatic retries, and pending continuation turns
     produce no suggestion) — the controller lazily imports `completeSimple`
     from the remapped legacy pi-ai module and calls
     `completeSimple(model, { systemPrompt: [prompt], messages }, { apiKey: modelRegistry.resolver(model), signal, reasoning })`.
     OMP emits the extension `agent_end` **before** the session fully unwinds
     (`ctx.isIdle()` is still false at handler time), so the terminal event
     itself is treated as the settle signal on OMP; Pi keeps its
     `agent_settled` + real-time idle gates. Stale-output protection is
     identical on both hosts (input-generation bumps, aborts, and render-time
     guards). Pi never loads the OMP module; OMP never calls
     `modelRegistry.complete`.
3. The returned text is sanitized (terminal controls stripped, trimmed, de-quoted,
   de-fenced, collapsed to one line, capped) and shown — via `setWidget`
   (widget/both), via the inline ghost overlay (ghost/both), or both.
4. The accept key is intercepted **before** the base editor: if a suggestion is
   showing, it fills the editor via `ctx.ui.setEditorText` and swallows the key.
   Other input is delegated to the base editor and checked afterward: focus, mouse,
   Escape, and arrow sequences leave an empty editor and keep the suggestion visible;
   entered text dismisses it and invalidates any in-flight request. Deleting back to
   empty re-arms the last suggestion after `rearmDelayMs` (no new model call).

## Develop

Clone and run the checks with [Bun](https://bun.sh):

```bash
bun install
bun run typecheck        # Pi API types (default tsconfig.json)
bun run typecheck:omp    # OMP 17.2.13 API types (tsconfig.omp.json)
bun test
bun run verify:package
```

The extension imports `@earendil-works/pi-coding-agent`, `@earendil-works/pi-tui`, and
`@earendil-works/pi-ai`. These are provided by your pi installation — list them in
`peerDependencies` with `"*"` (do not bundle). If your editor's TypeScript LSP can't
resolve them, link them from your pi install's `node_modules` (git-ignored here;
nothing is downloaded). The pinned `@oh-my-pi/*` packages are **dev-only** test
dependencies for `tsconfig.omp.json` — they never appear in `peerDependencies` or
runtime imports; on OMP the legacy `@earendil-works/pi-*` imports are remapped to
the host's bundled packages at load time.

Edit `next-prompt.ts` in place and restart the host (pi or OMP) to pick up changes.

## Compatibility

**Pi:** supported range **0.84.0 – current** (0.84.0 minimum; 0.85.1 validated
live on 2026-09-09).
`ModelRegistry.complete()` — which the extension calls directly — was added in
pi 0.84.0, so older 0.80–0.83 releases are not supported. CI runs the unit suite
and typecheck against both the oldest supported (0.84.0) and the current
(0.85.1) `@earendil-works/pi-*` packages (`.github/workflows/verify.yml` —
`compat` matrix) on every PR and before every tag publish.

**OMP:** supported range **17.2.12 – 17.2.13** (17.2.12 was the researched API
version; 17.2.13 validated live on 2026-09-09). OMP
runs the extension through its legacy `pi.extensions` manifest and
`@earendil-works/pi-*` import remapping — the same published package works on both
hosts. OMP-specific behavior:

- Lifecycle: suggestions compute on a **terminal `agent_end`** only;
  `willContinue: true` events (tool-loop continuations, automatic retries) never
  compute. Pi keeps its `agent_settled` contract.
- Transport: OMP completes via `completeSimple` + the model registry's auth
  resolver; Pi keeps `modelRegistry.complete`.
- Rendering: OMP supports `widget`, `ghost`, and `both`; because OMP has no
  editor-owner getter, a failed ghost install restores the default editor (Pi
  restores the captured prior owner) and another custom-editor extension
  installed first in the same session is not detected.
- Trust: OMP has no project-trust API; project config follows the loader default
  (global privacy floors and consent are unchanged).

CI runs a minimum/current matrix for BOTH hosts (`.github/workflows/verify.yml`,
shared by PRs and tag publishing): pi 0.84.0/0.85.1 typecheck + unit suite;
OMP 17.2.12/17.2.13 `typecheck:omp` + unit suite; the packed artifact is
installed on both hosts (isolated OMP profile + `omp plugin doctor` must report
zero errors; `pi install` + `pi list` must register the extension). Releases
additionally require a dated manual TUI smoke record for the exact version in
`TUI_SMOKE_TEST.md` — the tag cannot publish without it.

## Manual TUI smoke tests

Terminal-rendering and live-interaction behavior cannot be fully verified in unit
tests. Before a release, run the checklist in
[`TUI_SMOKE_TEST.md`](./TUI_SMOKE_TEST.md) in a real pi session AND a real OMP
session, and record the results.

## Design & development

Source and design discussion live in the
[GitHub repository](https://github.com/CrazyCoder/next-prompt-extension).

## License

MIT — see [LICENSE](https://github.com/CrazyCoder/next-prompt-extension/blob/main/LICENSE).
