---
name: azure-devops-create-pr
description: Create or update a pull request in Azure DevOps Repos from the current branch using the az CLI, including a correctly formatted merge commit message, squash merge, and auto-complete. Use when the user asks to raise, open, create, refresh, or update a PR (or pull request) in Azure DevOps / ADO / dev.azure.com, or asks whether the current branch already has a PR.
icon: git-branch
color: blue
---

# Create or Update an Azure DevOps Pull Request

Create a pull request in Azure DevOps Repos from the current git branch to the target branch — or bring an existing PR for that branch up to date — using `git` to discover context and `az repos pr` to talk to Azure DevOps.

**PR creation is always a two-step process: `az repos pr create`, then `az repos pr update`.** The required merge commit message embeds the PR number, which does not exist until the PR has been created, so the merge commit message, squash merge, and auto-complete can only be set by a follow-up update.

## Non-negotiable rules

1. **Run every `git` and `az` command outside the sandbox.** The `az` CLI is already signed in to Azure DevOps and relies on credentials and config that are unreachable from the sandbox. Pass `required_permissions: ["all"]` on the very first command and on every command after it. Do not "try it in the sandbox first" — a sandboxed `az` call wastes time and fails with a confusing auth error.
2. **Never run `az login`, `az devops login`, or ask for a PAT.** Authentication is already handled. If a command fails with an auth error, report it and stop rather than attempting to re-authenticate.
3. **Never pass `--auto-complete`, `--delete-source-branch`, or `--squash` to `az repos pr create`.** They belong to the step 6 update, alongside `--merge-commit-message`. A PR that auto-completes before step 6 runs will merge with the wrong commit message, which is the whole problem this skill exists to prevent.
4. **Always squash-merge.** Every step 6 `az repos pr update` must include `--squash true`. Do not omit it, do not pass `--squash false`, and do not leave merge strategy to repo defaults. Auto-complete must complete as a squash merge.
5. **Never create a second PR for a branch that already has an open one.** Step 3 decides whether step 5 runs at all; if a PR exists, skip step 5 and let step 6 update it.
6. **Argument scoping differs per subcommand.** `create` and `list` need `--organization`, `--project`, and `--repository`. `update`, `show`, and `set-vote` identify the PR by `--id` and accept only `--organization` — passing `--project` or `--repository` to them fails with `unrecognized arguments`. Never rely on `--detect` or on configured defaults for any of them.

## Workflow at a glance

```
Step 1  resolve org / project / repo   (git remote -v)
Step 2  resolve source / target branch (git rev-parse)
Step 3  look for an existing PR        (az repos pr list)
Step 4  draft title + description      (git log + git diff)
Step 5  az repos pr create             (draft, no auto-complete, no delete-source-branch, no squash)
        -- skipped entirely if step 3 found an existing PR --
Step 6  az repos pr update             (final title/description + merge commit message
                                        + auto-complete + delete-source-branch + squash
                                        + draft state)
Step 7  az repos pr show               (state check), then az repos pr set-vote (self-approve)
        -- vote skipped unless status is active and isDraft is false --
Step 8  report PR id, URL, and what changed
```

### Step 1 — Resolve organization, project, and repository from the git remote

```bash
git remote -v
```

Parse the `origin` fetch URL. Azure DevOps remotes come in these shapes:

| Remote URL                                                                           | Organization                         | Project  | Repository                 |
| ------------------------------------------------------------------------------------ | ------------------------------------ | -------- | -------------------------- |
| `git@ssh.dev.azure.com:v3/colortokens/Xcloud/terraform-infrastructure`               | `https://dev.azure.com/colortokens/` | `Xcloud` | `terraform-infrastructure` |
| `https://colortokens@dev.azure.com/colortokens/Xcloud/_git/terraform-infrastructure` | `https://dev.azure.com/colortokens/` | `Xcloud` | `terraform-infrastructure` |
| `https://colortokens.visualstudio.com/Xcloud/_git/terraform-infrastructure`          | `https://dev.azure.com/colortokens/` | `Xcloud` | `terraform-infrastructure` |

