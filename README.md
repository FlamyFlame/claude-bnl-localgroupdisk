# bnl-localgroupdisk

Claude Code plugin for migrating ROOT files to `BNL-OSG2_LOCALGROUPDISK` on BNL SDCC.

## What it does

Moves ROOT analysis files from local/personal pnfs storage to the ATLAS
LOCALGROUPDISK RSE at BNL, then creates a symlink farm so your analysis code
reads them transparently — no code changes, no grid proxy needed at runtime.

### The problem

LOCALGROUPDISK is permanent, backed-up storage managed by Rucio, but:
- You can't write directly to it (must upload to scratchdisk first, then replicate)
- It stores files in hash-based subdirectories, breaking your directory layout
- The procedure involves ~7 Rucio commands with easy-to-forget flags and names
- After migration, analysis code may need path updates if the symlink farm
  is placed at a different location than the original files

### The solution

This plugin encodes the full procedure into reusable skills that handle
pre-flight checks, upload, replication, monitoring, symlink farm creation,
and optional codebase adaptation with safe rollback.

## Requirements

- BNL SDCC account with ATLAS environment (`/cvmfs/atlas.cern.ch/`)
- Valid grid proxy: `voms-proxy-init -voms atlas -valid 96:00`
- `/atlas/usatlas` VOMS group membership (for LOCALGROUPDISK quota)
  - Request at https://atlas-auth.cern.ch/ if you don't have it
  - Verify: `voms-proxy-info --all` should show `/atlas/usatlas`
- Rucio account with scratchdisk and LOCALGROUPDISK quotas

## Installation

From GitHub:

```bash
git clone git@github.com:FlamyFlame/claude-bnl-localgroupdisk.git
claude --plugin-dir claude-bnl-localgroupdisk
```

Or add to your project's `.claude/plugins/` for persistent use:

```bash
git clone git@github.com:FlamyFlame/claude-bnl-localgroupdisk.git .claude/plugins/bnl-localgroupdisk
```

## Recommended settings

The plugin runs Rucio commands, creates symlinks, and (optionally) edits
source code. To avoid excessive permission prompts, add these to your
project's `.claude/settings.json`:

```jsonc
{
  "permissions": {
    "allow": [
      // Rucio & grid environment
      "Bash(rucio *)",
      "Bash(voms-proxy-info *)",
      "Bash(lsetup *)",
      "Bash(source */atlasLocalSetup.sh*)",
      "Bash(export ATLAS_LOCAL_ROOT_BASE=*)",

      // File operations (survey, symlink farm, verify)
      "Bash(ls *)",
      "Bash(du *)",
      "Bash(wc *)",
      "Bash(head *)",
      "Bash(ln -s *)",
      "Bash(mv *)",
      "Bash(mkdir *)",
      "Bash(rm *)",
      "Bash(cat *)",
      "Bash(echo *)",

      // ROOT verification
      "Bash(root -b *)",

      // Replication monitoring (for long FTS waits)
      "Monitor(*)",

      // Full integration path only — read/edit source code
      // Scope these to your repo and data paths:
      // "Read(/path/to/your/repo/**)",
      // "Edit(/path/to/your/repo/**)",
      // "Bash(git *)"
    ],
    "deny": [
      // Safety: don't delete original data or init proxies without asking
      "Bash(rm -rf /pnfs/*)",
      "Bash(voms-proxy-init *)"
    ]
  }
}
```

**Minimal setup** (same-path swap, no code changes): only the Rucio, file
operations, and ROOT verification lines are needed.

**Full integration** (different path + code changes): uncomment the
Read/Edit/git lines and scope them to your repository path.

## Quick start

The typical workflow is one command:

```
/bnl-localgroupdisk:migrate /pnfs/usatlas.bnl.gov/users/<you>/my_data my_dataset
```

This runs the full procedure end-to-end: pre-flight checks, upload, replication,
symlink farm, and (optionally) codebase path updates. It asks you interactive
questions at decision points — you don't need to run the other skills separately.

To skip all prompts and use defaults (upload + symlink, same-path swap):

```
/bnl-localgroupdisk:migrate /pnfs/usatlas.bnl.gov/users/<you>/my_data my_dataset, no confirmation
```

The other skills (`preflight`, `check-rule`, `build-symlinks`) exist for
partial workflows — e.g., you already uploaded but need to re-check a rule,
or you already have data on LOCALGROUPDISK and just need symlinks.

## Skills

### `/bnl-localgroupdisk:migrate <source_dir> <dataset_name>`

**Primary skill — handles everything.** End-to-end migration with interactive
decision points:

