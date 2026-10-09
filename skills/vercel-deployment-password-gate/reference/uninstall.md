# Remove the deployment password gate

Use this guide when asked to remove, uninstall, or revert the DIY gate. Complete
steps 1–5 in order. Choose **either** step 2A or 2B, then continue to step 3.
Deleting an environment variable alone is not a complete uninstall.

## 1. Identify the installation and intended scope

Read `docs/deploy-gate-installation.md` if present. Inspect Git history and the
current code to confirm actual paths, installation commits, Mode A/B, Vercel
project, and affected environments. Older installs may have no record.

Distinguish the DIY gate from Vercel's platform Password Protection or Vercel
Authentication. This guide removes the DIY code; it does not disable independent
platform protection or cancel a paid add-on. Do not remove application login,
authorization, or unrelated middleware.

An explicit request to uninstall this gate authorizes its removal; explain
which deployments will lose the wall and proceed. If the user only wants one
domain public, use the skill's domain-exception workflow. If they only want
production public while keeping previews gated, restore the default preview-only
predicate and Mode B production-strip wiring instead of uninstalling everywhere.
Clarify only when the target or intended scope is ambiguous.

Inspect `git status --short`; use a clean feature branch/worktree and preserve
unrelated edits. Do not reset or discard the user's work.

## 2A. Preferred: revert isolated gate commits

1. Identify the gate installation and later gate-only fixes from the record and
   history. Do **not** put the separate installation-record SHA bookkeeping
   commit in that revert sequence: it only records SHAs the install commit
   could not contain. Reverting the install commit while that later commit
   remains can conflict on the record file; step 4 keeps the file. Inspect
   **each** candidate with `git show --stat <sha>` and `git show <sha>`; a
   commit title alone does not prove it is gate-only. Use SHAs that actually
   exist in the target branch's history. After a squash, use the final
   gate-only squash commit, not both it and the original commits.
2. Keep prerequisite migrations and unrelated features. If a commit mixes them
   with the gate, or later application work depends on a file it introduced,
   use manual removal (2B). A clean revert can still break later code.
3. Revert each gate-only commit **newest first**, one command at a time:

   ```bash
   # Replace each placeholder with a verified, actual commit SHA.
   git revert <newest-gate-only-sha>
   git revert <older-gate-only-sha>
   git revert <initial-gate-install-sha>
   ```

   Run only the lines applicable to your installation; a single installation
   commit needs one revert. Do not revert a broad commit range: it can include
   unrelated work. Do not use `git reset --hard`, force-push, or restore entire
   files from the pre-install revision.
4. If a revert conflicts, resolve only when you can preserve current application
   behavior and remove just the gate, then `git revert --continue`. A
   modify/delete conflict on the installation record is expected when the later
   SHA bookkeeping commit touched it: keep that file (`git add` the version Git
   left in the tree) and do not `git rm` it. Otherwise `git revert --abort` and
   use step 2B on the current tree. Previously completed reverts remain; do not
   apply them twice. Do not guess `git revert -m` for a merge commit; use
   manual removal unless its parent and complete diff are understood.
5. Review the resulting diff against the manual checklist below for leftovers
   and unintended removals. If the revert deleted the installation record,
   recreate it from the template. Then mark the record removed, describe what
   was removed and what was kept, and record the uninstall commit(s). Keep the
   record as an audit trail.

Continue to step 3. Reverting the installation is not the end of uninstall.

## 2B. Fallback: remove the integration manually

Use this when SHAs are missing, changes were mixed/squashed with unrelated work,
or the gate and host code have evolved together.

