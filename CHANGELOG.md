# Changelog

## 0.1.0 (2026-10-28)

- First release: a forwarding client for the hosted Synap MCP server. The four captured tools are answered locally for tools/list, and every tools/call is forwarded upstream over a streamable-HTTP JSON-RPC session (initialize, notifications/initialized, tools/call, honouring the Mcp-Session-Id header when the upstream issues one).
- Tools: `synap_chat_completion`, `synap_list_models`, `synap_get_model` and `synap_estimate_cost`. Three tools are free catalogue and price reads; `synap_chat_completion` bills prompt and completion tokens to the caller's own Synap API key. No tool buys credit, tops up a balance, subscribes, checks out or mints a key.
- Upstream results pass through verbatim, including the isError flag; upstream failures surface as `modelrelay error:` text results.
- One bin serves the CLI (one subcommand per tool, plus `--version`) and the stdio MCP server (`mcp` subcommand). The upstream endpoint can be overridden via `SYNAP_MCP_URL`; when `SYNAP_API_KEY` is set it is sent as an `Authorization: Bearer` header on every upstream request.
