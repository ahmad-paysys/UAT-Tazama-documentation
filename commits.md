# commits.md — instructions for agents

Rules for committing work in this repository. These override any general defaults.

## 1. One file per commit

Every commit contains exactly one file. If you have edited five files, that is five separate commits — in the order the user asked for, or, if unspecified, in the order the files were modified (earliest first).

No exceptions. Not "these two files are logically the same change" — one file per commit.

## 2. Do not commit anything inside nested repositories

The following paths contain other people's repositories or the user's own working state. **Never** `git add` or `git commit` anything under them:

- `repos/`
- `solutions/`
- `UAT-BOARD/`

If your work modifies files inside these paths, that work is the user's to commit, not yours. Report the changes you made and stop.

This rule applies to every file inside those directories, including seemingly-trivial ones (`.gitignore`, `README.md`, config files). The user is the only person who commits inside those trees.

## 3. Never commit secrets

Before every `git add`, check what you are about to stage for anything that looks like a secret. Refuse to commit if you see:

- API tokens, GitHub PATs, DockerHub tokens (`ghp_...`, `dckr_pat_...`)
- Private keys (`-----BEGIN ... PRIVATE KEY-----`)
- Passwords in plaintext
- OAuth client secrets
- Cloud credentials (AWS access keys, GCP service-account JSON contents, Azure connection strings)
- `.env` files that contain filled-in values (as opposed to `.env.example` with placeholders)
- Session tokens, JWTs pasted for debugging
- Any string the user pasted into chat and asked you to save (unless it is obviously non-secret documentation)

If a file has a mix of documentation and a secret, remove or redact the secret before committing. If you cannot cleanly separate them, ask the user.

## 4. Never push to a remote

Do not run `git push` under any circumstance. The user pushes.

This applies to `git push`, `git push --force`, `git push --tags`, `gh pr create`, `gh pr merge`, and any other command that moves refs onto a remote. If you have a bundle of commits ready and think they should be pushed, tell the user and wait.

## 5. Signing and DCO

Every commit is both cryptographically signed and DCO-signed:

```
git commit -S -s -m "<subject>"
```

The `-S` produces a `Good signature` line in `git log --show-signature`. The `-s` appends a `Signed-off-by:` trailer.

If the user's git config does not have signing set up, stop and tell them. Do not commit unsigned.

## 6. No AI attribution

Never add any AI co-author, generator, or assistant trailer to a commit message. This includes but is not limited to:

- `Co-Authored-By: Claude ...`
- `Co-Authored-By:` any assistant or product name
- `Generated with ...`
- `Assisted by ...`
- Emoji or banners indicating AI involvement

The commit body ends with the DCO `Signed-off-by:` line. Nothing after it.

## 7. Commit messages

- Subject under 72 characters.
- Prefer `docs:`, `fix:`, `feat:` prefixes when the change is obviously one of those. Follow this repo's existing style if it differs (check `git log --oneline` for recent commits).
- Body (optional) explains *why*, not *what*. The diff already shows what.
- Never claim in the message that a change does something it doesn't. If the commit only touches one thing, the message says only that one thing.

## 8. When in doubt, stop and ask

If you are unsure whether a file falls under one of the excluded paths, whether a value is a secret, or whether the user meant to include a specific change in a commit — ask before you commit. The cost of pausing is small. The cost of a bad commit that has to be reverted or scrubbed from history is large.
