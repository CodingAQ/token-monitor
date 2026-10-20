---
summary: "CodeArts provider notes: OpenCode-schema SQLite stores (CLI, IDE extension and desktop agent), fork-only ownership, and watch behaviour."
ids: [codearts]
read_when:
  - Changing the CodeArts tracked client, its source roots or its parser policy
  - Debugging a CodeArts scan that reports zero usage or a stale workspace
---

# CodeArts

## Identity and ids

`codearts` is the tracked client for Huawei Cloud's CodeArts (`codearts`) —
the standalone CLI, plus the IDE extension and the desktop agent whose shared
store it also scans.
It is a fork-only client (`forkOnly: true` in `CLIENT_CATALOG`): upstream
tokscale has no such id, and the parse lives in Token Monitor's tokscale fork
(`crates/tokscale-core/src/token_monitor/codearts.rs`), like Proma and Qoder
CN. It is not a limits provider — quota for CodeArts stays out of scope until
a local source for it exists.

## Data sources

CodeArts is built on OpenCode's storage. Two stores carry real usage and both
are scanned by default, merged under the one `codearts` id:

- `~/.codeartsdoer/cli-data/opencode.db` — the standalone CLI.
- `~/.codeartsdoer/codearts-data/opencode.db` — the IDE extension and the
  desktop agent, which share one store.

Both use the same `session`/`message` tables and the same JSON payload shape
(`$.role`, `$.modelID`, `$.providerID`, `$.tokens.{input,output,reasoning,cache}`,
`$.time.created/completed` in milliseconds). The fork parses them through the
shared `opencode_schema` driver, probing both OpenCode query groups so a future
CodeArts build that moves to v2 tables (`session_message`) keeps working
without a parser change. A missing store is a valid empty source; the
`TOKEN_MONITOR_CODEARTS_DB_PATH` environment variable overrides both defaults
with one explicit database for tests and relocated homes.

The VS Code extension home (`~/.codearts-doer-for-coding`) holds no usage data
and is deliberately not scanned.

Token usage is authoritative from that database; there is no session-metadata
provider. Timestamps and project attribution come from the scan itself (the
`session.directory` join supplies the workspace), so `providers/codearts/`
holds no JS reader — unlike opencode, whose legacy JSON sources need one.

## Source precedence

Two default stores, no fallbacks between them: both are scanned on every pass
and a missing store is a valid empty source. The
`TOKEN_MONITOR_CODEARTS_DB_PATH` environment variable overrides both defaults
with one explicit database, mirroring the Qoder CN override the hermetic
contract fixture clears.

## Credentials and transport

None. The store is read locally; no credentials, accounts or network calls are
involved.

## Invariants and known gaps

- The id must stay a fixed point of `normalizeClientName()` and appears in the
  partition invariants like every tracked client (`codearts` aliases nothing).
- Owned clients have no message-cache lane (the cache is keyed on upstream
  `ClientId`s), so every scan re-reads the database. The store is small; do
  not add a client-side cache without evidence it is needed.
- Policy differences from OpenCode are declared in the fork's
  `token_monitor/codearts.rs` (`OpenCodeSchemaConfig` built from the shared
  `base()`): a payload may omit `cache` (or its read/write) without dropping
  the message, and epoch seconds are scaled to milliseconds. The store is
  re-scanned in full — no incremental state is carried across scans. Zero
  embedded costs stay unpriced.
- The store is young: if CodeArts starts writing epoch seconds, channel
  databases (`opencode-<channel>.db`) or payload shapes the driver rejects,
  the parser policy — not a new parser — is the place to absorb it.

## Verification

The fork's own tests plus the Token Monitor contract gate:

```bash
cargo test -p tokscale-core --lib token_monitor::codearts
node --test tests/shared/tokscaleTokenContracts.test.js
```

The contract fixture in `scripts/verify-vendored-tokscale.js` writes a
CodeArts-shaped database into the fixture home and runs the vendored binary
with `--client codearts`; it fails against any binary without the fork
parser, which is the signal to update the vendor pin.
