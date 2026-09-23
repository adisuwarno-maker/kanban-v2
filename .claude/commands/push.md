---
description: Security-scan the project, update the README and GitHub About section, set up GitHub Pages via Actions, then commit and push
argument-hint: <github-repo-url or owner/repo> [commit message]
---

Publish this project to GitHub and deploy it to GitHub Pages.

Arguments: `$ARGUMENTS`

- The first argument is the target repo, as an HTTPS URL (`https://github.com/<owner>/<repo>.git`) or as `owner/repo`.
- Anything after it is the commit message.
- If no repo was given, use the existing `origin` remote (`git remote get-url origin`). If there is none, ask the user for the repo and stop until they answer.

Work through the steps in order. **Step 1 is a gate: if it finds a blocker, stop and do not push anything.**

## 1. Security scan (before anything leaves this machine)

Scan every file that would be published: the tracked and untracked files git would add, which you can list with `git ls-files --cached --others --exclude-standard`. If the repo already has history, also scan `git log -p --all`, because pushing publishes the whole history.

**Blockers** (stop, report the file and line, and suggest a fix):
- Secrets and credentials. Look for GitHub tokens (`ghp_`, `gho_`, `ghs_`, `github_pat_`), AWS keys (`AKIA[0-9A-Z]{16}`), private keys (`-----BEGIN .*PRIVATE KEY-----`), Slack tokens (`xox[abpr]-`), Google keys (`AIza`), OpenAI or Anthropic keys (`sk-`, `sk-ant-`), and `password=`, `secret=`, `api_key=` or `token=` with a literal value.
- Files that must never be published: `.env*`, `*.pem`, `*.key`, `id_rsa*`, `.claude/settings.local.json`, `.DS_Store`. Make sure `.gitignore` covers them. If one is already tracked, remove it with `git rm --cached` and tell the user. If it appears in the history, warn them that the history must be rewritten and the secret rotated.
- If `gitleaks` or `trufflehog` is installed, run it as well (`gitleaks detect --source . -v`). Otherwise say that you ran a pattern-based scan only.

**App-specific checks** (read `CLAUDE.md` for the rules):
- User-supplied strings must go through `escapeHtml()` before they reach `innerHTML`. Flag any new unescaped interpolation.
- The code must not use `localStorage`, `sessionStorage`, `indexedDB` or `document.cookie`, and must not load external `<script src>` or `<link>` resources. The only allowed network call is the FormSubmit endpoint.
- No real logos or trademarks, and no page that imitates an official bank system.

**Warnings** (report them and continue unless the user objects):
- A real email address in `FORMSUBMIT_ENDPOINT`. Remind the user that it becomes public if the repo is public.
- Any other personal data, such as emails or phone numbers, in the source.

Before continuing, give a short scan report that lists blockers, warnings, and what was checked.

## 2. README

Create `README.md`, or update it if it already exists, keeping any sections the user wrote. Base it on the actual code and `CLAUDE.md`, not guesses. Include:
- the title and a one-line summary, including that this is an internal demo/training tool that is not affiliated with any official system
- the live site link: `https://<owner>.github.io/<repo>/`
- the features: board columns, drag-and-drop with the keyboard fallback, filters, summary strip, and FormSubmit notifications
- how to run it: open `index.html` directly, no build step
- how to configure `FORMSUBMIT_ENDPOINT`, including the one-time activation email
- a note that data is kept in memory only and resets on refresh
- how deployment works, via the Pages workflow on push to `main`

## 3. GitHub Pages workflow

Create `.github/workflows/pages.yml`, or update it if it already exists. It should deploy the repo root as a static site on every push to `main` and allow manual runs (`workflow_dispatch`). Use `permissions: contents: read, pages: write, id-token: write`, a `concurrency: group: pages` block, and these steps:
- `actions/checkout@v4`
- `actions/configure-pages@v5`
- `actions/upload-pages-artifact@v3` with `path: .`
- `actions/deploy-pages@v4`, in a `github-pages` environment

Add a `.nojekyll` file at the repo root. Make sure `.github/`, `.claude/` and `CLAUDE.md` are harmless to publish: they are served as static files, so they must contain no secrets, which Step 1 already covers.

The Pages source must be set to **GitHub Actions**:
- If `gh` is installed and authenticated (`gh auth status`), run `gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow`. If Pages already exists, run `gh api -X PUT repos/<owner>/<repo>/pages -f build_type=workflow` instead.
- Otherwise, tell the user to open `https://github.com/<owner>/<repo>/settings/pages` and set **Source** to **GitHub Actions**.

## 4. GitHub "About" section

Prepare these values:
- **Description**: one line, no more than 350 characters, taken from the README summary
- **Website**: `https://<owner>.github.io/<repo>/`
- **Topics**: lowercase and hyphenated, for example `kanban`, `project-management`, `vanilla-js`, `github-pages`, `html-css-javascript`

If `gh` is authenticated, apply them with `gh repo edit <owner>/<repo> --description "…" --homepage "…" --add-topic kanban --add-topic …`.

Otherwise, give the user the exact values to paste into the ⚙️ next to **About** on the repo home page. Never ask for, accept or handle the user's token or password yourself.

## 5. Commit and push

1. If the folder is not a git repo, run `git init -b main`.
2. Set `origin` to the target repo, adding it or updating it with `git remote set-url` if it differs.
3. Show `git status --short`, then stage only the intended files.
4. Commit with the message the user gave. If they didn't give one, write a concise message describing the changes.
5. If `origin/main` exists and has commits you don't have, run `git pull --rebase origin main` first, and resolve any conflicts with the user.
6. Run `GIT_TERMINAL_PROMPT=0 git push -u origin main`.

If the push fails because no credentials are saved:
- Tell the user to run `git push -u origin main` in their own terminal and sign in there, using a personal access token as the password (https://github.com/settings/personal-access-tokens/new, with **Contents: Read and write** on this repo).
- Or tell them to install and sign in to the GitHub CLI (`brew install gh && gh auth login`).
- Wait for them to confirm the push, then continue.

## 6. Verify and report

- Confirm that the remote has your commit: `git ls-remote origin main` should match `git rev-parse HEAD`.
- If `gh` is available, run `gh run list --workflow pages.yml --limit 1` and wait for the run to finish.
- Poll `https://<owner>.github.io/<repo>/` until it returns 200, checking every 15s for up to about 3 minutes. Then open it in the browser pane and confirm that the page title and the seeded cards render.
- End with a short summary covering:
  - the scan result
  - the files created or changed
  - the commit hash
  - the About values and whether they were applied or still need to be pasted in
  - the live site URL
  - anything the user still needs to do by hand