Rules for parsing:

- The organization is always the URL form `https://dev.azure.com/<org>/` — build it even when the remote is SSH or `visualstudio.com`.
- Strip the `v3/` prefix (SSH) and the `_git/` segment (HTTPS) before splitting on `/`.
- Strip a trailing `.git` from the repository name if present.
- If the project or repository name contains spaces (URL-encoded as `%20`), decode it and quote the value when passing it to `az`.

If `origin` is not an Azure DevOps remote, stop and tell the user this is not an Azure DevOps repository.

### Step 2 — Resolve the source and target branches

```bash
git rev-parse --abbrev-ref HEAD                 # source branch
git rev-parse --abbrev-ref origin/HEAD          # e.g. "origin/main" -> target branch is "main"
```

- **Source branch**: the current branch. If it is the same as the target branch, stop and tell the user there is nothing to raise a PR from.
- **Target branch**: strip the `origin/` prefix from `git rev-parse --abbrev-ref origin/HEAD`. If that command fails (`origin/HEAD` not set), run `git remote set-head origin --auto` and retry. Use whatever target branch the user explicitly asked for instead, if they named one.
- **The source branch must exist on the remote.** Check and push if needed:

```bash
git ls-remote --exit-code --heads origin "<source-branch>"
git push -u origin HEAD          # only if the branch is missing on the remote
```

Also confirm there are no unpushed local commits (`git status -sb` shows `ahead`); if there are, push them so the PR reflects the full branch, and mention it to the user.

### Step 3 — Look for an existing PR for this branch

```bash
az repos pr list \
  --organization "https://dev.azure.com/<org>/" \
  --project "<project>" \
  --repository "<repo>" \
  --source-branch "<source-branch>" \
  --status active \
  --output json
```

- **Empty list** → no PR yet. Continue to step 4, then step 5 to create one.
- **Non-empty list** → record `pullRequestId`, `title`, `description`, and `mergeCommitMessage` from the first entry. Continue to step 4, then **skip step 5** and go straight to step 6, which updates the existing PR. Do not create another PR.

If `description` or `mergeCommitMessage` are missing from the list output, fetch the full record:

```bash
az repos pr show --id <pr-id> --organization "https://dev.azure.com/<org>/" --output json
```

### Step 4 — Draft the title and description

Do this whether or not a PR already exists. For a new PR it is the text you create with; for an existing one it is what tells you in 6a whether the current title and description are still accurate.

**Gather the material first** (all outside the sandbox):

```bash
git log --pretty=format:'%h %s%n%b' "origin/<target-branch>..<source-branch>"
git diff --stat "origin/<target-branch>...<source-branch>"
git diff "origin/<target-branch>...<source-branch>"
```

The three-dot form shows only what the source branch introduced relative to its merge base with the target, which is exactly what the PR will contain. If the full diff is very large, rely on `--stat` plus the diffs of the most significant files rather than reading everything.

**Title**: one line of plain, readable English that describes what the branch accomplishes. Use the user's title verbatim if they gave one.

**Write it as normal text, not as a commit subject.** A PR title is read in a list by people deciding whether to open it, so it should read like a short headline: sentence case, no trailing period, no `type(scope):` prefix, no tag. Do not copy the conventional-commit style of the branch's commits, and do not compress it into telegraphic shorthand — spend the extra few words to make it descriptive, while staying concise.

| Instead of                                                 | Write                                                                           |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `docs(compliance): relocate frameworks and add onboarding` | Move framework coverage onto each scan type page and add cloud onboarding guide |
| `docs(compliance): add scanning and Tanium guides`         | Document compliance scanning and the Tanium integration                         |
| `feat(ssh): generate site PEM keys automatically`          | Generate site PEM keys automatically during SSH setup                           |
| `fix(vault): use xcloudkv site names under 24 chars`       | Keep Vault xcloudkv site names within the 24-character limit                    |
| `refactor(tfvars): unify site and cluster var files`       | Unify site and cluster tfvars into one file layout                              |