| Installation | Required edit |
| --- | --- |
| **Mode A: existing host middleware/proxy** | Remove the gate helper import, `previewGate` call, block-return branch, and `withUnlockCookie` wrapper. Remove gate-only bypass-header stripping and request reconstruction; reconnect the original request/response flow while preserving later application behavior. Delete the helper only after all its consumers are removed. **Keep the host middleware/proxy file**, i18n, redirects, rewrites, auth, and other cookies. |
| **Mode B: standalone gate** | Remove the gate-only file at the path that was actually installed. Next 16: `proxy.ts` or `src/proxy.ts`, including a `pageExtensions` name such as `proxy.page.ts`. Next ≤15: `middleware.ts` or `src/middleware.ts` (the export is `middleware`), including a `pageExtensions` name such as `middleware.page.ts`. Non-Next Routing Middleware: root `middleware.ts` only; `src/middleware.ts` is not loaded. Inspect the contents first: `@deploy-gate:managed` or the legacy `@preview-gate:managed` marker is not proof that no application logic was added later. If shared, remove only the gate and retain host logic as in Mode A. |
| **Production build strip** | Remove `node scripts/remove-proxy-on-prod.mjs &&` from every build entry that uses it, including package scripts, CI, and a Vercel dashboard Build Command override. Preserve the rest of the build command. Then delete the unused script. **Do not run the script as an uninstaller**: it only strips marked files during production builds. |
| **Matchers/runtime/cache configuration** | Remove gate-only matcher branches and exceptions for `/__deploy-unlock` and the legacy `/__preview-unlock`; retain coverage needed by the host. Remove gate-only runtime/cache entries only if unused elsewhere. Keep middleware-to-proxy migrations by default. Do not delete `VERCEL_ENV` or `VERCEL_TARGET_ENV` from shared configuration merely because the gate used them. |
| **Dependencies and supporting files** | Remove `@vercel/functions` only if the gate introduced it and no remaining code uses it. Use the project's package manager to update its lockfile. Remove gate-only tests, copied hash/token helpers, branded assets, docs/examples, and env declarations; preserve shared resources. |

Use the actual installed names, which may differ from the templates. Commit the
reviewed removal separately, e.g. `revert(deploy-gate): remove password protection`.
Update the installation record, mark it removed, and describe what was removed and what was kept.

## 3. Validate and deploy the code removal

1. Run the project's relevant typecheck/lint/build and existing tests. Exercise
   any retained host middleware (especially i18n, redirects, auth, APIs, and
   request bodies). Check that build commands no longer reference the stripper.
2. Search tracked source/configuration for leftovers without printing secrets:

   ```bash
   git grep -l -E 'DEPLOY_GATE_|PREVIEW_PASSWORD|PREVIEW_GATE_|previewGate|withUnlockCookie|deploy-gate|preview-gate|deploy_gate|preview_gate|/__deploy-unlock|/__preview-unlock'
   ```

   This prints filenames only. Inspect matches carefully; the retained audit
   record is expected, live gate imports/config/CI wiring need cleanup. Exit 1
   means no matches; other errors are failed checks, not a clean result. Also
   inspect ignored local env files by variable name without dumping values.
3. Deploy the code removal to every intended environment using the project's
   normal release process. Before an alias is pointed at a build that no longer
   strips the bypass header, stop callers from sending `x-deploy-gate-bypass`
   and the legacy `x-preview-gate-bypass` (header or query) to that alias.
   Those callers can be locked out of the still-gated deployment until the
   alias moves; that is preferable to the token reaching the app and its logs.
   Then verify the intended aliases point to the new build. Keep the gate
   credentials until this succeeds, so rollback remains possible and an
   unrelated build from old code does not accidentally ship ungated.

## 4. Clean up configuration outside Git

After the new code is deployed, inventory and remove the following **gate-owned**
variables from the correct Vercel project. Check Preview, Production (if used),
Development, each custom environment, and branch-specific overrides separately.
Remove them from local `.env*` files, CI, and secret-manager entries where present.
Use the dashboard or the installed CLI's verified syntax; do not bulk-delete
variables by prefix or expose their values.