1. Runs pre-flight checks (Rucio account, proxy, RSE names, quotas, pnfs mount)
2. Uploads files to scratchdisk, creates dataset, adds replication rule
3. Waits for replication to complete
4. Asks: **upload + symlink swap** (default) or **upload only**?
5. If symlink: asks **same-path swap** (default) or **different path**?
6. If different path: asks **full integration** (scan code, update paths,
   test, rollback-safe) or **symlink only**?

**Same-path swap** (default for most users): renames original directory to
`<dir>_orig`, places symlink farm at the original path. Analysis code works
without any changes.

**Full integration** (different path + code changes): creates a git branch,
searches your codebase for path references (including constructed paths like
`base_dir + subdir`), proposes edits, tests compilation and TTree entry
counts, rolls back on failure. Before touching your git state, asks whether
to commit your current work or stash it.

Example:
```
/bnl-localgroupdisk:migrate /pnfs/usatlas.bnl.gov/users/<username>/my_data my_dataset
```

### `/bnl-localgroupdisk:preflight`

Run pre-flight checks only: Rucio account, proxy, quotas, RSE names, pnfs
mount. Use this if you want to verify your environment before starting, or
if you're troubleshooting a failed migration. (`migrate` runs these
automatically, so you don't need to call this separately in the normal flow.)

### `/bnl-localgroupdisk:check-rule <rule_id>`

Monitor a Rucio replication rule. Use if your session was interrupted during
replication and you need to check whether it completed, or if you want to
monitor a rule from a previous migration.

### `/bnl-localgroupdisk:build-symlinks <scope:dataset_name> <farm_dir>`

Build a symlink farm from an already-replicated dataset. Use if you already
have data on LOCALGROUPDISK (e.g., from a previous migration or a colleague's
upload) and just need the local symlinks.

## How it works

```
Local files ──rucio upload──> BNL-OSG2_SCRATCHDISK
                                     │
                              rucio add-rule
                                     │
                                     v
                          BNL-OSG2_LOCALGROUPDISK
                                     │
                              PFN extraction
                                     │
                                     v
                    ┌── Decision Point 1 ──────────────┐
                    │                                   │
                Upload only                    Upload + symlink swap
                  (done)                                │
                                     ┌── Decision Point 2 ──────────────┐
                                     │                                   │
                              Same-path swap                      Different path
                              (rename _orig,                             │
                              farm at source_dir,            ┌── Decision Point 3 ──┐
                              no code changes)               │                       │
                                                      Symlink only          Full integration
                                                        (done)            (scan code, update
                                                                          paths, test, rollback)
```

Key facts:
- Source files are **never modified or deleted** during upload
- **Same-path swap** (default): renames original dir to `_orig`, places symlinks at the original path — zero code changes needed
- **Full integration**: creates a git branch for rollback, scans codebase for path references, updates code, tests in a temp directory (never overwrites original outputs), rolls back on failure
- LOCALGROUPDISK files live at `/pnfs/usatlas.bnl.gov/LOCALGROUPDISK/rucio/user/<account>/<hash>/<hash>/<file>`
- No grid proxy needed to read files via symlinks on SDCC nodes
- FTS replication queue wait: 1–12 hours (user priority); actual transfer: minutes
- Default quota: 50 TB per user

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `insufficient quota` on `add-rule` | Not in `/atlas/usatlas` VOMS group | Request membership at https://atlas-auth.cern.ch/ |
| `Database error` on `upload` | File DID already registered in Rucio | Check `rucio list-dids`; reuse existing DID or rename |
| Replication stuck at 0/N for hours | Normal FTS queue delay | Wait; check with `/bnl-localgroupdisk:check-rule` |
| Symlinks don't resolve | pnfs mount not available | Verify `/pnfs/usatlas.bnl.gov/LOCALGROUPDISK/` exists |
| `voms-proxy-init` with `/atlas/usatlas` fails | Not authorized for group | Request at ATLAS IAM |

## Worked example

Suppose you have 25 Monte Carlo ROOT files (98 GB total) at
`/pnfs/usatlas.bnl.gov/users/<you>/mc_sample/` and want them on
LOCALGROUPDISK with a symlink farm at the original path.

### 1. Pre-flight

```
/bnl-localgroupdisk:preflight
```

Expected output (all OK):

| Check | Status | Details |
|-------|--------|---------|
| Rucio account | OK | `<your_account>` |
| Grid proxy | OK | 95h remaining, `/atlas/usatlas` present |
| RSE names | OK | `BNL-OSG2_LOCALGROUPDISK`, `BNL-OSG2_SCRATCHDISK` |
| LGD quota | OK | 50 TB limit |
| pnfs mount | OK | `/pnfs/usatlas.bnl.gov/LOCALGROUPDISK/` accessible |

