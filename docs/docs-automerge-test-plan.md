# Docs-only auto-merge: sandbox test plan

Written for: you, creating the test repo and GitHub App by hand, and Claude, who runs the test phases from the command line.

Workflow under test: `.github/workflows/docs-only-automerge.yml` at core-baseline commit `60ad6dac8e`, plus the `DOCS_PATH` change in section 1.

Results and App setup steps: see `docs-automerge-test-report.md`.

---

## 1. What the workflow does

Triggers on PR `opened`, `reopened`, `synchronize`, `ready_for_review`.

1. The job runs only if the base branch is `master`, the PR is not a draft, and the repo variable `DOCS_MERGE_BOT_APP_ID` is set. Otherwise the job is skipped and no token is minted.
2. It mints a GitHub App installation token from `DOCS_MERGE_BOT_APP_ID` and the secret `DOCS_MERGE_BOT_PRIVATE_KEY`.
3. It lists the PR files. Every file must match the docs rule. Every non-deleted file must be a regular file, not a symlink.
4. If all checks pass, it squash-merges at the PR head SHA. The merge is performed by the App.

**Docs rule.** A path matches `**/docs/**/*.md`: a folder named `docs` at any depth, with any number of sub-folders under it, ending in a `.md` file. Folders starting with `.` never count, at any level, so `.github/docs/a.md` and `docs/.hidden/a.md` do not match. The file extension must be lowercase `.md`. `docs/a.md`, `src/docs/a.md` and `docs/sub/deep/a.md` match.

**Renames** require both the old and the new path to match. **Deletes** are always allowed.

**Log lines to look for** (in the job log, prefixed `#<PR number>:`):

| Log line | Meaning | Job result |
|---|---|---|
| `merged docs-only PR` (notice) | Merged | green |
| `skipped (changes outside docs/*.md)` (info) | Not docs-only, not merged | green |
| `merge rejected (405)` or `(409)` (warning) | GitHub refused the merge | green, PR stays open |
| Token step or any other error | Failure | red, nothing merged |

---

## 2. Gotchas that break tests

- **The workflow runs from the master copy.** `pull_request_target` uses the workflow file on the base branch. Editing the workflow on a PR branch does nothing for that PR. The file must be on `master` before you open test PRs.
- **Hard-coded target.** The token step has `owner: XodoDocs` and `repositories: PDFTronCore`. In the sandbox these must be deleted (Part A3). Otherwise the token mint fails because the App is not installed on `XodoDocs/PDFTronCore`.
- **Default branch must be `master`.** The job checks `base.ref == 'master'`.
- **App ID is a repo variable, not a secret.** Name: `DOCS_MERGE_BOT_APP_ID`.
- **The job decides when the run starts.** A PR opened before the variable exists will not run until a new commit is pushed (`synchronize`). Push an empty commit to re-trigger.
- **Squash merging must be enabled** in repo settings. The bot always squashes.
- **Your token needs the `workflow` scope** (classic PAT) or `Workflows: Read and write` (fine-grained) to push anything that touches `.github/workflows/`. Without it, the initial push in Part A is rejected.
- **Branch rulesets** (needed only for E1–E2) are not available on private repos of a personal Free account. Use a public sandbox, or a private one on a plan that supports rulesets.
- **Tested version matters.** The sandbox copy of the workflow must match the local copy with the `DOCS_PATH` change. Re-copy it if the file changes. Deploy by pushing to `master`: a workflow edit is not docs-only, so the bot will not merge it.

---

## 3. Part A: create the sandbox repo (manual)

**A1. Create the repo on GitHub**

1. Click `+` → **New repository** (your personal account).
2. Repository name: `docs-automerge-sandbox`.
3. Visibility: Public, or Private if your plan supports rulesets.
4. **Do not** tick "Add a README", ".gitignore", or "license". The repo must be empty so the first push sets the default branch.
5. Create repository.

**A2. Create the local folder with the workflow copy** (Git Bash):

