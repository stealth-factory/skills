---
name: skill-publishing
description: How to author and publish agent skills for the skills.sh / `npx skills` ecosystem — the SKILL.md format, why the frontmatter description is the trigger, multi-skill repo layout, the `skills` CLI (init/add/list/find/update/remove with flags), publishing to GitHub with SEO metadata, and public vs private repos (it git-clones, so private works with auth). Use when asked to "create/write/author a skill", "publish a skill", "set up a skills repo", "make a skill installable/discoverable", "use npx skills", or "can I use a private skills repo".
metadata:
  author: stealth-factory
  co-author: wiiiimm
  version: "1.2.2"
---

# skill-publishing

How to write a good agent skill and publish it so others (and your other
machines) can install it with `npx skills`.

## What a skill is

Reusable, **on-demand context** for an AI coding agent. One skill = one folder
with a `SKILL.md`:

```markdown
---
name: my-skill            # kebab-case, matches the folder
description: <one line>    # the TRIGGER — see below; most important field
metadata:
  author: you
  version: "1.0.0"        # semver; bump on meaningful change
---

# Title

<the body the agent loads only when this skill fires>
```

**Context economy:** only the `description` of every installed skill stays in the
agent's context. The body loads **only when the description matches** the task.
That's the whole design — so the description does the heavy lifting.

## The `description` is the trigger (get this right)

Write it as **concrete triggers**: what it does, then "Use when …" listing
symptoms, situations, and phrases a user might actually say.

- Good: `…Use when an iPhone page shows black bars, content is cut off at the bar
  edge, a cropped shadow, or the page jumps after the keyboard closes.`
- Bad: `Helps with iOS Safari styling.`

If the agent won't *match* it to the right moment, the skill is dead weight.

## The body

- Write for the **agent**, not end users: actionable facts, steps, rules.
- **Scannable**: headings, short sections, tables, code blocks.
- **Progressive disclosure**: keep `SKILL.md` focused; put heavy reference docs or
  scripts in sibling files and link them so they load only when needed.
- Prefer **facts + case-dependent guidance** over one rigid recipe, so it stays
  useful across situations.
- Note **provenance** for non-obvious claims (how it was verified) to earn trust.
- One concern per skill — split rather than overload.

## Bundling files in a skill (progressive disclosure)

A skill is a **directory**, not just one file. `SKILL.md` is the entry point; you
can ship reference docs, scripts, templates, and assets next to it:

```text
skills/my-skill/
  SKILL.md                 # entry: frontmatter + concise body
  reference/
    api.md                 # deep detail / long tables — read on demand
  scripts/
    init.sh                # a runnable helper the skill invokes
  templates/
    config.example.json    # boilerplate the skill copies
```

Only `SKILL.md`'s body loads when the skill fires; **everything else loads,
reads, or runs only when `SKILL.md` tells the agent to**, referenced by relative
path. Keep `SKILL.md` short and high-signal; push heavy material into siblings:

```markdown
For the full option list, read `reference/api.md`.
Scaffold with `scripts/init.sh <name>`.
```

The CLI installs the **whole directory** (symlink or `--copy`), so relative links
keep working. Split only when a skill gets large or needs runnable helpers — a
small skill is perfectly fine as a single `SKILL.md`.

## Repo layout (bundling many skills)

```text
skills/<skill-name>/SKILL.md            # flat (preferred)
skills/<category>/<skill-name>/SKILL.md # categorised (only if you have enough)
SKILL.md                                # a single skill at repo root also works
```

- **No manifest required** — the CLI auto-discovers those paths.
- **Don't put a root `SKILL.md` alongside a `skills/` dir**: a root `SKILL.md`
  **short-circuits discovery** — the CLI treats the repo as one skill and never
  scans subdirectories unless consumers pass `--full-depth`. Use one layout or
  the other.
- `README.md` = the **human** index (a table of skills) + SEO. The machine index
  is just each skill's `description`; no separate index file.
- `AGENTS.md` (+ `CLAUDE.md` symlink) = authoring conventions for the repo.
- A `.claude-plugin/marketplace.json` manifest is **optional**, only for
  plugin-marketplace features or non-standard skill paths.

