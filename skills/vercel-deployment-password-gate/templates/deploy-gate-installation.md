# Deployment password gate installation record

Fill in every field; write `none` when not applicable. Keep this record free of
credential values (including password hashes), cookies, and bypass URLs.

- Skill version:
- Mode: A (integrated into existing middleware) / B (standalone middleware)
- Framework, app directory, and gate/helper paths:
- Existing host middleware/proxy path and responsibilities to preserve:
- Gate-only files added (including scripts, tests, and assets):
- Shared files modified and gate-specific changes (including matcher changes):
- Build command before / after (include Vercel dashboard overrides):
- Dependencies added and package manager / lockfile:
- Vercel project/team, environments, and branch-specific variable overrides:
- Environment variable names and scopes added (NO values):
- CI/monitoring secret names, consumers, headers/query parameters added:
- Platform settings changed, previous state, and reason (NO secrets):
- Gate-only installation/follow-up commit SHAs, oldest first:
  - Pending: fill in after committing, in a separate documentation commit.
- Installation PR and final squash/rebased SHA(s):
- Separate prerequisite commit SHAs to KEEP by default:
- Verification performed and relevant deployment URLs (NO bypass parameters):
- Removal guide: `skills/vercel-deployment-password-gate/reference/uninstall.md`
  in [stealth-factory/skills](https://github.com/stealth-factory/skills/blob/main/skills/vercel-deployment-password-gate/reference/uninstall.md).

Revert verified gate-only commits newest first, then complete external cleanup
and redeployment using the guide. Git reverts do not change Vercel variables or
existing deployments. Update this record after gate changes and mark it removed
after uninstall; keep it as an audit trail.