```bash
mkdir -p ~/docs-automerge-sandbox && cd ~/docs-automerge-sandbox
git init -b master
mkdir -p .github/workflows docs src
cp "/c/Users/SergeChernetsov/source/core-baseline/.github/workflows/docs-only-automerge.yml" .github/workflows/
printf '# Sandbox\n' > README.md
printf '# Guide\n'   > docs/guide.md
printf '# Old\n'     > docs/old.md
```

**A3. Edit the copy.** In `.github/workflows/docs-only-automerge.yml`, delete these two lines under the `Create GitHub App token` step:

```yaml
          owner: XodoDocs
          repositories: PDFTronCore
```

With both lines removed, the action defaults to the current repo owner and the current repo only. Change nothing else.

**A4. Commit and push:**

```bash
git add -A
git commit -m "Initial sandbox: docs-only auto-merge workflow"
git remote add origin https://github.com/<OWNER>/docs-automerge-sandbox.git
git push -u origin master
```

**A5. Repo settings** (repo → Settings):

- General → Default branch: confirm it shows `master`.
- General → Pull Requests: tick **Allow squash merging**. Keep at least one other merge method enabled.
- Actions → General: defaults are fine (Allow all actions; read-only workflow permissions).

No workflow run happens on push. Runs start only from PR events.

---

## 4. Part B: create the GitHub App (manual)

**B1.** Avatar → **Settings** → **Developer settings** → **GitHub Apps** → **New GitHub App**. Direct link: `https://github.com/settings/apps/new`.

**B2. Basic fields**

| Field | Value |
|---|---|
| GitHub App name | `docs-merge-bot-sandbox-<your-initials>` (must be globally unique) |
| Homepage URL | `https://github.com/<OWNER>/docs-automerge-sandbox` |
| Callback URL, Setup URL | leave empty |
| Request user authorization during installation | off |
| Webhook → **Active** | **untick** (the workflow does not use webhooks) |

**B3. Repository permissions**

| Permission | Access | Why |
|---|---|---|
| Contents | Read and write | creates the merge commit |
| Pull requests | Read and write | reads PR files, merges PR |
| Metadata | Read-only | added automatically, required |

All other repository permissions: **No access**. Organization and account permissions: none.

**B4.** "Where can this GitHub App be installed?" → **Only on this account**.

**B5.** Click **Create GitHub App**.

**B6.** On the App page, under **About**, copy the **App ID** (a number). This goes into the variable in Part C.

**B7.** Left sidebar → **Credentials** → **Key pairs** → **New key**. A `.pem` file downloads. Treat it as a password: do not commit it or share it. The App is limited to 25 keys; delete old ones under **Key pairs** when needed. (There is no "Private keys" section on the General tab.)

**B8.** Left sidebar → **Install App** → **Install** next to your account → **Only select repositories** → choose `docs-automerge-sandbox` → **Install**.

---

## 5. Part C: connect the App to the repo (manual)

**C1.** Repo → Settings → Secrets and variables → Actions → **Variables** tab → **New repository variable**:

- Name: `DOCS_MERGE_BOT_APP_ID`
- Value: the App ID number from B6

**C2.** Same page → **Secrets** tab → **New repository secret**:

- Name: `DOCS_MERGE_BOT_PRIVATE_KEY`
- Value: the entire `.pem` file contents, including the `-----BEGIN ...-----` and `-----END ...-----` lines.

**C3.** Nothing else. The workflow needs no org-level secrets.

---

## 6. Part D: test plan (executable by Claude)

### D0. Setup

Prerequisites:

- The clone from A2 at `~/docs-automerge-sandbox`, with `origin` pointing at the sandbox.
- GitHub CLI installed (`winget install --id GitHub.cli`). It is **not** installed on this machine yet.
- Your `GITHUB_TOKEN` has `repo` and `workflow` scopes.

Environment, run once per shell:

```bash
export OWNER=<OWNER>
export R="$OWNER/docs-automerge-sandbox"
export GH_TOKEN="$GITHUB_TOKEN"
gh auth status
```

Create the base branch needed by N11:

```bash
git push origin master:refs/heads/release
```

**Standard case procedure (P).** Each case is one branch and one PR.

