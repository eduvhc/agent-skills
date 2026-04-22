---
name: lyzer-workflow
description: "End-to-end development workflow: read a Jira ticket, implement the changes, push to Azure DevOps, create a PR, and link everything back to Jira. Use this skill whenever the user gives a Jira ticket key and wants to go from ticket to PR, or when they say things like 'work on EPB-123', 'start EPB-456', 'open a PR for EPB-789', or 'do the full workflow for this ticket'. Also trigger when the user wants to push code and create a PR for a Jira ticket in the Lyzer platform."
version: 1.0.0
---

# Lyzer Workflow

End-to-end development workflow that automates the full cycle from Jira ticket to Azure DevOps PR with bidirectional linking.

## Input

The user provides a Jira ticket key as the argument (e.g., `/lyzer-workflow EPB-123`). Optionally they may specify a target branch (defaults to `dev`).

## Critical: Present a plan before doing anything

After reading the Jira ticket (Step 1), present a plan to the user and wait for approval before writing any code. The plan should include:

- What the ticket asks for (your interpretation)
- Which files you intend to modify or create
- A brief outline of the approach
- Any open questions or ambiguities

Only proceed to Step 2 after the user confirms the plan. This avoids wasted work from misunderstanding the ticket and gives the user a chance to steer the approach before any code is written.

## Workflow Steps

### Step 1 — Read the Jira ticket

Fetch the ticket details so you understand what needs to be done.

```
mcp__atlassian__jira_get_issue(issue_key="<TICKET_KEY>")
```

Read the summary, description, acceptance criteria, and any linked tickets. This context drives the implementation.

**Then present the plan and wait for the user to approve before continuing.**

### Step 2 — Implement the changes

After the user approves the plan:

1. **Create a branch** from the current branch:
   ```bash
   git checkout -b <TICKET_KEY>
   ```

2. **Do the work** — read the relevant code, make the changes, verify them (build, lint, tests as appropriate).

3. **Commit** with a message that references the ticket but focuses on the *what* and *why*:
   ```bash
   git commit -m "$(cat <<'EOF'
   <Short summary of the change>

   <Optional longer explanation if the change is non-trivial>
   EOF
   )"
   ```

Stage only the files you changed — don't use `git add -A`.

### Step 3 — Push to Azure DevOps

Regular git push may fail due to auth. Use the Azure CLI access token as the credential helper:

```bash
git -c credential.helper='!f() { echo username=azrepos; echo "password=$(az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query accessToken -o tsv)"; }; f' push -u origin <TICKET_KEY>
```

If this fails with an auth error, tell the user to run `az login` first and retry.

### Step 4 — Create the PR

Detect the repository name from the git remote:

```bash
git remote get-url origin
# Extract repo name from: https://360hyper@dev.azure.com/360hyper/360hyper/_git/<REPO_NAME>
```

Create the PR with `az repos pr create`:

```bash
az repos pr create \
  --org https://dev.azure.com/360hyper \
  --project 360hyper \
  --repository <REPO_NAME> \
  --source-branch <TICKET_KEY> \
  --target-branch dev \
  --title "<TICKET_KEY>: <Jira ticket summary>" \
  --description "$(cat <<'EOF'
# Ticket

https://lyzer.atlassian.net/browse/<TICKET_KEY>

## Summary

<Bullet points describing what changed and why>

## Test plan

<Checklist of verification steps>
EOF
)"
```

Extract the PR URL from the response. The format is:
```
https://dev.azure.com/360hyper/360hyper/_git/<REPO_NAME>/pullRequest/<PR_ID>
```

The `pullRequestId` field in the JSON response gives you `<PR_ID>`.

### Step 5 — Link the PR back to Jira

Add a comment to the Jira ticket with the PR link:

```
mcp__atlassian__jira_add_comment(
  issue_key="<TICKET_KEY>",
  body="PR: https://dev.azure.com/360hyper/360hyper/_git/<REPO_NAME>/pullRequest/<PR_ID>"
)
```

### Step 6 — Report to the user

Summarize what was done:
- The PR number and URL
- The reviewers auto-assigned (from the `az repos pr create` response)
- Confirmation the Jira comment was added

## Configuration

| Setting | Value |
|---------|-------|
| Azure DevOps Org | `https://dev.azure.com/360hyper` |
| Azure DevOps Project | `360hyper` |
| Jira Base URL | `https://lyzer.atlassian.net/browse/` |
| Default target branch | `dev` |

## Notes

- Target branch defaults to `dev`. If the user specifies a different target, use that instead.
- If the branch already exists locally or on the remote, ask the user how to proceed rather than force-pushing.
- The repository name is auto-detected from `git remote get-url origin`.