**Description**: markdown, **strictly under 4000 characters** (the Azure DevOps field limit — the command fails if you exceed it). Bias heavily toward _why_ over _what_: the diff is already visible to reviewers, the motivation is not. A good shape:

```markdown
## Summary

Briefly explain what this PR does and why.

## Why

Explain Problem, motivation, requirement, or ticket this addresses, and the consequence of not doing it. Include relevant context, constraints, or links to the issue/ticket.

## What changed

A handful of bullets at the level of behavior and design decisions, not a file-by-file replay of the diff.

## Notes for reviewers

Trade-offs considered, follow-ups deliberately left out, migration or rollout steps, blast radius, anything to test manually.
```

Drop any section that would be empty. Count the characters before creating the PR and trim the "What changed" bullets first if you are over the limit.

**Never mention the PR's own state or the mechanics of raising it.** The description documents the change; it is not a status report on the pull request. Write nothing about the PR being a draft or published, about auto-complete, about the merge commit message, about the source branch being deleted, or about the steps in this skill. A line like `The PR stays draft.` is wrong on both counts: draft status is temporary and already visible in the UI, so the sentence goes stale as soon as the PR is published and then has to be edited out by hand. "Notes for reviewers" is for the risk, trade-offs, and testing of the change, never for its publication state.

**Pass the description safely.** Write it to a temp file, then substitute the file contents so newlines and markdown survive shell quoting:

```bash
cat > /tmp/pr-description.md <<'EOF'
## Summary
...
EOF
wc -m /tmp/pr-description.md    # confirm < 4000
```

Then use `--description "$(cat /tmp/pr-description.md)"`. Never inline a multi-line description directly into the command string.

### Step 5 — Create the pull request

**Skip this step entirely if step 3 found an existing PR** and go straight to step 6, which updates it.

```bash
az repos pr create \
  --organization "https://dev.azure.com/<org>/" \
  --project "<project>" \
  --repository "<repo>" \
  --source-branch "<source-branch>" \
  --target-branch "<target-branch>" \
  --title "<title>" \
  --description "$(cat /tmp/pr-description.md)" \
  --draft true \
  --output json
```

**`--draft true` is the default.** Creating as a draft keeps reviewers and build policies idle while the PR is still being set up, and the PR stays a draft through step 6 unless the user asked otherwise.

Pass `--draft false` instead only when the user explicitly asks for a published PR (for example "publish it", "not a draft", "ready for review"). Carry that same choice into step 6.

**No `--auto-complete`, no `--delete-source-branch`, and no `--squash` here** — see rule 3. Squash is set in step 6 with the other completion flags.

Add work item linking **only when the user supplies work item IDs**:

```bash
  --work-items 12345 12346 \
  --transition-work-items true
```

`--work-items` takes space-separated IDs. `--transition-work-items true` moves those items to their next state (e.g. Active → Resolved) when the PR completes.

Read `pullRequestId` from the JSON response and go to step 6.

### Step 6 — Set the final text, merge commit message, squash, and completion flags

Every run ends up here, whether the PR was just created in step 5 or already existed. A single `az repos pr update` call carries the final title and description, the merge commit message, and the completion flags.

#### 6a — Settle the final title and description

- **PR created in step 5** — the title and description you just set are final. Nothing to decide; omit `--title` and `--description` from the update below.
- **PR already existed** — compare its `title` and `description` from step 3 against what you drafted in step 4 and judge whether the existing text is **still complete and relevant** for the commits now on the branch. Treat it as stale when any of these hold:
  - Commits added since the PR was opened introduce work the description does not mention.
  - The description describes work that is no longer in the diff.
  - The title no longer covers the scope of the branch (e.g. it names one fix but the branch now carries three).
  - The description is a placeholder, a bare commit subject, or empty.
  - The description mentions the PR's own state — draft, published, auto-complete, merge commit message, branch deletion. Strip those lines out even when the rest of the description is accurate and worth keeping.

  **If it is still accurate**, keep it — do not rewrite good prose just because you generated an alternative — and omit `--title` and `--description` from the update. **If it is stale**, include them, keeping the author's own wording where it is still true rather than replacing the whole body.

