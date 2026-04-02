# gt-tools

CLI tools for [Graphite](https://graphite.dev) stacked PRs.

---

## What's included

| Tool | What it does |
|------|-------------|
| `gt-drain` | Merges a Graphite stack bottom-up, one PR at a time, safely |
| `gt-rescue` | Restores branches from snapshots taken before any drain |

---

## The problem

Graphite is great for stacked PRs. But when you try to merge a deep stack, the standard workflow falls apart fast:

- Enqueue multiple PRs at once → the first merge kicks off a restack on the rest → merge conflicts cascade
- You resolve one conflict, push, and now the next PR is broken
- By the time PR #5 is ready, you've spent an hour fire-fighting rebases

`gt-drain` fixes this by merging **one PR at a time**, waiting for each merge to complete before restacking and submitting the next one. No conflict cascade. No manual babysitting.

---

## Install

### Homebrew (recommended)

```bash
brew tap john-farina/gt-tools
brew install gt-drain
```

That's it. Both `gt-drain` and `gt-rescue` are installed.

### Manual

```bash
curl -fsSL https://raw.githubusercontent.com/john-farina/gt-tools/main/gt-drain -o /usr/local/bin/gt-drain
curl -fsSL https://raw.githubusercontent.com/john-farina/gt-tools/main/gt-rescue -o /usr/local/bin/gt-rescue
chmod +x /usr/local/bin/gt-drain /usr/local/bin/gt-rescue
```

### Requirements

- [Graphite CLI](https://graphite.dev/docs/graphite-cli) (`gt`)
- [GitHub CLI](https://cli.github.com) (`gh`) — authenticated with your GitHub account
- Bash 5+ (macOS ships with Bash 3; Homebrew installs a modern version)

---

## Update

```bash
brew upgrade gt-drain
```

`gt-drain` also checks for updates automatically in the background once per day and prints a notice when a new version is available.

---

## Usage

### gt-drain

```
gt-drain                    Merge all branches in the current stack
gt-drain branch1 branch2    Merge specific branches in order
gt-drain --prep             Restack + submit only, no merge
gt-drain --status           Show merge status of in-flight PRs
```

**Flags**

```
--no-wait      Enqueue all PRs with --merge-when-ready, then exit (don't poll)
--continue     Resume after manually resolving a restack conflict
--no-squash    Skip squashing multi-commit branches before restack
--force        Continue even if post-restack drift is detected
--clear-cache  Wipe all saved state and snapshots (~/.gt-drain)
```

---

## Use cases

### Draining your full stack

The most common workflow. You're on any branch in your stack, all PRs are approved, and you want to merge everything cleanly:

```bash
gt-drain
```

`gt-drain` will:
1. Snapshot every branch's diff + SHA (safety net)
2. Squash multi-commit branches down to one commit
3. Sync + restack the whole stack
4. Submit updated PRs to GitHub
5. Merge the **bottom** PR
6. Wait for the merge to land
7. Restack everything above it
8. Repeat up the stack

You walk away and come back to a fully merged stack.

---

### Draining just part of a stack

You only want to merge the bottom two branches and leave the rest open:

```bash
gt-drain feat/auth-base feat/auth-tokens
```

Branches are merged in the order you specify.

---

### Prep without merging

You want to get the stack clean and submitted (all PRs up to date) but aren't ready to merge yet:

```bash
gt-drain --prep
```

Useful before a code review session — gets everything rebased and submitted so reviewers see the latest.

---

### Fire and forget

You want to kick off the drain and not wait at your terminal:

```bash
gt-drain --no-wait
```

All PRs are enqueued with `--merge-when-ready`. They'll merge sequentially as CI passes. Note: this doesn't handle the post-merge restack between each PR — use the full drain if your stack needs that.

---

### Recovering from a conflict mid-drain

If a restack conflict stops `gt-drain` mid-run:

1. Resolve the conflict manually
2. Run `gt-drain --continue`

The drain resumes from where it left off.

---

## Safety net: gt-rescue

Before every run, `gt-drain` snapshots every branch in your stack to `~/.gt-drain/snapshots/`. If anything goes wrong, `gt-rescue` gets you back.

```bash
gt-rescue                          # List all snapshots
gt-rescue show --last              # Show what the most recent snapshot contains
gt-rescue diff --last              # Compare snapshot state vs current state
gt-rescue restore --last           # Restore all branches to snapshot state
gt-rescue restore --last feat/foo  # Restore a single branch
gt-rescue clean                    # Delete old snapshots (keeps last 10)
```

---

## How it works

```
Stack:  main ← feat/A ← feat/B ← feat/C
```

Standard approach (broken):
```
Enqueue A, B, C → A merges → B and C rebase fails → 💥
```

`gt-drain` approach:
```
1. Snapshot all branches
2. Restack + submit entire stack
3. Merge A → wait for GitHub merge
4. Restack B and C onto main
5. Submit B → merge B → wait
6. Restack C onto main
7. Submit C → merge C → done ✓
```

---

## Uninstall

```bash
brew uninstall gt-drain
brew untap john-farina/gt-tools
```

---

## License

MIT
