# Security policy

## What is in scope

This repository publishes agent skills: Markdown instructions plus some
templates and helper scripts that agents copy into other projects. A security
issue here is anything that makes a project less safe once a skill has been
followed, for example:

- a template that fails open, leaks a credential, or can be bypassed (such as
  the middleware in `vercel-deployment-password-gate`);
- a helper script that runs something the skill does not say it runs;
- skill instructions that lead an agent to expose secrets, weaken access
  controls, or run destructive commands without saying so.

Bugs in third-party tools a skill mentions (the `skills` CLI, Vercel, GitHub
and so on) belong with those projects.

## Supported versions

Only the latest commit on `main` is supported. Skills are installed from the
repository, not from releases, so a fix reaches users when they run
`npx skills update`.

That refreshes the installed skill only. Some skills tell the agent to **copy**
template code into your project (for example `templates/deploy-gate*.ts`), and
those copies are not updated. When an advisory names a template, review your
copy and apply the fix there too.

## Reporting a vulnerability

Please **do not open a public issue or pull request** for a security problem.

Report it privately through GitHub instead:
[Report a vulnerability](https://github.com/stealth-factory/skills/security/advisories/new).

Include the skill name, the file and line if you have them, and the steps that
show the problem. We aim to acknowledge reports within 5 working days and will
let you know whether we accept the issue and when a fix lands.

## Credentials

Skills in this repository never contain real credentials. Placeholder values
and environment-variable **names** appear in templates on purpose. If you find
something that looks like a real secret, report it as above.