Whichever text ends up final is what the merge commit message in 6b must be built from. That is why the two happen in one step: a title or description change and the merge commit message that quotes them can never drift apart.

#### 6b — Build the merge commit message

**Required merge commit message format:**

```
Merged PR <PR_Number>: <PR_Title>

<PR_Description_Or_Squashed_Commit_Messages>
```

The first line is the commit subject. The body is the PR description. These PRs always squash-merge (rule 4); if the description is thin, use the list of commit subjects from the branch instead. Keep the whole message under 4000 characters — if the subject plus the full description exceeds that, use only the Summary/Why sections plus commit subjects as the body.

The blank line after the subject is deliberate: git and every tool that reads git history treat the first paragraph as the subject, so without it the message renders as one run-on subject line.

Build it in a temp file, using the PR ID (from step 5 for a new PR, from step 3 for an existing one) and the **final** title and description settled in 6a:

```bash
{
  printf 'Merged PR %s: %s\n\n' "<pr-id>" "<title>"
  cat /tmp/pr-description.md
} > /tmp/pr-merge-message.txt
wc -m /tmp/pr-merge-message.txt    # confirm < 4000
```

#### 6c — Apply the update

```bash
az repos pr update --id <pr-id> \
  --organization "https://dev.azure.com/<org>/" \
  --merge-commit-message "$(cat /tmp/pr-merge-message.txt)" \
  --auto-complete true \
  --delete-source-branch true \
  --squash true \
  --draft true \
  --output json
```

- `--merge-commit-message` — the message from 6b; the reason this second step exists.
- `--auto-complete true` — merge as soon as policies pass.
- `--delete-source-branch true` — clean up the branch on merge.
- `--squash true` — **required on every update** (rule 4). Squash source commits into one commit on merge. Never omit this flag and never pass `false`.
- `--draft true` — the default for a PR created in step 5. On `update`, `--draft` is the publish switch: `true` keeps or returns the PR to draft, `false` publishes it. See the table below — do not copy this line blindly onto an existing PR.
- `--title "<title>"` and `--description "$(cat /tmp/pr-description.md)"` — **only** when 6a found the existing text stale.
- `--transition-work-items true` — only if work items are linked and the user wants them advanced on completion.

New work item IDs are linked separately with `az repos pr work-item add`; `az repos pr update` has no `--work-items` flag.

Which `--draft` value to send:

| Situation                                                       | Send                                                     |
| --------------------------------------------------------------- | -------------------------------------------------------- |
| PR created in step 5, no instruction about publishing (default) | `--draft true` — created as a draft, stays a draft       |
| PR created in step 5, user explicitly asked for a published PR  | `--draft false`                                          |
| PR already existed, no instruction about its state              | **omit `--draft`** — never flip a state the author chose |
| User explicitly asked to publish, or to return to draft         | `--draft false` / `--draft true` as asked                |

Omitting `--draft` on an existing PR matters: sending `--draft true` would demote a PR the author had already published, and `--draft false` would publish one they deliberately left as a draft. Keep this consistent with step 5 — never publish a PR the user wanted left as a draft, and never leave a draft when they asked for it published.

Because the default end state is a draft, **auto-complete will not fire until someone publishes the PR** — drafts never complete, even with all policies green. That is intended, not a bug. Say so when reporting in step 8 so the user knows the PR is armed but waiting on publication.

Verify the result: confirm the response has `autoCompleteSetBy` populated, `mergeCommitMessage` set, `completionOptions.deleteSourceBranch` set as intended, `completionOptions.squashMerge` true, and `isDraft` matching what you intended (`true` for a PR created in step 5, `false` only on an explicit publish request, and unchanged for a PR that already existed). If `mergeCommitMessage` came back empty or truncated, or squash is not set, re-run this step rather than leaving the PR to merge with a default message or a non-squash merge.

### Step 7 — Self-approve the PR, but only if it is open and published

Voting is conditional. **Check the PR's current state first** and only vote if it is still open and not a draft:

