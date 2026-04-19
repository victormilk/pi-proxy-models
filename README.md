# pi-proxy-models

A [pi-coding-agent](https://github.com/badlogic/pi-mono) extension that exposes
[CLIProxyAPIPlus](https://github.com/router-for-me/CLIProxyAPIPlus) models to
`pi`'s model picker and routes each model family through its native streaming
API (Anthropic Messages, OpenAI Chat Completions, or Google Generative AI).

That means you can `/login` to Claude Code, Gemini CLI, OpenAI Codex, GitHub
Copilot, Kiro, GLM, etc. inside CLIProxyAPIPlus once, and then consume all of
those subscriptions from `pi` with their native features intact — prompt
caching for Claude, thinking for Gemini, and so on.

## Why three providers?

`pi.registerProvider()` forces a single `baseUrl` per provider, but the
Anthropic, OpenAI, and Google SDKs expect different base paths (`/`, `/v1`,
`/v1beta`). The extension therefore partitions the discovered models across up
to three providers:

| Provider          | Family             | pi API              | base path          |
| ----------------- | ------------------ | ------------------- | ------------------ |
| `cliproxy`        | Claude / Anthropic | `anthropic-messages`  | `<url>`            |
| `cliproxy-openai` | OpenAI / Codex / Copilot / Kiro / GLM / Qwen … | `openai-completions`  | `<url>/v1`         |
| `cliproxy-gemini` | Google / Gemini    | `google-generative-ai`| `<url>/v1beta`     |

Providers with no matching models are **not** registered. If you only run
Claude accounts through CLIProxy, you only get `cliproxy/…` models.

## Install

> Requires a running CLIProxyAPIPlus instance. See
> [the upstream README](https://github.com/router-for-me/CLIProxyAPIPlus) for
> Docker/`docker-compose` setup.

Drop the single file into pi's global extension directory:

```bash
# from this repo
mkdir -p ~/.pi/agent/extensions/cliproxy
ln -sf "$(pwd)/index.ts" ~/.pi/agent/extensions/cliproxy/index.ts
```

or copy instead of symlinking if you prefer:

```bash
mkdir -p ~/.pi/agent/extensions/cliproxy
cp index.ts ~/.pi/agent/extensions/cliproxy/index.ts
```

For quick one-shot testing without installing:

```bash
pi -e ./index.ts
```

## Configure

The extension reads its config in this order (first match wins):

1. Environment variables `CLIPROXY_URL` and `CLIPROXY_API_KEY`
2. `~/.pi/agent/cliproxy.json`:
   ```json
   {
     "baseUrl": "http://localhost:8317",
     "apiKey": "your-api-key"
   }
   ```
3. Default: `baseUrl = http://localhost:8317`, no API key

A missing/empty API key is **tolerated** — the extension passes a placeholder
downstream. CLIProxyAPIPlus accepts any value when its own `api-keys:` list is
empty. When `api-keys:` is populated, set `CLIPROXY_API_KEY` to one of those
values.

Examples:

```bash
# Env-based (remote proxy with auth)
export CLIPROXY_URL=https://my-proxy.example.com
export CLIPROXY_API_KEY=abc123
pi

# File-based (persistent local config)
cat > ~/.pi/agent/cliproxy.json <<EOF
{ "baseUrl": "http://localhost:8317", "apiKey": "dev-key" }
EOF
pi
```

## Usage

Start `pi` and pick a model with `Ctrl+P` or `/model`:

```
cliproxy/claude-sonnet-4-5
cliproxy/claude-opus-4-5
cliproxy-gemini/gemini-2.5-pro
cliproxy-openai/gpt-5-codex
...
```

Or via flag:

```bash
pi --provider cliproxy --model claude-sonnet-4-5
pi --provider cliproxy-gemini --model gemini-2.5-pro
pi --provider cliproxy-openai --model gpt-4o
```

### Slash commands

| Command             | Description                                              |
| ------------------- | -------------------------------------------------------- |
| `/cliproxy-status`  | Ping the proxy, show model count + auth info             |
| `/cliproxy-models`  | List all discovered models grouped by `owned_by`         |
| `/cliproxy-refresh` | Re-fetch the model list and re-register all providers    |

### Listing models from the CLI

```bash
pi --list-models cliproxy         # Claude-family models
pi --list-models cliproxy-gemini  # Gemini-family models
pi --list-models cliproxy-openai  # everything else
```

## Behaviour notes

- **Model metadata** (`contextWindow`, `maxTokens`, `reasoning`, image input)
  is inferred from the model ID; costs are set to `0` because upstream accounts
  are paid via subscription, not tokens.
- **Startup resilience** — if the proxy is unreachable at launch, the
  extension still loads with a small static fallback list and warns the user.
  Run `/cliproxy-refresh` once the proxy is back online.
- **No Bearer header is added by pi** — each native SDK sends its own auth
  (Anthropic `x-api-key`, OpenAI `Authorization: Bearer`, Google
  `x-goog-api-key`) using the configured key.

## Troubleshooting

**`CLIProxy unreachable`** — verify the proxy is listening:
```bash
curl -s http://localhost:8317/v1/models | jq '.data | length'
```

**`302 Found` / `unauthorized` from Gemini or OpenAI** — CLIProxyAPIPlus is
forwarding to the upstream API with an unauthenticated token. Check that you
have an account linked for that provider in your proxy's `auths/` directory,
or set a valid key that matches the proxy's `api-keys:` list.

**Models don't appear after starting CLIProxy** — run `/cliproxy-refresh` in a
running pi session, or restart pi.

## License

MIT
