---
name: copilot-review
description: This skill should be used when the user asks to "run copilot review", "start a copilot review cycle", "run the copilot review loop", "have copilot review the PR", "let copilot review this", or wants to repeatedly request GitHub Copilot reviews on a pull request until no new comments are generated.
user-invocable: true
---

Run a Copilot review cycle on the current branch's open pull request. Repeat until Copilot generates 0 new inline comments.

## Step 0 — Identify the PR

Resolve the current repo and open PR:

```bash
gh repo view --json nameWithOwner --jq '.nameWithOwner'   # → "owner/repo"
gh pr view --json number,url
```

If no open PR exists for the current branch, stop and tell the user.

Derive `OWNER`, `REPO`, and `PR_NUMBER` from these outputs. Use them in all subsequent API calls.

## Step 1 — Snapshot the current review count

Before requesting, record how many Copilot reviews already exist so the poll in Step 2 has a baseline:

```bash
gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/reviews \
  --jq '[.[] | select(.user.login == "copilot-pull-request-reviewer[bot]")] | length'
```

Also record the current UTC timestamp as `REQUEST_TIME` — use it in Step 3 to filter only new comments.

## Step 2 — Request a Copilot review

```bash
gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/requested_reviewers \
  -X POST -f 'reviewers[]=copilot-pull-request-reviewer[bot]'
```

## Step 3 — Poll until the review appears

Poll every 30 seconds (max 20 iterations ≈ 10 minutes). Stop when the Copilot review count exceeds the baseline from Step 1:

```bash
for i in $(seq 1 20); do
  count=$(gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/reviews \
    --jq '[.[] | select(.user.login == "copilot-pull-request-reviewer[bot]")] | length')
  echo "$(date '+%H:%M:%S') — reviews: $count"
  [ "$count" -gt $BASELINE ] && echo "New review detected!" && break
  sleep 30
done
```

If the timeout is reached without a new review, stop and report the timeout to the user.

## Step 4 — Fetch new inline comments

Retrieve only comments created after `REQUEST_TIME`:

```bash
gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/comments \
  --jq '[.[] | select(.created_at > "REQUEST_TIME") | {id, path, line, body}]'
```

**If 0 new inline comments → the cycle is complete.** Report a summary (total rounds, total fixes applied) and stop.

## Step 5 — Evaluate each comment (be opinionated)

For each comment:

1. Read the referenced file at the indicated path and line for full context.
2. Make a **fix or skip** decision:
   - **Fix** when: the comment identifies a real bug, incorrect semantics, a meaningful inconsistency, or a clear improvement aligned with existing patterns.
   - **Skip** when: the comment is purely stylistic, contradicts established codebase conventions, relies on incomplete context, or addresses a scenario that cannot occur in practice.
3. State the reasoning explicitly before acting — do not silently skip or silently fix.

## Step 6 — Apply fixes

- Edit the relevant files directly.
- If a test suite covers the changed area, run it and confirm it passes before committing.
- Stage only the files that were changed as part of fixing Copilot comments.
- Commit with a concise message describing what was addressed and push:

```bash
git add <changed files>
git commit -m "Address Copilot review comments: <summary>"
git push
```

## Step 7 — Resolve addressed threads, leave skipped ones open

Get the thread node IDs via GraphQL (needed because REST has no resolve endpoint):

```bash
gh api graphql -f query='{
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 50) {
        nodes {
          id
          isResolved
          comments(first: 1) { nodes { databaseId } }
        }
      }
    }
  }
}'
```

Match each thread's `comments.nodes[0].databaseId` to the comment IDs from Step 4 to identify which thread corresponds to which comment. Resolve only the threads whose comments were fixed:

```bash
gh api graphql -f query='mutation {
  resolveReviewThread(input: { threadId: "PRRT_..." }) {
    thread { id isResolved }
  }
}'
```

Leave skipped threads unresolved so they remain visible for the user.

## Step 8 — Loop

Return to Step 1 and run another cycle.

The loop exits only when Step 4 finds 0 new inline comments.
