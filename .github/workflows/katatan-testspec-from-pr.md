---
name: KATATAN Test Spec from PR
description: >
  When a pull request is merged, read what it implemented, design test cases for it,
  and append them to a KATATAN test specification.

on:
  pull_request:
    types: [closed]
  workflow_dispatch:
    inputs:
      pr_number:
        description: "Number of an already merged PR to generate test cases for"
        required: true
        type: string

# Only run for merged PRs (closed-without-merge is ignored). Manual runs are always allowed.
if: github.event_name == 'workflow_dispatch' || github.event.pull_request.merged == true

# The agent itself is read-only. Writing to KATATAN happens in the custom safe-output job below.
permissions:
  contents: read
  pull-requests: read
  copilot-requests: write

tools:
  github:
    toolsets: [pull_requests, repos]
  bash:
    - "cat /tmp/gh-aw/katatan/*"
    - "jq *"

# Runs before the agent. Fetches the existing test cases so the agent can avoid duplicates.
# KATATAN_TOKEN is only exposed to this step, never to the agent.
steps:
  - name: Fetch existing KATATAN test cases
    env:
      KATATAN_TOKEN: ${{ secrets.KATATAN_TOKEN }}
      KATATAN_SPEC_ID: ${{ vars.KATATAN_SPEC_ID }}
    run: |
      mkdir -p /tmp/gh-aw/katatan
      out=/tmp/gh-aw/katatan/existing-cases.json
      if [ -z "$KATATAN_TOKEN" ] || [ -z "$KATATAN_SPEC_ID" ]; then
        echo "::warning::KATATAN_TOKEN or KATATAN_SPEC_ID is not set; continuing without existing cases."
        echo '[]' > "$out"
        exit 0
      fi
      if npx -y @katatan/cli@0.1.0 --json test-case list --spec-id "$KATATAN_SPEC_ID" > /tmp/katatan-list.json; then
        # Keep only the fields the agent needs to detect duplicates.
        jq '[.[] | {caseNumber, testSubject, expectedResult}]' /tmp/katatan-list.json > "$out"
      else
        echo "::warning::Could not fetch existing test cases; continuing with an empty list."
        echo '[]' > "$out"
      fi
      echo "Existing test cases: $(jq length "$out")"

