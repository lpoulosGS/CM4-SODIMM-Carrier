---
name: kicad
description: Use whenever working in this repo (a KiCad 10 project) — editing/reviewing .kicad_sch, .kicad_pcb, .kicad_pro, .kicad_prl, fp-lib-table, sym-lib-table, or footprint/symbol .pretty libraries; investigating "KiCad conflicts" (duplicate " - Copy" files, "_restore_backup_*" folders); resolving git diffs/merge conflicts in KiCad files; or running ERC/DRC checks. Triggers on: kicad, .kicad_sch, .kicad_pcb, .kicad_pro, .kicad_prl, fp-lib-table, sym-lib-table, .pretty, footprint, schematic, PCB, ERC, DRC, netlist, "kicad conflict", "restore backup".
---

# KiCad project conventions (CM4-SODIMM-Carrier)

## File formats
All KiCad 10 project files are plain-text S-expressions (Lisp-like `(key value)` trees) — readable and diffable, but verbose:
- `.kicad_pro` — project settings (JSON, not S-expr)
- `.kicad_prl` — per-user local project state (open panels, UI column widths, selected layers). **Diffs here are almost always cosmetic UI state, not design changes.** Don't spend review time on these unless something functional (like net class visibility) changed.
- `.kicad_sch` — schematic sheet. Large diffs are common even for small visible edits, since every symbol instance carries a UUID and full property list.
- `.kicad_pcb` — board file (footprints, tracks, zones, layers). Same UUID-heavy verbosity as schematics.
- `fp-lib-table` / `sym-lib-table` — project-level footprint/symbol library tables. Format: `(fp_lib_table (version 7) (lib (name "...") (type "KiCad") (uri "...") (options "") (descr "...")))`.
- `.pretty` folders — footprint libraries (one `.kicad_mod` file per footprint). `.kicad_sym` files are symbol libraries.

**Path convention:** library `uri` fields should use `${KIPRJMOD}/RelativePath.pretty` (project-root-relative) for anything checked into the repo, never an absolute path like `C:/Users/<name>/Downloads/...` — those break for every other machine/user. Absolute paths pointing outside the repo (e.g. to a teammate's GitHub checkout) are a known pre-existing wart in this project (`CM4IO` lib) — flag but don't silently "fix" without asking, since it may be intentional shared-library plumbing.

## The "KiCad conflict" recovery pattern
When KiCad has a file open and that file changes on disk underneath it (e.g. `git pull`, `git checkout`, or another KiCad instance saving), KiCad will not silently discard your in-memory edits. On next save/close it produces recovery artifacts instead of overwriting:
- `<name> - Copy.<ext>` — KiCad's in-memory version saved under a new name
- `_restore_backup_<timestamp>/` — timestamped backup folders of the whole project state

These are **not git merge conflicts** (no `<<<<<<<` markers, no unmerged paths in `git status`) — they're KiCad-side artifacts. Resolution approach:
1. `git status` first — if there's no "Unmerged paths" section, this isn't a git conflict at all.
2. Diff each `- Copy` file against its tracked counterpart to see what actually differs before deleting anything.
3. Once the correct version is confirmed and kept, delete the `- Copy` files and `_restore_backup_*` folders — they're not meant to be committed.
4. To prevent recurrence: close KiCad (or at least the affected file) before running git operations that touch tracked KiCad files.

## Reviewing diffs
- Treat `.kicad_prl` diffs as low-priority UI noise unless a reviewer flags something specific.
- For `.kicad_sch`/`.kicad_pcb`, a large textual diff doesn't imply a large design change — check for actual property/position/net changes rather than line count. When in doubt whether a diff is meaningful, it's fine to say so and suggest opening the file in KiCad to visually confirm, rather than over-interpreting raw S-expressions.
- `.gitattributes` in this repo normalizes line endings; `warning: LF will be replaced by CRLF` on `git add`/`diff` is expected noise, not an error.

## kicad-cli (headless checks)
Installed at `C:\Users\GS General\AppData\Local\Programs\KiCad\10.0\bin\kicad-cli.exe` (not on PATH by default — invoke with the full path, or add it to PATH). Useful for verifying changes without opening the GUI:
- `kicad-cli sch erc <file>.kicad_sch` — Electrical Rules Check, writes a report
- `kicad-cli pcb drc <file>.kicad_pcb` — Design Rules Check, writes a report
- `kicad-cli sch export netlist|pdf|bom ...` / `kicad-cli pcb export gerbers|drill|pos ...`
- `kicad-cli pcb render` — 3D render to PNG/JPEG, useful for visually sanity-checking a board diff without opening the full GUI

Run ERC/DRC after any change that touches nets, footprints, or library references (e.g. after adding a footprint library like `MCP25625T_E_ML.pretty`) to catch broken references headlessly.
