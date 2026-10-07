---
name: git-crypt-add-gpg-collaborator
description: >
  Adds a collaborator's GPG key from GitHub to the current repo's git-crypt keyring on
  a new branch so they can decrypt the encrypted config files, and reports the key
  identity for the user to confirm. Use when someone asks for git-crypt access by
  GitHub username: "add <user>'s gpg key", "give <user> git-crypt access", "fix the
  GPG request from Slack".
---

# Add GPG collaborator to git-crypt

End-to-end flow for adding one GitHub user to the current repository's git-crypt
keyring. The user supplies a GitHub username; do everything else.

## Inputs

- `username` — the collaborator's GitHub username (required). If not provided
  in the invocation, ask for it before doing anything.

## Preflight

Run these in parallel and verify before touching anything:

1. `git status` — working tree must be clean.
2. `git branch --show-current` — note the current branch.
3. `git config user.email` — derive the branch prefix from the local-part
   (everything before `@`), lowercased, non-alphanumerics stripped. E.g.
   `firstname.lastname@tv4.se` → `firstnamelastname`. Call this `PREFIX`.

   If the user has a shorter preferred handle visible in recent branches,
   prefer that — but never hardcode another user's handle.

If the working tree isn't clean, stop and ask the user how to proceed.

If the current branch is not `main`, ask whether to (a) create the new branch
from current HEAD, or (b) switch to `main` first. Default suggestion is (b).

## Steps

1. **Create branch from main.** Branch name: `<PREFIX>/add-gpg-<username>`.
   If already on main with clean tree:

   ```sh
   git checkout -b <PREFIX>/add-gpg-<username>
   ```

2. **Import the key from GitHub.**

   ```sh
   curl -sS https://github.com/<username>.gpg | gpg --import
   ```

   Capture stderr — gpg prints one `key <KEY_ID>: public key "<name> <email>" imported`
   line per key. If the output says `Total number processed: 0`, the user has no GPG
   key on GitHub — stop and report back.

   **If multiple keys are imported**, pick the one whose uid is `@tv4.se`. Some
   collaborators have a personal key (e.g. `@gmail.com`) on the same GitHub
   account — it should not go into the repo. If no uid is `@tv4.se`, stop and
   ask the user which key to use rather than guessing.

   **Use the imported uid to confirm identity.** GitHub profiles can have empty
   name/email fields, in which case the imported uid (`<name> <email>`) is the
   strongest signal that the right person's key is being added. Surface the uid
   in your reply so the user can verify before the commit lands.

3. **Get the full fingerprint and expiry.**

   ```sh
   gpg --fingerprint <KEY_ID>
   ```

   The fingerprint is the 40-hex line under `pub`. Strip spaces to get
   `FINGERPRINT`. Note any `[expires: YYYY-MM-DD]` on the `pub` line — capture
   it for the final report. If the key is already expired, stop and ask the
   user to have the collaborator renew before retrying.

4. **Set ultimate ownertrust (non-interactive).** The README shows an
   interactive `gpg --edit-key` flow, but `/dev/tty` isn't available in this
   harness. Use `--import-ownertrust` instead — trust value `6` is ultimate:

   ```sh
   echo "<FINGERPRINT>:6:" | gpg --import-ownertrust
   ```

   Verify with `gpg --list-keys <KEY_ID>` — the uid line should read
   `[ultimate]`.

5. **Add to git-crypt.** This creates one commit on the current branch:

   ```sh
   git-crypt add-gpg-user <KEY_ID>
   ```

   Expected: `Add 1 git-crypt collaborator` commit, one new file under
   `.git-crypt/keys/default/0/<FINGERPRINT>.gpg`.

## After

Report back to the user with:

- Key ID, fingerprint, and uid (name + email) imported
- Key expiry date if set – flag if it's within ~3 months so the user can nudge
  the collaborator to renew
- Branch name and commit sha
- Ask whether to push + open a PR, or leave for them to do

Do **not** push or open a PR automatically — git-crypt membership is a
shared-state change and the user should confirm the identity before it lands.

### If the user says yes to the PR

- @-mention the collaborator's GitHub handle in the PR summary so they're
  notified.
- The test plan should ask the collaborator to verify against the PR branch –
  not after merge. `git-crypt unlock` works on a checkout of the branch, so a
  failure caught pre-merge is much cheaper to fix. Suggested checkboxes:

  ```
  - [ ] Both keys present under `.git-crypt/keys/default/0/`
  - [ ] @<username>: check out this branch, run `git-crypt unlock`, confirm
        that encrypted files decrypt
  ```

## References

- git-crypt docs: https://github.com/AGWA/git-crypt
