# holst-cli

A stateful command-line interface to **[Holst](https://holst.so)** — a hosted
collaborative whiteboard (a Russian alternative to Miro). Built so that AI
agents can drive the product without a browser.

## What "the real software" means here

Holst has no local installation. This CLI does **not** re-implement boards: it
speaks the same HTTP API the official web client at `https://app.holst.so` uses,
and reads board contents by decoding the same CRDT snapshots the app restores
from. The Holst service is a hard dependency — without credentials the CLI fails
with instructions rather than degrading to a fake local mode.

## Requirements

- macOS Apple Silicon or Linux x86-64 for prebuilt releases
- Python 3.10+ only when installing from source
- A Holst account
- Network access to `https://app.holst.so` (or your on-premise host)
- Node 21+ plus the `yjs` package for source installations; prebuilt releases include both, to read boards live and to write to them
  (see [How board contents work](#how-board-contents-work))

## Installation

Prebuilt releases include Python, Node and Yjs:

```bash
brew install AntonLisovoy/tap/holst-cli
```

Or install into `~/.local/bin` without Homebrew:

```bash
curl -LsSf https://raw.githubusercontent.com/AntonLisovoy/holst-cli/main/install.sh | sh
holst --version
```

Run the same command to update. Add `~/.local/bin` to your `PATH` if needed.
For a specific version, use `sh -s -- 1.0.0` instead of `sh`.

For development from a source checkout:

```bash
uv sync --dev
uv run holst --help
```

Dependencies install automatically. Source installations additionally need Node
21+ and `npm install --prefix ~/.config/holst-cli/node yjs`.

## Authentication - read this first

Two things to know before you start.

**A personal access token will not work.** The Holst UI issues PATs with a single
scope, `organization:audit_events_read` - they cannot list or create boards.

**A token alone is not enough either.** Verified against the live service: Holst
checks the session cookie *and* the bearer token together.

| Credential sent to `GET /gapi/current_user` | Result |
|---|---|
| Bearer token alone | 401 |
| Cookie alone | 401 |
| **Cookie + token** | **200** |

Signing in stores both halves, so this is usually invisible - but it explains why
`auth login --token <TOKEN>` by itself fails, and why the value in `localStorage`
(a CSRF token, not the session token) authenticates nothing.

```bash
# Recommended: email and password. The password is never stored;
# the resulting token and cookie are.
holst auth login --email you@example.com

# Non-interactive, where there is no terminal for a hidden prompt:
printf '%s' "$PASSWORD" | holst auth login --email you@example.com --password-stdin

# Pair like the desktop app does
holst auth pair

# Or copy both halves from the browser: DevTools -> Network -> any /gapi/
# request -> its Cookie and Authorization request headers
holst auth login --cookie "<COOKIE>"
holst auth login --token <TOKEN> --no-verify
```

Not sure what your account allows, or why a login failed?

```bash
holst auth methods --email you@example.com   # permitted sign-in methods
holst auth diagnose                          # which credential form works
```

The token is stored in `~/.config/holst-cli/credentials.json` with mode
`0600` and is never printed in full. `HOLST_TOKEN` overrides the stored value.

Verify it:

```bash
holst auth status --verify
holst auth whoami
```

## Usage

### Interactive (default)

```bash
holst
```

Starts a REPL with history, a status prompt, and the packaged `SKILL.md` path in
the banner. `help` lists the command groups; `exit` leaves.

### One-shot commands

```bash
# Workspaces
holst workspace list
holst workspace use <WORKSPACE_ID>
holst workspace tree

# Boards
holst board list
holst board list --search retro
holst board create --name "Sprint 42 Retro" --use
holst board info
holst board rename "Sprint 42 Retro (archived)"
holst board duplicate --name "Sprint 43 Retro"
holst board url
holst board members
holst board recent
holst board favorites --add <BOARD_ID>
holst board delete --board <BOARD_ID>            # to trash
holst board restore <BOARD_ID>                   # back from trash
holst board delete --board <BOARD_ID> --permanently

# Reading board contents
holst content summary
holst content objects --type sticker
holst content text
holst content raw -o board.json
holst content objects --snapshot-only     # skip the live session

# Writing board contents
holst content add-frame --ratio 16:9 --name "Sprint 43"
holst content add-sticker --text "Ship the harness" --color yellow4
holst content add-sticker --text "Inside the frame" --frame <FRAME_ID>
holst content add-text --text "Heading" --x 40 --y 40
holst content add-shape --kind flowchart-decision --text "Ship?"
holst content add-card --text "Decision record" --color blue3
holst content add-arrow --from-object <ID> --to-object <ID>
holst content shapes                       # the 62 shape kinds
holst content apply --file objects.json     # many at once

# Changing and removing them
holst content update --object <ID> --text "Reworded" --color red6
holst content update --object <ID> --x 400 --y 200 --to-front
holst content update --object <ID> --frame <FRAME_ID>
holst content delete --object <ID>          # a frame takes its contents

# Snapshots
holst backup list
holst backup download board.snapshot

# Templates
holst template list --scope standard
holst template categories

# Session
holst session status
holst session history
```

### JSON mode for agents

Every command accepts `--json` and emits a uniform envelope:

```bash
holst --json board list
```

```json
{ "ok": true, "boards": [ ... ], "count": 12, "workspace_id": "..." }
```

Failures are equally machine-readable and exit non-zero:

```json
{ "ok": false, "error": "…", "error_type": "auth_error" }
```

`error_type` is one of `auth_error`, `not_found`, `api_error`, `config_error`,
`invalid_input`.

## How board contents work

Holst stores board contents as a Yjs CRDT document; there is no "GET board as
JSON" endpoint, and no create-object endpoint either. The CLI reads two ways and
writes one way.

**Reading** prefers the live document and falls back to a snapshot:

```
live      wss://app.holst.so/hud/ws   ->  join, sync, decode
snapshot  GET /gapi/backup/list       ->  snapshot keys
          GET /gapi/backup/download   ->  binary Yjs update -> pycrdt
```

Snapshots are written about every ten minutes, so they can lag the board, and a
board nobody has opened has none at all. When a read falls back to one, the CLI
says so and why. `--snapshot-only` forces that route; `--key` implies it.

**Writing** joins the board's live document, applies every operation in one Yjs
transaction, and waits for the server to acknowledge the update. Nothing is
reported as written until that acknowledgement arrives.

Six object types can be created: `frame`, `sticker`, `text`, `shape`, `card` and
`arrow`. Everything else on a board is readable but not writable -- see
[Limitations](#limitations).

`content apply` takes a JSON list. An entry with no `op` creates an object of
its `type`, with the remaining keys matching that `add-*` command's options; an
entry with `"op": "update"` or `"op": "delete"` names an existing `id`:

```json
[
  {"type": "frame", "frame_id": "3f1c9d20-0000-4000-8000-000000000001",
   "ratio": "16:9", "name": "Sprint 43", "x": 0, "y": 0},
  {"type": "sticker", "text": "Ship it", "color": "yellow4", "x": 40, "y": 120,
   "parent_id": "3f1c9d20-0000-4000-8000-000000000001"},
  {"type": "shape", "kind": "flowchart-decision", "text": "Ready?", "x": 500, "y": 120},
  {"type": "arrow", "start": {"x": 260, "y": 200}, "end": {"x": 500, "y": 200}},

  {"op": "update", "id": "…", "text": "Reworded", "color": "red6"},
  {"op": "delete", "id": "…"}
]
```

Ids are not allocated by the server, so giving the frame an explicit `frame_id`
lets the notes name it as their `parent_id` in the same file. Omit it and the
frame gets a generated id, which only exists after the write.

If any id in the file is wrong -- an edit or delete naming an object that is not
there, or a `parent_id` naming a frame that is not -- **nothing** is written and
the error names the id. A file may name a frame it creates further down.

**Editing** rewrites the fields named and leaves the rest alone; `--text`
retypes the body in place, so anyone with that board open sees the change
without reloading. **Deleting** a frame takes its contents with it, as it does
in Holst; `--keep-children` leaves them behind. Neither can be undone from the
CLI: a board's undo history belongs to the browser session that made the change.

Colours are Holst palette tokens -- one of `white`, `amber`, `blue`, `brown`,
`crimson`, `cyan`, `grass`, `gray`, `green`, `indigo`, `jade`, `lime`, `mint`,
`orange`, `pink`, `plum`, `purple`, `red`, `sky`, `teal`, `tomato`, `violet`,
`yellow` followed by a shade `1`-`12` -- or a hex value, except on a card, which
takes tokens only. Frame presets are `16:9`, `4:3`, `1:1`, `a4-portrait` and
`a4-landscape`; pass `--width`/`--height` for a custom size. Shape kinds are
listed by `content shapes` -- 19 basic, 25 flowchart and 18 BPMN.

One thing to know when reading back what you wrote: a text object's type on the
wire is `simple-text`, so `content objects --type simple-text` is what filters
for it. `add-text` and `"type": "text"` in an apply file both accept the
friendlier name.

**Both live reads and all writes need Node 21+ and the reference `yjs` package** (included in prebuilt releases),
because Holst's `hud` boards speak the Yjs v2 encoding that `pycrdt` does not.
Install it once:

```bash
npm install --prefix ~/.config/holst-cli/node yjs
```

Without it, reads fall back to snapshots and writes fail with that exact install
command -- never silently. A v2 update decodes as *empty*, not as an error, under
a v1 reader, so "this board is empty" is never reported without checking.
Override the binary with `HOLST_NODE` and the package location with
`HOLST_YJS_PATH`.

Writing is implemented for the `hud` provider, which is what every current board
uses. A `hocuspocus` board is refused with a message naming the reason rather
than written to blindly; its contents remain readable.

## Examples

Two runnable scripts in `examples/`, both shaped the way an agent would drive
the CLI — `--json` in, ids out:

| Script | What it does |
|---|---|
| `read_board.sh [query]` | Find a board by name and print its object counts and every string on it |
| `write_board.sh [name]` | Create a board, put a 16:9 frame and five notes on it, read them back, print the URL |
| `diagram_board.sh [name]` | Draw a flowchart of shapes and arrows, relabel one node, delete another, read it back |

## Configuration

| Variable | Purpose | Default |
|---|---|---|
| `HOLST_TOKEN` | Session token, overrides the stored file | — |
| `HOLST_BASE_URL` | Service host (on-premise installs) | `https://app.holst.so` |
| `HOLST_AUTH_SCHEME` | Token header form: `bearer`, `token`, `raw`, `x-auth-token` | `bearer` |
| `HOLST_CLI_STATE_DIR` | Session/credentials directory | `~/.config/holst-cli` |
| `HOLST_LIVE_TESTS` | Set to `1` to enable the live test suite | unset |

## Running the tests

```bash
# from the repository root
pip install -e ".[test]"

# Offline suite: unit tests, CRDT pipeline, local HTTP contract, CLI subprocess
HOLST_CLI_FORCE_INSTALLED=1 python3 -m pytest holst_cli/tests/ -v -s

# Live suite (creates and deletes boards in YOUR workspace)
HOLST_LIVE_TESTS=1 HOLST_CLI_FORCE_INSTALLED=1 \
  python3 -m pytest holst_cli/tests/test_full_e2e.py -v -s -k Live
```

See `tests/TEST.md` for the plan, results, and known coverage gaps.

## Limitations

- **Six object types can be created**: frames, sticky notes, text, shapes, cards
  and arrows. The rest of Holst's catalogue -- tables, images, kanban boards,
  mind maps, code blocks, dice and the others -- decodes and lists fine, and any
  of them can be moved, resized, restacked, reparented and deleted, but there is
  no builder to create one and no way to set its colour or text.
- **Editing covers position, size, colour, text, parent and stacking.** Rotation,
  fonts, arrow labels and per-type styling are not exposed; `content raw` shows
  the fields that exist on any given object. An arrow is the exception to
  moving: it follows its endpoints, and re-routing one means deleting it and
  drawing another.
- **Writing needs the `hud` provider**, and Node with the `yjs` package. A
  `hocuspocus` board is refused explicitly.
- **No server-side image export.** Holst renders PNG/SVG/PDF in the browser, so
  there is no endpoint to ask for a rendered board.
- **Unofficial.** Holst publishes no public API reference; the endpoints used
  here were read from the official web client. They may change without notice.
  Response fields are read defensively and the raw payload is always preserved.
  When something does break, `docs/WHEN_THE_API_CHANGES.md` is the recovery
  procedure: how to identify which layer drifted, how to read the current
  bundle with `tools/bundle_search.py`, and where each fact lives in the code.

## Attribution

Not affiliated with or endorsed by Holst.