```bash
git switch master && git pull --ff-only
git switch -c t/<ID>
# apply the case change (see the case list)
git add -A && git commit -m "<ID>: <description>"    # or: git commit --allow-empty -m "<ID>: retrigger"
git push -u origin t/<ID>
N=$(gh pr create -R "$R" --base master --head t/<ID> --title "<ID>: <description>" --body "auto-merge test" [--draft] | grep -oE '[0-9]+$')
```

Wait about 20 seconds for the run to start. If the newest run is older than the PR, wait another 20 seconds and retry.

**Verify block (V)**, run after P:

```bash
RUN=$(gh run list -R "$R" --workflow docs-only-automerge.yml -L 1 --json databaseId -q '.[0].databaseId')
gh run watch "$RUN" -R "$R" --exit-status      # non-zero exit means the run failed
gh run view "$RUN" -R "$R" --json conclusion,jobs -q '.conclusion, (.jobs[] | "\(.name): \(.conclusion)")'
gh run view "$RUN" -R "$R" --log | grep -E "#$N:|##\[(error|warning)\]"
gh pr view "$N" -R "$R" --json state,isDraft,mergedBy -q '"state=\(.state) draft=\(.isDraft) mergedBy=\(.mergedBy.login)"'
```

A **merged** result has `state=MERGED` and `mergedBy` ending in `[bot]`. That confirms the App did the merge, not you.

A **job skipped** result has the job conclusion `skipped` and no `#N:` line. Draft and wrong base both produce this.

### D2. Must merge

| ID | Branch | Change | Expect |
|---|---|---|---|
| H1 | `t/h1-add` | add `docs/new-guide.md` | merged |
| H2 | `t/h2-edit-multi` | modify `docs/guide.md`; add `docs/a.md` and `docs/b.md` | merged |
| H3 | `t/h3-delete-rename` | `git rm docs/old.md`; `git mv docs/guide.md docs/guide-renamed.md` | merged (delete and in-docs rename both allowed) |
| H4 | `t/h4-nested` | add `src/docs/nested.md` | merged. This documents current behavior: any non-dot parent folder qualifies. See section 7. |
| H7 | `t/h7-deep` | add `docs/sub/deep.md` | merged. Nested under sub-folders of `docs/`, any depth. |
| H5 | `t/h5-draft` | add `docs/draft.md`, open with `--draft` | job `skipped` while draft. Then `gh pr ready "$N" -R "$R"` → merged. |
| H6 | `t/h6-sync` | add `docs/sync.md` **and** change `README.md`, push, open PR | first run: `skipped`. Then `git checkout master -- README.md`, commit, push (`synchronize`) → merged. |

### D3. Must NOT merge

Each case must end with the log line `#N: skipped (changes outside docs/*.md)`, the job green, and the PR open.

| ID | Branch | Change | Why it is rejected |
|---|---|---|---|
| N1 | `t/n1-mixed` | `README.md` change plus `docs/mixed.md` | root file outside docs |
| N2 | `t/n2-dot-in-docs` | add `docs/.hidden/deep.md` | dot folder under `docs/` |
| N3 | `t/n3-image` | add `docs/image.png` (`printf 'x' > docs/image.png`) | not `.md` |
| N4 | `t/n4-dot` | add `.github/docs/hidden.md` | dot folder |
| N5 | `t/n5-txt` | add `docs/notes.txt` | not `.md` |
| N6 | `t/n6-case` | add `docs/Upper.MD` | extension is case-sensitive |
| N7 | `t/n7-rename-in` | `git mv README.md docs/README.md` | old path outside docs |
| N8 | `t/n8-rename-out` | `git mv docs/guide.md guide.md` | new path outside docs |
| N9 | `t/n9-symlink` | add symlink `docs/link.md` (see below) | symlink, mode 120000 |
| N10 | `t/n10-workflow` | append `# test` to `.github/workflows/docs-only-automerge.yml`; add `docs/wf.md` | workflow file is outside docs. The branch copy of the workflow must not affect the run. |
| N11 | `t/n11-release` | add `docs/rel.md`, open PR with `--base release` | base is not `master`. Expect job `skipped` with no `#N:` line. |

N9 symlink creation (works from Git Bash without symlink privileges):

