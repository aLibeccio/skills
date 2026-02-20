---
name: commit
description: Automate validating, staging, committing, and pushing code changes with Golang pre-commit checks, strict conventional commit message generation, and OTP-aware push handling. Use when the user asks to commit, save changes, commit and push, run the commit skill, 提交代码, or 保存改动.
---

# Commit

Execute this workflow in strict order. If any command in Step 1 or Step 2 returns a non-zero exit code, STOP immediately and report the error.

## Step 1: Detect project type and run pre-commit checks (Golang only)

1. Detect whether the current directory is a Golang project:
   - Treat as Golang project if `go.mod` exists, or any `*.go` file exists.
2. If it is a Golang project:
   - Run `go test ./...`.
   - If tests fail, STOP and report that tests must pass before committing.
   - Run `golangci-lint run`.
   - If lint fails, STOP and report that lint issues must be fixed before committing.
3. If it is not a Golang project, skip Step 1 checks and continue.

## Step 2: Check and stage changes

1. Run `git diff --staged`.
2. If staged diff is empty:
   - Run `git diff`.
   - If unstaged diff is not empty, run `git add -A`.
   - If unstaged diff is also empty, STOP and report that the working tree is clean.

## Step 3: Generate commit message

1. Analyze staged changes with `git diff --staged`.
2. Summarize the staged changes concisely.
3. Generate a strict conventional commit message in the format `type(scope): description`.
   - Valid `type` values: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`.
   - Keep `description` under 50 characters.
   - Keep `scope` short and concrete. If no clear scope exists, use `repo`.

## Step 4: Commit

1. Run `git commit -m "<message>"` using the message generated in Step 3.

## Step 5: Push to remote

1. Get current branch by running `git branch --show-current`.
2. Check upstream state:
   - Run `git rev-parse --abbrev-ref --symbolic-full-name @{u}`.
   - If this command fails, treat branch as not yet pushed and continue.
   - If it succeeds, compare local and upstream to determine whether push is needed.
3. Attempt push with `git push origin <current_branch>`.
4. Monitor push output:
   - If push fails or hangs due to OTP/2FA authentication, STOP and instruct the user:
     - `Push failed or requires OTP. Please manually run git push origin <current_branch> in your terminal to complete the authentication.`

## Step 6: Final output

1. Show the successful commit hash and commit message.
2. If push succeeds, confirm the changes were pushed to the remote repository.
