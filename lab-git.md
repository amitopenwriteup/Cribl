# Cribl Stream — Remote Git Repositories: Summary

## Overview
- Available for **on-prem deployments only**; requires an **Enterprise Cribl license**.
- Connecting to Git is optional, used for version control of configuration changes.
- Configured via: **Settings → Global → System → Git Settings**.

## Supported Remote Formats
- URI must match regex: `(?:git|ssh|ftps?|file|https?|git@[-\w.]+):(\/\/)?(.*?)(\.git\/?)?$`
- Examples:
  - `<protocol>://git@example.com/<username>/<reponame>.git`
  - `git://<host.xyz>:<port>/<user>/path/to/repo.git`

## Security Warnings
- **Back up `$CRIBL_HOME`** before configuring a remote.
- Create remotes as **empty repos** (no README/commit history) to avoid push errors.
- Sensitive files (`cribl.secret`, SSH keys) get pushed with the full directory — secure the repo accordingly.

## Connecting Over HTTPS (Recommended)
- Use **Basic auth** with a **personal access token** in the Password field (not embedded in the URL).
- **Never embed tokens in the Remote URL** — Cribl stores/returns it in plaintext and commits it to history (secret scanners will flag it). If already done, **revoke** the token (rotating isn't enough).
- GitHub requires a **classic** PAT with `repo` scope (fine-grained tokens don't work).
- Username/password auth only works for hosts still accepting account passwords (GitHub deprecated this Aug 13, 2021).
- Basic auth keeps credentials out of `cribl.yml`, but **not full encryption at rest**:
  - Decrypted password is written to `$CRIBL_HOME/.git/config`.
  - SSH private keys are stored in plaintext at `$CRIBL_HOME/local/cribl/auth/ssh/git.key`.

## Connecting Over SSH
- Set up via CLI (required if using a passphrase) or UI.
- Remote URL must use full `ssh://` format (Git-style `user@server:path.git` not supported).
- Must run `ssh-keyscan -H github.com >> ~/.ssh/known_hosts`.
- Watch for an appended newline after pasting the private key (deleting it causes fetch errors).

## GitLab-Specific Notes
- Create a **blank project** (no README).
- Use a **project access token** with `write_repository` scope; username format: `project_{project_id}_bot`.
- Push creates `master` branch on remote — don't select `main` if it appears.
- **"Prevent committing secrets to Git"** push rule can break commits involving `.pem`/`.key` files.

## Additional Git Settings
- **Branch/GitOps workflow** dropdowns (GitOps-specific).
- **Collapse Actions**: combines Commit+Push (or Commit+Deploy) into one button.
- **Default commit message** and **Git timeout** (default 600,000 ms).
- **Scheduled Actions**: cron-based Commit/Push/Commit & Push.
- **Copilot tab**: auto-generate commit messages.

## Restoring Leader from Remote Repo
Key steps:
1. Untar the **matching** Cribl version (don't start it).
2. `git init` in `$CRIBL_HOME`.
3. Configure SSH key if needed via `GIT_SSH_COMMAND` or `git config core.sshCommand`.
4. Verify access: `git ls-remote <repo>`.
5. `git remote add origin <repo>`.
6. `git fetch origin && git reset --hard origin/master`.
7. Verify with `git show --abbrev-commit`.
- **Important**: Verify `cribl.secret` is restored — required to decrypt sensitive config values.

## `.gitignore` File
- Contains two managed sections:
  - **CRIBL SECTION**: Cribl-managed, auto-synced on Leader start (don't remove rules, can comment out).
  - **CUSTOM SECTION**: User-defined patterns, untouched by Cribl.
