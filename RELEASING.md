# Releasing ezmvc/cli

Versioning is driven by **[Conventional Commits](https://www.conventionalcommits.org)** +
**[cocogitto](https://docs.cocogitto.io/)** (`cog`), with a manual release trigger.

**The git tag is the single source of truth for the CLI version.** Nothing in the
source holds a copy of it, and `composer.json` has no `version` field (Packagist derives it
from the tag).

---

## One-time setup (per machine)

`cog` is a single self-contained binary — no runtime needed. It is **not** on winget, so grab
the release binary:

1. Download the Windows build from
   <https://github.com/cocogitto/cocogitto/releases> (`cocogitto-<ver>-x86_64-pc-windows-msvc.tar.gz`),
   extract `cog.exe`, and put it in a folder that's on your `PATH`
   (e.g. `%LOCALAPPDATA%\Microsoft\WindowsApps`, or a `~/bin` you've added to PATH).
   *(Alternatively, if you ever install Rust: `cargo install cocogitto`.)*
2. Confirm it's found: `cog --version`
3. From the repo root, install the git hooks:
   ```
   cog install-hook --all
   ```
   This installs a **commit-msg** hook that rejects any message which isn't a valid
   Conventional Commit. (The hook calls `cog`, which is why step 1 must come first.)
   Skip this step when `.git/hooks/commit-msg` already exists, for instance from a git
   template (`init.templateDir`): `cog install-hook` overwrites it.

---

## Day-to-day: branching & commits (GitHub Flow)

- Every change goes on its own branch off `main`: `git switch -c feat/short-name`.
- Merge back into `main` **preserving individual commits** (merge commit or rebase-merge —
  not squash), because the changelog is built from the individual commit messages.
- Merge commits themselves never become changelog entries (`ignore_merge_commits`).

### Commit message format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

| Type | In changelog? | Version effect | Use for |
|------|:---:|---|---|
| `feat`     | ✅ Features   | minor bump | a new capability |
| `fix`      | ✅ Bug Fixes  | patch bump | a fix to released behaviour |
| `perf`     | ✅            | patch bump | performance improvement |
| `revert`   | ✅            | patch bump | reverting a previous change |
| `chore`    | ❌ hidden     | none | tidy-up, deps, "fixing a feat that isn't released yet" |
| `refactor` | ❌ hidden     | none | internal restructure, no behaviour change |
| `docs`     | ❌ hidden     | none | docs / comments |
| `test`     | ❌ hidden     | none | tests |
| `ci`/`build`/`style` | ❌ hidden | none | pipeline / formatting |

**Breaking changes** — append `!` after the type or add a `BREAKING CHANGE:` footer:
```
feat!: rename `ezmvc generate` to `ezmvc build`
```
Pre-1.0 this bumps the **minor** (0.x); post-1.0 it bumps the **major**.

> Tip for in-branch corrections: `git commit --fixup=<sha>` then
> `git rebase -i --autosquash` before merging — the noise never reaches `main`.

---

## Cutting a release (manual)

Nothing is ever committed directly to `main`, a release included. `cog bump` runs only on a
branch named `chore/release` (the one entry in `cog.toml`'s `branch_whitelist`), where it
writes `CHANGELOG.md`, commits it and tags that commit; a pull request brings it into `main`.

```bash
git switch main
git pull
git switch -c chore/release
cog bump --auto --dry-run              # prints the next version, e.g. 0.2.0
cog bump --auto                        # writes CHANGELOG.md, commits it, tags v0.2.0
git push -u origin chore/release
```

Open a pull request and merge it with **"Create a merge commit"**. Rebase-merge and squash
both replace the tagged commit with a new one, which would leave the tag pointing at a commit
that is not on `main`. Once merged, publish the tag and delete the branch, so the next
release starts it fresh from `main`:

```bash
git push origin v0.2.0
git switch main
git pull
git branch -d chore/release
git push origin --delete chore/release   # unless GitHub already deleted it after the merge
```

- Preview the release notes first with `cog changelog` (shows the unreleased entries).
- Force a specific version instead of `--auto`: `cog bump --version 1.0.0`
  (or `--major` / `--minor` / `--patch`).

### After the tag is pushed

- Create the **GitHub Release** for the tag; paste that version's section from `CHANGELOG.md`
  as the body (or generate it with `cog changelog --at vX.Y.Z`).
- **Packagist** picks up the new version automatically from the tag — no `composer.json`
  edit needed.

---

## Version numbering

- Starts at **v0.1.0** (pre-1.0 signals the API may still move; breaking changes stay minor).
- Cut **v1.0.0** deliberately, when you want to signal a stable public API — from then on,
  `feat` → minor and breaking → major, per semver.
