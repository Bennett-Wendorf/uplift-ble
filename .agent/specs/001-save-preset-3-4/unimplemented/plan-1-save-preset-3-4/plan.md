# Plan: 1 - Save height presets 3 and 4

## Task Description

Add support for saving the current desk height to memory presets 3 and 4 on
Uplift/Jiecang standing desks controlled via the Uplift BLE adapter. Today
this repository (Bennett-Wendorf's fork of `librick/uplift-ble`, checked out
at the v0.5.2 state) exposes save commands for presets 1 and 2 only
(`save_height_preset_1()` → opcode `0x03`, `save_height_preset_2()` →
opcode `0x04` in `src/uplift_ble/desk_controller.py`). There are no
controller methods, CLI commands, or documentation for saving presets 3 and
4.

Task type: **feature** · Complexity: **medium** (small code delta; the
dominant risk is protocol uncertainty around which opcodes the firmware
accepts for save-into-preset-3/4).

## Objective

1. Add `DeskController.save_height_preset_3()` and
   `DeskController.save_height_preset_4()` factory methods (mirroring
   presets 1/2) that send the correct vendor opcodes with an empty payload.
2. Expose matching CLI subcommands in `src/uplift_ble_cli/cli.py`.
3. Add unit tests following existing patterns in `tests/` (exact wire-format
   packets + controller-method behavior).
4. Update `README.md` protocol tables and the compatibility matrix, marking
   the new commands with their verification status.
5. Bump the package version per repo conventions (`0.5.2` → `0.6.0`,
   feature ⇒ minor bump; no changelog file exists in this repo).

