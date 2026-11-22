# wsync

**Manual, git-style folder sync over SSH — with a hardlinked snapshot time-line. Zero daemons, zero databases, zero magic: everything you can lose is a folder with a date on it.**

`wsync` keeps a remote folder and a local *workingdir* in sync, one human at a time, one deliberate step at a time. Unlike backup tools, every state that a destructive operation is about to overwrite is photographed **before** it is destroyed — and the photos are browsable directories, not archives.

## Why

- You work on files that live on another machine (NAS, home server, work box).
- You want *redundancy without restores*: a copy — and a history — you can just `cd` into.
- You hate background sync daemons racing you while you edit.
- Git is too heavy or not available on the other side; rsync mirror alone is too dangerous.

`wsync` is the middle point: rsync transport, git *mental model* (baseline / my changes / their changes), and a snapshot ribbon that makes every mistake reversible.

## The world, on disk

```
wsync-folder/
├── wsync                  ← the whole tool (one bash script, constants on top)
├── workingdir/            ← where you actually work
├── server-<ts>/           ← 📷 photograph of the SERVER (taken at pull, and at diff when drift appears)
├── server-<ts>-E/         ← 🚨 server files "victimized" during a push (recovered from rsync --backup)
├── push-<ts>/             ← 🏁 the reconciled world, dated at push time
└── local-<ts>/            ← 🛟 emergency save of un-pushed work
```

All snapshots are **hardlink-based** (`rsync --link-dest`): identical files across snapshots share the same disk blocks, so dozens of snapshots of a text project cost almost nothing.

## The cycle

```
PULL    server → workingdir            + 📷 snapshot R₁ (this cycle's baseline)
WORK    edit anything in workingdir
STATUS  your changes vs R₁             (purely local, instant, no network)
DIFF    server drift vs last photo     (+ 📷 automatic snapshot if the server moved)
PUSH    workingdir → server            (guardians + victim backup) + 🏁 snapshot
```

The only risky window in this design — the server changing between your last `diff` and your `push` — is closed *inside* the transfer itself: `rsync --backup --backup-dir` moves every file it is about to overwrite or delete into a staging area, which `wsync` then rescues as a `server-<ts>-E` snapshot and cleans up. Nothing gets clobbered silently, ever.

## Commands

| Command | What it does |
|---|---|
| `wsync init` | Verify host/remote path, create `workingdir/` |
| `wsync pull` | Sync server → workingdir. Guardian: refuses (or emergency-saves) if you have un-pushed local changes |
| `wsync status` | Your changes vs the cycle baseline. Local-only, instant |
| `wsync diff` | Server drift vs last photo (auto-photographed when found) + status |
| `wsync push` | Sync workingdir → server with victim backup, then a 🏁 dated snapshot |
| `wsync show <file>` | Unified diff of one file (photo ↔ workingdir, or between two snapshots) |
| `wsync hist` | The full ribbon: 🏁 milestones, 📷 photos, 🚨 incidents, 🛟 rescues |
| `wsync diff-snap A B` | What changed between any two snapshots |
| `wsync keep [N]` | Prune, keeping the newest N snapshots |
| `wsync repair` | Recover or discard staging leftovers from an interrupted push |

Non-interactive use: set `WSYNC_YES=1` to auto-confirm prompts.

## Safety design

| Threat | Mitigation |
|---|---|
| `pull` overwriting un-pushed work | dirty-guardian vs cycle baseline + emergency `local-<ts>` snapshot |
| `push` clobbering files you never saw | drift-guardian refuses until you acknowledge |
| server changing *during* the push | `--backup-dir`: overwritten/deleted files are rescued into `server-<ts>-E` atomically at transfer time |
| crashed push leaving debris | `wsync repair` finds and recovers/discards staging dirs |
| "what did it look like back then?" | every state is a plain directory: `cd`, `diff -r`, `wsync diff-snap` |

## Install

1. Copy the `wsync` script into its own folder (that folder becomes the sync node).
2. Edit the constants at the top: `SSH_HOST` (an `~/.ssh/config` entry is strongly recommended; empty = local path mode) and `REMOTE_DIR`.
3. `chmod +x wsync && ./wsync init`
4. `./wsync pull` — first sync copies everything; afterwards all snapshots are incremental/hardlinked.

Requirements: `bash`, `rsync` on **both** ends, `ssh` (with key auth; password auth works but you'll type it on every command). Nothing else — no server-side agent, no sftp, no database, works fine on minimal/atomic distros.

## Design notes

- **Single-writer model.** wsync assumes one human working through the pull→work→push cycle. Two people editing the same project at the same time want git, not this.
- **Baseline = latest non-`-E` snapshot.** There is no privileged "base" directory: history is a flat ribbon of dated folders, and semantics come from the name prefix.
- **Chronology is sorted by the timestamp embedded in names**, not lexically (`push-…` must not sort after `server-…`).
- **`-E` snapshots are partial by nature** (victims only) and are therefore excluded from drift baselines.
- The `.wsync-cycle` pointer file remembers which photo is the current cycle's baseline.

## License

MIT — do whatever, attribution appreciated.
