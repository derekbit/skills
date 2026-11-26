---
name: debate-review
description: >
  Two-model debate review of a Longhorn pull request (longhorn-manager, longhorn-engine,
  longhorn-instance-manager, longhorn-spdk-engine, go-spdk-helper, longhorn-share-manager,
  backing-image-manager, go-common-libs, types, backupstore, longhorn chart) or of a local branch.
  A main reviewer finds issues, a second model from another family tries to refute them, the main
  reviewer makes the final call. Produces a local Markdown report only; it never comments on,
  reviews, or otherwise writes to the PR. Use for any review request on a Longhorn repository.
compatibility: Requires git, gh (authenticated), and GitHub Copilot CLI (copilot).
metadata:
  author: longhorn
  version: 0.2.0
---

# debate-review

Three reviewer passes, each a separate `copilot -p` process:

1. **Main** finds issues against the Longhorn guide, the component profile, the repo's `AGENTS.md`,
   and the linked `longhorn/longhorn` issue.
2. **Debate**, on a model from another family, confirms, refutes, or downgrades each finding and
   adds blocking gaps the main pass missed.
3. **Final**, on the main model in a new process, decides each finding: `agreed`, `contested`, or
   `withdrawn`.

You orchestrate. You do not review the diff yourself and never change the code under review.

**Local report only.** This skill never writes to GitHub: no review, no comment, no approval, no
label, no reaction, on the PR or its issue. Use `gh` only for reads (`gh api` without `--method`,
which is a GET). Never run `gh api --method POST|PATCH|PUT|DELETE`, `gh pr review`, `gh pr comment`,
`gh issue comment`, or `gh pr edit`. If the user asks to post the result, say this skill only
produces a local report and point them to the report file; they can copy from it themselves.

`<skill-dir>` is the directory of this file. It holds `references/longhorn-review-guide.md`,
`references/components.md`, `references/format.md`, and the three briefs in `assets/prompts/`.

## 1. Target and models

**Target.** A PR URL, `<owner>/<repo>#<n>`, or `<n>` against the cwd's `origin` is **PR mode**.
"My branch", "before I open a PR", or no PR is **local mode**, against the base the user named, else
`origin/HEAD` or `origin/master`. `<component>` is the repo name if `references/components.md` has a
`## <repo>` section, else `generic`.

**Models.** The user may write `main=<model> debate=<model> final=<model>`, or the same in plain
words. Otherwise: main is the strongest available Claude model, debate is the strongest available
model of another family (GPT, then Gemini), final is the main model. Check every name against the
`--model` list in `copilot help`. If one is unavailable, ask; do not substitute. If main and debate
are the same family, warn in one line. Always pass `--model` per pass; never rely on `/model` or
`COPILOT_MODEL`, which would give every pass the same model. Tell the user the three models before
starting.

## 2. Workspace

Everything lives under `~/.cache/debate-review/<owner>__<repo>/`, never in the user's own clone.

**PR mode.** Each PR has one checkout that is moved to the PR's **latest head** on every run and is
**kept after the review**, so a human reviewer can read the code next to the report. Each head gets
its own run directory, so earlier reports stay too.

```bash
gh api repos/<owner>/<repo>/pulls/<n> --jq '{title, body, url: .html_url, author: .user.login, head: .head.sha, base_ref: .base.ref}'
ROOT=~/.cache/debate-review/<owner>__<repo>
CLONE=$ROOT/clone
WT=$ROOT/pr-<n>/checkout                  # the PR at its latest head, kept for the human reviewer
RUN=$ROOT/pr-<n>/<head7>                  # this head's briefs, replies, JSON, report.md
mkdir -p $RUN
[ -d $CLONE ] || git clone --quiet --filter=blob:none --no-checkout https://github.com/<owner>/<repo>.git $CLONE
git -C $CLONE fetch --quiet --force origin +refs/pull/<n>/head:refs/dr/pr-<n> +refs/heads/<base_ref>:refs/dr/base-<base_ref>
git -C $CLONE rev-parse refs/dr/pr-<n>    # must equal <head>; otherwise the PR moved: re-read it
if [ -e $WT/.git ]; then git -C $WT checkout --quiet --force --detach <head>
else git -C $CLONE worktree add --quiet --detach $WT <head>; fi
BASE=$(git -C $CLONE merge-base refs/dr/base-<base_ref> <head>)
DIFF="git diff $BASE...<head>"
rm -rf $WT/.github/hooks $WT/.github/agents $WT/.github/skills $WT/.claude $WT/.mcp.json
```

The `rm` removes project hooks, agents, skills, and MCP config a PR could use to run code in the
reviewer processes; instruction files and the built-in GitHub MCP server are switched off by flags in
step 4. Step 6 restores them, so the kept checkout matches the PR exactly.

If `report.md` already exists in `$RUN`, this head was reviewed before: show that report and ask
before running again.

**Local mode.** Review the user's tree in place. Their own code is trusted and the reviewers cannot
write:

```bash
WT=$(git rev-parse --show-toplevel)
BASE=$(git -C $WT merge-base <base-ref> HEAD)
DIFF="git diff $BASE"          # committed + staged + unstaged; untracked files need `git add -N` first
RUN=~/.cache/debate-review/<owner>__<repo>/local-<branch>/<head7>
mkdir -p $RUN
```

## 3. Context in `$RUN`