Secondary objective (recommended, upstream parity — see Open Questions):
add the recall commands `move_to_height_preset_3()` /
`move_to_height_preset_4()` (opcodes `0x27`/`0x28`, hardware-verified
upstream in librick/uplift-ble PR #53) because they are absent from this
checkout and are needed for end-to-end hardware verification of the new save
commands.

## Problem Statement

Users of desks with 4 memory presets cannot program slots 3/4 from this
library or CLI; they must use the physical keypad (or the vendor app, which
stores presets on the phone rather than in the adapter — upstream PR #46).
Reading preset values for all four slots already works: notification
handlers `_process_notification_0x25`…`_process_notification_0x28` populate
`self._height_preset_1..4` and emit `HEIGHT_PRESET_1..4` events, and the
read-only properties `height_preset_1..4` are exposed. Only the write path
for slots 3/4 is missing — but the correct opcodes for it are **not
documented by the project** and must be researched (done in this plan) and
empirically confirmed on hardware before they can be marked verified.

### Current State (verified against code at checkout `feat/save-preset-3-4`)

| Item | Status |
| --- | --- |
| Save commands preset 1/2 (`0x03`/`0x04`, empty payload) | Exist (`desk_controller.py` L576-584) and have CLI commands. Verified by upstream community on `0xFE60` + `0xFF00` (PRs #46, #48). |
| Save commands preset 3/4 | **Do not exist anywhere.** |
| Recall commands preset 1/2 (`0x05`/`0x06`) | Exist in controller and CLI. |
| Recall commands preset 3/4 (`0x27`/`0x28`) | **Do NOT exist in this checkout** (they were added upstream only in v0.6.0, PR #53; this fork is at the v0.5.2 state). Note: `0x27`/`0x28` *do* appear here as incoming notification opcodes for preset 3/4 reports (`_process_notification_0x27/0x28`, handler map L265-268). |
| Preset report handling + properties (all four) | Complete: `HEIGHT_PRESET_1..4` events, `height_preset_1..4` properties. |
| Version / releases | `pyproject.toml` = 0.5.2; no CHANGELOG file in repo; GitHub Releases notes are auto-generated ("What's Changed"). This fork currently has zero tags/releases on GitHub. |
| Upstream (librick/uplift-ble) is at v0.7.0 and already contains recall 3/4 (`move_to_height_preset_3/4`, `0x27`/`0x28`) but **no** save 3/4 either — so this feature would be new upstream as well. |

## Solution Approach

### Opcode research findings (the critical open question)

Question: which opcodes does the firmware accept to SAVE into presets 3 and
4? Presets 1/2 use consecutive `0x03`/`0x04`, while recalls jump to
`0x05`/`0x06` (slots 1/2) and `0x27`/`0x28` (slots 3/4), so the save opcodes
for slots 3/4 are not guessable from the basic block alone.

Research performed:

- **This repo**: README protocol tables document commands only through
  `0x06`; notifications `0x25`–`0x28` report preset heights. No save-3/4
  opcodes anywhere in code, comments, or tests.
- **Upstream librick/uplift-ble** (issues #4, #52; PRs #46, #48, #53):
  recall for slots 3/4 was hardware-verified as `0x27`/`0x28`
  (`move_to_height_preset_3/4`, merged in v0.6.0). Issue #52 (cbusillo)
  confirms slot recalls follow reprogrammed handset memories, and notes
  "Opcodes 0x27 and 0x28 are also used by incoming preset reports; BLE
  characteristic direction distinguishes outgoing recalls from incoming
  reports." No save-3/4 opcodes found upstream either.
- **gabe565's ESPHome wiki, "Uplift Desk Automation"**
  (https://wiki.gabe565.com/books/esphome/page/uplift-desk-automation):
  explicit command table:
  `0x03` save preset 1, `0x04` save preset 2, `0x05`/`0x06` recall 1/2,
  **`0x25` save preset 3, `0x26` save preset 4**, `0x27`/`0x28` recall
  3/4; and "a sync request (`0x07`) gets a response with cmd `0x25`,
  `0x26`, `0x27`, `0x28`, then a regular `0x01` height."
- **Rocka84 / Jiecang Reverse Engineering** (https://gitlab.com/Rocka84/jiecang-reverse-engineering),
  based on decompiling the Jiecang dongle app (FCC-ID 2ANKDJCP35NBLT1):
  `MEMORY 1` = `0x03`, `MEMORY 2` = `0x04`, `GO TO MEMORY 1/2` =
  `0x05`/`0x06`, **`MEMORY 3` = literal packet `F1 F1 25 00 25 7E`**
  (`cmdMemory3 = "F1F12500257E"`), `GO TO MEMORY 3` =
  `F1 F1 27 00 27 7E`. That app's table stops at memory 3, but it documents
  an alternate absolute-set mechanism: `SET MEMORY HEIGHT` = opcode
  `0xAD`, payload `[slot (0x01-0x03), height_hi, height_lo]` — the vendor
  app writes explicit heights into memories this way.

**Recommendation**: implement **save preset 3 → `0x25`**,
**save preset 4 → `0x26`**, empty payload. Confidence: **medium-high**.
Basis: (a) two independent community sources agree on exactly these
opcodes; (b) the extended block is structurally consistent — recalls
`0x27`/`0x28` in that same block are hardware-verified, and opcodes are
already reused between command and report spaces (`0x25` as an outgoing
save colliding with `0x25` as the incoming preset-1 *report* is exactly
analogous to upstream's verified `0x27`/`0x28` recall/report reuse); (c)
Rocka84's literal packet string for memory 3 matches our
`create_command_packet(0x25, b"")` byte-for-byte. Residual uncertainty:
neither source is an official Uplift/Jiecang document, and no published
test of these save opcodes on real hardware exists — therefore all code,
README entries, and tests in this plan must label them as
**pending hardware verification** (compat-table ⚠️, docstring note) until
the manual verification below succeeds. Fallback candidates if `0x25`/
`0x26` turn out inert are documented in "Open Questions and Risks" (try
`0x29`/`0x2A` first; never brute-force broad ranges — e.g. `0x12` is known
potentially dangerous).

Design: mirror existing conventions exactly (`@command_writer()`, return a
zero-argument-payload `DeskCommand`), so no packet-format, event, or
notification-handler changes are required. Save opcodes produce no direct
acknowledgement; stored values surface via the already-handled
`0x25`–`0x28` preset reports (triggered on V3-class hardware by writing
opcode `0x07`, which this repo exposes as `request_height_limits()`).

## Relevant Files

- `src/uplift_ble/desk_controller.py` — add 4 command factory methods
  (`save_height_preset_3/4`; plus parity recalls
  `move_to_height_preset_3/4`).
- `src/uplift_ble_cli/cli.py` — add matching CLI subcommands.
- `tests/test_packet.py` — exact wire-format regression tests for the new
  opcodes (existing pattern: `test_command_packet_opcode_*`).
- `README.md` — "Known Commands" table rows, compatibility matrix rows,
  verification-status notes.
- `pyproject.toml` — version bump `0.5.2` → `0.6.0`.
- `uv.lock` — must be re-generated after the version bump (it pins the
  project's own version; upstream did this as a separate
  "chore: tick uplift ble version in uv lock" change).

### New Files

- `tests/test_desk_controller.py` — first test file for
  `DeskController.command_writer` methods, using a fake BLE client
  (pattern adapted from the fakes in `tests/test_desk_validator.py`).

## Team Orchestration

> **Worktree Isolation**: The team-lead creates an isolated git worktree
> for this spec using `~/.config/opencode/scripts/worktree-create.sh`. All
> builders work inside this worktree. After final validation, changes are
> merged back via `~/.config/opencode/scripts/worktree-merge.sh`.

The team-lead agent will orchestrate execution using these team members:

### Team Members

- **Builder**
  - Name: preset34-builder
  - Role: Implement controller methods, CLI commands, tests, README and
    version-bump changes
  - Agent: builder

- **Validator**
  - Name: preset34-validator
  - Role: Verify implementation meets acceptance criteria (tests, lint,
    format, spell, docs consistency)
  - Agent: validator

- **Documenter**
  - Name: preset34-documenter
  - Role: Final README/protocol-doc review and draft release notes
  - Agent: documenter

## Step by Step Tasks

### 1. Add preset 3/4 save (and recall) command methods
- **Task ID**: add-preset-34-controller-methods
- **Depends On**: none
- **Assigned To**: preset34-builder
- **Agent**: builder
- **Actions**:
  - In `src/uplift_ble/desk_controller.py`, after
    `move_to_height_preset_2` (L591-594), add the two recall methods
    exactly as upstream v0.6.0+ does (hardware-verified opcodes; keep
    docstrings identical to upstream for merge parity):
      ```python
      @command_writer()
      def move_to_height_preset_3(self) -> DeskCommand:
          """Recall height preset 3 (move the desk to the stored position).

          Opcode 0x27 also identifies incoming preset 3 reports; the BLE
          characteristic direction distinguishes the command from the report.
          """
          return DeskCommand(opcode=0x27, payload=b"")
      ```
      and `move_to_height_preset_4` with opcode `0x28` (same docstring
      shape).
  - Add the two new save methods in the same preset-command region
    (keep the file's opcode-ordered grouping; place them so the
    `0x25`/`0x26` entries sit between `set_current_height_as_height_limit_min`
    (`0x22`) and `stop_movement` (`0x2B`), or immediately after the recall
    methods if that reads better — do not reorder existing methods):
      ```python
      @command_writer()
      def save_height_preset_3(self) -> DeskCommand:
          """Save the current desk height as height preset 3.

          Opcode 0x25 also identifies incoming preset 1 reports; the BLE
          characteristic direction distinguishes the command from the report.
          Pending hardware verification (community-sourced opcode).
          """
          return DeskCommand(opcode=0x25, payload=b"")
      ```
      and `save_height_preset_4` with opcode `0x26` (analogous docstring:
      "Opcode 0x26 also identifies incoming preset 2 reports…").
  - No changes to notification handlers, events, properties, or
    `packet.py` are needed.
- **Acceptance Criteria**:
  - `save_height_preset_3()` returns
    `DeskCommand(opcode=0x25, payload=b"")`;
    `save_height_preset_4()` returns
    `DeskCommand(opcode=0x26, payload=b"")`.
  - Recall methods return opcodes `0x27`/`0x28` with empty payloads.
  - Code follows AGENTS.md (type hints, Google-style docstrings on public
    methods, no unnecessary comments) and passes ruff check + format.

### 2. Add CLI commands
- **Task ID**: add-preset-34-cli
- **Depends On**: add-preset-34-controller-methods
- **Assigned To**: preset34-builder
- **Agent**: builder
- **Actions**:
  - In `src/uplift_ble_cli/cli.py`, mirror the existing
    `save_height_preset_1/2` and `move_to_height_preset_1/2` commands
    (L199-232) for presets 3 and 4: four new
    `@cli.command()` + `@with_desk_controller` async functions
    (`save_height_preset_3`, `save_height_preset_4`,
    `move_to_height_preset_3`, `move_to_height_preset_4`) that await the
    controller method and echo a confirmation to stderr, e.g.:
      ```python
      @cli.command()
      @with_desk_controller
      async def save_height_preset_3(controller: DeskController):
          """Save the current desk height as height preset 3."""
          await controller.save_height_preset_3()
          click.echo(
              "Sent command to save desk's current height as height preset 3",
              err=True,
          )
      ```
    (click auto-derives subcommand names `save-height-preset-3`, etc.)
- **Acceptance Criteria**:
  - `uplift-ble-cli --help` lists the four new subcommands.
  - Each command connects via the existing `_get_desk_controller` flow and
    sends exactly one command packet (plus wake burst when
    `requires_wake`).
  - ruff clean; message style matches neighboring commands.

### 3. Add tests
- **Task ID**: add-preset-34-tests
- **Depends On**: add-preset-34-controller-methods
- **Assigned To**: preset34-builder
- **Agent**: builder
- **Actions**:
  - `tests/test_packet.py`: add exact-bytes regression tests following the
    existing `test_command_packet_opcode_*_empty_payload` pattern:
      ```python
      def test_command_packet_opcode_0x25_empty_payload():
          """Save preset 3 command (`F1 F1 25 00 25 7E`)."""
          packet = create_command_packet(0x25, b"")
          assert packet == bytes([0xF1, 0xF1, 0x25, 0x00, 0x25, 0x7E])
      ```
      plus the same for `0x26` (`F1 F1 26 00 26 7E`),
      `0x27` (`F1 F1 27 00 27 7E`), and
      `0x28` (`F1 F1 28 00 28 7E`).
  - New file `tests/test_desk_controller.py`: construct a real
    `DeskController` around a fake client (no BLE, no timers):
      ```python
      class FakeBleakClient:
          def __init__(self) -> None:
              self.written: list[tuple[str, bytes]] = []

          async def write_gatt_char(self, uuid: str, data: bytes, response: bool = True) -> None:
              self.written.append((uuid, bytes(data)))
      ```
    Build with
    `DeskController(client=fake, input_char_uuid="in", output_char_uuid="out", requires_wake=False, notification_timeout=0)`
    (constructor signature per L87-99). Write one
    `@pytest.mark.asyncio` test per new method asserting
    (a) the returned packet equals the expected bytes and
    (b) `fake.written == [("in", <same bytes>)]`. Cover
    `save_height_preset_3/4` and `move_to_height_preset_3/4`.
  - No notification-parsing tests are needed: save opcodes generate no new
    report types; the `0x25`–`0x28` report handlers already exist.
- **Acceptance Criteria**:
  - All new tests pass with `uv run pytest`; existing tests unaffected.
  - Tests assert behavior (packet bytes written to the input
    characteristic), not private implementation details.
  - Async tests use explicit `@pytest.mark.asyncio` (repo runs
    pytest-asyncio in strict mode; no `asyncio_mode` configured).

### 4. Update README protocol documentation
- **Task ID**: update-readme
- **Depends On**: add-preset-34-controller-methods
- **Assigned To**: preset34-builder
- **Agent**: builder
- **Actions**:
  - "Known Commands" table: insert rows (keeping the table's opcode
    ordering, after the `0x23` rows and before the `0x2B` row):
      | Opcode | Length | Attribute Value | Purpose |
      | ------ | ------ | --------------- | ------- |
      | 0x25   | 0      | `0xF1,0xF1,0x25,0x00,0x25,0x7E` | Save current height as height preset 3 (community-sourced; pending hardware verification) |
      | 0x26   | 0      | `0xF1,0xF1,0x26,0x00,0x26,0x7E` | Save current height as height preset 4 (community-sourced; pending hardware verification) |
      | 0x27   | 0      | `0xF1,0xF1,0x27,0x00,0x27,0x7E` | Move to height preset 3 |
      | 0x28   | 0      | `0xF1,0xF1,0x28,0x00,0x28,0x7E` | Move to height preset 4 |
    Add a sentence under the table noting that save-3/4 opcodes come from
    community reverse engineering (Jiecang app decompilation) and remain
    unverified by this project, while recall 3/4 were verified on an
    official Uplift V3 adapter (`0x00FF` service profile, FF00 control
    characteristics) — cite the upstream v0.6.0 validation precedent.
  - Compatibility matrix: add rows `save_height_preset_3`,
    `save_height_preset_4` (all columns ⚠️ until field-verified), and
    `move_to_height_preset_3`, `move_to_height_preset_4`. For the recall
    rows mirror upstream v0.7.0's marks (✅ for the Uplift service
    `0x00FF` column — hardware-verified there; ⚠️ elsewhere). Add a short
    "Hardware validation" note line as upstream did ("Preset 3 and 4 recall
    verified on … with an official Uplift V3 adapter").
  - No changes needed to the notifications table (0x25–0x28 already
    documented) or release instructions.
- **Acceptance Criteria**:
  - `uv run --frozen mdformat --check README.md` and
    `uv run codespell README.md` pass with the repo's locked tool versions
    (pre-commit pins ruff 0.14.1 / mdformat 1.0.0 — use `uv run`, not a
    newer system install, to match CI).
  - Table content is factually consistent with code opcodes and with the
    verification-status claims made in docstrings.

### 5. Version bump
- **Task ID**: bump-version
- **Depends On**: update-readme
- **Assigned To**: preset34-builder
- **Agent**: builder
- **Actions**:
  - Edit `pyproject.toml`: `version = "0.5.2"` → `version = "0.6.0"`.
    (Feature ⇒ minor bump, matching every prior feature release in this
    project's history: 0.3.0 preset recalls, 0.5.0 new services,
    upstream 0.6.0 recall-3/4.)
  - Regenerate the lockfile so it pins the new version
    (`uv.lock` currently contains `name = "uplift-ble", version =
    "0.5.2"`): run `uv sync --group dev` (or `uv lock`) and confirm
    `uv.lock` now shows 0.6.0 for the project entry.
  - Commits (CI enforces exact formats — see
    `.github/workflows/commit-message-check.yml` and
    `branch-name-check.yml`; current branch `feat/save-preset-3-4` is
    valid):
      - `feat: add save commands for height presets 3 and 4`
        (controller + CLI + tests + README; lowercase, ≤72 chars)
      - optionally a separate
        `feat: add recall commands for height presets 3 and 4`
      - `chore: bump version to 0.6.0` (pyproject.toml + uv.lock together;
        this exact message format is required by CI — no "v" prefix)
- **Acceptance Criteria**:
  - `pyproject.toml` and `uv.lock` agree on version 0.6.0.
  - Commit messages pass the CI regexes (subject ≤72 chars, lowercase
    body after type; bump message matches
    `^chore: bump version to X.Y.Z$`).

### 6. Final Validation
- **Task ID**: validate-all
- **Depends On**: add-preset-34-cli, add-preset-34-tests, update-readme, bump-version
- **Assigned To**: preset34-validator
- **Agent**: validator
- **Checks**:
  - Run all validation commands (see below) in the worktree.
  - Verify each acceptance criterion from tasks 1–5.
  - Cross-check: opcodes in code == opcodes in tests == opcodes in README
    tables; version consistent across `pyproject.toml`/`uv.lock`.
  - Confirm no unrelated files changed and nothing was committed outside
    the worktree branch (do not push tags or releases — that is a
    maintainer action per README "Making a New Release").

### 7. Documentation
- **Task ID**: generate-docs
- **Depends On**: validate-all
- **Assigned To**: preset34-documenter
- **Agent**: documenter
- **Actions**:
  - Read the plan file and implementation files; re-read
    `README.md` end-to-end for consistency (protocol tables, compatibility
    matrix, warnings).
  - Fix any doc/code inconsistencies found (docs-only edits).
  - Draft a ready-to-paste GitHub release description for `v0.6.0`
    (upstream releases use auto-generated "What's Changed" notes; a short
    human note is optional) and save it next to this plan at
    `.agent/specs/001-save-preset-3-4/unimplemented/plan-1-save-preset-3-4/release-notes-0.6.0.md`,
    otherwise append it to this file under "Release Notes Draft".
  - Do not create an `app_docs/` tree — this repo documents protocol
    details only in `README.md`.

## Acceptance Criteria

1. `DeskController.save_height_preset_3()` and
   `save_height_preset_4()` exist, are async (via `@command_writer`), take
   no arguments, send opcodes `0x25`/`0x26` with empty payloads
   (`F1 F1 25 00 25 7E` / `F1 F1 26 00 26 7E`).
2. (Parity) `move_to_height_preset_3()/4()` send opcodes
   `0x27`/`0x28`, matching upstream v0.6.0+ naming and docstrings.
3. CLI exposes `save-height-preset-3`, `save-height-preset-4`,
   `move-to-height-preset-3`, `move-to-height-preset-4`.
4. New unit tests pass; full suite green (`uv run pytest`).
5. README documents the four opcodes with correct verification status
   (saves = pending/community-sourced; recalls = verified upstream).
6. Version is 0.6.0 in both `pyproject.toml` and `uv.lock`; commit
   messages conform to CI rules.
7. ruff check, ruff format --check, codespell, and mdformat (README) all
   pass with locked tool versions.

## Validation Commands

### Automated (run in the worktree after `uv sync --group dev`)

- `uv run pytest -v` — full test suite (CI runs plain `uv run pytest`).
- `uv run ruff check .`
- `uv run ruff format --check .`
- `uv run codespell`
- `uv run mdformat --check README.md`
- `git diff --check`

### Manual hardware verification of the save opcodes (opcode protocol)

This cannot be automated by the team; it requires a real desk + BLE adapter
and someone ready to stop the desk. **Safety first**: clear the desk path,
keep hands/feet clear, and have `stop_movement()` (`0x2B`) or the physical
stop handy. Unknown opcodes are generally ignored by the firmware, but do
NOT brute-force opcode ranges — at least one undocumented opcode (`0x12`)
is known to be potentially dangerous; probe only the candidates listed in
"Open Questions and Risks", one at a time.

Suggested script (run from repo root after `uv sync --group dev`; adjust
UUIDs if not on the FE60/FF00 variant — see `desk_configs.py`):

```python
import asyncio, logging
from bleak import BleakClient
logging.basicConfig(level=logging.DEBUG)
from uplift_ble.desk_configs import DESK_CONFIGS_BY_SERVICE
# pick the DeskConfig matching your adapter's service UUID
CONFIG = DESK_CONFIGS_BY_SERVICE["0000fe60-0000-1000-8000-00805f9b34fb"]

async def main():
    async with BleakClient("AA:BB:CC:DD:EE:FF") as client:
        if not client.is_connected:
            raise SystemExit("not connected")
        from uplift_ble.desk_controller import DeskController
        c = DeskController(client, CONFIG.input_char_uuid,
                           CONFIG.output_char_uuid,
                           requires_wake=CONFIG.requires_wake)
        await c.start()
        events = []
        for ev in ("HEIGHT_PRESET_3", "HEIGHT_PRESET_4"):
            getattr  # (register via c.on(DeskEventType.HEIGHT_PRESET_3, lambda *a: print("P3:", a)))
        async def save_and_read(slot: int):
            before = getattr(c, f"height_preset_{slot}")
            await getattr(c, f"save_height_preset_{slot}")()
            # Trigger preset reports (0x07 "settings/sync query"):
            await c.request_height_limits()
            await asyncio.sleep(2.0)
            after = getattr(c, f"height_preset_{slot}")
            print(f"preset {slot}: before={before} after={after}")
        # 1) Move to a known height A, wait for motion to stop:
        await c.move_to_specified_height(1050)   # mm; pick a safe value
        await asyncio.sleep(8.0)
        print("height_raw:", c.height_raw)
        await save_and_read(3)
        # 2) Move to a clearly different height B, wait, save again:
        await c.move_to_specified_height(1150)
        await asyncio.sleep(8.0)
        await save_and_read(3)
        # 3) Recall and watch HEIGHT events return near A
        await c.move_to_height_preset_3()
        await asyncio.sleep(10.0)
        print("height_raw after recall:", c.height_raw)
        # 4) Repeat steps for slot 4, then stop:
        await c.stop_movement()
        await c.stop()

asyncio.run(main())
```

Decision criteria (robust to the "units vary by hardware" caveat — do not
assert absolute equality against `height_mm`):
- **PASS** if `height_preset_3/4` changes when saved at two different
  heights, is stable across repeated saves at one height, and recall moves
  the desk back toward the saved position (observable in raw
  `HEIGHT`/height_raw samples).
- **FAIL/inert** if preset values never change after candidate save
  commands; then retry with fallback candidates per "Open Questions and
  Risks".
- Record which opcodes worked, on which service variant/firmware
  (`uplift-ble-cli about`, adapter label), in the PR description — this
  becomes the compat-matrix ✅ evidence (project precedent: upstream
  issues #52/#53).
- After a verified PASS: flip the relevant matrix cells from ⚠️ to ✅ and
  add the hardware-validation note in a follow-up commit/PR.

## Notes

- **Upstream parity / merge hygiene**: this fork is at v0.5.2 while
  upstream librick/uplift-ble is at v0.7.0 (which already contains
  `move_to_height_preset_3/4`). Using the exact same method names, opcodes,
  and docstrings as upstream minimizes future merge conflicts when this
  fork syncs. The save methods (`0x25`/`0x26`) are new to both repos — if
  they get verified, consider contributing them back upstream (open an
  issue/PR on librick/uplift-ble referencing the community sources).
- **Release process** (when the maintainer is ready; not part of this
  plan's commits): branch `release/0.6.0` (CI forbids a "v" prefix in
  release branch names — CONTRIBUTING.md's `release/vX.Y.Z` example
  contradicts CI and must be ignored), PR to main, merge, tag
  `git tag -a v0.6.0 -m "Release v0.6.0"`, GitHub Release (auto-generated
  notes), then `uv build && uv publish`. There is no CHANGELOG file in
  this repo; release notes live only on GitHub Releases/PyPI.
- **Preset semantics** (from upstream PR #46): the vendor app stores
  presets on the phone, not in the adapter — desks configured only via the
  app may have empty slots (`0x00` reports), which is exactly why
  BLE-side saves are useful.
- **Extended-preset hardware support**: Bennett-Wendorf's
  `hass-uplift-desk` integration gates preset 3/4 recall to service
  variants `0x00FF`, `0xFE60`, `0xFF00` (excludes `0xFF12`). Expect the
  same split for saves; keep matrix cells ⚠️ per variant until tested.
- **Opcode reuse across directions** is normal in this protocol:
  outgoing save-3 (`0x25`) shares its opcode with the incoming preset-1
  *report*; direction (write to `0xFE61`/input char vs notification on
  `0xFE62`/output char) disambiguates — same as upstream's verified
  recall/report reuse of `0x27`/`0x28`.

## Open Questions and Risks

**OQ-1 (primary): Save opcodes for presets 3/4 are community-sourced, not
officially verified.** Recommended: `0x25` (save preset 3),
`0x26` (save preset 4) — confidence **medium-high** (two independent
sources; structural consistency with hardware-verified recall opcodes
`0x27`/`0x28`; literal packet corroboration from a Jiecang app
decompilation). Until the manual verification above passes on real
hardware, treat them as unproven: keep ⚠️ in the compatibility matrix and
the "pending hardware verification" docstring note. If `0x25`/`0x26` are
inert, test candidates in this order (one at a time, with the read-back
protocol; stop immediately if anything unexpected happens to desk state):
1. `0x29` / `0x2A` — "next free" opcodes after the recall block
   (plausible but no source);
2. Opcode `0xAD` ("SET MEMORY HEIGHT", from Jiecang app decompilation)
   with payload `[slot, hi, lo]` absolute heights — note it is documented
   only for slots 1–3 in that firmware and may not exist on Uplift
   hardware; would imply a different API shape (`set_preset_height(slot,
   height)`) rather than save-current-height;
3. Presets 3/4 may be keypad-only on some firmwares (no BLE save path at
   all). In that case document the limitation and consider dropping the
   methods behind an explicit "unsupported" note rather than shipping
   dead commands.

**R-1: Safety.** These commands move a heavy desk. Unknown-opcode probes
must be limited to the documented candidates above; never sweep opcode
ranges (e.g., `0x12` is known potentially dangerous). All hardware testing
per the safety preconditions in "Validation Commands".

**R-2: Hardware/firmware variance.** Behavior may differ across service
variants (`0x00FF`/`0xFE60`/`0xFF00`/`0xFF12`) and firmware revisions;
preset report units are explicitly "unknown/varies" per README. Tests must
not assert absolute preset values, only relative change + recall behavior
on hardware.

**R-3: Report-trigger uncertainty.** Writing `0x07`
(`request_height_limits()`) returns preset reports on V3/FF00-class
hardware (kwalb field data) but is documented upstream as "request height
limits" with a `0x07` response — on other desks it may not emit
`0x25`–`0x28` at all. The verification protocol therefore also accepts
preset reports arriving spontaneously after save/recall actions; if no
reports arrive, fall back to the recall-then-watch-height check as the
primary evidence.

**R-4: Fork drift.** This checkout predates upstream v0.6.0/v0.7.0
(recalls 3/4, fallback units). The parity task here re-adds recalls with
upstream's exact names/opcodes to keep future syncs clean; if the team
decides instead to merge upstream history first, drop the recall portion
of task 1 and keep only the save methods.

**R-5: No ack semantics known.** Save commands appear fire-and-forget
(no direct acknowledgement documented). The `@command_writer` wrapper's
post-write notification wait (default 1 s) is retained for consistency;
no timeout/retry logic is added since there is nothing to retry against.
