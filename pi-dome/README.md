# pi-dome

Don Hubi's standalone Pi sandbox kit, using **`openai` → Sign in with ChatGPT
on the host**. The sandbox requests the official OpenAI Responses API with
`OPENAI_API_KEY=proxy-managed`; the sandbox proxy supplies the current host-owned
OAuth access token. Neither refresh tokens nor host `auth.json` are mounted,
copied, or templated into the sandbox.

The upgrade was successfully tested on the host, as reported by the user.
Configuration is now deliberately simple: **seed missing files, preserve existing
files, and run no startup upgrade or migration task**. The kit does not register
host secrets. Upgrade closeout and rollback guidance are in [UPGRADE.md](UPGRADE.md).

## Baseline

| Component | Selection / verification |
| --- | --- |
| Image | `docker.io/sbx/pi-image@sha256:b278e3e25c3cae780efd9b5b4df047209a91cebafa45426c241356fa53c05cfd` |
| Runtime | Linux amd64: Pi `1.1.0`, Node `22.22.1`; `curl`, fd, rg and Docker present |
| Other architectures | OCI index contains arm64; its runtime was not verified in the recorded baseline |
| Coding default | `openai/gpt-6.1-sol`, present in the image's bundled catalog |
| Thinking / transport | `xhigh` / `sse` |
| Search default | OpenAI Responses, `gpt-6.1-sol`, `workflow: "none"` |
| Packages | `pi-web-access@0.39.0`, `@juicesharp/rpiv-ask-user-question@2.13.5` |

Pins are defaults for new settings, not an automatic update policy. Changing the
image, package versions, or model defaults is an explicit edit and re-test.
The host's `deviceId` and credentials are not distributed as kit defaults.

## Host authentication and usage

Run `sbx` and OAuth setup on the **host**, not inside a sandbox. Commands below
assume the repository root. Replace example sandbox names and absolute paths
with your own.

1. Authenticate host Pi in a dedicated, private agent directory with
   `/login openai`, selecting **Sign in with ChatGPT**. Keep its `auth.json` and
   corresponding `settings.json` together, with owner-only permissions.
2. Protect that directory, the Pi/Node executables and their dependencies, and
   an absolute token helper outside **all writable sandbox mounts**. The helper
   must use that agent directory and emit only the output of
   `pi auth print-bearer-token --provider openai --min-expiry 5m`.
3. Validate and create a new sandbox using this **standalone kit only**:

   ```console
   sbx kit validate ./pi-dome/
   sbx create --name pi-dome-new ./pi-dome/ "$PWD"
   ```

4. Review approval for credential injection into `api.openai.com`. If this new
   sandbox needs a scoped source, register the protected host helper:

   ```console
   sbx secret set openai --sandbox pi-dome-new \
     --command '/absolute/host/path/pi-dome-token' --refresh on-demand
   ```

   Replace the path first. An existing approved global source may already cover
   the sandbox; do not overwrite it or change its scope without reviewing other
   consumers. Do not substitute `sbx secret set openai --oauth`: that built-in
   flow was not established as compatible with modern Pi OAuth during review.

5. Check placeholder delivery and real inference, then attach:

   ```console
   sbx exec pi-dome-new -- sh -c 'test "$OPENAI_API_KEY" = proxy-managed'
   sbx exec pi-dome-new -- pi --no-extensions --no-tools --no-session \
     --model openai/gpt-6.1-sol -p 'Reply with exactly OK.'
   sbx run --name pi-dome-new
   ```

The credential is deliberately not `required` during creation, allowing a scoped
resolver to be registered afterwards. An unbound sandbox is **not usable** until
placeholder delivery, approved injection, and actual inference pass. A banner
or non-empty error message is not proof of inference.

`on-demand` resolves the host credential on each use; host Pi owns refresh and
persistence. Never put a real token into sandbox environment variables or copy
host auth files into it. A stored sandbox `openai` credential can override the
environment placeholder; review any such conflict privately. There is no startup
helper to detect or remove it.

