# Engine capability contract: offer Codex and Gemini without losing the office's walls

Status: proposal. Builds on issue #48 (a provider neutral policy compiler with an enforcement
matrix and conformance tests), issue #139 (a typed `isolation` capability on each preset, closed
as a duplicate of #48), PR #125 (`test/office-privacy-conformance.test.cjs`) and TODOS.md,
Engines, "Make a second engine (Codex first) ready for an office" (gaps 3, 6 and 9).

#48 and #139 already propose a capability record per engine, a member limited where its engine
can't hold a rule (with the reason on the Access tab), and a conformance bar before an engine
joins `BUILD_ENGINES`. This adds:

- mail and background judgments as fields of that record, beside isolation, so the three
  Claude only branches (`index.ts:4201`, `hive.ts:1426`, `runHiddenClaude`) become one lookup;
- a `platform` override in the record, so Windows `deny-only` and Linux are data, not prose;
- one pure `effectiveAccess(grants, enforcement, platform)` read by both spawn code and the
  Access tab, so the screen cannot show more walls than the member has;
- starting values limited to what the code proves today.

## The problem

`BUILD_ENGINES` in `src/shared/agentProvider.ts` says that bringing an engine back means adding its
id there and nothing else. The preset half of that is true: `test/engine-readiness.test.cjs`
already checks Codex and Gemini CLI. The runtime half is not. Three promises the README makes to
an owner are wired for Claude Code only:

- **Mail.** `src/main/index.ts:4201` passes the md-mail server only when `claudeProvider`. A Codex
  member given a mailbox starts with no mail tools, while its Access tab still shows email on.
- **Folder privacy and connector deny.** `src/main/hive.ts:1426` returns before `hookSettings` for
  every engine but Claude. Codex keeps its workspace-write sandbox and `--add-dir`, so its writes
  are walled, but nothing stops it reading a teammate's folder. The README says a team member
  "opens only its own folder".
- **Background judgments.** `memoryTidy.ts`, `standingCheck.ts`, `hireCheck.ts` and
  `workStyleConvert.ts` all call `runHiddenClaude`. On an office with no Claude, every email under
  a standing approval falls back to Needs you and memory never tidies.

Adding `'codex'` to `BUILD_ENGINES` today fails only `test/build-engines-everywhere.test.cjs:21`,
which pins the list to `['claude']`. Update that pin and nothing checks mail, folder privacy or
background calls for the new engine, so those promises break quietly. The owner this product is
for never opens a terminal, so they would never see it.

## The proposal

Each engine declares what it can enforce, in one typed record. The app checks that record in
three places: when an engine is admitted to the build, when a member starts, and on the Access tab.

### 1. The record

In `src/shared/agentProvider.ts`, on each preset:

```ts
export type Isolation =
  | 'os-sandbox+deny'   // shell walled by the OS, file tools held by deny rules (Claude on macOS)
  | 'deny-only'         // file tools held, shell not walled (Claude on Windows today)
  | 'write-sandbox'     // writes walled, reads open (Codex today)
  | 'none';

export interface EngineEnforcement {
  isolation: Isolation;
  mail: 'mcp-config' | 'codex-config-toml' | 'none';
  connectorDeny: boolean;
  backgroundRunner: boolean;
  /** Where a platform differs from the default above. */
  platform?: Partial<Record<'darwin' | 'win32' | 'linux', Partial<EngineEnforcement>>>;
}
```

Starting values are only what the code proves today:

| Engine      | isolation                                | mail       | connectorDeny | backgroundRunner |
| ----------- | ---------------------------------------- | ---------- | ------------- | ---------------- |
| Claude Code | os-sandbox+deny (win32: deny-only)       | mcp-config | yes           | yes              |
| Codex       | write-sandbox                            | none       | no            | no               |
| Gemini CLI  | none (until a test proves otherwise)     | none       | no            | no               |

