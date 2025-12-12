---

description: 仅用 Git 分析改动并自动生成 conventional commit 信息（可选 emoji）；必要时建议拆分提交，默认运行本地 Git 钩子（可 --no-verify 跳过）
agent: build
confirm: true
argument-hint: "[--no-verify] [--all] [--amend] [--signoff] [--emoji] [--scope <scope>] [--type <type>]"

# allowed-tools: Read(**), Exec(git status, git diff, git add, git restore --staged, git commit, git rev-parse, git config), Write(.git/COMMIT_EDITMSG)

---

Analyze the current git repository and produce Conventional Commits-style commit message(s) and an actionable plan to apply them.

Context (these shell commands will be executed and their output injected into the prompt):

* Current repo check: !`git rev-parse --is-inside-work-tree 2>/dev/null || echo "NOT_A_REPO"`
* Current branch & HEAD: !`git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "UNKNOWN"`
* Repo state (porcelain): !`git status --porcelain --untracked-files=normal`
* Staged file list: !`git diff --staged --name-only`
* Unstaged file list: !`git diff --name-only`
* Staged diff summary (stat): !`git diff --staged --stat`
* Recent commit subjects (for language/style signals): !`git log -n 50 --pretty=%s`

User-provided flags (these are passed to the command via $ARGUMENTS):
$ARGUMENTS

TASK — produce the following outputs **in this exact structure**:

1. QUICK VERDICT (1 line)

   * One of:

     * "single-commit" (safe to commit all changes as one),
     * "suggest-split" (recommend splitting into N commits — follow with groups),
     * "abort-due-to-state" (e.g., conflicts, not a git repo).

2. IF abort-due-to-state: short human-friendly reason and explicit steps to resolve.

3. If single-commit OR suggest-split: produce commit proposal(s).

   For each commit proposal, include:

   * Commit label: `Commit #k`
   * Type & optional scope (e.g., `feat(auth):`), follow Conventional Commits.
   * Subject (<=72 chars, imperative).
   * Body: bullet points — motivation, what changed, files/dirs affected, potential risks, tests/verification steps.
   * BREAKING CHANGE section if applicable.
   * If `--emoji` appears in $ARGUMENTS, also provide the emoji prefix line (e.g., "emoji: ✨").
   * Suggested `git` commands to create it (pathspecs + git add + git commit flags). Example:

     ```
     git add path1 path2
     git commit -m "feat(scope): subject" -m "body line 1\nbody line 2" [--signoff] [--no-verify-if-specified]
     ```

   If you recommend splitting, group files into coherent pathspecs and label each group with the likely commit type.

4. If `--all` is present in $ARGUMENTS and staged area is empty, include the explicit command we will run:

   * `git add -A` (and mention consequences).

5. If `--amend` is present, produce the amended commit message only (mention the amended commit hash from git rev-parse if available).

6. SAFETY / CHECKS

   * If detecting rebase/merge/conflict/detached HEAD, set QUICK VERDICT to "abort-due-to-state" and list commands to fix.
   * If very large diffs (> 300 LOC changed or > 5 top-level directories), prefer "suggest-split".

7. OUTPUT FORMATS (for automation)

   * A plain-text section delimited by `===COMMIT-MESSAGES===` that lists the exact commit messages to be written to `.git/COMMIT_EDITMSG` (one message per commit, separated by `---COMMIT---`).
   * A plain-text section delimited by `===ACTIONABLE-CMDS===` containing the exact shell commands to run if the user agrees (one command per line).

IMPORTANT:

* Use the injected shell outputs above as evidence. Quote filenames/pathspecs exactly as shown.
* Keep the subject lines short and imperative. Body lines should be clear and actionable.
* If repository language or prior commits are mostly non-English, adapt language accordingly (use recent commit subjects as a hint).
* Always avoid making filesystem changes in this prompt — only produce suggested `git` commands. (OpenCode will run them if the user opts in.)

## Language Detection Rules

Before generating any commit messages, detect the preferred commit language:

- Analyze the recent commit messages shown above.
- Determine whether English or Chinese is more frequently used.
- Use the majority language when generating commit message subjects and bodies.
- Keep the entire output for each commit in a single consistent language.
- If no clear majority exists, default to English.

Examples of accepted final structure (abbreviated):

QUICK VERDICT: suggest-split

Commit #1
type(scope): subject
body...
files: src/auth/*, tests/auth/*
commands:
git add src/auth/* tests/auth/*
git commit -m "feat(auth): add login flow" -m "..."

Commit #2
...

===COMMIT-MESSAGES===
feat(auth): add login flow

* Implement login endpoint and client flow
* Add unit tests

---

fix(ci): adjust lint config

* ...

===ACTIONABLE-CMDS===
git add src/auth/* tests/auth/*
git commit -m "feat(auth): add login flow" -m "Implement login endpoint..."
git add .github/workflows/ci.yml
git commit -m "fix(ci): adjust lint config"
