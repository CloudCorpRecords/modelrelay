# BUILD REPORT — modelrelay

Clean-room rebuild of `com.linkrra/synap` v1.2.0 (remote-only MCP
server at `https://pool.linkrra.com/mcp`), built 2026-10-28 as a
forwarding client. Package: `@reneturcios/modelrelay` 0.1.0,
registry name `io.github.CloudCorpRecords/modelrelay`.

## Surface built

Full surface built: **4/4 tools**, no omissions.

| Tool | Title (as served) | Required fields as served |
|------|-------------------|-----------------------------|
| `synap_chat_completion` | Run a chat completion on Synap | `model`, `messages` |
| `synap_list_models` | List Synap models and prices | *(no `required` key in schema)* |
| `synap_get_model` | Get one Synap model in full | `model` |
| `synap_estimate_cost` | Estimate the cost of a Synap call | `model`, `prompt_tokens` |

The upstream `synap_` name prefix is part of the captured tool names
and is served verbatim. No tool's inputSchema carries a `$schema`
marker; those are served exactly as captured. Tool literals were
generated programmatically from
`harness/fixtures/synap-tools-original.json` by the script at
`/tmp/b41-modelrelay-gen.py` (parse + re-serialise, substituted into
`src/forward.ts`) — never retyped — and a post-generation check (the
registration-integrity test suite, plus the parity harness) confirmed
the served literals deep-equal the fixture's 4 tool objects in order.
Tools are registered via the MCP SDK low-level Server API
(`ListToolsRequestSchema` / `CallToolRequestSchema` handlers), not
McpServer+zod, per the task instruction.

## Fixture conflicts

**None.** The spec and the fixture agree on tool names, order, titles,
required fields (present on exactly the three tools listed above,
absent on `synap_list_models`), and descriptions.

## Labelled reconstructions (from the spec's Wire section)

- Upstream endpoint pinned: `https://pool.linkrra.com/mcp`.
- Env override `SYNAP_MCP_URL` — labelled reconstruction.
- Env forwarding `SYNAP_API_KEY` — labelled reconstruction (the registry
  record pins a secret `Authorization` header on its remote and the
  capture was taken keyless). When set, `Authorization: Bearer <value>`
  is sent on every upstream request. When unset, no key header is sent.
  Both branches are tested.
- Error rendering `modelrelay error:` text for upstream failures —
  labelled reconstruction.
- Upstream results pass through verbatim, including `isError` and extra
  fields. tools/call is forwarded as streamable-HTTP JSON-RPC:
  initialize (protocolVersion `2025-03-26`) → notifications/initialized →
  tools/call, honouring `Mcp-Session-Id` when issued; plain-JSON and SSE
  replies are both parsed. The forwarding core takes an injectable fetch;
  tests stub the upstream — no live network calls in tests.

## Deviations

**None.**

## Lockfile

- Route: **Route 3** — generated clean from birth in the build dir with
  no `node_modules` and no pre-existing lockfile:
  `npm install --package-lock-only --ignore-scripts --offline`.
- Parses as JSON; `name` == `@reneturcios/modelrelay`.
- Package entries: **128**.
- sha256: `02a0e3162cc20f4c49d4d65fd38550ce96909996b7ecb3286392128caa03bbeb`
  (re-verified unchanged after the subsequent offline `npm install`).

## server.json

- Description exactly: "Open AI models through one API: chat completions,
  live model catalogue, prices, cost estimates." — **95 characters**.
- npm package entry carries `"version": "0.1.0"`; the OCI entry
  (`ghcr.io/cloudcorprecords/modelrelay:0.1.0`) carries **no** `version`
  field. Dockerfile (node:20-alpine, both stages) carries the label
  `io.modelcontextprotocol.server.name="io.github.CloudCorpRecords/modelrelay"`.
  Container workflow `IMAGE` is `ghcr.io/cloudcorprecords/modelrelay` and
  its smoke test asserts the server name `modelrelay`.

## Verification (all run 2026-10-28)

1. `npx tsc --noEmit` → exit 0.
2. `npm run build` → exit 0; real `dist/` produced (cli, forward, index,
   mcp, types — .js, .d.ts, .js.map each).
3. `node dist/cli.js --version` → `0.1.0`; unknown subcommand → Usage on
   stderr, exit 2.
4. `npm test` → **57 tests, 57 pass, 0 fail** (registration-integrity
   suite deep-equals the served tools/list against the fixture's 4
   tools in order, plus session handling/reuse, env override, key-header
   present/absent/blank branches, verbatim argument forwarding and
   result pass-through including upstream isError, SSE parsing, error
   rendering, MCP-layer validation, and CLI behaviour).
5. Parity: `python3 harness/parity.py --original
   harness/fixtures/synap-tools-original.json --rebuild
   "stdio:node /home/hatch/workspace/mcp-rebuild-100/builds/modelrelay/dist/cli.js mcp"
   --out builds/modelrelay/parity-out` → **verdict PASS, parity 100.0**
   (4/4 identical, 0 missing, 0 extra, no schema diffs). Output in
   `parity-out/`.
