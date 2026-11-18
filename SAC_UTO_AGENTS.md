# SAC Uto fork of Zulip - instructions for agents

This repository is SAC Uto's fork of Zulip. It carries a small set of
organization-specific patches on top of upstream Zulip releases. This
file describes how those patches are carried forward when upgrading to
a new Zulip release, and the rules an AI agent must follow while doing
so. It takes precedence over the upstream instructions in `AGENTS.md`.

The upstream guidelines in `AGENTS.md` and `.claude/rules/` still
apply to code style, but their testing, commit-series and PR
instructions do **not** apply to this fork's upgrade work (see "No
local execution" below).

## Remotes and branches

Remotes:

- `upstream`: the official Zulip repository (`zulip/zulip`).
- `origin`: our fork (`sac-uto/zulip`).

Branches:

- `main`: mirror of `upstream/main`. Never contains our patches.
- `stable`: points at the latest upstream release tag (no beta, RC or
  dev tags). Never contains our patches.
- `sac-dev`: `stable` plus our patch commits. Deployed to the **test**
  server.
- `sac-prod`: the same commit as `sac-dev`, once that has been
  verified on the test server. Deployed to the **live** server.

Every organization-specific patch is its own commit whose summary
starts with `SAC Uto: `. To list the current patches:

```bash
git log --oneline stable..sac-dev
```

If that range contains any commit not prefixed with `SAC Uto: ` (for
example a cherry-picked upstream fix), point it out to the developer
during the upgrade; it may already be included in the new release and
need to be dropped.

## Upgrade procedure

1. **Update `main`.** Check out `main`, fetch `upstream`, and
   fast-forward `main` to `upstream/main`. Push `main` to `origin`.

   ```bash
   git checkout main
   git fetch upstream --tags
   git merge --ff-only upstream/main
   git push origin main
   ```

2. **Update `stable`.** Find the latest release tag that is not a
   beta, RC or dev tag (e.g. `13.2`, not `14.0-rc1` or `14.0-dev`).
   **Before moving `stable`, record the commit it currently points
   at**; step 3 needs it. Then point `stable` at the new tag and push
   it to `origin`.

   ```bash
   git tag --list '[0-9]*' --sort=-version:refname | grep -Ev 'beta|rc|dev' | head
   git rev-parse stable   # record this as <old-stable>
   git branch -f stable <tag>
   git push --force origin stable
   ```

3. **Rebase the patches onto the new `stable`.** Only the commits
   after `<old-stable>` may be moved:

   ```bash
   git checkout sac-dev
   git rebase --onto stable <old-stable> sac-dev
   ```

   Do **not** use a plain `git rebase stable`. Releases are tagged on
   maintenance branches (`12.x`, `13.x`, ...), so the old and new
   release usually live on different branches, and a plain rebase
   would replay upstream maintenance commits (version bumps,
   backports) on top of the new release.

   This usually produces conflicts. **Even when it does not**, every
   location touched by a `SAC Uto: ` commit must be re-examined
   against the new upstream code: the surrounding code may have
   changed meaning, been moved, renamed, or gained new call sites that
   also need the patch. A clean rebase is not evidence of a correct
   result. Review each patch commit with e.g. `git show <commit>` and
   read the new upstream code around it, including callers, related
   code paths, and signatures of the upstream functions the patch
   calls.

4. **Push `sac-dev`.** Once all patches have been applied to the new
   version, force-push `sac-dev` to `origin`. The developer then
   deploys it to the test server and tests it there.

   ```bash
   git push --force-with-lease origin sac-dev
   ```

5. **Promote to production.** Only after the deployment has been found
   to work without issues on the test server, point `sac-prod` at the
   same commit as `sac-dev` and push it. The developer then deploys it
   to the live server.

   ```bash
   git branch -f sac-prod sac-dev
   git push --force-with-lease origin sac-prod
   ```

## Rules for agents

### No local execution

There is no local development or test environment for this fork.
**Only make code changes. Do not attempt to run the project, its
tests, its linters, `tools/provision`, or any other tooling locally.**
Verification during the upgrade is done purely by reading the code
statically; actual testing happens only after deployment to the test
server.

### Ask before any Git write

Before performing **any** Git operation that is not read-only (for
example `checkout`, `fetch`, `merge`, `rebase`, `commit`, `branch -f`,
`reset`, `cherry-pick`, `push`, tagging), interactively ask the
developer for permission. Read-only commands (`log`, `show`, `diff`,
`status`, `grep`, `describe`, `branch --list`, ...) may be run freely.

### Ask instead of assuming

If there is more than one non-trivial way to apply a `SAC Uto: ` patch
to the new version of Zulip (for example, upstream refactored the
patched code, split it into several places, or introduced a new
mechanism that could replace the patch), present the options to the
developer and ask which one to use. Do not pick one silently.

### No attribution trailers

Never add `Co-Authored-By:` (or any other AI attribution) lines to
commit messages in this repository, including when creating, amending
or rewording `SAC Uto: ` commits.

### Keep patches minimal

- Keep the number of code locations touched and the amount of patched
  code to a minimum. Do not reformat, refactor, or "improve" upstream
  code beyond what the patch needs; every extra changed line is a
  future rebase conflict.
- Keep each patch in its own `SAC Uto: ` commit, editing the existing
  commit when adapting it rather than adding fixup commits on top.
- Every patch site must have a comment line starting with
  `SAC Uto patch: ` immediately before it, briefly explaining what the
  patch does or changes there (use the file's comment syntax, e.g.
  `#`, `//`, `{# #}` or `<!-- -->`). When adapting a patch, keep these
  comments accurate, and add one to any new patch site. Exception:
  `zproject/prod_settings_template.py` gets no such comments.
- Where a patch disables upstream code, the existing patches keep it
  as commented-out code; when adapting such a patch, update the
  commented-out code to match the current upstream version.
- If an abstraction would reduce the coupling between a patch and the
  underlying upstream implementation (for example, a single hook point
  or settings-driven switch instead of edits scattered across several
  functions), propose it to the developer and ask whether they want it
  implemented. Do not introduce it without their agreement.