## The `skills` CLI (vercel-labs/skills)

```bash
npx skills init skills/<name>        # scaffold a new skill folder + SKILL.md

npx skills add owner/repo            # interactive: pick skills + agents
npx skills add owner/repo --list     # browse the repo, install nothing
npx skills add owner/repo --skill a b   # specific skills (space-separated, NOT a,b)
npx skills add owner/repo --all -g   # all skills, all agent targets, global (see caveats below)
npx skills add owner/repo --copy     # copy instead of symlink into agent dirs

npx skills list                      # installed skills (-g for global)
npx skills find <keyword>            # search
npx skills update [name...]          # update
npx skills remove [name...]          # remove
npx skills use owner/repo@skill      # one-off prompt without installing
```

### Multi-value flags are space-separated, never comma-separated

`-s/--skill` and `-a/--agent` are **variadic**: separate values with spaces, or
repeat the flag. A comma-separated list is not parsed as a list — both flags
reject it and **exit 1**:

```bash
npx skills add owner/repo -s a b -a claude-code codex          # works
npx skills add owner/repo -s a -s b -a claude-code -a codex    # works (repeated)

npx skills add owner/repo -a claude-code,codex -y   # ■ Invalid agents: claude-code,codex   → exit 1
npx skills add owner/repo -s a,b -y                 # ■ No matching skills found for: a,b   → exit 1
```

Both fail cleanly: nothing is installed and no lock file is written. The `-a`
message is the confusing one — it prints a valid-agent list **containing the very
names you just passed**, so it reads like a bug in the tool rather than a parsing
problem.
The examples use `-y` so a multi-skill source cannot stop at the skill-selection
prompt before agent validation; cancelling that prompt is a different exit path.

**Agent names are not the informal ones.** It's `claude-code`, not `claude`
(`codex`, `cursor`, `github-copilot`, `gemini-cli` are themselves valid). `*`
selects all. The full accepted list is printed by any invalid `-a` value.

> **Don't read this CLI's status through an unguarded pipe.** A shell pipeline
> returns the *last* stage's status unless `pipefail` is set, so the common
> `npx skills add … | grep …` idiom reports **0** while `npx` itself exited **1**:
> measured `npx=1, grep=0, pipeline $?=0`, and `set -o pipefail` correctly yields 1.
> That is how an earlier version of this section came to claim a silent fail-open
> that does not exist — the trap was in the measurement, not the tool. Check
> `PIPESTATUS`/`pipefail`, or better, **verify what landed on disk rather than
> trusting any exit status.**

### `--all` writes more than you think

`--all` is shorthand for `--skill '*' --agent '*' -y`. It selects all agent
targets, but the number of directories depends on scope and existing agent
directories; several targets share a directory. In a clean project, v1.5.23
created `.agents/`, `.claude/`, and a **non-hidden top-level `agent/`** for Eve.
The latter holds a second full copy of every skill; it is not a symlink.
Other non-universal project targets can be skipped when their directories do
not already exist. In a repo that extra copy is `git add -A` bait, and
`npx skills remove --all` leaves the empty `agent/skills/` directory behind.
Global `--all -g` has different destinations: it does not create that project
`agent/` directory, and Eve and PromptScript report unsupported global installs
even though the command exits 0. Prefer naming the agents you actually want and
checking their installed files.

### Lock files: project and global differ

| | Project scope | Global scope (`-g`) |
| --- | --- | --- |
| Path | `skills-lock.json` at the project root | `~/.agents/.skill-lock.json` by default; `$XDG_STATE_HOME/skills/.skill-lock.json` when set |
| Version | `1` | `3` |
| Per-skill (source-dependent) | `computedHash`, `skillPath`, `source`, `sourceType`, `sourceUrl` for generic Git sources, … | `skillFolderHash`, `installedAt`, `updatedAt`, `source`, `sourceUrl`, … |

