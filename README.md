# modelrelay

Open AI models are spread across providers with incompatible APIs and price lists. modelrelay is a small local bridge to the hosted Synap MCP service: you can run a chat completion on one of the open models Synap serves, browse the live catalogue with per-million-token prices, pull the full record for a single model, and estimate what a call will cost before you spend anything.

Nothing is stored or computed on your own machine. This package answers `tools/list` locally from the captured tool definitions and forwards every `tools/call` to the hosted service over a streamable-HTTP JSON-RPC session, returning whatever the service sends back unchanged.

## The tools

Three of the four tools are free catalogue and price reads. The fourth, `synap_chat_completion`, is the only one that costs money: it bills prompt and completion tokens to the Synap API key supplied with the call, against your own pre-existing Synap account. No tool buys credit, tops up a balance, starts a subscription, or creates an API key.

- **`synap_chat_completion`** — Sends a conversation to a model and returns the standard OpenAI `chat.completion` JSON: the reply sits in `choices[0].message.content` and token counts in `usage`. The call is not streamed and keeps no memory between calls, so send the full message history each time. The model id `synap-v1` lets Synap pick the cheapest model that can handle the request. `temperature` (0 to 2) and `max_tokens` are optional.
- **`synap_list_models`** — The live list of models Synap serves right now, each with its price per million tokens, context window and what it is good at. An optional `query` filters case-insensitively against model id, name and capability. Auto-routing ids such as `synap-v1` carry no fixed price; they bill at the price of the model they route to.
- **`synap_get_model`** — The complete catalogue record for one model: pricing, context and output limits, supported parameters, quantisation, data-retention policy and readiness. An unknown id comes back as an error naming the id.
- **`synap_estimate_cost`** — Works out what a chat completion would cost from token counts, using the live price list: prompt and completion dollars and the total. `completion_tokens` defaults to 0; use your `max_tokens` for a worst case. Auto-routing ids have no fixed price and return an error saying so.

## Install and run

As an MCP server via npm:

```bash
npx -y @reneturcios/modelrelay mcp
```

MCP client configuration:

```json
{
  "mcpServers": {
    "modelrelay": {
      "command": "npx",
      "args": ["-y", "@reneturcios/modelrelay", "mcp"]
    }
  }
}
```

As a container:

```bash
docker run -i ghcr.io/cloudcorprecords/modelrelay
docker run --rm ghcr.io/cloudcorprecords/modelrelay synap_get_model '{"model":"qwen3-coder-30b"}'
```

## CLI

One binary serves both the command line and the MCP server. Each tool is a subcommand taking a single JSON string with that tool's arguments:

```bash
modelrelay synap_list_models '{"query":"coder"}'
modelrelay synap_chat_completion '{"model":"synap-v1","messages":[{"role":"user","content":"hello"}]}'
modelrelay synap_get_model '{"model":"qwen3-coder-30b"}'
modelrelay synap_estimate_cost '{"model":"qwen3-coder-30b","prompt_tokens":500,"completion_tokens":200}'
modelrelay --version
```

Exit codes: `0` when the forwarded call succeeds, `1` when the upstream result is an error, and `2` for usage problems such as an unknown subcommand or a malformed JSON argument.

## Configuration

- `SYNAP_MCP_URL` — Overrides the upstream endpoint. The default is the pinned hosted endpoint, `https://pool.linkrra.com/mcp`. (The override variable is a reconstruction added by this rebuild.)
- `SYNAP_API_KEY` — When set, its value is sent to the upstream on every request as an `Authorization: Bearer <value>` header. When it is not set, no key header is sent at all, matching the keyless behaviour the tool surface was captured with. (The hosted service's registry record pins a secret `Authorization` header on its remote; the environment-variable forwarding is a reconstruction added by this rebuild.) Without a key, `synap_chat_completion` returns an error and nothing is spent; the three catalogue tools need no key.

Upstream failures — a JSON-RPC error from the service, a non-2xx response, an unreadable body, or a request that fails outright — are reported to the caller as an error result whose text starts with `modelrelay error:`.

## How forwarding works

The first forwarded call opens a session: an `initialize` request (protocol version `2025-03-26`), followed by a `notifications/initialized` notification, then the `tools/call` itself. If the service answers the initialize request with an `Mcp-Session-Id` header, that value is carried on every later request, and the open session is reused for the rest of the process. Replies are accepted either as plain JSON or as server-sent events. Successful results pass through verbatim, including any `isError` flag and any extra fields the service includes.

## Development

```bash
npm install
npm run build   # tsc -> dist/ (dist/cli.js is the bin)
npm test        # tsx --test test/*.test.ts — stubbed upstream, no network
npm run dev     # run the CLI from source
```

The test suite checks the served tool list against the captured surface tool by tool, exercises the session handshake and its reuse, covers both key-header branches, and verifies that arguments go upstream untouched and results come back untouched.

## License

MIT — see `LICENSE`. Copyright (c) 2026 Rene Turcios.