```bash
az repos pr show --id <pr-id> \
  --organization "https://dev.azure.com/<org>/" \
  --query "{status:status, isDraft:isDraft}" \
  --output json
```

Vote only when **both** conditions hold:

| Field     | Required value | Skip when                                                                   |
| --------- | -------------- | --------------------------------------------------------------------------- |
| `status`  | `active`       | `completed` (already merged — auto-complete got there first) or `abandoned` |
| `isDraft` | `false`        | `true` (still a draft, which is the default end state of step 6)            |

If either check fails, **skip the vote entirely** and say why in step 8. Do not try to publish the PR, reactivate it, or work around the condition to make the vote possible — the state is the answer.

**In the default flow the PR is left as a draft, so this step normally does not run.** That is expected: a draft is not ready for approval, and its `isDraft: true` is exactly the signal to hold off. Voting typically only happens when the user asked for a published PR.

When both checks pass, record the approval as the signed-in user — the last action of the run, outside the sandbox like every other `az` call (rule 1):

```bash
az repos pr set-vote --id <pr-id> \
  --organization "https://dev.azure.com/<org>/" \
  --vote approve \
  --output json
```

Like `update`, this command identifies the PR by `--id` and accepts only `--organization` — no `--project` or `--repository`.

Accepted `--vote` values are `approve`, `approve-with-suggestions`, `reject`, `reset`, and `wait-for-author`. Always use `approve` here unless the user asks for something else.

Two outcomes are still possible after a passing check and are **not** failures — report them and move on to step 8 rather than retrying:

- **The PR merged in between.** The state check and the vote are two separate calls, so a published PR with auto-complete armed and no blocking policies can complete in the gap. Voting on a completed PR is rejected. Nothing is wrong; the PR is merged.
- **The vote does not satisfy the reviewer policy.** If the branch has a minimum-reviewers policy with "prohibit requestor from approving their own changes" enabled, the approval is recorded but does not count toward the policy. Say so plainly so the user knows another reviewer is still needed.

### Step 8 — Report back

From the step 6 update response (the step 7 vote response only describes the reviewer), read `pullRequestId`, `title`, `sourceRefName`, `targetRefName`, and `status`, and build the **PR web URL**:

```
https://dev.azure.com/<org>/<project>/_git/<repo>/pullrequest/<pullRequestId>
```

Tell the user:

- Whether you created a new PR or updated an existing one, and on the update path what you changed (title, description, merge commit message, flags) or that you deliberately left the existing text alone.
- PR ID, title, and the branches involved.
- That auto-complete, delete-source-branch, and squash merge are on, and the merge commit message that was set.
- Whether the PR is a draft or published. If it is a draft (the default), state plainly that it will not auto-complete until it is published.
- Whether the self-approval was recorded, skipped, or refused. If it was skipped, name the reason from the step 7 check — the PR is a draft, or it has already merged or been abandoned. If it was recorded but does not count toward a reviewer policy, that another reviewer is still needed.
- Any linked work items, and the URL.

## Asking the user

Use the ask-questions tool instead of guessing when:

- The remote has multiple Azure DevOps remotes and it is unclear which one to use.
- The default branch cannot be determined and no target branch was given.
- An existing PR's description contains hand-written context that your regenerated description would drop, and it is unclear whether to keep or replace it.

Do **not** ask about the title or description in the normal case — draft them from the commits and diff, create or update the PR, and let the user ask for edits afterward. Do **not** ask whether to update an existing PR; 6a decides that from the diff.

## Command reference (`az repos pr`, azure-devops extension)

Only the parts of the CLI this workflow needs. Full docs: `az repos pr create --help`.

### `az repos pr create`

