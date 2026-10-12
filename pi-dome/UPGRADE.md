# pi-dome upgrade plan and execution guide

> **Status: proposed, not implemented.** This guide consolidates the upstream
> review and the host-authentication proposal. The current kit still uses native
> in-sandbox `openai-codex` login and a restart-time settings overlay.
> Following this guide changes that architecture; adding this document does not.

## Recommended target

Use Pi's **`openai` provider → Sign in with ChatGPT**, authenticated once on the
host. Host Pi owns the OAuth credential and refreshes it. Docker Sandboxes
resolves a current access token through a dynamic secret and injects it into
requests to `api.openai.com`. Sandbox Pi sees only `OPENAI_API_KEY=proxy-managed`.

```text
Host, outside all writable sandbox mounts
  dedicated Pi agent directory: auth.json + settings.json/deviceId
       │
       │ pi auth print-bearer-token --provider openai --min-expiry 5m
       ▼
  sbx dynamic secret resolver (--refresh on-demand)
       │
       │ proxy replaces Bearer proxy-managed with the current access token
       ▼
  https://api.openai.com/v1/responses
       ▲
       │ request with placeholder only
Sandbox: Pi openai provider + SSE + pinned extensions
```

This is **not** legacy `openai-codex`, not an OpenAI API-key purchase, and not
`sbx secret set openai --oauth`. Docker's built-in OpenAI OAuth flow is documented
for Codex; compatibility with Pi's newer dynamic-registration flow has not been
established. Do not substitute it for the host Pi resolver.

### Decisions to carry into implementation

1. **Keep the kit standalone.** Reuse the upstream image, not `extends: pi` or a
   composition with `./pi/`. Additive composition cannot remove its Anthropic
   credentials or credential-file customizations.
2. **Keep OAuth on the host.** Do not mount/copy the host `auth.json`, distribute
   refresh tokens, or write an engine-managed sandbox `auth.json`.
3. **Use durable settings.** Remove the overlay at
   `files/home/.pi/agent/settings.json`; seed only when missing, then migrate
   narrowly and idempotently. Preserve user choices and `deviceId`.
4. **Set both provider and model.** `defaultProvider` alone is not a reliable
   provider-selection constraint. Validate a specific model under `openai`.
5. **Pin a tested baseline.** Pin the image by digest and the two npm packages by
   version. Verify the image's actual Pi/Node versions, not just host npm metadata.
6. **Start with sandbox-scoped credentials.** A global `openai` binding can affect
   unrelated OpenAI/Codex sandboxes. Choose production scope explicitly.
7. **Make web search explicit.** Select OpenAI search and its model, configure
   extension proxy transport, and avoid browser-curator workflows by default.
8. **Gate rollout on real tests.** A mocked provider request and a non-empty TCK
   prompt result do not prove authenticated inference or token refresh.

## Review baseline and confidence

| Item | Observed result | What remains to prove |
| --- | --- | --- |
| Repository | Fork HEAD `c4bc3a36f08715a199735a3020c123ffb9a4496d` (`Add pi-dome`) | Upgrade branch and merge |
| Fetched upstream | `358a89d1cda7f47b37a5a4b60b2ce63be4eab818`; 67 new commits after the previous fetched tip | Re-review any newer upstream commits |
| Changes in `pi/` | Only a Dockerfile comment change, `db5d885` / PR #280; no functional kit/auth change | Rolling image contents can still change independently |
| Pi runtime | Installed Pi and npm metadata reported `1.1.0` | Published image's Pi/Node versions |
| Spec/TCK | Upstream spec tests and pi-dome pure-logic TCK checks passed in an isolated snapshot | Container TCK and real sbx e2e |
| Modern provider | A credential-free mocked Pi 1.1.0 request used `Bearer proxy-managed` at `/v1/responses` and omitted ChatGPT-unsupported `temperature`, `max_output_tokens`, and `prompt_cache_retention` | Actual proxy replacement and subscription inference |
| Host secret mechanism | Docker documents `--command`, sandbox scope, and `--refresh on-demand` | Installed host sbx support, daemon execution, refresh/concurrency |
| Extensions | Candidates: `pi-web-access@0.39.0`, `@juicesharp/rpiv-ask-user-question@2.13.5` | Compatibility, search entitlement, and proxy behavior in the target image |