safe-outputs:
  jobs:
    add-katatan-test-cases:
      description: >
        Append test cases to the KATATAN test specification, open a QA review issue,
        and comment the result on the PR.
        Call this at most once, with every test case in a single JSON array.
      runs-on: ubuntu-latest
      output: "Test cases were submitted to KATATAN."
      permissions:
        issues: write
        pull-requests: write
      inputs:
        cases_json:
          description: >
            JSON array (as a string) of test cases. Each item:
            {"testSubject": string, "preconditions": string, "steps": [string],
            "expectedResult": string, "notes": string}.
            testSubject, steps and expectedResult are required.
          required: true
          type: string
        summary:
          description: "One or two sentences describing what was covered, shown in the QA review issue."
          required: false
          type: string
      env:
        KATATAN_TOKEN: ${{ secrets.KATATAN_TOKEN }}
        KATATAN_SPEC_ID: ${{ vars.KATATAN_SPEC_ID }}
        KATATAN_WORKSPACE_ID: ${{ vars.KATATAN_WORKSPACE_ID }}
        KATATAN_QA_ASSIGNEES: ${{ vars.KATATAN_QA_ASSIGNEES }}
        PR_NUMBER: ${{ github.event.pull_request.number || github.event.inputs.pr_number }}
        GH_TOKEN: ${{ github.token }}
      steps:
        - name: Validate test cases
          run: |
            set -euo pipefail
            if [ -z "${KATATAN_TOKEN:-}" ] || [ -z "${KATATAN_SPEC_ID:-}" ]; then
              echo "::error::Set the KATATAN_TOKEN secret and the KATATAN_SPEC_ID variable."
              exit 1
            fi
            item=$(jq -c '[.items[] | select(.type == "add_katatan_test_cases")] | last' "$GH_AW_AGENT_OUTPUT")
            if [ "$item" = "null" ]; then
              echo "The agent did not request any test cases."
              exit 0
            fi
            echo "$item" | jq -r '.summary // ""' > /tmp/summary.txt
            # cases_json must be a JSON array that matches KATATAN's batch-create limits.
            echo "$item" | jq -r '.cases_json' | jq '
              if type != "array" then error("cases_json must be a JSON array") else . end
              | if length < 1 or length > 30 then error("expected 1-30 test cases, got \(length)") else . end
              | map(
                  if (.testSubject | type) != "string" or (.testSubject | length) == 0 or (.testSubject | length) > 2000
                    then error("invalid testSubject") else . end
                  | if (.expectedResult | type) != "string" or (.expectedResult | length) == 0 or (.expectedResult | length) > 2000
                    then error("invalid expectedResult in \(.testSubject)") else . end
                  | if (.steps | type) != "array" or (.steps | length) == 0 or (.steps | length) > 100
                      or any(.steps[]; type != "string" or length > 1000)
                    then error("invalid steps in \(.testSubject)") else . end
                  | {testSubject, steps, expectedResult}
                    + (if (.preconditions | type) == "string" then {preconditions} else {} end)
                    + (if (.notes | type) == "string" then {notes} else {} end)
                )' > /tmp/cases.tmp.json
            mv /tmp/cases.tmp.json /tmp/cases.json
            echo "Validated $(jq length /tmp/cases.json) test cases."
        - name: Append test cases to KATATAN
          run: |
            set -euo pipefail
            [ -f /tmp/cases.json ] || exit 0
            npx -y @katatan/cli@0.1.0 --json test-case batch-create \
              --spec-id "$KATATAN_SPEC_ID" \
              --file /tmp/cases.json \
              --author "GitHub Agentic Workflows (PR #$PR_NUMBER)" > /tmp/created.json
            echo "Created $(jq '.created' /tmp/created.json) test cases."
        - name: Build the review report
          run: |
            set -euo pipefail
            [ -f /tmp/created.json ] || exit 0
            # Link to the spec in KATATAN when the workspace ID is known (spec ID is "<projectId>.<specId>").
            spec_ref="\`$KATATAN_SPEC_ID\`"
            if [ -n "${KATATAN_WORKSPACE_ID:-}" ]; then
              spec_ref="[\`$KATATAN_SPEC_ID\`](https://app.katatan.com/workspace/$KATATAN_WORKSPACE_ID/project/${KATATAN_SPEC_ID%%.*}/spec/${KATATAN_SPEC_ID#*.})"
            fi
            {
              echo "The following test cases were generated from #$PR_NUMBER and appended to the KATATAN test specification $spec_ref."
              echo "Please review each case in KATATAN, fix or delete it if needed, and tick it off here."
              echo
              if [ -s /tmp/summary.txt ]; then echo "> $(tr '\n' ' ' < /tmp/summary.txt)"; echo; fi
              echo "### Review checklist"
              echo
              jq -r '.testCases[] | "- [ ] **case-\(.caseNumber)**: \(.testSubject | gsub("<"; "&lt;") | gsub("\n"; " "))"' /tmp/created.json
              echo
              echo "### Test case details"
              echo
              # Escape "<" so generated text cannot break the <details> blocks; keep line breaks in long fields.
              jq -r '
                def esc: gsub("<"; "&lt;");
                def block: esc | gsub("\n"; "  \n");
                .testCases[]
                | "<details>\n<summary><b>case-\(.caseNumber)</b>: \(.testSubject | esc | gsub("\n"; " "))</summary>\n\n"
                  + (if (.preconditions // "") != "" then "**Preconditions**\n\n\(.preconditions | block)\n\n" else "" end)
                  + "**Steps**\n\n"
                  + ([.steps | to_entries[] | "\(.key + 1). \(.value | esc | gsub("\n"; " "))"] | join("\n"))
                  + "\n\n**Expected result**\n\n\(.expectedResult | block)\n\n"
                  + (if (.notes // "") != "" then "**Notes**\n\n\(.notes | block)\n\n" else "" end)
                  + "</details>\n"' /tmp/created.json
              echo
              echo "<sub>Generated by GitHub Agentic Workflows. Run: $GITHUB_SERVER_URL/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID</sub>"
            } > /tmp/report.md
            cat /tmp/report.md >> "$GITHUB_STEP_SUMMARY"
        - name: Open a review issue for QA
          run: |
            set -euo pipefail
            [ -f /tmp/report.md ] || exit 0
            count=$(jq '.created' /tmp/created.json)
            pr_title=$(gh pr view "$PR_NUMBER" --repo "$GITHUB_REPOSITORY" --json title --jq .title)
            title="Review $count KATATAN test case(s) from #$PR_NUMBER: $pr_title"
            gh label create katatan-review --repo "$GITHUB_REPOSITORY" --color FBCA04 \
              --description "AI-generated KATATAN test cases waiting for QA review" --force
            args=(--repo "$GITHUB_REPOSITORY" --title "$title" --body-file /tmp/report.md --label katatan-review)
            # KATATAN_QA_ASSIGNEES is an optional comma-separated list of GitHub users.
            if [ -n "${KATATAN_QA_ASSIGNEES:-}" ] && issue_url=$(gh issue create "${args[@]}" --assignee "$KATATAN_QA_ASSIGNEES"); then
              :
            else
              [ -n "${KATATAN_QA_ASSIGNEES:-}" ] && echo "::warning::Could not assign $KATATAN_QA_ASSIGNEES; creating the issue unassigned."
              issue_url=$(gh issue create "${args[@]}")
            fi
            echo "$issue_url" > /tmp/issue-url.txt
            echo "Review issue: $issue_url"
        - name: Comment on the PR
          run: |
            set -euo pipefail
            [ -f /tmp/issue-url.txt ] || exit 0
            {
              echo "### 🧪 KATATAN test cases added"
              echo
              echo "$(jq '.created' /tmp/created.json) test case(s) were appended to the KATATAN test specification."
              echo "QA review is tracked in $(cat /tmp/issue-url.txt)."
              echo
              echo "| Case | Test subject |"
              echo "| --- | --- |"
              jq -r '.testCases[] | "| case-\(.caseNumber) | \(.testSubject | gsub("\\|"; "\\|") | gsub("\n"; " ")) |"' /tmp/created.json
            } > /tmp/comment.md
            gh pr comment "$PR_NUMBER" --repo "$GITHUB_REPOSITORY" --body-file /tmp/comment.md

# Give each run its own concurrency slot so several merges in a row are all processed.
concurrency:
  job-discriminator: ${{ github.run_id }}

timeout-minutes: 15
---

# Write KATATAN test cases for a merged pull request

You are a QA engineer. Pull request #${{ github.event.pull_request.number || github.event.inputs.pr_number }}
in `${{ github.repository }}` has been merged. Your job is to design manual test cases that verify
the behavior this PR introduced or changed. The cases are added to a test specification in
[KATATAN](https://katatan.com), a test management service.

## 1. Understand the change

Use the GitHub tools to read the pull request:

- title and description (the author's intent, linked issues, acceptance criteria)
- the list of changed files and the diff

Focus on behavior a user or API client can observe. Ignore pure refactors, formatting, and
comment-only changes.

## 2. Check what is already covered

Existing test cases in the target specification are in `/tmp/gh-aw/katatan/existing-cases.json`
(an array of `{caseNumber, testSubject, expectedResult}`). Read it with `cat` or `jq`.
Do not add a case that checks the same thing as an existing one.

## 3. Design the test cases

- One case checks one thing. Keep `testSubject` short and specific
  (for example, "Login fails with an expired password").
- Cover the normal path, error handling, and boundary values that the change introduced.
- `preconditions`: the state required before the first step (data, settings, user role).
- `steps`: concrete actions a tester can follow, one action per item (do not number them).
- `expectedResult`: an observable outcome (what is shown, returned, or stored), not "works correctly".
- `notes`: `Source: PR #<number>` followed by the main files involved.
- Write every field in English.
- Add between 1 and 30 cases. Prefer fewer, meaningful cases over many shallow ones.

## 4. Submit

- If the PR has behavior worth testing, call `add_katatan_test_cases` **exactly once**:
  - `cases_json`: all cases as a single JSON array string
  - `summary`: one or two sentences on what the cases cover
- If the PR has nothing to test (documentation, CI configuration, dependency bumps with no
  behavior change) or every relevant case already exists, call `noop` with a short reason instead.