```bash
B=$(printf 'guide.md' | git hash-object -w --stdin)
git update-index --add --cacheinfo 120000,$B,docs/link.md
git commit -m "N9: symlink in docs"
```

### D4. Failure and edge cases

**E1: approval required, App not bypassing.**

1. Repo → Settings → Rules → Rulesets → New branch ruleset. Name `master-review`. Enforcement: Active. Target: default branch. Enable **Require a pull request before merging**, required approvals `1`. No bypass actors. Create.
2. Case `t/e1-approval`, add `docs/e1.md`, open PR (P). Do not approve.
3. Expect: run green with `##[warning]#N: merge rejected (405)`. PR stays open.

**E2: App on bypass list.**

1. Edit the ruleset → **Bypass list** → add the App. If the UI does not offer the App here, skip E2 and say so in results.
2. Case `t/e2-bypass`, add `docs/e2.md`, open PR (P).
3. Expect: merged, `mergedBy` ends in `[bot]`.
**E3: squash disabled.**

1. Settings → General → Pull Requests: untick **Allow squash merging**. Keep merge commits enabled.
2. Case `t/e3-squash`, add `docs/e3.md`, open PR (P).
3. Expect: green with `merge rejected (405)` warning. PR stays open.
4. Re-tick **Allow squash merging**.

**Not tested:** 409 (head moved between the check and the merge). It is hard to trigger on purpose. Covered by reading the code only.

### D6. Results

Record each row after running it.

| ID | Expected | Actual | PR # | Run ID | Notes |
|---|---|---|---|---|---|
| H1 | merged | | | | |
| H2 | merged | | | | |
| H3 | merged | | | | |
| H4 | merged | | | | |
| H7 | merged | | | | |
| H5 | skipped, then merged | | | | |
| H6 | skipped, then merged | | | | |
| N1–N11 | skipped, open | | | | |
| E1 | green, 405 warning, open | | | | |
| E2 | merged | | | | |
| E3 | green, 405 warning, open | | | | |

If a clean PR gets a 405 warning, re-run the job before concluding the merge is blocked. A fresh PR may not have its mergeability computed yet. This is possible, not verified.

### D7. Cleanup

1. Delete the sandbox repo: Settings → Danger Zone → **Delete this repository**.
2. Delete the App: `https://github.com/settings/apps/<slug>` → **Advanced** → **Delete GitHub App**. This removes its keys and installations.
3. Delete the local clone `~/docs-automerge-sandbox` and the `.pem` file.

---

## 7. Decisions for production (found while reading the workflow)

These are not part of the sandbox test. Decide them before the workflow goes to PDFTronCore.

1. **Fork PRs auto-merge.** There is no check that the PR head is in the same repo. `pull_request_target` gives fork PRs access to the base repo's secrets. Fix: add `github.event.pull_request.head.repo.full_name == github.repository` to the job `if:`.
2. **Any non-dot `docs/` folder qualifies, at any depth.** Rule is now `**/docs/**/*.md` (H7 covers sub-folders). The old count of 12 files under `breakpad/docs/*.md` is now a lower bound; re-count in core-baseline. That is third-party code. If only root `docs/` should count, change `DOCS_PATH` to `/^docs\/(?:[^./][^/]*\/)*[^/]+\.md$/`. Test H4 shows `src/docs/` qualifies today.
3. **Required reviews.** If `master` requires approvals, the bot's merge is rejected (405) and logged as a warning. It merges only if the App is on the ruleset bypass list. Decide whether the bot may bypass review.
4. **Rejections are silent.** A 405 or 409 leaves the PR open with a green job. Nothing retries it.

## 8. Production repo: what to configure (PDFTronCore)

Not part of the sandbox test. Needed later:

- Install the App on `XodoDocs` with access to `PDFTronCore`. Use a separate production App if policy requires one.
- Repo variable `DOCS_MERGE_BOT_APP_ID` and secret `DOCS_MERGE_BOT_PRIVATE_KEY` on `XodoDocs/PDFTronCore`.
- Keep `owner: XodoDocs` and `repositories: PDFTronCore` in the workflow. Do not apply the A3 edit to production.
- Squash merging enabled on `PDFTronCore`.
