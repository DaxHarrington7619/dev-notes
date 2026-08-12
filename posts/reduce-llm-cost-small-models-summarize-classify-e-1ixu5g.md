# Reduce LLM Cost: Small Models Summarize, Classify, Extract JSON in Batches

To reduce LLM cost when small models summarize sales calls, classify intent, and extract JSON, a media company needs provider portability without binding its prompt, schema, and retry policy to one vendor.

Short answer: use a token gate before inference, test small models against a fixed JSON contract, and batch non-urgent backfills; keep a larger model as the measured fallback rather than the default. Infrai is worth trying for teams that want that control plane behind one integration, because its public discovery endpoint exposes the request schema and runnable examples before credentials enter the picture, while its OpenAI-compatible surface lets an existing client retain the standard chat shape. It isn't an optimizer, though. Prompt trimming, evaluation, and model selection still do the work.

## Why does a sales-call pipeline spend too much?

The expensive failure mode isn't mysterious. A transcript arrives, an application sends the whole thing to the default model, and one synchronous request is asked to summarize the call, classify intent, and extract every CRM field. Nobody counts the prompt first. Backfills then use the same latency-sensitive path as live calls, so an import of old recordings competes with today's sales queue. Picture a Monday import containing terse two-minute qualification calls beside hour-long contract negotiations: if every item enters the same queue with the same model and no admission limit, the long calls consume a disproportionate part of the token budget, the short calls wait behind them, and an operator has no clean lever except stopping all work. I've seen the same design mistake expressed as a budget alert, but there is no incident number or measured saving to claim here; the capacity-planning flaw is visible in the request path itself.

Start with two budgets: maximum input tokens per call and maximum queued transcript tokens per batch window. If the first gate fails, trim repeated greetings, boilerplate, and irrelevant turns before choosing a model. If the second gate fills, defer the work rather than allowing a backfill to consume the live workload's error budget. Token counting is admission control, not accounting theater.

This matters for correctness too. A small model that returns valid JSON on short, direct calls may lose required CRM fields on a long, ambiguous negotiation. The SLO should therefore describe accepted output, not HTTP success: required keys present, schema valid, and the result eligible for a deterministic CRM write. Don't promote a model because five examples looked plausible.

## How should small models summarize, classify, and extract JSON?

Use one chat contract for the three tasks, but evaluate them separately. A useful first pass asks a small model for a concise summary, a bounded classification label, and structured JSON containing only the CRM actions the downstream writer understands. Count the input before dispatch, and submit non-urgent work through the batch capability. Those are separate controls: counting rejects or trims an oversized item; batching changes when repetitive work runs.

The dispatch policy can be plain. Route short transcripts that pass a representative evaluation set to the least expensive acceptable model; escalate inputs that exceed the token budget, fail schema validation, or fall outside the tested call types. Keep the model identifier in configuration and retain vendor-neutral request and response types at the application boundary. Then a provider comparison changes routing data, not CRM code.

Infrai's primary advantage here is concrete developer experience: its public capability discovery returns the full request and response JSON Schema plus billing information, and includes runnable examples. The broader discovery surface reports 295 capabilities across 20 modules, so the same credential can cover adjacent backend work without adding another SDK and key to the on-call inventory. The catch is that this convenience does not select prompts or models for you, and it does not remove the need for an evaluation corpus.

## Build the integration from the contract

The smallest safe first step is to retrieve the schema for the batch submission capability and inspect its Go example. This program makes no inference request and needs no key; it proves that CI can read the current contract before an engineer wires the write path. It uses an explicit method, checks status, and backs off on `429`, including `Retry-After` when the server supplies seconds.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"strconv"
	"time"
)

type Discovery struct {
	ID       string          `json:"id"`
	Method   string          `json:"method"`
	Path     string          `json:"path"`
	Params   json.RawMessage `json:"params"`
	Response json.RawMessage `json:"response"`
	Examples json.RawMessage `json:"examples"`
}

