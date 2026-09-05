# pi-dome

Don Hubi's standalone [Pi](../pi/) sandbox kit with **OpenAI ChatGPT/Codex
subscription login**, not Anthropic auth. Reuses upstream's rolling
`docker.io/sbx/pi-image:latest` (Pi, fd, ripgrep and Docker) and npm proxy
configuration. Keeps `pi-web-access` and `@juicesharp/rpiv-ask-user-question`.

## Usage and migration

From this checkout:

```console
sbx run --kit ./pi-dome/ pi-dome
```

Create a **new sandbox** with this kit alone; do not combine it with `./pi/`.
The old `pi-dome` mixin could add OpenAI network access but could not remove
the base kit's Anthropic credentials. In particular, upstream now writes an
Anthropic OAuth `auth.json` at startup, replacing locally saved provider logins.
Changing the kit of an existing sandbox does not remove its old bindings.
No changes to the host's Anthropic secrets are necessary.

If published through this repository's pipeline, the equivalent references are:

```console
sbx run --kit docker.io/sbx/pi-dome-kit:latest pi-dome
sbx run --kit 'git+https://github.com/docker/sbx-kits-contrib.git#dir=pi-dome' pi-dome
```

These references are only usable once this directory is published upstream;
use the local path for an unpublished customization.

## OpenAI subscription login

1. In Pi, run `/login` and select **OpenAI (ChatGPT Plus/Pro)** / Codex.
2. Prefer **Device code login (headless)** when offered by the installed Pi
   version. Open the displayed URL in your host browser and enter the code.
   Follow OpenAI's instructions if device-code access needs enabling.
3. Alternatively, use browser login: open Pi's authorization URL on the host.
   If the final `http://localhost:1455/auth/callback?...` page cannot connect,
   copy the **entire redirect URL** from the address bar into Pi's login prompt.
   Host localhost is not sandbox localhost; no callback port publishing is
   needed with this manual flow. Do not paste the URL into an agent conversation.
4. Use `/model` to select an `openai-codex` model available to your account.

The default provider is `openai-codex`, **not** `openai`. An `OPENAI_API_KEY`
or `sbx secret set openai` is not a ChatGPT subscription login. Ordinary OpenAI
API use (including extension web-search features that require it) needs separate
API credentials and billing; this kit does not provision them.

Pi saves tokens and account metadata in `~/.pi/agent/auth.json` and refreshes
expired tokens through `auth.openai.com`. This kit neither seeds nor rewrites
that file, so login is retained across restarts of the same sandbox. After
sandbox removal/recreation, log in again unless you separately preserve its
state. Use `/logout` to remove the local login; revoke access with OpenAI when
needed. Do not commit or print `auth.json`, or share refresh tokens between
concurrent sandboxes.

**Security trade-off:** this is native, in-sandbox OAuth, not proxy-managed
secret isolation. Real tokens are readable by Pi, the agent, and its extensions.
Only install packages and run workloads you trust. Unlike the upstream
Anthropic kit, this kit intentionally permits `/login` inside the sandbox.

## Network and settings

- `auth.openai.com`: authorization, device-code polling, token exchange/refresh.
- `chatgpt.com`: Codex subscription inference.
- `api.openai.com`: optional OpenAI API/web-search use.
- `registry.npmjs.org`: configured packages and runtime Pi package operations.

SSE is the default transport to avoid dependence on WebSocket proxy support.
The kit does not disable Pi's other built-in providers; it simply declares no
Anthropic credentials or Anthropic network exceptions.

The shipped `settings.json` is a home-file overlay, just as in the original
customization: restarting reapplies its defaults (including the package list).
It does **not** contain auth data. The image tracks upstream releases; see
[the Pi image documentation](../pi/README.md#building-and-publishing).

## Verification

```console
sbx kit validate ./pi-dome/
./scripts/test-kit.sh pi-dome
./scripts/test-kit-e2e.sh pi-dome
```

There is intentionally no automated TCK prompt configuration: a fresh sandbox
requires interactive subscription login, which the harness cannot supply.
After logging in, verify manually from a host terminal:

```console
sbx exec <sandbox-name> -- pi --provider openai-codex -p 'Reply with OK'
```

Repeat after stopping and starting the same sandbox to check login retention.
Never dump credentials for diagnostics. A 401 after a successful login may need
`/logout` followed by `/login`; for blocked requests inspect `sbx policy log`
rather than broadening the allowlist blindly.
