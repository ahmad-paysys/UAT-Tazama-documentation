# `solutions/` — Instructions for agents

This file governs work performed under any repository in `solutions/`. Any
agent making changes inside a `solutions/<repo>/` subtree must follow every
rule below. These are not defaults — they are requirements.

If any instruction here conflicts with a general instinct, the instruction
here wins. If any instruction here conflicts with a user request in a
specific conversation, ask the user before acting.

---

## Setting up the working copy

Every task begins by ensuring the target repository is present at
`solutions/<repo>/` in a known-clean starting state. Do not skip this step
even if the directory already looks correct.

### If `solutions/<repo>/` does not exist

Clone it from its upstream on GitHub:

```bash
git clone https://github.com/tazama-lf/<repo>.git solutions/<repo>
```

Do not clone from a fork, a mirror, or a local copy. If the user names a
specific fork, use that; otherwise the canonical `tazama-lf/<repo>` remote
is the source of truth.

### If `solutions/<repo>/` already exists

Do not reset, discard, or overwrite the existing state. Any branches or
uncommitted work already present may belong to the user or to prior work
that is still in flight. Instead:

1. `git fetch origin` to sync remote refs.
2. Create a new branch off `origin/dev` without touching the current
   branch, working tree, or index. Use `git worktree add` if a fresh
   worktree is cleaner than switching branches in place.
3. Confirm the new branch is based on `origin/dev` (not `main`, not a
   stale local `dev`) before making any changes.

If the working tree contains uncommitted changes or untracked files that
would be affected by the operation, stop and ask the user before doing
anything destructive. Never `git reset --hard`, `git clean -fd`,
`git checkout --`, or delete files to "start fresh" without explicit
permission.

### Remote URL must be SSH

The `origin` remote must use SSH, not HTTPS. If `git remote -v` shows an
`https://github.com/...` URL, switch it before doing any other work:

```bash
git remote set-url origin git@github.com:tazama-lf/<repo>.git
```

This applies whether the repo was just cloned or already existed. HTTPS
remotes prompt for credentials and break signed pushes in this
environment; SSH is the only supported transport.

---

## Branching

Every branch created for work in `solutions/<repo>/` starts with the exact
prefix `feat-paysys-`, followed by a short kebab-case slug that describes
the work.

Examples of valid names:

- `feat-paysys-fix-ems`
- `feat-paysys-tenant-scoping-cache`
- `feat-paysys-issue-47`

No other prefix is acceptable. This is not "prefer" — this is the only
allowed form. If you cannot think of a good slug, ask the user rather than
guessing.

Branches are created off `origin/dev`. Never off `main`, `staging`, `rc*`,
or any other integration branch, unless the user explicitly names a
different base.

---

## Commits

### Prefix

Every commit subject begins with either `fix: ` or `feat: `. Exactly those
two, with the trailing space, in lowercase. Not `fix:`, not `Fix: `, not
`fix(lint): `, not `chore: `, not `docs: `, not `refactor: `, not `test: `,
not `ci: `. If the change genuinely does not fit either `fix` or `feat`,
stop and ask the user.

Use `fix: ` for changes that correct broken behaviour or restore intended
behaviour that was missing. Use `feat: ` for changes that add new
behaviour, new capability, or new tests that did not previously exist.

A parenthetical scope after the prefix is acceptable when it clarifies the
area touched, e.g. `fix: (lint) scope lint:eslint to explicit glob`.

### Signing and DCO

Every commit must be both cryptographically signed **and** carry a DCO
sign-off trailer. This is non-negotiable.

- Signing: `git commit -S ...`. The commit must show `G` (good signature)
  in `git log --show-signature`.
- DCO: `git commit -s ...`. This appends
  `Signed-off-by: <Name> <email>` as the final trailer of the commit
  message.

Both flags are always used together: `git commit -S -s`. If the user's
local git config does not have signing set up, stop and tell the user —
do not commit unsigned.

### Authorship

Commits are authored **only** by the user. No AI co-author trailer of any
kind is ever added. This includes but is not limited to:

- `Co-Authored-By: Claude <...>` in any form
- `Co-Authored-By: <any assistant, any model, any product name>`
- Any "Generated with" or "Assisted by" footer
- Any signature, banner, or emoji indicating AI involvement

