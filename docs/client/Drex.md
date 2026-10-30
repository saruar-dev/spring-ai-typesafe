# Using Drex

[Drex](https://console.nace.ai/docs/drex) is nace.ai's family of hosted decision models. They
serve the same `POST /v1/systemone` wire protocol as Jev, at `https://console.nace.ai`, so
`TypeSafeClient` and everything built on it runs against Drex with a different base URL, key
and model name. `drex-max`, the most accurate of them, also reads images; this SDK sends text
only, see [what differs](#what-differs).

!!! warning "A different model, not a different endpoint"
    The protocol matches closely; the model is another one. The thresholds in this project's
    examples and tests were tuned on Jev, so check your criteria against the Drex model you
    pick before relying on its decisions. Drex also rejects some requests Jev accepts; see
    [what differs](#what-differs).

## Pick a model

Create a key on the [console](https://console.nace.ai) and keep it in `NACE_API_KEY`, the
variable the official docs use. Then name the model: Drex has no `jev-latest`, and
`drex-latest` points to `drex-v2.0`, not to `drex-max`.

| Model | Reads | Questions per request | Options per choice, levels per score |
|---|---|---|---|
| `drex-max` | text and images | 512 | 255 |
| `drex-max-v1.1` | text and images | 512 | 255 |
| `drex-v2.0`, also `drex-latest` | text | 512 | 255 |
| `drex-v1.5` | text | 512 | 255 |

`listModels()` returns the same list, with the aliases and retired ids after the versions.
Pricing is per input token; output tokens are not billed. See the official
[models](https://console.nace.ai/docs/drex/reference/models),
[limits](https://console.nace.ai/docs/drex/reference/limits) and
[pricing](https://console.nace.ai/docs/drex/reference/pricing) pages, and the
[Drex Max guide](https://console.nace.ai/docs/drex/guides/drex-max).

## Point the client at it

The client appends `/v1/systemone` and `/v1/models` to the base URL, so give it the host
only.

=== "Spring Boot"

    ```properties
    spring.ai.typesafe.base-url=https://console.nace.ai
    spring.ai.typesafe.api-key=${NACE_API_KEY}
    spring.ai.typesafe.model=drex-max
    spring.ai.typesafe.timeout=60s
    spring.ai.typesafe.retry.total-timeout=150s
    ```

    Drex gives the model up to 55 s on a heavy request, so the SDK's default 10 s per-attempt
    timeout can cut a large one short. The retry budget must stay above the timeout, or no
    retry can ever fit in it.

=== "Plain Java"

    ```java
    TypeSafeClient client = TypeSafeClient.builder()
            .baseUrl("https://console.nace.ai")
            .apiKey(System.getenv("NACE_API_KEY"))
            .defaultModel("drex-max")
            .timeout(Duration.ofSeconds(60))
            .retryPolicy(RetryPolicy.builder()
                    .totalTimeout(Duration.ofSeconds(150))
                    .build())
            .build();
    ```

    `RetryPolicy` keeps a 30 s budget by default, which a 60 s timeout already exceeds, so
    without the explicit policy no retry ever fits. The Spring Boot property above sets the
    same 150 s budget.

    Or leave all three out and export `TYPESAFE_BASE_URL=https://console.nace.ai`,
    `TYPESAFE_API_KEY` and `TYPESAFE_DEFAULT_MODEL=drex-max`, which is how the official
    [migration guide](https://console.nace.ai/docs/drex/guides/migrate-from-typesafe) points
    the TypeSafe SDKs at Drex.

## What differs

Verified against `https://console.nace.ai` with `drex-max` on 2026-10-26, and taken from the
official [questions](https://console.nace.ai/docs/drex/guides/questions),
[errors](https://console.nace.ai/docs/drex/reference/errors) and
[limits](https://console.nace.ai/docs/drex/reference/limits) pages:

| Area | Drex | Effect |
|---|---|---|
| `model` | must be a `drex-*` name; `jev-latest` is 422, with a message that lists the accepted names | Set `spring.ai.typesafe.model` or `defaultModel(...)`. The `TypeSafeModels` constants are rejected. |
| `instructions` | must be a string; a JSON object or array is 422 `must be a string` | Write instructions as text. The components that send structured instructions fail; see [below](#components-that-send-structured-instructions). |
| Criteria descriptions | a choice option or a noul `true` / `false` criterion must be a string or `null`; a score level must be a string | Use the `String` overloads of `option(...)`, `whenTrue(...)`, `whenFalse(...)` and `level(...)`, not the `Map` or `List` ones. |
| Score `legend` | an array of labels in level order, where Jev sends an object keyed by level | Read into the same `legend()` map, so `labelOf(level)` and `nearestLabel()` work as on Jev. |
| `GET /v1/models` | `{"models": [...]}` with `name`, `description`, `release_date` and `alias_for` | `listModels()` works. |
| Request id | sent as `x-typesafe-request-id`, and as `x-request-id` and `request_id` in the body | `requestId()` is set, on errors too. |
| Error body | `{"error": {"type": "...", "message": "...", "issues": [{"path": "...", "message": "..."}]}}` for every status | `errorType()` and `errorMessage()` are set. A 422's `issues` are the `validationErrors()`, with `path()` as Drex writes it and `type()` `null`. |
| Statuses | 402 (`insufficient_credit`, `payment_required`) is new; 529 `overloaded` carries `retry-after` and `retry-after-ms`; every validation failure is 422 | 402 is a plain `TypeSafeApiException` and is not retried. 529 is `TypeSafeOverloadedException`, retried within the budget on the computed backoff; this SDK honours a server delay (`retry-after-ms`) only for `TypeSafeRateLimitException`, so the 529's `retry-after` is not read. Don't rely on `TypeSafeBadRequestException` for a malformed request. |
| Limits | 512 questions per request and 255 options or levels per question on every model; 4,194,304-byte body; `drex-max` 262,144 input tokens per question; `drex-v2.0` 131,072-token state and 262,144-token row; `drex-v1.5` 131,072-token state and 139,264-token row | Over a limit is 422 with a message that names it. A `JevToolIndex` choice has one option per candidate, so more than 255 candidates in one search must be split across searches. |
| Timeout | the model is given up to 55 s under load | Set `timeout` to 60 s and keep the retry budget above it. |
| Rate limits | per account: 120 requests per minute and 8 in flight on the free tier, 600 and 16 when paid; 429 carries `retry-after-ms` | `JevBatchOptions.defaults()` runs four requests at a time, within either limit. |
| Images | `drex-max` and `drex-max-v1.1` read up to 8 images per request from a top-level `images` array; the other models reject a non-empty one | `SystemOneRequest` has no such field, so this SDK sends text only. |

Everything else, including the noul, choice and score answers, probabilities, confidence
and token usage, has the same shape as Jev's.

## Components that send structured instructions

Some of this project's components write `instructions` as a JSON object, to separate the
question from its focus. Drex refuses those with 422 `must be a string`. Checked by reading
what each component sends; only the request shapes above were sent to Drex.

| Component | Against Drex |
|---|---|
| `JevJudge`, `JevEvaluator`, `JevSelfRefineAdvisor` | String instructions and descriptions in your criteria are accepted. |
| `JevGuardrailAdvisor` | Its questions are strings; accepted. |
| `JevChatModel` | String instructions and descriptions in your questions are accepted. |
| `JevDocumentReranker` | The default question has structured instructions. Pass your own `Noul` with string instructions to `question(...)`. |
| `JevDocumentFilter` | Its `contains_prompt_injection` question has structured instructions and cannot be replaced, so every passage fails. |
| `JevToolIndex` | The applicability question asked alongside two or more candidates has structured instructions, so a search over more than one candidate fails. |

## Integration tests against Drex

The live ITs follow `TYPESAFE_BASE_URL`, but they pin `TypeSafeModels.JEV_LATEST` as the
model, which Drex rejects. So they don't run against Drex as they are.

## See Also

- [Using Ollama](Ollama.md) and [Using Laya](Laya.md): local System One servers
- [TypeSafeClient](TypeSafeClient.md): builder settings and environment variables
- [Spring Boot Starter](SpringBootStarter.md): properties
- [Errors and Retries](ErrorsAndRetries.md): the exception types above
- [Drex documentation](https://console.nace.ai/docs/drex), and its
  [Migrate from TypeSafe](https://console.nace.ai/docs/drex/guides/migrate-from-typesafe) guide