Before removing a shared Preview or Production credential, list every branch,
custom environment, and commit source that can still deploy and still contains
the gate. Absent configuration fails open: the next build of that old code
publishes ungated. Land the code removal on each of those sources first, retire
them so they cannot deploy, or keep a branch-specific credential until their
removal is deployed. Deleting the shared scope does not change an immutable
deployment that already built with the credential.

| Variable | Why it must be checked |
| --- | --- |
| `DEPLOY_GATE_PASSWORD_HASH` | Current password credential |
| `DEPLOY_GATE_PASSWORD` | Legacy plaintext credential |
| `DEPLOY_GATE_BYPASS_TOKENS` | Named automation credentials |
| `DEPLOY_GATE_UNPROTECTED_HOSTS` | Domain exceptions; left alone in old gate code it can trigger a 503 on other hosts |
| `PREVIEW_PASSWORD_HASH` | Legacy fallback can keep the gate active |
| `PREVIEW_PASSWORD` | Legacy plaintext fallback |
| `PREVIEW_GATE_BYPASS_TOKENS` | Legacy token fallback |
| `DEPLOY_GATE_BYPASS_SECRET` | Optional CI/monitoring credential convention; not a gate runtime variable |

The caller update belongs in step 3, before the alias change, not after it.
Update tests, monitors, scripts, and saved links so they no longer send
`x-deploy-gate-bypass` or the legacy `x-preview-gate-bypass` header or query
parameter to aliases that are about to become public. The installed gate strips
that header before the app sees it; once the stripper is gone, a request that
still carries it exposes the token to the application and to request logs. A
caller that must still open an old gated deployment URL may keep using the
token against that URL only. Inspect custom-named CI secrets recorded during
installation too. Do not delete shared credentials or Vercel's
`VERCEL_AUTOMATION_BYPASS_SECRET`, which belongs to independent platform
protection. Removing a caller copy does not revoke the token on an immutable
deployment that already has it.

**Unset obsolete variables; do not replace them with empty strings or `{}`.**
Present-but-empty credentials can fail closed in remaining old gate code. Check
legacy aliases as well as current names. Removing variables alone leaves the
middleware, dependencies, and build wiring installed.

Git cannot revert dashboard changes. Review recorded platform changes and restore
only settings required by the requested final state. Do not blindly turn Vercel
Authentication back on: that could replace the removed password wall with SSO
when the user wanted a public site. Do not cancel subscriptions as an incidental
cleanup step.

## 5. Verify removal and report remaining old deployments

Test a fresh browser session and HTTP requests without gate cookies, bypass
headers, or bypass query parameters on every intended new environment/alias:

- The page shows the application (or its legitimate application login), not the
  gate form or its configuration-error 503. Check content as well as HTTP status;
  a bare 200 or a platform login page is not proof of successful removal.
- A nested page and an API route still behave correctly. Retained host rewrites,
  redirects, auth, and cookies work. No new `deploy_gate` or `preview_gate`
  cookie is issued. Neither `/__deploy-unlock` nor the legacy
  `/__preview-unlock` serves the gate handler (a framework fallback/404 is
  acceptable).
- Independent Vercel protection may still block anonymous access. Report that
  separately and follow the user's intended scope; do not silently disable it.

Existing browser gate cookies become unused once the code is gone; they do not
require keeping a cleanup middleware installed. Clear them in the test browser
to avoid misleading results.

**Existing immutable deployment URLs retain their old code and credentials.**
Neither a revert nor deleting project env vars changes them. A redeploy creates
a new deployment; moving an alias does not erase the old URL. Identify relevant
old previews/production deployments and report which remain gated. If retiring
them is within the requested scope, use the project's deployment-retirement
process; do not delete deployments indiscriminately. Removing local or CI token
copies also does not revoke those tokens on old deployments.

Finish with: removal commit/PR, environments and aliases verified, external
cleanup completed or still pending, retained platform/application protection,
and historical-deployment limitations. Do not claim full removal while required
deployment or configuration steps remain blocked.
