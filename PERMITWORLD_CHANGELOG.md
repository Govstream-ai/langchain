# PermitWorld fork changelog

Divergences this fork (`Govstream-ai/langchain`) carries on top of upstream
[`brainlid/langchain`](https://github.com/brainlid/langchain). Upstream's own
changes live in `CHANGELOG.md`; this file tracks **only what we added**, so that
after each upstream rebase we can tell "a behavior we changed" from "a behavior
they changed" without re-reading the full diff.

**How to maintain:** add an entry when you land a fork-only change; when you
rebase onto a new upstream tag, bump _Current base_ and drop any entry whose
change got upstreamed (delete the line, don't mark it — the file is the live
set of divergences, not a history).

**Current base:** upstream `v0.8.14` (`7d1d2b7`, prep for v0.8.14).

---

## Gemini file / PDF content — `ChatGoogleAI`

- Send `%ContentPart{type: :file}` (PDF, CSV, images) inline to Gemini via an
  `inline_data` `for_api/1` clause. Lets permitworld attach documents to Gemini
  requests, which upstream's `ChatGoogleAI` didn't support. (`51d377f`)
- Accept a **raw MIME string** in that `:file` clause — permitworld callers pass
  `attachment.mimetype` verbatim (`"application/pdf"`, `"text/csv"`,
  `"image/png"`) rather than a `:media` option. (`e527548`)
  - _Upstream disposition:_ permitworld-specific — couples to our
    `attachment.mimetype` convention; would need a normalizing layer before
    contributing upstream.
- Files: `lib/chat_models/chat_google_ai.ex`, `test/chat_models/chat_google_ai_test.exs`

## Gemini Vertex serialization — `ChatVertexAI`

- Wrap non-object tool results so `functionResponse.response` stays a JSON
  object — Vertex 400s on a top-level array (our GIS tools return them), which
  `ChatGoogleAI`'s object wrapper had masked. (`7e5a35c`)
- Add the missing `:file` `for_api/1` clause so inline documents (PDFs)
  serialize to Vertex's `inlineData`/`mimeType` instead of raising
  `FunctionClauseError`; mirrors `ChatGoogleAI`. (`130cce1`)
- Implement `cache/4` + `cached_content` for Gemini context caching;
  `LLMChain.cache/2` previously hit `:undef` on Vertex. POSTs to the
  `cachedContents` endpoint (derived from `:endpoint`, OAuth bearer);
  under-minimum payloads return `{:ok, :noop}` and run uncached.
  (`1386c08`, `eaab2cf`)
  - _Upstream disposition:_ the array-response fix is a genuine upstream defect
    (regression from brainlid/langchain#491) — contribute as-is. The `:file`
    clause and `cache/4` are the Vertex twins of `ChatGoogleAI` features; caching
    has no permitworld coupling, but the `:file` clause's raw-MIME-string
    acceptance carries the same `attachment.mimetype` coupling as the
    `ChatGoogleAI` entry above — normalize before contributing.
- Files: `lib/chat_models/chat_vertex_ai.ex`, `test/chat_models/chat_vertex_ai_test.exs`

## Gemini context caching — `ChatGoogleAI` + `LLMChain`

- `LLMChain.cache/1` → `ChatGoogleAI.cache/3`: creates a Gemini cached-content
  entry (POST to `cachedContents`), stores the handle in a new `cached_content`
  field, and blanks `messages`/`tools` on the chain so already-cached content
  isn't resent. Google's 400-on-too-small is swallowed as a `{:ok, :noop}` so
  under-minimum payloads pass through uncached. (`6f4be7c`)
- `:ttl` support: `LLMChain.cache/2` threads a `cache_opts` `:ttl` into the
  cache-create body. (`c0e9dce`)
  - _Upstream disposition:_ candidate to contribute upstream — no permitworld
    coupling; a general Gemini caching feature.
- Files: `lib/chains/llm_chain.ex`, `lib/chat_models/chat_google_ai.ex`, `lib/function.ex`

## OpenAI tool-message construction fix — `ChatOpenAI`

- Reorder the `for_api/2` clauses so a `role: :tool` message carrying
  `tool_results` with content is matched before the generic clause (previously a
  tool call constructed with content crashed / serialized wrong); carry a
  `cached_content` key through the request map. (`2636ae9`)
  - _Upstream disposition:_ candidate to contribute upstream — looks like a
    genuine upstream bug.
- Files: `lib/chat_models/chat_open_ai.ex`

---

_Not in this fork:_ Bedrock / SigV4 support in `ChatAnthropic` is **upstream**,
not ours — obtaining it natively is the reason for this rebase.
