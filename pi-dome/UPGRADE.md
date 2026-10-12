# pi-dome upgrade closeout and rollback guide

> **Status: upgrade successfully tested on the host (user-reported); stage 10
> kit/documentation cleanup completed.** The user removed the startup upgrade
> task to keep the kit simple and already cleaned up sbx secrets. The current kit
> seeds missing configuration files only. No rollback, credential changes, host
> OAuth deletion, or sandbox retirement was performed during this cleanup.

The original execution plan has been condensed into this closeout record and
retained rollback guidance. [README.md](README.md) describes current setup and
usage; `spec.yaml` is the source of truth for kit behavior.

## Final kit behavior

- Standalone kit, using a digest-pinned upstream Pi image; no additive composition
  with `pi` and no inherited Anthropic credential configuration.
- Modern `openai` provider with host-owned **Sign in with ChatGPT** credentials.
  A protected host token helper remains the dynamic secret source; it is not the
  removed sandbox startup task. Sandbox Pi uses `OPENAI_API_KEY=proxy-managed`.
- Credential injection is declared only for `api.openai.com`. No host auth-file
  mount, auth overlay, or real token in sandbox environment/configuration is
  needed by the kit.
- Private `settings.json` and `web-search.json` seeds use `onlyIfMissing: true`.
  Existing settings, package choices, and `deviceId` are left unchanged.
- No startup migration, package reconciliation, upgrade, backup/marker management,
  credential-conflict detection, or automatic search-proxy configuration.
- Install hooks prepare the agent directory and npm proxy settings only. Configure
  the search extension's proxy manually if needed; npm proxy setup does not
  configure extension transport. See [README.md](README.md#web-search-and-network).

The earlier migration-helper acceptance criteria and fixture/lifecycle commands
are no longer instructions for this simplified kit. Updating an existing sandbox
is a deliberate manual operation, with private backups of any non-secret settings
being changed. Never copy or overwrite host OAuth files as part of that work.

## Recorded baseline

| Item | Recorded selection / evidence |
| --- | --- |
| Image | `docker.io/sbx/pi-image@sha256:b278e3e25c3cae780efd9b5b4df047209a91cebafa45426c241356fa53c05cfd` |
| Runtime | Pi `1.1.0`, Node `22.22.1`, Linux amd64 |
| Architecture limit | OCI index includes arm64; arm64 runtime verification was not recorded |
| Coding model | `openai/gpt-6.1-sol` |
| Thinking / transport | `xhigh` / `sse` |
| Search | Provider `openai`, model `gpt-6.1-sol`, `workflow: "none"` |
| Packages | `pi-web-access@0.39.0`, `@juicesharp/rpiv-ask-user-question@2.13.5` |
| Credential service | `openai`; host-owned dynamic source, proxy-managed injection |
| Final host credential scope | Not recorded here; do not infer or change it during cleanup |
| Host sbx version / published upgrade SHA | Not recorded here |

## Completion record

- [x] User reported that the upgrade was successfully tested.
- [x] Startup upgrade task removed by the user; current `spec.yaml` has no
      `setup.startup` task and the mounted kit has no migration helper artifacts.
- [x] sbx secret cleanup already completed by the user; no secret commands rerun.
- [x] README aligned with the seed-only kit, including manual search-proxy setup.
- [x] Obsolete migration-helper and removed test-file instructions retired from
      current documentation; rollback safeguards retained below.
- [x] Local YAML/JSON and documentation consistency checks passed for the mounted
      kit during cleanup. These are not sbx runtime or authentication tests.

The previous execution record reported spec/unit, container TCK, and Docker
stop/start checks before the startup helper was removed. Those historical results
do not certify the final simplified configuration. The user's successful host
upgrade report is retained without inventing per-gate results for natural token
expiry, concurrent refresh, hosted search, TUI behavior, or authenticated e2e.

This sandbox exposes only the kit files, not Git metadata or the full repository's
test tooling, and has no `sbx` executable. No new host lifecycle/authentication
checks or Git operations were possible during cleanup. Host sandbox retirement
was explicitly excluded from the requested kit/documentation-only scope.

### 10. Rollback and cleanup

#### Successful-rollout cleanup

1. **Credential cleanup is already done**, as reported by the user. Do not repeat
   removal commands, change production scope, or touch unrelated global sources.
2. **Kit/documentation cleanup is complete.** The kit retains only missing-file
   seeding and install-time preparation. Documentation no longer promises a
   startup migration/helper or references removed helper tests as current tests.
3. **Keep production authentication intact.** The host auth directory, its stable
   settings/deviceId, Pi/Node dependencies, and protected token helper are runtime
   dependencies, not disposable upgrade artifacts.
4. **Retire old/pilot sandboxes separately if still needed.** First confirm that
   their non-secret settings/sessions and any rollback state are no longer needed.
   Retirement was not requested or performed here; no sandbox names are assumed.
5. **Keep rollback information until it is no longer needed.** Preserve any saved
   pre-upgrade kit revision/image and private non-secret settings backups. Nothing
   in this cleanup deletes them or asserts that an old sandbox is still available.

#### If a later rollout needs rollback

Run host lifecycle and credential commands on the **host**. Repository operations
belong in a checkout with Git metadata. Do not execute this failure path merely
because successful cleanup is complete.

1. Stop the affected new/pilot sandbox and preserve private diagnostics plus any
   required non-secret settings/session state.
2. Reattach to a retained known-working sandbox, or recreate it from the saved
   kit revision and image reference. A legacy rollback may require native login;
   it is not a solution to the modern host-owned authentication requirement.
3. Only if a newly added pilot-scoped resolver still exists and its removal is
   intended, remove that exact binding:

   ```bash
   # Example only: use the actual pilot name and review its current scope first.
   sbx secret rm openai --sandbox pi-dome-upgrade-pilot
   ```

   Removing a scoped source exposes any global source for the same service again.
   Do not resume the sandbox assuming it is credential-free. Remove the sandbox
   itself only after required state has been saved. Do not rerun this example for
   bindings already cleaned up.
4. If an approved global source was changed, restore its previous source/scope
   through the user's credential-management flow. Git rollback does not restore
   host credentials. Never use `secret rm --all` or `sbx reset` for this cleanup.
5. Restore user settings from a private backup only with consent. Never overwrite
   or delete host OAuth files as routine rollback; logout/revocation requires an
   explicit decision to remove the grant.
6. Revert implementation commits on shared branches rather than force-resetting
   history. Retain any pre-upgrade backup branch and keep the upstream merge
   separate from implementation reversals when available.

## Future upgrades

Changes to the image, package pins, models, proxy transport, or credential scope
require deliberate re-testing. At minimum, record host sbx/runtime versions,
confirm placeholder-only isolation and real `openai` inference, check extensions
and search through the proxy, and verify settings survive stop/restart. Exercise
expiry/concurrent refresh when changing authentication or resolver behavior.

Keep test-daemon credentials/policy separate from production. Content-only TCK or
non-empty prompt output is not proof of inference. Publish only an existing,
verified Git commit or OCI digest; this closeout does not claim a new release was
published.