| Flag                                           | Notes                                                                                                                      |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `--source-branch` / `-s`                       | Source branch name, e.g. `dev`. Accepts the short name; `refs/heads/dev` also works.                                       |
| `--target-branch` / `-t`                       | Target branch name. Defaults to the repo's default branch if omitted — still pass it explicitly.                           |
| `--title`                                      | PR title.                                                                                                                  |
| `--description` / `-d`                         | Markdown body. Each value passed becomes a new line, so a single quoted multi-line string is fine. Max 4000 characters.    |
| `--organization` / `--org`                     | Org URL, e.g. `https://dev.azure.com/colortokens/`.                                                                        |
| `--project` / `-p`                             | Project name or ID.                                                                                                        |
| `--repository` / `-r`                          | Repository name or ID.                                                                                                     |
| `--draft {false,true}`                         | Create as draft / work in progress. Default to `true`; pass `false` only when the user explicitly asks for a published PR. |
| `--work-items`                                 | Space-separated work item IDs to link.                                                                                     |
| `--transition-work-items {false,true}`         | Advance linked work items to their next state on completion. Default `false`.                                              |
| `--required-reviewers`, `--optional-reviewers` | Space-separated users or groups.                                                                                           |
| `--labels`                                     | Space-separated labels.                                                                                                    |
| `--output` / `-o`                              | `json` (default), `jsonc`, `table`, `tsv`, `yaml`, `none`.                                                                 |
| ~~`--auto-complete`~~                          | **Do not use on create.** Belongs to step 6 — see rule 3.                                                                  |
| ~~`--delete-source-branch`~~                   | **Do not use on create.** Belongs to step 6 — see rule 3.                                                                  |
| ~~`--squash`~~                                 | **Do not use on create.** Belongs to step 6 — always `--squash true` there (rule 4). CLI default is `false`.               |
| ~~`--merge-commit-message`~~                   | Accepted here, but useless: the PR number it must contain does not exist yet. Set it in step 6.                            |
| ~~`--open`~~                                   | Opens a browser. Avoid in an agent context.                                                                                |

### `az repos pr update`

Identifies the PR by `--id` and takes `--organization` only — **no `--project`, `--repository`, or `--work-items`**.

