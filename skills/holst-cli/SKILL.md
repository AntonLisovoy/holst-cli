---
name: holst-cli
description: >-
  Drive the Holst (holst.so) collaborative whiteboard from the command line:
  list and manage workspaces and boards, read what is on a board, put frames,
  sticky notes, text, shapes, cards and arrows onto one, change or remove them,
  download backups, and browse templates. Use when the user asks to inspect,
  create, rename, duplicate, or delete Holst boards, to read what is on a Holst
  board, or to add, edit or remove content on a board without opening a
  browser.
---

# holst-cli

Command-line interface to **Holst** (<https://holst.so>), a hosted collaborative
whiteboard. The CLI calls the same HTTP API the official web client uses — it
does not re-implement Holst, and it requires a real account.

If the holst MCP server is connected (`holst mcp`), prefer its tools to running `holst` commands.

## Installation

```bash
brew install AntonLisovoy/tap/holst-cli
# Alternative:
# curl -LsSf https://raw.githubusercontent.com/AntonLisovoy/holst-cli/main/install.sh | sh
holst --help
```

Prebuilt releases support macOS Apple Silicon and Linux x86-64 and include
Python, Node and Yjs. Network access to `https://app.holst.so` is required.
For a source checkout: `pip install -e .` (Python 3.10+), plus Node 21+ and
`npm install --prefix ~/.config/holst-cli/node yjs`.

Install this skill for your agent:

```bash
holst skill install -a claude-code --project -y
```

`-a` is repeatable (`claude-code`, `cursor`, `codex`, `copilot`, `gemini`, `universal`); `-g` installs for the whole user account instead of this directory. `holst skill path` prints where the bundled file lives.

## Authentication - the traps to avoid

Verified against the live service:

- A **personal access token** from the Holst UI carries a single scope,
  `organization:audit_events_read`. It **cannot** list or create boards. If a
  user offers "my Holst API token", expect this one and expect it to fail.
- **A token alone is never enough.** Holst checks the session cookie *and* the
  bearer token together; either alone returns 401. Signing in stores both, and
  `auth status` warns when only one half is present.
- **Stored** credentials are bound to the host you signed in to. With
  `--base-url` pointing elsewhere, holst refuses to send them before any
  request is made (checked with `--base-url https://example.invalid`) and
  tells you to sign in for that host.
- `holst auth pair` stores a working session (token plus the `sessionToken`
  and `holst-signed-user` cookies); `auth status --verify` and `board list`
  work with it.
- `holst auth logout` revokes the session on the server (`server_revoked:
  true`) and removes the local token, cookie and host; the old token then
  gets 401.
- `holst auth sessions revoke ID -y` signs out one session (401 for it
  afterwards); without `-y` and without a terminal it is refused.
- REPL history masks a token typed on the command line.

Implemented, not yet verified against the live service:

- `HOLST_BASE_URL` pointing elsewhere is treated like `--base-url`.
  `HOLST_TOKEN` and `HOLST_COOKIE` from the environment are explicit and are
  not host-bound: they are sent to whatever host is in use.
- `holst auth logout --local` skips the server. If the server revoke fails,
  local credentials are still cleared; the command exits 0 with a warning and
  `server_revoked: false` under `--json`.
- `holst auth logout --all-others -y` signs out all other sessions and keeps
  the current one.

**Pairing without the desktop app.** Run `holst auth pair` (or
`holst --json auth pair`, then later `holst auth redeem CODE` from the same
machine within 15 minutes). Open `/app-auth/<request id>/grant` on the Holst
host in a signed-in browser with DevTools -> Network open, press the approve
button, and copy `data.code` from the `app_auth/<id>/grant` response; the
`holst://` link it then opens needs the desktop app and can be ignored. The
grant link works once: a second click returns `app-auth-grant-expired`, so
start a new `holst auth pair` instead.

```bash
holst auth login --email you@example.com    # prompts; password never stored
printf '%s' "$PW" | holst auth login --email you@example.com --password-stdin
holst auth pair                             # desktop-style pairing
holst auth methods --email you@example.com  # permitted sign-in methods
holst auth diagnose                         # which credential form works
```

Most agent contexts have no terminal for a hidden prompt: use `--password-stdin`
rather than letting the prompt abort with `error_type: "no_input"`.

Check state before doing anything else:

```bash
holst --json auth status --verify
```

## Agent usage rules

1. **Always pass `--json`.** Every command emits a uniform envelope:
   - success: `{"ok": true, ...payload}`
   - failure: `{"ok": false, "error": "...", "error_type": "..."}` with a non-zero exit
2. **Branch on `error_type`**, not on message text:
   | `error_type` | Meaning | What to do |
   |---|---|---|
   | `config_error` | No token configured | Tell the user to run `auth pair` |
   | `auth_error` | Token rejected or lacks access (401/403) | Token may be a PAT or expired; re-authenticate |
   | `not_found` | Board/workspace does not exist (404) | Re-list to get valid ids |
   | `api_error` | Rate limit, server error, unreachable | Retry or report |
   | `invalid_input` | Bad arguments | Fix the command |
   | `invalid_select` | A `--select` path is empty (`""` or `a,,b`) | Fix the path |
   | `io_error` | A local file could not be read or written (`--out`) | Check the path and permissions |
   | `skill_install_error` | `skill install` could not write the skill | Read the message; fix the target directory |
   | `doctor_failed` | `holst doctor` found a failing check | Read the `checks` list and follow each `hint` |

   When something fails and the cause is unclear, run `holst --json doctor` first: it checks versions, the state directory, credentials, Node, yjs and the API, and names a fix for each problem.
   If `doctor` is clean and a live read or write still fails, run the same command again with `HOLST_DEBUG=1` (or `--debug`) and read stderr: one redacted line per HTTP request, socket protocol message and phase, with timings; stdout and `--json` output are unchanged.
3. **Inspect before mutating.** Run `board info` or `content summary` before
   renaming or deleting.
4. **Deletion is confirmed by default.** Non-interactive calls need `-y`, and
   `--permanently` destroys instead of trashing. Never pass `--permanently -y`
   unless the user asked for permanent deletion.
5. **Select context once.** `workspace use <ID>` and `board use <ID>` persist in
   the session file, so later commands can omit ids.

## Keeping results out of your context

Global flags go **before** the subcommand.

- `--select` keeps only the dotted fields you name; lists are walked for you and the shape is kept:

  ```bash
  holst --select workspaces.id,workspaces.name workspace list
  ```

  A field that does not exist comes back as `null`, never as an error, so a `null` may be a typo: check the spelling against the full output once.
- `--out FILE` writes the whole JSON result to a file and prints one line, `{"ok": true, "output": "/abs/path", "bytes": N}`; read the file with your own tools only when you need it:

  ```bash
  holst --out boards.json board list
  ```

Both imply `--json`. When a command fails (an error envelope) no file is written and the envelope is printed as usual; a failing `holst doctor` report is still written, and the summary line carries `"ok": false`, `error_type` and `error`.

## Command groups

| Group | Purpose |
|---|---|
| `auth` | Token management, pairing, `whoami`, sessions, personal access tokens |
| `workspace` | List workspaces, select one, show the space/board tree, member counts |
| `board` | List, search, create, rename, duplicate, move into a folder or to the root, move to trash, delete, share, members, comments, recents, favorites |
| `content` | Read a board (`summary`, `objects`, `text`, `raw`, `shapes`) and write to it (`add-frame`, `add-sticker`, `add-text`, `add-shape`, `add-card`, `add-arrow`, `update`, `delete`, `apply`) |
| `backup` | List and download board snapshots |
| `template` | Browse standard/personal/workspace templates and categories |
| `session` | Show, clear, and inspect the persisted selection and history |

Running `holst` with no subcommand opens an interactive REPL.

## Common workflows

### Find a board and read what is on it

```bash
holst --json workspace list
holst --json board list --search retro
holst --json board use <BOARD_ID>
holst --json content summary          # object counts by type
holst --json content text             # every readable string
holst --json content objects --type sticker
```

### Create a board

```bash
holst --json board create --name "Sprint 42 Retro" --use
# response includes "id" and "url"
```

### Put something on a board

```bash
holst --json content add-frame --ratio 16:9 --name "Sprint 42" \
  --x -445.5 --y 71
# response includes "created": ["<FRAME_ID>"]

holst --json content add-sticker --text "Deploys were flaky" \
  --color red6 --x 285 --y 317 --frame <FRAME_ID>
```

Many objects at once, in a single transaction — prefer this over a loop of
`add-sticker` calls, which joins the board once per note:

```bash
cat > objects.json <<'EOF'
[
  {"type": "frame", "frame_id": "3f1c9d20-0000-4000-8000-000000000001",
   "ratio": "16:9", "name": "Sprint 42", "x": 0, "y": 0},
  {"type": "sticker", "text": "What went well", "color": "yellow4",
   "x": 40, "y": 200, "parent_id": "3f1c9d20-0000-4000-8000-000000000001"},
  {"type": "sticker", "text": "What did not", "color": "red6",
   "x": 300, "y": 200, "parent_id": "3f1c9d20-0000-4000-8000-000000000001"}
]
EOF
holst --json content apply --board <BOARD_ID> --file objects.json
```

To put notes inside a frame created in the same file, give the frame an explicit
`frame_id` (any fresh uuid) and use it as the notes' `parent_id` — ids are not
allocated by the server, so one generated up front is as valid as one the CLI
would have made. Without it the frame's id only exists after the write, and the
notes need a second call.

Beyond notes: `add-text` for a bare text block, `add-shape --kind <kind>` for
one of 62 shapes (`content shapes` lists them), `add-card` for a document block,
and `add-arrow` to connect two objects:

```bash
holst --json content add-shape --kind flowchart-decision \
  --text "Ship it?" --x 40 --y 200
holst --json content add-arrow --from-object <ID> --to-object <ID> \
  --type elbow --head-end triangle-filled
```

### Change or remove something

```bash
holst --json content objects            # find the id first
holst --json content update --object <ID> --text "Reworded" --color red6
holst --json content update --object <ID> --x 400 --y 200 --to-front
holst --json content update --object <ID> --frame <FRAME_ID>   # or --frame none
holst --json content delete --object <ID> -y   # -y is required
```

`update` rewrites only the fields named; `--text` retypes the body in place, so
a collaborator with the board open sees the change without reloading. `delete`
takes a frame's contents with it unless `--keep-children` is passed. At a
terminal it asks first; without one (an agent, a pipe) it does not prompt and
fails with `error_type: "no_input"` ("pass -y"), so an agent has to pass `-y`
explicitly. A piped `y` is not accepted.

One transaction can create, change and remove at once:

```json
[
  {"type": "sticker", "text": "New", "x": 0, "y": 0},
  {"op": "update", "id": "<ID>", "color": "blue6"},
  {"op": "delete", "id": "<ID>"}
]
```

If any id in the file is not on the board, **nothing** is written and the error
names the id.

Give every created object an `id` (`--id`, or `"id"` in an apply item). If a
write fails with `write_unconfirmed`, re-run the same call with the same ids;
do not re-read and rebuild. Use `dry_run` / `--dry-run` to preview deletes and
large batches.

### Archive a board offline

```bash
holst --json backup list --board <BOARD_ID>
holst --json backup download board.snapshot --board <BOARD_ID>
holst --json content raw --snapshot board.snapshot -o board.json
```

### Clean up

```bash
holst --json board delete --board <BOARD_ID> -y              # to trash
holst --json board trash                                    # verify
holst --json board restore <BOARD_ID>                       # undo the trashing
holst --json board delete --board <BOARD_ID> --permanently -y
```

## How board contents work

Holst stores boards as **Yjs CRDT documents**; there is no endpoint returning a
board as JSON. `content` commands download the newest snapshot
(`/gapi/backup/download`) and decode it locally with `pycrdt`. Consequences an
agent should know:

- Reads prefer the **live board** and fall back to the newest **snapshot**.
  Snapshots are written about every ten minutes, so a fallback read can be
  stale; the CLI says when it fell back and why. `--snapshot-only` forces it.
- Writes join the live document and are reported only once the server
  acknowledges them.
- Six types can be **created**: `frame`, `sticker`, `text`, `shape`, `card`,
  `arrow`. Any object, of any type, can be **moved, resized, restacked,
  reparented and deleted**; colour and text can be changed on the six above.
  Everything else reads fine. Writing needs the `hud` provider (every current
  board); a `hocuspocus` board is refused with a message saying so.
- A text object's type on the wire is **`simple-text`**, so that is what
  `content objects --type ...` filters on. `add-text` and `"type": "text"` in an
  apply file take the friendly name.
- **Deleting a frame deletes its contents**, as in Holst itself. Pass
  `--keep-children` to leave them on the board. Nothing deleted through the CLI
  can be undone from the CLI.
- **Positions are absolute board coordinates**, even for an object inside a
  frame. `--frame <id>` sets the parent; it does not shift the coordinates.
- Colours are palette tokens (`yellow4`, `red6`, `blue6`): one of 23 names with
  a shade 1-12, or a hex value. An unknown token is rejected with the list.
- A board that has never been opened may have **no snapshot**; the error says so.
- There are two snapshot encodings and the CLI picks the right one automatically
  (asking for the wrong provider makes the server return HTTP 500). Boards on
  Holst's `hud` provider use Yjs **v2**, which needs Node plus the `yjs`
  package: `npm install --prefix ~/.config/holst-cli/node yjs`. If it is
  missing, the error says so - do not report the board as empty.
- Objects without a recognisable type are reported as `unknown`.
- Object shapes are Holst's own. `content objects` reports `id`, `type`, and
  extracted `text`, and keeps the full object under `raw` for anything else.

## Output examples

```bash
$ holst --json board list
{
  "ok": true,
  "boards": [{"id": "…", "name": "Sprint 42 Retro", "space": "Design", …}],
  "count": 12,
  "workspace_id": "…"
}

$ holst --json board list      # without credentials
{
  "ok": false,
  "error": "No Holst access token configured. …",
  "error_type": "config_error"
}
```

## Environment variables

| Variable | Purpose |
|---|---|
| `HOLST_TOKEN` | Session token; overrides the stored credentials file |
| `HOLST_BASE_URL` | Service host, for on-premise installations; overrides the stored host (`--base-url` overrides both) |
| `HOLST_CLI_STATE_DIR` | Where session and credentials are kept |
| `HOLST_AUTH_SCHEME` | Token header form (`bearer` by default) |
| `HOLST_COOKIE` | Session cookie; Holst SaaS needs it together with the token |
| `HOLST_NODE` | Path to the Node binary used for board contents and live edits |
| `HOLST_YJS_PATH` | Directory that directly contains the `yjs` package (normally a `node_modules` directory) |
| `HOLST_DEBUG` | `1` traces HTTP requests, board-session frames and timings to stderr, redacted |

## Cautions

- Boards are shared with other people. Creating, renaming, deleting **and
  writing, editing or removing objects** are visible to the whole workspace
  immediately — there is no local staging and no undo in this CLI. Confirm the
  target board before writing.
- `content delete` removes other people's work as readily as your own, and
  deleting a frame deletes everything inside it. Read the board first
  (`content objects`), name ids explicitly, and never pass `-y` on a board the
  user has not confirmed.
- Tokens are secrets. They are stored at mode `0600` and masked in output; never
  echo a full token back to the user or into a file.
- Holst publishes no official API. Endpoints were read from the web client and
  may change without notice. When a command starts failing, or a board reads as
  emptier than it looks in the browser, **read `docs/WHEN_THE_API_CHANGES.md`
  before changing any code** — it covers how to tell which layer drifted, how to
  read the current client bundle (`tools/bundle_search.py`), and the one place
  each fact lives. Never guess a replacement shape: a wrong one decodes cleanly
  and renders nothing.