**Common failure:** LGD quota shows "NO LGD QUOTA" — you need `/atlas/usatlas`
VOMS group membership. Request at https://atlas-auth.cern.ch/, then re-init
your proxy with `voms-proxy-init -voms atlas:/atlas/usatlas -valid 96:00`.

### 2. Migrate

```
/bnl-localgroupdisk:migrate /pnfs/usatlas.bnl.gov/users/<you>/mc_sample mc_sample_truth
```

What happens step by step:

1. **Survey**: reports 25 files, 98 GB. Asks you to confirm.
2. **DID check**: verifies filenames aren't already registered in Rucio.
   - If they are (e.g., from a previous grid job): skill stops and offers
     options (reuse existing DID or rename).
3. **Upload**: `rucio upload --rse BNL-OSG2_SCRATCHDISK` — ~30s per 4 GB file,
   ~12 min total. Source files are NOT touched.
4. **Dataset**: creates `user.<you>:mc_sample_truth`, attaches all 25 files.
5. **Replication rule**: adds rule to `BNL-OSG2_LOCALGROUPDISK`.
   - Returns a rule ID (e.g., `86adb3d1...`).
   - **This is the slow step.** FTS queue wait: 1–12 hours. 0/25 locks for
     hours is normal (user-priority transfers are deprioritized). Actual
     data transfer once FTS picks it up: ~5 min for 100 GB.
6. **Decision point 1**: asks upload-only or upload+symlink. Choose symlink.
7. **Decision point 2**: asks same-path swap or different path. Choose same-path.
8. **Symlink farm**: renames original dir to `mc_sample_orig`, creates symlink
   farm at the original path. Each symlink points to:
   `/pnfs/usatlas.bnl.gov/LOCALGROUPDISK/rucio/user/<you>/<hash>/<hash>/<file>.root`
9. **Verify**: confirms 25 symlinks, spot-checks one resolves, ROOT opens it OK.

### 3. After migration (same-path swap)

- Your analysis code reads from the same path as before — no changes needed.
- Original files preserved at `mc_sample_orig/` — delete when satisfied.
- No grid proxy needed to read via symlinks on SDCC.
- To check the rule later: `/bnl-localgroupdisk:check-rule <rule_id>`

### Alternative: full integration (different path + code changes)

If you choose "different path" at decision point 2 and "full integration" at
decision point 3, the skill additionally:

1. **Saves your git state**: asks whether to commit current work (recommended)
   or stash it. If commit, the agent commits for you with a WIP message.
2. **Creates a migration branch**: `lgd-migrate-<dataset>` off your current branch.
3. **Searches your codebase**: finds path references — not just literal strings,
   but also constructed paths (e.g., `base_dir + subdir`), config files, and
   filename patterns.
4. **Proposes edits**: classifies each hit (literal, variable, config, etc.)
   and proposes the appropriate edit. Asks for confirmation before applying.
5. **Tests automatically**: compares TTree entry counts between original and
   farm, attempts compilation of modified code, runs a minimal test in a temp
   directory (never overwrites your outputs).
6. **Finalizes or rolls back**: on success, commits and restores stash (handling
   merge conflicts if needed). On failure, reverts all code changes, deletes the
   branch, restores stash, and reports what went wrong.

If no code references are found, the skill asks you which file defines the
path. If you don't know, it exits with clear manual instructions and leaves
the symlink farm intact.

### What can go wrong

| Step | Problem | What the skill does |
|------|---------|---------------------|
| 3 | `Database error` (duplicate DID) | Stops, offers reuse or rename options |
| 5 | `insufficient quota` | Stops, directs to ATLAS IAM for VOMS group |
| 5 | Rule stuck at 0/N for 12+ hours | Normal; skill documents expected timing |
| 8 | Symlinks don't resolve | Skill checks pnfs mount; reports if unavailable |
| 8 | Crash between rename and symlink build | Farm built in staging dir first, then atomic swap |
| 9 | ROOT can't open file | Reports failure; original at `_orig` is intact |
| (full) Code search finds nothing | Asks user; if no answer, exits with manual instructions |
| (full) Test fails after code edits | Rolls back all changes, deletes branch, restores stash |
| (full) Stash pop has merge conflict | Reports conflicting files, asks user to resolve |

## Tested

Pilot-tested May 2026 on BNL SDCC with 25 ROOT files (98 GB).
Full analysis (TChain, 1M events) ran successfully on symlink farm.