### 2. Admission: `BUILD_ENGINES` refuses an engine that has not proved its record

Extend `test/engine-readiness.test.cjs`: every engine in `BUILD_ENGINES` must declare
`EngineEnforcement`, and each declared level must be checked against the settings and hook
decisions the app generates for that engine, extending `test/office-privacy-conformance.test.cjs`
(no live agent, so it runs in CI, as asked on #48): a read of another member's folder, a write
outside its own, a denied connector call and, when `mail` is not `none`, a gated mail call. It
passes only when each is blocked or the record says the engine cannot hold it.

### 3. Start: one pure function decides what a member actually gets

`src/shared/effectiveAccess.ts`:

```ts
effectiveAccess(grants, enforcement, platform) => { granted, withheld: { what, reason }[] }
```

- `mail: 'none'` → email off for that member.
- `connectorDeny: false` → every connector off for that member.
- isolation weaker than the member's folder needs → that member is not started on this engine.
- `backgroundRunner: false` → the office's background calls use another engine in the office
  whose `backgroundRunner` is true, else their existing fallback.

Spawn code in `index.ts` and `hive.ts` reads `granted` instead of testing `claudeProvider`, so the
Claude-only branches become one data lookup.

### 4. The Access tab says what was withheld, in the owner's words

"Email is off for Kelly: she works on Codex, which can't use the office mailbox yet." Michael puts
it on Needs you once, not on every start. An owner never sees more walls on screen than the
member actually has.

## Then close the gaps, one field at a time

Each step flips one value in the table and ships with its runtime check.

1. **Codex mail.** Write `[mcp_servers.md-mail]` into the member's private
   `CODEX_HOME/config.toml` with a fresh broker token each start, removed at teardown like the
   Claude file. Widen the mail gate in `hooks.ts` to Codex's tool names (TODOS gap 3, issue #109).
2. **Background runner.** `runHiddenModel` with a Codex runner, using the `codex exec` command
   already tested in #85 (TODOS gap 6).
3. **Codex read privacy.** Codex reads mostly through its shell, which workspace-write leaves
   free to read anywhere, and the PreToolUse hook it posts (`hive.ts:4568`) sees a shell command,
   not the paths it touches. A hook can deny Codex's file read tools but cannot make shell reads
   private. Codex stays `write-sandbox`, and is not started for a member whose folder needs read
   privacy, until its sandbox can deny reads. Put the private mail token folder
   (`agentPrivatePaths.ts`, #63) in its denied set either way (#139).

Then add `'codex'` to `BUILD_ENGINES`. Gemini CLI follows the same path.

## More operating systems

The `platform` field is how the same record covers Linux and Windows.
`electron-builder.yml` already has a Linux AppImage target, `folderPolicy` already handles Linux
case-sensitive paths, and the sandbox comment in `hookSettings` names bubblewrap for Linux. Linux
can be declared `os-sandbox+deny` for Claude once the same runtime check passes there. Windows
stays `deny-only` until it has a shell sandbox (#187), and now the Access tab says so instead of
the README.

## Why this first

Every other step of "support Codex, Gemini and more operating systems" passes through this.
Without it, each new engine or platform is a fresh audit of `index.ts` and `hive.ts`. With it,
adding one is filling in a record and passing one test, and the owner is never told a member is
walled in when it isn't.

## Out of scope

Building gaps 1 to 3, Outlook mailboxes, and a Windows shell sandbox.

## How this was found

I read the README's promises against the spawn paths in `src/main/hive.ts` and
`src/main/index.ts`, then checked each gap against TODOS.md (Engines), issue #48 and issue #139,
which already name them and the record idea. This proposal folds mail and background calls into
that record and adds one function that both spawn code and the Access tab read. I also ran REA's
static Electron analysis (`rea-agents analyze-javascript-application`) to map the app's boundary
between the window and the main process; one result from that is reported separately through
private vulnerability reporting, as SECURITY.md asks.