The review environment has no `sbx` executable. No authenticated end-to-end test
has been performed, and no host secret has been registered.

## Step-by-step execution

1. [Check prerequisites](#0-record-state-and-check-prerequisites) and
   [synchronize upstream](#1-create-an-upgrade-branch-and-synchronize-upstream).
2. [Verify the runtime](#2-choose-and-verify-the-runtime-baseline) and
   [authenticate host Pi](#3-authenticate-once-with-host-pi).
3. [Implement the kit changes](#4-implement-the-kit-changes) and
   [protect the credential helper](#5-create-a-protected-host-credential-helper).
4. [Create a pilot](#6-create-a-fresh-pilot-and-register-the-dynamic-secret),
   [verify lifecycle behavior](#7-prove-authenticated-inference-and-lifecycle-behavior),
   and [run automated tests](#8-run-automated-tests-keeping-daemon-scopes-explicit).
5. [Approve scope and roll out](#9-choose-production-credential-scope-then-roll-out),
   retaining the [rollback path](#10-rollback-and-cleanup).

Run repository operations in the checkout. Run all `sbx` commands and host OAuth
setup on the **host**, not inside the sandbox. Commands below are instructions,
not actions already taken. Replace all `/absolute/...` and model/digest
placeholders before use. Do not run later steps until their prerequisites pass.

### 0. Record state and check prerequisites

Record the current kit revision, existing sandbox names, relevant credential
scope/source, and runtime versions. Keep notes free of credentials; do not export
or paste secret-store contents or `auth.json` into a transcript.

```bash
# Repository
 git status --short --branch
 git remote -v
 git rev-parse HEAD upstream/main

# Host
 sbx --version
 sbx secret set --help
 sbx create --help
 sbx run --help
 pi --version
 pi auth print-bearer-token --help
 node --version
 docker version
```

Confirm the installed sbx supports the current v2 spec, `--command`,
`--refresh on-demand`, `--sandbox`, and standalone kit references as the first
positional argument to `create`/`run`. Upgrade sbx if necessary; do not invent a
minimum version before testing it. These instructions target local Sandboxes,
not `sbx --cloud`.

Record absolute paths to host Pi and Node. With nvm/asdf or another version
manager, the daemon may not inherit your interactive shell's PATH. Verify the
chosen Pi executable, Node runtime, helper, agent directory, and host temporary
directory are all outside writable sandbox mounts. This includes dependencies
and configuration they load, and mounts added later. Never mount the whole host
home directory to make authentication work.

**Gate:** host tooling supports the required interfaces; the checkout is clean
(commit this documentation separately first, if needed); the existing working
sandbox is retained for rollback.

### 1. Create an upgrade branch and synchronize upstream

```bash
# Repository; choose unused branch names
 git branch backup/pi-dome-before-upgrade
 git switch -c upgrade/pi-dome-host-chatgpt
 git fetch https://github.com/docker/sbx-kits-contrib.git \
   '+refs/heads/*:refs/remotes/upstream/*'
 git log --oneline HEAD..upstream/main
 git diff HEAD..upstream/main -- pi/
```

The HTTPS fetch avoids the SSH host-key failure encountered during review and
leaves configured remotes unchanged. Merge the reviewed upstream revision:

```bash
 git merge --no-ff 358a89d1cda7f47b37a5a4b60b2ce63be4eab818
```

If upgrading to a newer upstream tip, inspect and record that revision first,
then merge that reviewed revision instead. Keep the upstream merge separate
from pi-dome implementation commits. Do not expect the merge itself to fix
pi-dome authentication: the reviewed upstream `pi/` delta is only a comment.

### 2. Choose and verify the runtime baseline

Start with Pi `1.1.0` and the extension candidates in the evidence table, or
select a newer baseline and repeat the compatibility checks. Use a recent host
Pi that exposes **Sign in with ChatGPT under `openai`** and the bearer-token CLI.
Do not fall back to the legacy provider if the installed runtime lacks them.

For the image, resolve a candidate digest and inspect its runtime:

```bash
# Host; registry access may be required
 docker pull docker.io/sbx/pi-image:latest
 docker image inspect docker.io/sbx/pi-image:latest \
   --format '{{json .RepoDigests}}'
```

Select the digest from that result, then use that exact reference for subsequent
checks and in `sandbox.image`:

```bash
 IMAGE='docker.io/sbx/pi-image@sha256:REPLACE_WITH_VERIFIED_DIGEST'
 docker run --rm --entrypoint pi "$IMAGE" --version
 docker run --rm --entrypoint node "$IMAGE" --version
```

For multi-platform distribution, verify each supported architecture and pin the
appropriate manifest-list digest. If no published image has the required Pi,
use a separately built/published image with a pinned Pi version and base image.
The upstream Dockerfile has a `PI_VERSION` build argument, but do not assume a
`build:` block in the kit will be executed by the installed sbx release.

**Gate:** record image digest, architecture, Pi version, Node version, and package
versions. `latest` and a successful `pi --version` alone are not a compatibility
certification. Do not run automatic self/package updates in startup hooks;
updates become an explicit re-test and re-pin operation.

### 3. Authenticate once with host Pi

Choose a dedicated, private directory outside all writable sandbox mounts. The
following uses POSIX host paths; adapt paths for the host platform.

```bash
# Host
 AUTH_DIR="$HOME/.config/pi-dome-auth"
 umask 077
 mkdir -p "$AUTH_DIR"
 chmod 700 "$AUTH_DIR"
 cd "$AUTH_DIR"
 PI_CODING_AGENT_DIR="$AUTH_DIR" pi --no-extensions
```

In this **host Pi** session:

1. Run `/login openai`.
2. Select **Sign in with ChatGPT**, not an API-key method or `openai-codex`.
3. Complete the browser flow. If Pi asks for a callback URL/code, paste it only
   into its login prompt, never into an agent conversation or shared log.
4. Use `/model` to choose an available `openai` model. Record its exact model ID.
5. Exit Pi.

Keep both `auth.json` and the corresponding `settings.json` (including the
stable `deviceId`). Protect them with owner-only permissions. Do not periodically
replace settings with a template. Host logout, grant revocation, account-policy
changes, or invalidated refresh tokens can require host reauthentication later;
"once" means no login per sandbox, not a promise that OAuth never expires.

Verify without printing credentials:

```bash
# Host; use the same private agent directory
 PI_CODING_AGENT_DIR="$AUTH_DIR" pi auth check --provider openai --json
 PI_CODING_AGENT_DIR="$AUTH_DIR" pi auth print-bearer-token \
   --provider openai --min-expiry 5m >/dev/null

 OPENAI_MODEL='REPLACE_WITH_VALIDATED_OPENAI_MODEL_ID'
 PI_CODING_AGENT_DIR="$AUTH_DIR" pi --no-extensions --no-tools --no-session \
   --model "openai/$OPENAI_MODEL" -p 'Reply with exactly OK.'
```

The bearer-token command must succeed with OAuth, not merely an environment API
key. Do not add `--credentials` to `auth check`. The real inference test consumes
account usage; confirm the selected model is supported by the subscription.
Keep this host directory dedicated to authentication, without untrusted project
configuration or third-party extensions.

**Gate:** modern host OAuth and one real `openai` inference succeed.

### 4. Implement the kit changes

These are future edits, not changes made by this guide:

| File / artifact | Required change |
| --- | --- |
| `pi-dome/spec.yaml` | Modern description; verified image digest; proxy-managed credential below; durable settings setup; retain npm proxy setup |
| `pi-dome/files/home/.pi/agent/settings.json` | Remove this active settings overlay |
| Settings seeding/migration | Use `setup.files` with `onlyIfMissing: true`, plus a narrowly scoped migration helper if needed |
| Web-search configuration | Seed `web-search.json` only when missing; derive its proxy from the sandbox environment safely |
| Regression tests | Add credential/default/seed/migration checks and real lifecycle coverage |
| `pi-dome/README.md` | Replace legacy auth/usage guidance only when the new behavior is implemented and validated |
| `pi-dome/testdata/tck.yaml` | Optional authenticated smoke configuration, with the e2e daemon credential provisioned deliberately |

#### Credential declaration

Add this **top-level** block:

```yaml
credentials:
  - service: openai
    description: Host Pi ChatGPT access token, injected by the sandbox proxy
    apiKey:
      name: OPENAI_API_KEY
      proxyManaged: true
      inject:
        - domain: api.openai.com
          scheme: bearer
```

`apiKey` describes the **injection shape**, not the credential's billing/auth
origin. The host resolver supplies a ChatGPT OAuth access token. In this repo's
spec, `proxyManaged: true` is necessary to populate the in-container placeholder;
`name` alone is insufficient. Do not add an `oauth.credentialFile` or a sandbox
`auth.json` template. Initially leave `required` unset/false so the pilot can be
created before registering its sandbox-scoped resolver; reject an unbound pilot
at the validation gate before use.

Retain `api.openai.com` in the network allowlist and approve credential injection
for that host through sbx's user-controlled binding flow. The requested and
approved domains must intersect. Do not silently replace
`~/.config/sbx/credentials.yaml` or widen an existing binding.

For host-only authentication, `auth.openai.com` and `chatgpt.com` are no longer
needed for sandbox login/Codex inference. Remove them after tests establish they
are unnecessary, unless native sandbox login is an explicitly supported mode.
Host token refresh is separate from the sandbox egress allowlist. Keep
`registry.npmjs.org` for extension installation. Declare additional tool/model
catalog hosts only when required and reviewed; do not broadly allow everything.

#### Durable settings

Seed these defaults **only if settings are absent**, replacing the model marker
with the tested ID before shipping:

```json
{
  "defaultProvider": "openai",
  "defaultModel": "REPLACE_WITH_VALIDATED_OPENAI_MODEL_ID",
  "transport": "sse",
  "packages": [
    "npm:pi-web-access@0.39.0",
    "npm:@juicesharp/rpiv-ask-user-question@2.13.5"
  ]
}
```

Use `setup.files` with `path: /home/agent/.pi/agent/settings.json` and
`onlyIfMissing: true`, with a preceding directory-creation hook if needed. Choose
ordering consistent with sbx's lifecycle: install precedes startup file writes,
and startup commands run on every container start. Do not keep the original
home overlay alongside the new seeding logic; it would defeat preservation.

Migration rules:

- Back up existing settings privately before any intentional migration.
- On a fresh file, write the validated defaults. On existing customized files,
  preserve unknown keys, `deviceId`, models, transport, and resource lists.
- Migrate the recognized old kit-default profile, or explicitly opted-in fields;
  do not silently switch every user who intentionally chose `openai-codex`.
- Merge package entries by package identity, preserving user-added entries and
  object-form resource filters. Upgrade recognized unversioned kit defaults to
  the chosen pins; do not duplicate entries or downgrade user-selected pins.
- Use a versioned migration marker, atomic JSON writes, and owner-correct
  permissions. Re-running must converge without rewriting unchanged files.
- Invalid JSON must fail safely with a useful diagnostic, not be overwritten.
- Never alter `auth.json`. A stored sandbox `openai` credential can override the
  environment placeholder: report that conflict and require a user decision.
- Account for project `.pi/settings.json`/`models.json` and resumed-session model
  choices during validation. Project settings can override global defaults;
  resource lists are combined.

Preservation across **stop/restart** is required. Removing a sandbox does not
preserve its user settings/sessions automatically; preserve non-secret state
explicitly if desired. Host-owned authentication survives sandbox recreation.

#### OpenAI web search and extension transport

Seed, rather than overwrite, a configuration such as:

```json
{
  "provider": "openai",
  "openaiSearchProviders": ["openai"],
  "openaiSearchModel": "REPLACE_WITH_VALIDATED_SEARCH_MODEL_ID",
  "workflow": "none"
}
```

The search model must be tested for hosted web-search support; it need not match
the coding model. Keep the official Responses endpoint and do not enable the
legacy Codex alpha-search protocol for this upgrade.

Host-owned OAuth looks like an environment credential inside Pi. Automatic
subscription detection via `isUsingOAuth()` therefore cannot be relied on.
`openaiSearchProviders` constrains credential resolution; `provider: "openai"`
also prevents automatic routing to Exa or another unapproved provider.

**Configure the extension's `proxy` explicitly** from the current sandbox
`HTTPS_PROXY`/`HTTP_PROXY`, using a safe JSON writer and without overwriting an
intentional user proxy. Do not hardcode a pilot sandbox's proxy address into a
published kit. The extension documents that Node fetch does not automatically
honor environment proxies; Pi model calls and npm proxy setup do not configure
extension transport. Ensure the target image has `curl` for its proxy path.
If proxy-only DNS requires `ssrf.trustEnvProxy`, enable it only after diagnosis;
it adjusts DNS preflight, not proxy transport. Avoid per-call `proxy: ""`, which
forces direct access and prevents placeholder replacement.

OpenAI search can use the modern subscription credential, but entitlement and
end-to-end behavior remain test gates. `workflow: "none"` avoids an extra summary
model call and an inaccessible curator browser. Searching and fetching result
pages are different operations: fetching arbitrary URLs still needs narrowly
approved target hosts, and OpenAI search does not grant unrestricted egress.

### 5. Create a protected host credential helper

Use an absolute helper path, outside shared workspaces, for daemon execution.
The template below is **not ready to execute** until all paths are replaced:

```sh
#!/bin/sh
set -eu
export PI_CODING_AGENT_DIR='/absolute/host/path/pi-dome-auth'
export PATH='/absolute/host/node/bin:/usr/bin:/bin'
exec '/absolute/host/path/pi' auth print-bearer-token \
  --provider openai --min-expiry 5m
```

Store it, for example, at a private host path
`/absolute/host/path/pi-dome-token`, with mode `0700`. Adjust PATH for the Pi
executable's Node shebang and platform dependencies. The helper must emit only
the token to stdout; no shell banners, debugging, or `set -x`. Do not embed an
access/refresh token in the script or command text.

Test from a private directory, discarding secret output:

```bash
# Host; use the actual absolute helper path
 /absolute/host/path/pi-dome-token >/dev/null
```

sbx runs command secrets from a fresh host temporary directory, not the
checkout. Do not rely on relative paths or a project-controlled helper. The
helper's runtime, configuration, and dependencies need the same mount protection
as the helper itself. Pi handles refresh/persistence; do not implement a second
independent refresh-token owner.

### 6. Create a fresh pilot and register the dynamic secret

Validate the changed kit first:

```bash
# Host, repository root
 sbx kit validate ./pi-dome/
 sbx kit inspect ./pi-dome/ --output json
 sbx create --name pi-dome-upgrade-pilot ./pi-dome/ "$PWD"
```

Use a **new** sandbox name with this standalone kit only. Do not use
`--kit ./pi-dome/ pi-dome`, which treats the kit as an additional mixin on current
sbx. Do not reuse an old sandbox carrying Anthropic bindings or overlays, and do
not use `kit add` as a way to remove them. Review any credential approval prompts
and the effective binding/source; existing global OpenAI credentials may be
visible until the scoped source is installed.

With explicit user approval, register the source on the host:

```bash
 sbx secret set openai --sandbox pi-dome-upgrade-pilot \
   --command '/absolute/host/path/pi-dome-token' \
   --refresh on-demand
```

Registration verifies the command by default. Avoid `--no-verify` for rollout and
avoid `--show-error` unless privately reviewing potentially sensitive resolver
stderr. A sandbox-scoped source takes precedence over a global source, and
service-secret updates take effect without restarting existing local sandboxes.

`on-demand` resolves for every credential use. It avoids the default 55-minute
sbx cache outliving a short-lived OAuth token. `--min-expiry 5m` asks host Pi to
refresh before a near-expiry token is used. This costs a host process launch per
resolution; measure it before choosing any bounded cache. Do not infer a safe
cache duration from an assumed token lifetime.

**Gate:** default verification succeeds, the intended binding authorizes
`api.openai.com`, and sandbox Pi has the placeholder, not a real credential.
If an initially unbound kit did not receive the environment sentinel, do not
assume storing the secret also retrofits that environment variable. Verify the
installed engine's binding/recreation behavior before proceeding; use only its
documented placeholder configuration, never a real token in `--env` or settings.

### 7. Prove authenticated inference and lifecycle behavior

Set the validated ID in the host shell and run a real request:

```bash
 OPENAI_MODEL='REPLACE_WITH_VALIDATED_OPENAI_MODEL_ID'
 sbx exec pi-dome-upgrade-pilot -- pi --version
 sbx exec pi-dome-upgrade-pilot -- sh -c \
   'test "$OPENAI_API_KEY" = proxy-managed'
 sbx exec pi-dome-upgrade-pilot -- pi --no-extensions --no-tools --no-session \
   --model "openai/$OPENAI_MODEL" -p 'Reply with exactly OK.'
 sbx policy log pi-dome-upgrade-pilot
```

The placeholder check intentionally prints nothing. Do not dump the environment
or auth files. Confirm a real model reply, successful exit, modern provider, and
expected API destination. A startup banner, warning text, or non-empty output
alone does not count. Repeat with extensions enabled to verify package loading.

Attach with the documented current CLI syntax:

```bash
 sbx run --name pi-dome-upgrade-pilot
```

Test all of the following before rollout:

| Check | Required result |
| --- | --- |
| Fresh startup/default selection | A new session selects the configured `openai` model without `/login` |
| Settings customization | Change model/thinking/theme and add a package; kit startup does not reset them |
| Stop/restart | `sbx stop pi-dome-upgrade-pilot`, then `sbx run --name pi-dome-upgrade-pilot`; settings and inference still work |
| Recreation | Create a second fresh sandbox, register the same helper with its sandbox scope, and infer without login; do not delete the first pilot yet |
| Expiry/refresh | After a normal access-token expiry window, inference succeeds and host Pi persists refreshed credentials without a new login |
| Concurrency | Two sandboxes using the same host directory remain usable through refresh; no rotating-token race or corrupted auth store |
| Credential isolation | No host auth-directory mount, real token in sandbox files/env, or Anthropic credential customization |
| Failure handling | In a controlled pilot, an unavailable helper or invalidated host grant fails clearly; no unrelated credential/provider fallback |
| Web search | Force `provider: "openai"`, `workflow: "none"`; verify successful cited results, expected endpoint, and no legacy/Exa fallback |
| Search transport | Requests go through the sandbox proxy; no direct request carrying an unresolved placeholder |
| Question extension | `ask_user_question` loads and works in the TUI |
| Tools | `read`, `bash`, `edit`, `write`, `find`/fd, rg, and Docker work as expected |
| Migration helper | Fresh, partial, customized, malformed, and repeated-run settings fixtures behave as specified |

Wait for natural expiry or use a controlled test fixture; do not hand-edit a live
refresh token or expiry to force a refresh. Concurrency/locking was not verified
in the review, so it is a release gate, not a guarantee. If it fails, serialize
host resolver execution with a suitable cross-process lock and re-test; do not
copy refresh credentials into separate sandboxes.

Use policy diagnostics for blocks. Read the response body: pending approvals
must be reviewed on the host, explicit local denies must be inspected/removed,
and organization-policy blocks require IT approval. Only a default-deny case
without an allow rule should lead to adding an appropriate narrow allow rule.
Do not treat every 403 as a missing network allowance; OpenAI may also reject
account/model/tool entitlements.

### 8. Run automated tests, keeping daemon scopes explicit

```bash
# Repository: structural/unit coverage
 go test ./spec/...

# Host, repository root: container coverage
 ./scripts/test-kit.sh pi-dome

# Host: one-time setup for the separate e2e daemon
 sbx --app-name sbx-kits-contrib-tck login

# Host, repository root: real sbx content/startup coverage
 ./scripts/test-kit-e2e.sh pi-dome
```

The e2e wrapper uses `--app-name sbx-kits-contrib-tck`, its own secrets, and a
`deny-all` policy baseline. It does not inherit the pilot's sandbox-scoped secret
or the main daemon's global secret. Do not point it at the day-to-day daemon or
reset production policy to get green tests.

For an authenticated TCK prompt, deliberately register the helper under the
**test daemon**, after approving the test usage:

```bash
 sbx --app-name sbx-kits-contrib-tck secret set openai \
   --command '/absolute/host/path/pi-dome-token' --refresh on-demand
```

This is global **only within that test daemon**, because the harness generates
sandbox names. Confirm its binding consent supports non-interactive creation.
Then add `testdata/tck.yaml` with `promptArgs: ["-p"]` (no `binary` override is
needed for entrypoint `pi`) and run the e2e script. Without prompt configuration,
a green e2e run covers content/startup, not inference. Even with it, retain the
manual exact-provider, expiry, and search tests: the prompt assertion accepts
non-empty output rather than certifying a successful model response.

Add regression tests for the credential shape/domain, removal of the active
settings overlay, pinned provider/model/packages, only-if-missing behavior, and
migration idempotency/preservation. Container TCK file checks may need explicit
coverage for seeded files, which are no longer static home overlays.

Fork PR e2e jobs can be skipped because CI lacks upstream secrets. Record local
results; a green fork CI check is not proof of real sbx e2e.

### 9. Choose production credential scope, then roll out

After the pilot gates pass, choose one of these deliberately:

- **Sandbox-scoped `openai` (safest initial rollout):** register the same helper
  once for each new sandbox. No new Pi login is needed, but each instance needs
  a secret-source binding. Existing global OpenAI use remains unchanged.
- **Global `openai` (simplest once-per-host setup):** register the helper without
  `--sandbox`. Future appropriately bound pi-dome sandboxes can use it without
  repeating registration. This can change authentication for other sandboxes
  using service `openai`; review all consumers and existing secret precedence
  before overwriting anything.
- **Dedicated service, e.g. `pi-dome-openai` (preferred long-term isolation if
  validated):** change the kit's service identity, declare injection explicitly
  (and `provider: openai` if required), and approve/register it on the host.
  Docker documents kit-declared services, but verify custom-name resolution and
  bindings on the installed sbx before adopting it. Do not present it as an
  already-tested drop-in command.

Use the same host helper/agent directory for approved instances; never duplicate
its refresh token. Configure all credential changes with explicit consent. Do
not overwrite an existing global secret just to avoid one per-instance command.

Update README usage to the standalone positional form, modern host login,
placeholder isolation, tested versions, settings retention, and current model
CLI syntax (`--model openai/<validated-id>`, not `--provider` alone). Retain native
in-sandbox modern `/login openai` only as a documented opt-in fallback: it exposes
real credentials and restores per-sandbox login, so it does not meet this goal.

For Git distribution, publish the upgrade commit to the fork and use its full
commit SHA:

```bash
 sbx run --name pi-dome-new \
   'git+https://github.com/Domeee/sbx-kits-contrib.git#ref=REPLACE_WITH_40_HEX_UPGRADE_COMMIT&dir=pi-dome' .
```

Use that example only after the commit exists remotely. The upstream Docker
repository currently has no pi-dome directory. For OCI distribution, use a
verified published artifact digest, not an assumed
`docker.io/sbx/pi-dome-kit:latest` reference. Record the release kit revision,
image digest, runtime/package versions, model IDs, sbx version, credential scope,
and completed test results before replacing the old working sandbox.

### 10. Rollback and cleanup

Keep the previous sandbox stopped but available until the new path is proven.
If rollout fails:

1. Stop the pilot/new sandbox; preserve private diagnostics and non-secret
   settings/session state as needed.
2. Reattach to the known-working old sandbox, or recreate from the saved kit
   revision and image reference. Legacy rollback may require its native login;
   do not call it a solution to the modern-flow requirement.
3. Remove only the pilot's scoped resolver if it was newly added:

   ```bash
   sbx secret rm openai --sandbox pi-dome-upgrade-pilot
   ```

   Removing a scoped secret exposes any global secret for that service again.
   Do not resume the pilot assuming it has no credentials. Remove the pilot
   sandbox only after any required state has been saved.
4. If an approved global secret was changed, restore its previous source/scope
   through the user's credential-management flow. Git rollback does not restore
   host credentials. Do not run `secret rm --all` or `sbx reset`.
5. Restore user settings from the private backup only with consent. Never
   overwrite/delete host OAuth files as a routine code rollback; revoke/logout
   explicitly if the user wants the grant removed.
6. Revert implementation commits on shared branches rather than force-resetting
   history. The backup branch records the pre-upgrade code; keep implementation
   commits separate from the upstream merge for easier review and reversal.

After successful rollout, remove temporary pilot/test bindings deliberately and
retire the old sandbox only after confirming its state is no longer needed.

## Completion checklist

- [ ] Reviewed upstream revision merged on a dedicated branch.
- [ ] Image digest, Pi/Node versions, package pins, and model IDs recorded.
- [ ] One modern host `openai` login; private auth/settings/helper dependencies
      outside all writable sandbox mounts.
- [ ] Sandbox has only the placeholder; proxy injects only to approved OpenAI API
      destinations; no auth-file overlay or inherited Anthropic configuration.
- [ ] Default provider **and** model set; custom settings survive restart.
- [ ] Fresh instances work without `/login`; expiry and concurrent refresh pass.
- [ ] Search and question extensions pass real tests, including proxy transport.
- [ ] Spec, container TCK, real sbx e2e, and manual inference tests recorded.
- [ ] Production credential scope approved; rollback state retained.
- [ ] README and published references describe tested, implemented behavior.

## References

- [Dynamic secret sources](https://docs.docker.com/ai/sandboxes/configuration/credentials/#use-a-dynamic-secret-source)
- [sbx secret set](https://docs.docker.com/reference/cli/sbx/secret/set/)
- [sbx secret rm](https://docs.docker.com/reference/cli/sbx/secret/rm/)
- [sbx create](https://docs.docker.com/reference/cli/sbx/create/)
- [sbx run](https://docs.docker.com/reference/cli/sbx/run/)
- [sbx stop](https://docs.docker.com/reference/cli/sbx/stop/)
- [Repository credential schema](../spec/SPEC-v2.md)
- [Kit testing guide](../.agents/skills/kit-author/topics/testing.md)
- [Kit distribution guide](../.agents/skills/kit-author/topics/distribution.md)
- Pi baseline references: the installed package's `docs/cli.md`,
  `docs/providers.md`, `docs/settings.md`, and `docs/environment-variables.md`;
  `dist/cli/credential-print.js`; pi-ai's
  `dist/auth/oauth/openai-chatgpt.js` and `dist/api/openai-responses.js`.
- Extension baseline references: the pinned pi-web-access README and
  `openai-search.ts` (modern credential selection, routing, and proxy behavior).