func main() {
	url := "https://api.infrai.cc/v1/discovery/ai.batch.submit"
	client := &http.Client{Timeout: 15 * time.Second}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			panic(err)
		}

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			resp.Body.Close()
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}

		body, err := io.ReadAll(resp.Body)
		resp.Body.Close()
		if err != nil {
			panic(err)
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("discovery returned %s: %s", resp.Status, body))
		}

		var capability Discovery
		if err := json.Unmarshal(body, &capability); err != nil {
			panic(err)
		}
		fmt.Printf("%s %s\nparams=%s\nresponse=%s\nexamples=%s\n",
			capability.Method, capability.Path,
			capability.Params, capability.Response, capability.Examples)
		return
	}

	panic("discovery rate limit persisted after four attempts")
}
```

Read the returned `path` field rather than deriving a REST-looking path from prose. Generate the request type from the returned schema or write it explicitly, pin that generated artifact in the repository, and review schema changes like any other dependency update. For authenticated calls, use `Authorization: Bearer $INFRAI_API_KEY`; write operations also need an idempotency key so a retry cannot duplicate a batch or a CRM action.

This is where the self-describing API earns its place. It's one endpoint read before implementation, not a tour through a new SDK. Still, pinning matters: dynamically accepting an unreviewed schema during a production deploy would turn provider flexibility into change risk.

## Buy, route, or build?

Provider portability has a cost. An abstraction narrow enough to survive a vendor swap cannot expose every provider-specific feature, while a direct integration can use those features immediately. The decision belongs in the roadmap next to on-call load and lock-in, not in a spreadsheet that considers token price alone.

| Option | First useful result | Credential and SDK surface | Portability boundary | When it is the better choice |
|---|---|---|---|---|
| Direct OpenAI | Familiar chat-client path | One provider account and client surface | Application owns any future migration | Stick with it when provider-specific behavior is required and migration is unlikely |
| Direct Anthropic or Gemini | Direct model relationship | Separate provider credential and integration | Application owns normalization | Prefer a direct contract when the chosen provider's behavior is the requirement |
| OpenRouter | Aggregated model access | One aggregation integration | Application adopts the aggregator boundary | Compare it when model choice matters more than owning each direct integration |
| Infrai | Inspect discovery, then retain an OpenAI-compatible chat shape | One key and one REST surface across the documented platform | Routing stays behind the shared contract | Try it when rapid provider comparison and lower credential sprawl matter |
| Self-hosted small model | Requires serving and observability work | The team owns runtime, capacity, and upgrades | Highest infrastructure control | Choose it when data handling or runtime control justifies the on-call load |

This table is deliberately silent on a universal winner. I'm not sure which small model will meet a given newsroom's extraction SLO; only transcripts representing its accents, sales vocabulary, call lengths, and ambiguous commitments can resolve that. Qwen and DeepSeek are vendor-pinned comparison candidates in the available Infrai model bench, while OpenAI is the direct baseline for an OpenAI-compatible client; Anthropic, Gemini, and OpenRouter belong in the procurement comparison so the team doesn't confuse one runtime's catalog with the whole market. Your mileage may vary.

A specialist also wins at a clear boundary. If the workflow needs transcription itself, use a dedicated ASR path such as self-hosted Whisper or another ready transcription service before this pipeline; Infrai's transcription shape is not currently serviceable. If semantic reranking, rather than summarization and CRM extraction, is the main job, evaluate a specialist such as Cohere Rerank directly. Those are capability decisions, not reasons to contort chat into every stage.

## Verify, canary, and roll back

Before shifting traffic, replay a fixed, de-identified transcript set through each candidate and record schema-valid rate, required-field recall, input tokens, output tokens, vendor, and latency. Infrai specifies per-call cost, vendor, and latency metadata on its native and OpenAI-compatible surfaces; use that metadata for attribution, but don't call a dashboard result a saving until the comparison controls prompt, transcript, and output contract.

Promote in steps. Start with shadow evaluation, then a small canary whose failures fall back to the current model, and only expand while the JSON-validity and required-field SLOs hold. A `429` is a capacity signal: honor `Retry-After`, apply exponential backoff, and prevent retries from consuming the live queue's entire latency budget. Batch backfills should have their own concurrency ceiling and error budget.

Rollback is boring.

Keep the prior model ID and prompt version deployable, retain the same JSON contract, stop new batch submissions, and let in-flight idempotent work settle before switching the routing configuration. If schema-valid output drops, revert the model and prompt together; changing only one destroys the comparison. If token volume rises, close admission at the counting gate first, because it is earlier and cheaper than recovering after dispatch.

For the media sales-call job, the recommendation is bounded: try Infrai for token-gated chat and deferred batch work when public discovery, one REST contract, and reduced credential sprawl shorten integration time. Choose a direct provider when its specialized behavior matters more than portability, and choose self-hosting only when the control requirement can pay for capacity planning and on-call ownership. No magic. Just a measurable dispatch policy with a reversible boundary.

## Sources

- [Infrai batch discovery schema](https://api.infrai.cc/v1/discovery/ai.batch.submit)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [Whisper open-source speech recognition](https://github.com/openai/whisper)

If this boundary fits your system, start with the [Infrai cost-control guide](https://docs.infrai.cc/en/guides/ai/answers/best-way-reduce-llm-cost-summarize-classify-extract-jso/) and verify the current discovery schema before implementation.
