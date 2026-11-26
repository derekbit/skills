# Skills

[Agent Skills](https://agentskills.io) for GitHub Copilot. Each skill is a folder of Markdown
instructions under `skills/` that Copilot loads when a request matches it. A skill can be general or
tailored to one project; its `description` says which.

## Available skills

| Skill | Description |
|-------|-------------|
| [`debate-review`](skills/debate-review/) | Two-model review of a Longhorn PR or local branch: a main reviewer finds issues, a model from another family tries to refute them, the main reviewer makes the final call. Writes a local report; never comments on the PR. |

## Requirements

Most skills here use:

| Tool | Why |
|------|-----|
| [GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli) (`copilot`) | runs the skills |
| [`gh`](https://cli.github.com) | reads PRs and issues (`gh auth login`) |
| `git` | |

Each skill lists what it needs in the `compatibility` field of its `SKILL.md`.

## Install

**All skills, as a Copilot CLI plugin** (`plugin.json` at the root, skills under `skills/`):

```bash
/plugin install https://github.com/derekbit/skills
# or, from a terminal
copilot plugin install https://github.com/derekbit/skills
```

Update with `copilot plugin update agent-skills`.

**One skill**, with GitHub CLI 2.90 or later:

```bash
gh skill install derekbit/skills <skill-name>
```

**By hand.** Copilot discovers skills at `~/.copilot/skills/<name>/SKILL.md` (personal) and at
`.github/skills/<name>/SKILL.md` (one repository). Clone once and link the skills you want:

```bash
git clone https://github.com/derekbit/skills.git ~/.copilot/skills/.agent-skills
ln -sfn ~/.copilot/skills/.agent-skills/skills/<skill-name> ~/.copilot/skills/<skill-name>
```

`git -C ~/.copilot/skills/.agent-skills pull` updates every linked skill. Check that a skill
loaded with `/skills info <skill-name>` in Copilot CLI, or `/skills` in the VS Code chat input.

## Using a skill

Invoke it by name, for example `/debate-review <PR URL>`, or describe the task and let Copilot pick
the skill. Each skill's `SKILL.md` documents its inputs and options.

### debate-review

Tailored to Longhorn (longhorn-manager and the other Longhorn components).

```text
/debate-review https://github.com/longhorn/longhorn-manager/pull/1234
/debate-review longhorn/longhorn-instance-manager#567 main=<model> debate=<model>
/debate-review my current branch against master
```

- **Local report only.** The skill only reads from GitHub; it never posts a review, a comment, or
  anything else to the PR. Reviewer processes have no GitHub access at all. You get a summary in the
  chat and the full report on disk.
- **A checkout to read next to the report.** Each PR keeps one checkout, moved to the PR's latest
  head on every run and left in place afterwards. Reports are kept per head:

  ```
  ~/.cache/debate-review/<owner>__<repo>/
  ├── clone/                  # blobless cache clone (shared by all PRs of the repo)
  └── pr-<n>/
      ├── checkout/           # the PR at its latest reviewed head: open this in your editor
      ├── <head7>/report.md   # one report per reviewed head
      └── <older-head7>/report.md
  ```

  Remove a PR's checkout when you are done:

  ```bash
  R=~/.cache/debate-review/<owner>__<repo>
  git -C $R/clone worktree remove --force $R/pr-<n>/checkout
  ```
- **Report layout** follows longhorn-manager's `AGENTS.md`: Context Check, Issue Alignment, Key
  Findings (P0 critical, P1 major, P2 minor), Assessment, Testing Status, plus an appendix of
  findings withdrawn during the debate.
- **Models.** Each pass is its own `copilot -p --model <model>` process. Set them with
  `main=`, `debate=`, `final=`, or in plain words. By default main is the strongest Claude model,
  debate the strongest model of another family (GPT, then Gemini), final the main model. Names must
  be ones `copilot help` lists for `--model`.
- **What is reviewed.** PR mode reviews the latest PR head in the checkout above, never your own
  clone. Local mode reviews your tree in place, read-only (untracked files only after `git add -N`). `vendor/`,
  generated code, and `go.sum` are excluded. The repo's `AGENTS.md` is read from the base commit.
- Component profiles cover longhorn-manager, longhorn-engine, longhorn-instance-manager,
  longhorn-spdk-engine, go-spdk-helper, longhorn-share-manager, backing-image-manager,
  go-common-libs, types, backupstore, longhorn (chart, LEPs), and longhorn-ui.

## Layout

```
plugin.json                    # Copilot CLI plugin manifest (covers every skill)
skills/
└── <skill-name>/
    ├── SKILL.md               # required: frontmatter (name, description) + instructions
    ├── references/            # optional: material the skill reads on demand
    └── assets/                # optional: templates, prompts
```

## Adding a skill

1. Create `skills/<skill-name>/SKILL.md`. The frontmatter `name` must equal the directory name
   (lowercase, hyphens). The `description` says what the skill does and when to use it; Copilot
   matches requests against it, so name the projects, repos, or tasks it applies to.
2. Keep `SKILL.md` to the steps, and move long reference material into `references/`.
3. Prefer plain Markdown instructions that drive `git`, `gh`, and `copilot`. A skill that writes to
   GitHub (comments, labels, pushes) must say so in its `description` and ask before each write.
4. Add a row to the "Available skills" table above.