```bash
cd $WT
EXCL=(':(exclude)vendor/**' ':(exclude)**/go.sum' ':(exclude)go.sum' ':(exclude)**/zz_generated*.go'
      ':(exclude)k8s/pkg/client/**' ':(exclude)k8s/crds.yaml' ':(exclude)client/generated_*.go'
      ':(exclude)**/*.pb.go' ':(exclude)**/*_pb2*.py' ':(exclude)generated-py/**' ':(exclude)pkg/generated/**')
$DIFF -- . "${EXCL[@]}" > $RUN/diff.patch
$DIFF --name-status > $RUN/files.txt
git show $BASE:AGENTS.md > $RUN/AGENTS.base.md 2>/dev/null || rm -f $RUN/AGENTS.base.md
```

If `diff.patch` is empty, only vendor or generated files changed: say so and stop.

Write `$RUN/pr.md`: title, URL, author, body, and `git log --format='%h %s%n%b' $BASE..HEAD`.

Write `$RUN/issues.md` from up to three issues referenced in the title, body, or commits. Longhorn
issues live in `longhorn/longhorn`: `Issue #N`, `longhorn/longhorn#N`, `longhorn/longhorn-N`,
`longhorn/longhorn N`, and `github.com/longhorn/longhorn/issues/N` all mean `longhorn/longhorn#N`.
`longhorn/<component>#N` and other bare `#N` are PRs, not issues.

```bash
gh api repos/longhorn/longhorn/issues/<N> --jq '{title, url: .html_url, state, labels: [.labels[].name], body}'
gh api "repos/longhorn/longhorn/issues/<N>/comments?per_page=100" --jq '[.[] | select(.user.login | endswith("[bot]") | not) | {user: .user.login, body}] | .[-8:]'
```

If none is linked, write `Missing issue reference.`; if `gh` fails, write `Issue context unavailable:
longhorn/longhorn#<N>.` If the PR or issue links a LEP (`longhorn/longhorn/.../enhancements/*.md`)
or depends on another Longhorn PR, save it too (`gh api ... -H 'Accept: application/vnd.github.raw'`
or `-H 'Accept: application/vnd.github.diff'`) as `$RUN/lep.md` or `$RUN/related-<repo>-<n>.patch`.

## 4. Run the three passes

For each pass, copy the brief to `$RUN/brief-<pass>.md` and replace `{{REPO}}`, `{{COMPONENT}}`,
`{{HEAD}}`, `{{DIFF}}` (the `$DIFF` command), `{{RUN}}`, and `{{SKILL_DIR}}`. Then:

```bash
cd $WT && copilot -s --no-ask-user --no-custom-instructions --disable-builtin-mcps --model <model> \
  --add-dir=$RUN --add-dir=<skill-dir> \
  --allow-tool='read, shell(git diff), shell(git log), shell(git show), shell(git grep), shell(git blame), shell(grep), shell(rg)' \
  --deny-tool='write, url, shell(gh)' \
  -p "Read $RUN/brief-<pass>.md and follow it exactly. Do not modify any file." > $RUN/<pass>.reply.md
```

Reviewers get no GitHub access at all: `gh` and URLs are denied and the GitHub MCP server is off.
Never add `--allow-all-tools`, `--allow-all`, or `--yolo`; if `copilot` refuses to run without one,
stop and tell the user. Run each pass in the background and wait; a pass takes minutes. Extract the last ```json block of the reply
into `$RUN/<pass>.json`, check it, then run the next pass:

1. **main**: brief `review-main.md`, main model → `main.json`.
2. **debate**: brief `review-debate.md`, debate model → `debate.json`. Run it even when main found
   nothing: looking for what main missed is half its job.
3. **final**: brief `review-rebuttal.md`, final model → `final.json`.

Check each document against `references/format.md`:

- valid JSON with the right `schema`; every finding has `id`, `file`, `line_start <= line_end`,
  `severity` (`critical|major|minor`), `claim`; ids unique;
- main: drop findings with `confidence` below 0.5;
- debate: one verdict per `F*` (a missing one is `confirm`); drop `minor` new findings;
- final: every `F*` and `D*` comes back with a `status`, nothing new, no `D*` is `contested`;
- each `file` exists and the code quoted in `evidence` is really there.

If a reply fails a check, rerun that pass once with one added line naming the problem. If it fails
again, stop and tell the user which pass failed and where its reply is.

## 5. Write the report

Render `final.json` into `$RUN/report.md` as `references/format.md` section 4 shows: `agreed` and
`contested` findings in severity order, then the withdrawn ones in an appendix with their
`debate_note`. Mark a finding whose `file` is not in `diff.patch` as "outside the diff". For a
`security` finding that is a real vulnerability, add a line that it should go to
https://github.com/longhorn/longhorn/security, not a public PR comment.

Show the report to the user in the reply (the summary and the findings), and give the path of
`$RUN/report.md`.

## 6. Finish

In PR mode, put back what step 2 removed and leave the checkout in place:

```bash
git -C $WT checkout --quiet -- .          # restore the removed hook/agent/skill/MCP files
```

Do not remove the checkout or the clone. Tell the user the counts by severity, what was withdrawn or
contested, the report path (`$RUN/report.md`), and the checkout path (`$WT`, at `<head7>`) to open
next to it. Do not act on the findings unless asked.
