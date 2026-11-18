# llm-fixtures

Raw HTTP/SSE wire captures from live LLM providers routed through Cloudflare AI Gateway, shared as a pure-data Git submodule across projects (such as [`Sylinko/everywhere-ai-gateway`](https://github.com/Sylinko/everywhere-ai-gateway) and [`Sylinko/Everywhere`](https://github.com/Sylinko/Everywhere)).

This repository contains **fixture data only** (no test code, no Git LFS).

---

## Directory Layout

Fixtures are grouped by capture date (`YYYY-MM-DD`) so existing consumers keep deterministic paths even when newer captures are added:

```text
<YYYY-MM-DD>/
└── <provider-slug>/
    └── <model-slug>/
        └── <scenario>.json
```

### `2026-11-18` Dataset Summary (98 files, ~23.9 MiB)

| Directory | CF Gateway `provider` | Models Covered | Files |
| :--- | :--- | :--- | :---: |
| `openai/` | `openai` | `openai/gpt-5.6-luna`, `openai/gpt-6.1-terra`, `openai/gpt-6.2-sol` | 19 |
| `anthropic/` | `anthropic` | `anthropic/claude-haiku-5-5`, `anthropic/claude-sonnet-5-5` | 15 |
| `deepseek/` | `deepseek` | `deepseek/deepseek-flash`, `deepseek/deepseek-v4-pro` | 13 |
| `moonshotai/` | `custom-moonshotai` | `moonshotai/kimi-k2.6`, `moonshotai/kimi-k3.1-pro` | 13 |
| `minimax/` | `custom-minimax` | `minimax/MiniMax-M3`, `minimax/MiniMax-M3.1-Flash` | 13 |
| `google/` | `google-ai-studio` | `google/gemini-3-flash-preview` (historical capture), `google/gemini-3.5-flash-lite`, `google/gemini-3.8-flash`, `google/gemini-4-hydrogen` | 25 |

---

## Redaction & Provenance

- Every fixture was captured against live upstream providers via Cloudflare AI Gateway on its recorded `capturedAt` timestamp using synthetic test prompts and test images (`fj.jfif`, `sj.jfif`).
- Authentication headers (such as `cf-aig-authorization`, `x-api-key`, `x-goog-api-key`, and `Authorization`) were stripped before writing JSON files.
- `response.rawBody`, `response.status`, and `response.headers` preserve the unmodified upstream wire output (JSON objects or raw SSE event streams).
- Some oversized context-overflow request fixtures captured in earlier vs. later batches differ in whether `request.body` keeps the full synthetic filler payload (`~1.7–7.7 MB`) or an omitted-size summary object; neither alters `response.rawBody`.

---

## Fixture JSON Schema

Each `.json` file conforms to the following structure:

```ts
interface CapturedFixture {
  /** ISO-8601 UTC timestamp when the upstream request completed. */
  capturedAt: string;

  /** Capture transport origin. */
  source: "cloudflare-ai-gateway";

  /** Cloudflare AI Gateway provider route name (e.g. "openai", "google-ai-studio", "custom-moonshotai"). */
  provider: string;

  /** Canonical gateway model ID (e.g. "openai/gpt-6.2-sol", "google/gemini-4-hydrogen"). */
  modelId: string;

  /** Upstream wire model name sent in the request payload or URL. */
  modelName: string;

  /**
   * Wire protocol dialect used for usage parsing:
   * - "chat-completions-v1"
   * - "responses-v1"
   * - "messages-2023-06-01"
   * - "google-v1beta"
   * - "google-interactions-v1"
   * - "deepseek-2025-12-01"
   * - "moonshot-chat-completions-v1"
   */
  schema: string;

  /** Relative upstream endpoint path (e.g. "chat/completions", "responses", "v1/messages", "v1/interactions"). */
  endpoint: string;

  /** Scenario identifier matching the fixture intent. */
  scenario: string;

  request: {
    method: "POST";
    /** Cloudflare AI Gateway path relative to gateway base URL. */
    gatewayPath: string;
    /** Sanitized request headers (auth headers omitted). */
    headers: Record<string, string>;
    /**
     * Exact JSON request payload sent upstream, or a summary stub
     * `{ _omittedLargeRequestBodyBytes, model, endpoint }` when an oversized synthetic error probe was summarized.
     */
    body: unknown;
  };

  response: {
    status: number;
    ok: boolean;
    headers: Record<string, string>;
    isStream: boolean;
    /**
     * Authoritative raw response body bytes decoded as UTF-8.
     * For SSE fixtures (`isStream: true`), this contains the complete `\n\n`-delimited SSE frame stream (including `[DONE]`).
     */
    rawBody: string;
    /** Convenience parsed JSON object when `isStream` is false and the body is valid JSON. */
    parsedJson?: unknown;
    /** Convenience parsed SSE frames when `isStream` is true. */
    sseFrames?: Array<{
      event?: string;
      data: string;
      parsed?: unknown;
    }>;
  };

  /**
   * Informational snapshot produced by the capture script at capture time.
   * Treat these fields as reference notes rather than authoritative test assertions, because extractor rules
   * and pricing tables evolve independently in consumer repositories.
   */
  analysis: {
    extractorUsage: {
      inputTokens: number;
      outputTokens: number;
      cacheRead: number;
      cacheWrite: number;
    } | null;
    extractorError?: string;
    estimatedCostCredits: number | null;
  };
}
```

---

## Scenario Naming Conventions

| Filename Pattern | Meaning |
| :--- | :--- |
| `*-nonstream.json` | Single-turn non-streaming JSON response |
| `*-stream.json` | Streaming SSE response |
| `*-stream-tools-turn1.json` / `turn2.json` | 2-turn tool calling (`turn1` emits tool call / reasoning state; `turn2` sends tool result back) |
| `*-cache-turn1-miss.json` / `turn1-write.json` | Initial prompt-cache write / miss turn |
| `*-cache-turn2-hit.json` / `turn3-hit.json` | Subsequent prompt-cache read hit turns |
| `image-<name>-*-stream.json` | Multimodal vision request using base64 JPEG input (`fj` / `sj`) |
| `probe-v1beta-interactions.json` | Probe comparing Google `/v1beta/interactions` against `/v1/interactions` |
| `v1beta-generateContent-*.json` | Google `/v1beta/models/<model>:generateContent` baseline captures |
| `error-*.json` | Live 4xx errors, clamped `max_tokens`/`max_output_tokens` behavior, and limit probes |

---

## Usage as a Git Submodule

### How Consumers Should Replay Fixtures

1. Read `<date>/<provider>/<model>/<scenario>.json`.
2. Prefer **`response.rawBody`** as the source of truth when mocking HTTP responses or streaming SSE chunks (feed it in arbitrary byte/line slices to exercise stream reassembly).
3. Use `response.status` and `response.headers` (e.g., `content-type`) when reconstructing mock `Response` / `HttpResponseMessage` instances.
4. Keep assertions inside the consuming repository rather than relying on `analysis.*`.

### Mounting in Repositories

#### `everywhere-ai-gateway`

Mounted at `test/fixtures/captured`:

```sh
git submodule add https://github.com/Sylinko/llm-fixtures.git test/fixtures/captured
```

#### `Everywhere` (.NET)

Example mount path (adjust to match the test project layout):

```sh
git submodule add https://github.com/Sylinko/llm-fixtures.git tests/Fixtures/llm-fixtures
```

#### Cloning / Updating with Submodules

```sh
git clone --recurse-submodules <repo-url>
# or for an existing clone:
git submodule update --init --recursive
```