| Flag                                        | Notes                                                                                                                                                                             |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--id`                                      | PR ID. Required.                                                                                                                                                                  |
| `--merge-commit-message`                    | Message displayed when the commits are merged. The point of step 6.                                                                                                               |
| `--auto-complete {false,true}`              | Complete automatically once policies pass and the merge is clean.                                                                                                                 |
| `--delete-source-branch {false,true}`       | Delete the source branch after completion.                                                                                                                                        |
| `--squash {false,true}`                     | **Always `--squash true`** (rule 4). Squash source commits into one commit when merging into the target. Do not omit; CLI default if omitted is not squash.                       |
| `--draft {false,true}`                      | The publish switch: `false` publishes a draft, `true` keeps or returns it to draft. Default to `true` for a PR created in step 5; omit it entirely for a PR that already existed. |
| `--title`                                   | New title. Send only when 6a found the existing title stale.                                                                                                                      |
| `--description` / `-d`                      | New markdown body. Same 4000-character limit and same newline-per-value behavior as on `create`. Send only when 6a found the existing description stale.                          |
| `--status {active,abandoned,completed}`     | Set the PR state. Use with care — `completed` merges immediately.                                                                                                                 |
| `--transition-work-items {false,true}`      | Advance linked work items on completion.                                                                                                                                          |
| `--bypass-policy`, `--bypass-policy-reason` | Skip required policies. Do not use unless the user explicitly asks.                                                                                                               |

### `az repos pr set-vote`

Identifies the PR by `--id` and takes `--organization` only — **no `--project` or `--repository`**.

| Flag                                                                     | Notes                                                                        |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| `--id`                                                                   | PR ID. Required.                                                             |
| `--vote {approve,approve-with-suggestions,reject,reset,wait-for-author}` | Required. Use `approve` for the step 7 self-approval. `reset` clears a vote. |
| `--organization` / `--org`                                               | Org URL.                                                                     |

### `az repos pr list`

| Flag                                          | Notes                                                 |
| --------------------------------------------- | ----------------------------------------------------- |
| `--source-branch` / `-s`                      | Only PRs originating from this branch.                |
| `--target-branch` / `-t`                      | Only PRs targeting this branch.                       |
| `--status {active,completed,abandoned,all}`   | Filter by status. Use `active` for the step 3 lookup. |
| `--organization`, `--project`, `--repository` | Same as on `create`; always pass them.                |
| `--creator`, `--reviewer`                     | Filter by user.                                       |
| `--top`, `--skip`                             | Paging.                                               |

### Other commands in the family

| Command                                                     | Use                                                                                               |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `az repos pr show --id <id> --organization <org-url>`       | Full details of one PR: `status`, `isDraft`, `mergeCommitMessage`. Drives the step 7 state check. |
| `az repos pr work-item add --id <id> --work-items <ids...>` | Link work items after creation.                                                                   |
| `az repos pr work-item list --id <id>`                      | List linked work items.                                                                           |
| `az repos pr reviewer add --id <id> --reviewers <user...>`  | Add reviewers after creation.                                                                     |
| `az repos pr reviewer list --id <id>`                       | Check recorded votes, including your own.                                                         |
| `az repos pr policy list --id <id>`                         | Inspect why auto-complete is still waiting.                                                       |
| `az repos pr checkout --id <id>`                            | Check out a PR's source branch locally.                                                           |

## Troubleshooting

| Symptom                                                                                       | Cause and fix                                                                                                                                                     |
| --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Before you can run Azure DevOps commands, you need to run the login command`                 | The command ran inside the sandbox. Re-run it outside the sandbox; do not attempt to log in.                                                                      |
| Merged commit says `Merged PR 123: <branch name>` or has no body                              | Step 6 was skipped, or the PR auto-completed before step 6 ran. Never pass `--auto-complete` to `create`.                                                         |
| `unrecognized arguments: --project ... --repository ...`                                      | Those flags are not valid on `update`, `show`, or `set-vote`. Pass only `--id` and `--organization`.                                                              |
| `unrecognized arguments: --work-items`                                                        | `update` cannot link work items. Use `az repos pr work-item add`.                                                                                                 |
| `TF401398: The pull request cannot be activated because the source branch ... does not exist` | The source branch was never pushed. Push it, then retry.                                                                                                          |
| `TF401179: An active pull request for the source and target branch already exists`            | Step 3 was skipped or raced. Run `az repos pr list`, then skip step 5 and update that PR in step 6.                                                               |
| `--description` or `--merge-commit-message` rejected as too long                              | Exceeded 4000 characters. Trim and retry.                                                                                                                         |
| `The following arguments are required: --repository`                                          | Org/project/repo parsing from the remote failed. Re-check the `git remote -v` output against the table in step 1.                                                 |
| Auto-complete appears not to be set                                                           | `autoCompleteSetBy` is null, or branch policies are blocking. Check `az repos pr policy list --id <id>` and report the policy status to the user.                 |
| Merge is not a squash (multiple commits landed, or `squashMerge` is false)                    | Step 6 omitted `--squash true`. Re-run the update with `--squash true` before the PR completes. CLI default is not squash.                                        |
| PR is a draft and never merges despite green policies                                         | Expected: drafts never auto-complete, and draft is the default end state. Publish it (`az repos pr update --id <id> --org <url> --draft false`) when it is ready. |
| A previously published PR came back as a draft                                                | Step 6 sent `--draft true` for a PR it did not create. Omit `--draft` for an existing PR; republish with `--draft false`.                                         |
| Description mentions draft/publish state                                                      | The step 4 constraint was ignored. Strip those lines and re-run step 6 with `--title`/`--description` included.                                                   |
| `set-vote` fails because the PR is completed or abandoned                                     | The step 7 state check was skipped, or the PR merged in the gap between the check and the vote. Expected, not an error — report the state and stop.               |
| No approval recorded and the PR is a draft                                                    | Correct behaviour, not a bug: step 7 skips the vote while `isDraft` is `true`. Publish the PR first if an approval is genuinely wanted.                           |
| `set-vote` succeeds but the PR still needs approval                                           | A minimum-reviewers policy prohibits the requestor from approving their own changes. Tell the user another reviewer is required.                                  |