If a shell wrapper or default template inserts such a trailer, remove it
before committing. The commit body ends with the DCO `Signed-off-by:`
line, and nothing after it.

### Commit granularity

Keep each commit small and focused on one logical change. A commit that
touches config, tests, and production code across three unrelated concerns
is too large — split it. A branch with one enormous commit is a review
burden; a branch with ten focused commits is legible.

Concretely: if the commit subject needs "and" to describe what it does, it
is probably two commits.

---

## Pushing

Do not push to any remote. Ever. The user pushes. This applies to:

- The task branch itself.
- Any local branches created along the way.
- Tags, notes, or any other ref.

`git push` is not to be run by an agent under `solutions/`. If the user
asks you to push, that is a one-time authorisation for that specific
push — it is not a standing permission for future work.

Similarly, do not open, edit, comment on, or merge pull requests without
explicit instruction. Drafting a PR body for the user to paste is fine;
running `gh pr create` is not, unless directly requested.

---

## Dependencies

Do not add a new dependency to any project without the user's explicit
permission. This includes:

- `dependencies`, `devDependencies`, `peerDependencies`, `optionalDependencies`
- Transitive-only additions introduced by pinning a package that would
  otherwise not have been installed
- Any language ecosystem: `package.json`, `pyproject.toml`, `go.mod`,
  `Cargo.toml`, etc.
- Adding a new registry entry, GitHub Packages source, or private registry
  to `.npmrc` or equivalent

Before proposing a new dependency, explain to the user:

1. What problem it solves that cannot be solved with what is already
   installed.
2. Its licence, current maintenance status, and any known security
   advisories.
3. Approximate install footprint (size, transitive count) if it is a
   heavy addition.
4. Whether the alternative is to inline a small amount of code instead.

Wait for the user to say "yes" before running the install command.
Removing an unused dependency is not a new dependency and does not
require this ceremony — but do surface the removal in the PR body.

Updating an existing dependency's version pin (e.g., `^8.20.0 → ^8.41.0`
to satisfy a peer requirement) is not a new dependency, but should still
be explained in the commit message and PR body.

---

## Handling ambiguity

If any of the following is true, stop and ask the user rather than
guessing:

- Two or more reasonable interpretations exist for what the user asked.
- The change could be scoped narrowly or broadly, and the user did not
  say which.
- A file's intent is unclear from the code and there is no comment,
  documentation, or ticket resolving it.
- A test fails and the fix could reasonably be either "change the test
  to match the code" or "change the code to match the test."
- The user's instruction can be read as authorising a destructive
  action (force-push, reset, delete) but also as authorising a safer
  alternative.

The cost of pausing to confirm is small. The cost of guessing wrong on a
non-reversible action, or of drifting away from what the user actually
wanted, is large. When in doubt, ask.

Do not fabricate details to fill gaps: file paths, function signatures,
API shapes, third-party behaviour. Read the code or the docs first;
ask the user second; never invent.

---

## Verification before "done"

Before declaring a task complete:

1. Re-run the project's lint command. It must exit 0.
2. Re-run the project's test command. All suites must pass.
3. Re-run any coverage gate the project enforces. The threshold must be
   met.
4. `git log --show-signature -N` (for the N commits you authored) must
   show `Good signature` on each commit.
5. `git log -N --format='%(trailers:key=Signed-off-by)'` must show a
   `Signed-off-by:` trailer on each commit.
6. `git log -N --format='%an <%ae>'` must show only the user's name and
   email — no AI co-author lines anywhere in the message body or
   trailers.

If any of these fail, fix the underlying problem. Do not amend a
published commit; create a new one. Do not skip hooks with `--no-verify`
to work around a failing check — investigate the failure.

---

## Reporting to the user

When reporting completion of a task in `solutions/`:

- Name the branch you created.
- List the commit hashes (short SHA) in order.
- Confirm each commit is signed (`G`) and DCO-signed.
- State that nothing has been pushed.
- Note anything left in the worktree unstaged (e.g., an unrelated user
  edit that was preserved rather than bundled into your commits).

Do not include a summary of "what I intended to do." Report what
actually happened, verified against `git log` and the working tree.
