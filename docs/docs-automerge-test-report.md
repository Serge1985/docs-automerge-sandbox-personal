# Docs-only auto-merge: test report

Sandbox: `Serge1985/docs-automerge-sandbox-personal`. App: `docs-merge-bot-sandbox-sch` (App ID `5751982`). Plan: [docs-automerge-test-plan.md](docs-automerge-test-plan.md).

## Status

Pass means the result matched the expected result. Pending means the test has not run; the reason is in the details.

| ID | Case | Expected | Actual | PR | Run ID | Status |
|---|---|---|---|---|---|---|
| H1 | add `docs/new-guide.md` | merged | first run red (App not installed on repo); after install, merged | #1 | 44209221446 | Pass |
| H2 | modify `docs/guide.md`; add `docs/a.md`, `docs/b.md` | merged | first run `405 Base branch was modified`; re-run merged | #4 | 44211228593 | Pass |
| H3 | `git rm docs/old.md`; `git mv docs/guide.md docs/guide-renamed.md` | merged | merged | #17 | 44211926032 | Pass |
| H4 | add `src/docs/nested.md` | merged | merged | #5 | 44211240446 | Pass |
| H5 | `docs/draft.md` as draft, then ready | skipped, then merged | skipped while draft; merged after ready | #3 | 44209396612 (draft), 44211383523 (ready) | Pass |
| H6 | `docs/sync.md` plus `README.md` change, then revert README | skipped, then merged | skipped; merged after README restored | #6 | 44211253001 (skipped), 44211811224 (merged) | Pass |
| H7 | add `docs/sub/deep.md` | merged | merged | #22 | 44217823007 | Pass |
| N1 | `README.md` change plus `docs/mixed.md` | skipped | skipped | #7 | 44211264859 | Pass |
| N2 | add `docs/.hidden/deep.md` | skipped | skipped | #23 | 44217950658 | Pass |
| N3 | add `docs/image.png` | skipped | skipped | #9 | 44211287988 | Pass |
| N4 | add `.github/docs/hidden.md` | skipped | skipped | #10 | 44211301825 | Pass |
| N5 | add `docs/notes.txt` | skipped | skipped | #11 | 44211315940 | Pass |
| N6 | add `docs/Upper.MD` | skipped | skipped | #12 | 44211330040 | Pass |
| N7 | `git mv README.md docs/README.md` | skipped | skipped | #13 | 44211343331 | Pass |
| N8 | `git mv docs/guide.md guide.md` | skipped | skipped | #14 | 44211355359 | Pass |
| N9 | add symlink `docs/link.md` | skipped | skipped | #15 | 44211368051 | Pass |
| N10 | append to workflow file; add `docs/wf.md` | skipped | skipped | #16 | 44211380754 | Pass |
| N11 | add `docs/rel.md`, base `release` | skipped, no `#N:` line | skipped, no `#N:` line | #2 | 44209353631 | Pass |
| E1 | `master-review` approval rule, App not bypassing | green, 405 warning, open | `405 Repository rule violations found`, open (#20, before App was added to bypass) | #20 | 44214949132 | Pass (#20). Re-run needs App off bypass list |
| E2 | App on ruleset bypass list | merged by bot | merged, `app/docs-merge-bot-sandbox-sch` | #21 | 44215667402 | Pass |
| E3 | squash merge disabled | green, 405 warning, open | `405 Squash merges are not allowed on this repository.`, open | #19 | 44214721036 | Pass |

Removed from the plan: G1 and G2 (variable-missing gate test), E4 (broken private key), D5 (fork PR).

## Details

### H1: add docs file
- Branch `t/h1-add` adds `docs/new-guide.md`.
- First run (red): step `Create GitHub App token` failed with `Not Found - get-a-repository-installation`. The App had no installation on the repo. Fixed by installing the App on the repo.
- Re-run of the same run: `#1: merged docs-only PR`. `mergedBy` = `app/docs-merge-bot-sandbox-sch`.

### H2: edit and add
- Branch `t/h2-edit-multi` modifies `docs/guide.md` and adds `docs/a.md`, `docs/b.md`.
- First run: `#4: merge rejected (405): Base branch was modified. Review and try the merge again.` Job green, PR open. Master changed during the run because H1, H4 and H5 merged within minutes. Probably a race, not a rule failure.
- Re-run: `#4: merged docs-only PR`.

### H3: delete and rename
- Branch `t/h3-delete-rename`, cut from master after H2 merged: deletes `docs/old.md`, renames `docs/guide.md` to `docs/guide-renamed.md`.
- Merged on first run.

### H4: nested folder
- Adds `src/docs/nested.md`. Merged. Any non-dot parent folder qualifies.

### H5: draft, then ready
- Draft run `44209396612`: job `skipped`, no `#3:` line. PR open, draft.
- `gh pr ready 3`: run `44211383523`: `#3: merged docs-only PR`.

### H6: README change, then revert
- Run `44211253001`: `#6: skipped (changes outside docs/*.md)` because `README.md` changed.
- `README.md` restored to master's content (commit `744e4e7`). Run `44211811224`: `#6: merged docs-only PR`.

### H7: sub-folder under docs
- Branch `t/h7-deep` adds `docs/sub/deep.md`.
- Run `44217823007`: `#22: merged docs-only PR`. `mergedBy` = `app/docs-merge-bot-sandbox-sch`.
- The nested rule reached master as commit `3fa78c63c`, pushed directly to `master` at 01:12 by your account. The ruleset did not block that push at the time.

### N1, N3 to N10: must not merge
- Each run: `#N: skipped (changes outside docs/*.md)`, job green, PR open. Run IDs are in the status table.

### N2: dot folder under docs
- Branch `t/n2-dot-in-docs` adds `docs/.hidden/deep.md`.
- Run `44217950658`: `#23: skipped (changes outside docs/*.md)`. Job green, PR open.
- Earlier PR #8 used `docs/sub/deep.md`. That was the old N2 case and was skipped under the one-level rule. It is no longer the expected result.

### N11: base `release`
- Job `skipped`. The log has no `#2:` line, because the base-branch check runs first. As expected.

### E1: approval required
- PR #18 (`docs/e1.md`) merged. `master-review` did not apply: its branch include list was empty. Setting the target to the default branch fixed it. You made that change.
- PR #20 (`docs/e1-retry2.md`), after the target fix and before the App was added to the bypass list: `##[warning]#20: merge rejected (405): Repository rule violations found`. Job green, PR open. This is the valid E1 result.
- PR #24 (`docs/e1-retry3.md`) was a re-run after the bypass was added. It merged: `#24: merged docs-only PR`, `mergedBy` = `app/docs-merge-bot-sandbox-sch`. E1 needs the App off the bypass list, so this run is not a valid E1 result.

### E2: App on bypass list
- Bypass list: Integration `5751982`, mode `always`. You added it.
- PR #21 (`docs/e2.md`): `#21: merged docs-only PR`. `mergedBy` = `app/docs-merge-bot-sandbox-sch`.

### E3: squash disabled
- Set `allow_squash_merge=false` through the API.
- PR #19 (`docs/e3.md`): `##[warning]#19: merge rejected (405): Squash merges are not allowed on this repository.` Job green, PR open.
- Squash merge re-enabled right after the run.

## Repo changes made during testing
- `master` commit `a96e3eb` adds the fixtures `docs/guide.md` and `docs/old.md` (plan A2).
- `master` commit `3fa78c63c` (your direct push) puts the nested `DOCS_PATH` rule on master.
- Bot merges to `master`: #1, #3, #4, #5, #6, #17, #18, #21, #22, #24.
- `master-review` ruleset: target and bypass list changed by you.
- Squash merge setting: off during E3, back on now.
- `DOCS_MERGE_BOT_APP_ID` and `DOCS_MERGE_BOT_PRIVATE_KEY`: unchanged. The variable was never deleted.
- Test PRs still open: #2, #7 to #16, #19, #20, #23. Remote test branches are still there. Cleanup is plan D7.

## Open items
1. **E1 re-run needs the App off the bypass list.** The App is on the bypass list (added for E2), so E1 PRs merge. To re-run E1 as designed, remove the App from the `master-review` bypass list, run E1, then add it back.
2. A 405 leaves the PR open with a green job, and nothing retries it (H2 showed this). Plan section 7, item 4.
3. Plan section 7, item 2 is still open: any non-dot folder named `docs` qualifies, at any depth. Decide before production.

## App setup (re-iterated)

### Create the App
1. Go to Settings → Developer settings → GitHub Apps → New GitHub App. Direct link: `https://github.com/settings/apps/new`.
2. **GitHub App name:** `docs-merge-bot-sandbox-<initials>`. Must be globally unique. Used here: `docs-merge-bot-sandbox-sch`.
3. **Homepage URL:** `https://github.com/<OWNER>/<test-repo>`. Leave **Callback URL** and **Setup URL** empty. Leave **Request user authorization during installation** off.
4. **Webhook → Active:** untick. The workflow does not use webhooks.
5. **Repository permissions:**
   - Contents: Read and write (creates the merge commit).
   - Pull requests: Read and write (reads PR files and merges the PR).
   - Metadata: Read-only (added automatically).
   - All other repository permissions: No access. No organization or account permissions.
6. **Where can this GitHub App be installed?** Only on this account.
7. Click **Create GitHub App**.
8. On the App page, under **About**, copy the **App ID**. Here: `5751982`.
9. Left sidebar → **Credentials** → **Key pairs** → **New key**. A `.pem` file downloads. Treat it as a password and do not commit it. The App allows 25 keys; delete old ones under Key pairs when needed.

### Install on the test repo
10. Left sidebar → **Install App** → **Install** next to your account → **Only select repositories** → choose the test repo → **Install**.
11. Check: the App must be installed on the repo that runs the workflow. If it is not, the token step fails with `Not Found - get-a-repository-installation` and the job is red. This is what H1 showed before the install.

### Connect to the repo
12. Repo → Settings → Secrets and variables → Actions → **Variables** → **New repository variable**. Name: `DOCS_MERGE_BOT_APP_ID`. Value: the App ID.
13. **Secrets** → **New repository secret**. Name: `DOCS_MERGE_BOT_PRIVATE_KEY`. Value: the entire `.pem` file, including the `-----BEGIN ...-----` and `-----END ...-----` lines.
14. Nothing else. The workflow needs no organization-level secrets.

### If the default branch has rulesets
15. The ruleset target must include the default branch (`~DEFAULT_BRANCH`). A ruleset with an empty target list does not apply. PR #18 merged for that reason.
16. To let the bot merge past an approval rule, add the App to the ruleset's **Bypass list** (Integration, mode `always`). Verified by E2. Any approval-rule test (E1) must run with the App off this list.