Do not compose with `./pi/`, use `--kit ./pi-dome/ pi-dome`, or reuse a sandbox
carrying the old Anthropic bindings/overlays. Composition is additive and cannot
remove them. Use local references unless an exact upgrade revision has been
published; there is no assumed `pi-dome-kit:latest` artifact.

## Durable settings

`setup.files` seeds these files with mode `0600` **only when missing**:

- `/home/agent/.pi/agent/settings.json`: provider and model, thinking level, SSE,
  and the two pinned packages.
- `/home/agent/.pi/agent/web-search.json`: explicit OpenAI search routing and model,
  with `workflow: "none"`.

There is no active settings/auth overlay and no startup configuration task.
Existing files are not migrated, reconciled, backed up, or rewritten by the kit.
User-selected models, package versions, theme, `deviceId`, and other settings stay
as they are. The install hooks only prepare the private agent directory and npm's
proxy configuration; they do not upgrade Pi or its packages.

For an existing sandbox, adopt changed defaults manually after privately backing
up the relevant non-secret settings. Do not delete settings just to pick up new
kit defaults. Inspect project `.pi/settings.json` and `models.json` too: project
settings can override global defaults, resource lists are combined, and resumed
sessions can retain a different model.

Settings survive **stop/restart**, not sandbox removal. Preserve any required
non-secret settings/sessions before recreation. Host-owned auth survives sandbox
recreation; ensure each replacement has an approved credential binding.

## Web search and network

Fresh search configuration explicitly selects `provider: "openai"`,
`openaiSearchProviders: ["openai"]`, the chosen model, and `workflow: "none"`.
It uses the official Responses endpoint, not Codex alpha search. The default
workflow opens no curator browser and makes no extra summary model call.

**The kit no longer configures the extension's search proxy automatically.**
The npm proxy install hook is not extension transport configuration, and Node
fetch does not automatically honor environment proxies. Before using search,
configure the extension's `proxy` field in `web-search.json` with the current
sandbox `HTTPS_PROXY`/`HTTP_PROXY` URL, using a JSON-safe editor and preserving
intentional user settings. Recheck it if a recreated sandbox has a different
proxy URL. Do not set `proxy: ""` to force direct traffic: that bypasses credential
replacement. Enable `ssrf.trustEnvProxy` only after diagnosing proxy-only DNS;
it is not a substitute for proxy transport.

The allowlist includes:

- `api.openai.com`: model inference and hosted search; the only credential
  injection destination.
- `registry.npmjs.org`: pinned package installation, with npm proxy setup retained.
- `auth.openai.com` and `chatgpt.com`: retained login-related allowances; neither
  is needed by the default host-owned authentication path. Removing them is a
  separate network-policy change, not part of documentation cleanup.

Hosted search still depends on the selected account/model's entitlement and does
not grant arbitrary page-fetch egress. Approve target hosts narrowly and use
`sbx policy log <sandbox>` plus the block response for diagnosis. Do not broaden
the allowlist to work around an account/model entitlement error. Automatic
subscription detection is unreliable with an environment placeholder; explicit
search routing avoids automatic Exa/legacy-provider selection.

## Validation and maintenance

On the host, from the repository root:

```console
sbx kit validate ./pi-dome/
sbx kit inspect ./pi-dome/ --output json
```

When the full repository's test tooling is available:

```console
go test ./spec/... ./tck/...
./scripts/test-kit.sh pi-dome
```

Real inference, extension use, settings preservation through stop/restart, and
credential refresh must be checked on the host when changing the baseline.
Content-only tests do not certify authentication or search. Any e2e testing must
keep the separate `sbx-kits-contrib-tck` daemon's secrets and policy isolated from
production; do not reset production policy to get green tests.

The removed startup helper and its migration/lifecycle fixtures are not part of
the current kit. See [the closeout record](UPGRADE.md#completion-record) for the
reported upgrade outcome and the limits of checks run during documentation cleanup.