**Inspect lock files before committing or sharing them.** Source metadata can
disclose private repository identifiers and local paths. A generic Git URL with
embedded credentials was persisted verbatim in both scopes' `source` and
`sourceUrl` fields. Use credential-free source URLs with external Git auth, and
keep sensitive lock files out of public history; absence of absolute paths is
not a safety guarantee.

The project lock **records the installed set; it does not guarantee reproducible
contents or agent placement**. In v1.5.23:

- `experimental_install` targets universal `.agents/skills/`, not the original
  selected agents. Changing a Git source's HEAD or a local source after recording
  the lock caused it to install the changed contents and replace `computedHash`;
  the recorded hash was not enforced as an integrity pin.
- With an unchanged `node_modules` source and its recorded lock in a fresh
  checkout, both `experimental_install` and `experimental_sync -y` reported
  "All skills are up to date" and exited 0 while the destination skill was absent.
  `experimental_install --force` populated `.agents/skills/` in that fixture,
  but this does not make the contents immutable or restore other agent targets.
- `experimental_sync` scans `node_modules` for skills; it is not a general
  lock-file restore command.

Before ignoring installed directories, verify recovery from a clean checkout
using your actual sources and required agent destinations. If exact contents
matter, preserve them separately; do not rely on this lock alone.

Other flags: `-g/--global`, `-l/--list`, `-y/--yes` (skip prompts),
`--copy` (copy instead of symlink), `--full-depth` (scan every subdirectory even
when a root `SKILL.md` exists — needed for a repo that has both a root skill and a
`skills/` dir). Telemetry is on by default; set `DISABLE_TELEMETRY=1` (or
`DO_NOT_TRACK=1`) to opt out.

**Check copy layouts after updates.** In v1.5.23, `skills update` can replace
project skills installed with `--copy` with symlinks into
`.agents/skills/`. This was reproduced on Linux for a Git-source Claude Code
install with Claude Code detected; [an upstream report](https://github.com/vercel-labs/skills/issues/1199#issuecomment-5365810628)
also describes it on macOS. Inspect destination types after updates, especially
before committing copied skills. Re-running `skills add` with `--copy` and the
required agents restored real directories in the Linux fixture.

> Provenance: flag forms, project/global `--all`, lock metadata, and the restore
> cases above were checked against the npm package `skills` **v1.5.23** on
> 2026-10-09 using isolated homes and synthetic local, Git, and `node_modules`
> sources. These are version-specific observations; re-check your installed CLI.

## Publishing to GitHub

```bash
git init && git add -A && git commit -m "chore: bootstrap skills repo"
gh repo create owner/skills --public --source=. --push
# SEO: a clear description + topics make it discoverable
gh repo edit owner/skills \
  --description "Agent skills for … — install: npx skills add owner/skills" \
  --add-topic agent-skills --add-topic claude-skills --add-topic claude-code \
  --add-topic skills --add-topic ai-agents
```

Keep the README's skills table and each `description` sharp — those are what
both humans and `npx skills find` read.

## Public vs private

`npx skills add` **`git clone`s** the repo under the hood (defaulting to the
HTTPS URL) — apart from a no-clone GitHub-API fast path reserved for a few
allowlisted first-party owners (e.g. `vercel`), any other repo is cloned. So:

- **Public** → clones with no auth.
- **Private** → works too, **if the machine is authed to clone it**. For HTTPS,
  run `gh auth setup-git` once so git uses your GitHub login non-interactively;
  or pass an SSH URL directly: `npx skills add git@github.com:owner/repo.git`.
- Anyone you share the install command with also needs repo access + auth, and
  public discovery surfaces (skills.sh leaderboard/search) only index **public**
  repos. So: private = great for personal/studio use; public = shareable +
  discoverable.

## Checklist before publishing a skill

- [ ] Folder is `skills/<kebab-name>/` and `name:` matches.
- [ ] `description` reads as triggers ("Use when …"), not marketing.
- [ ] Body is actionable, scannable, one concern.
- [ ] `metadata.version` set (semver).
- [ ] Added to the README skills table.
- [ ] `npx skills add <repo> --list` shows it with the right description.
