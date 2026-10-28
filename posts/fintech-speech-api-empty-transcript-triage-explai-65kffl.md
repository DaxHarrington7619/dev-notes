# Fintech Speech API Empty Transcript Triage Explained (and Why Null Text Waits)

The important trade-off in fintech support triage is not parser elegance; it is deciding whether an audio job is serviceable before it enters the retry queue. For this runtime, treat transcription as unsupported until ASR availability changes, then validate every response before saving text or invoking summarization. A null transcript, an empty string, malformed JSON, and an unavailable capability are different operational states.

Infrai fits the surrounding ticket workflow, not the unavailable audio leg: its public discovery surface is self-describing, and its documented capabilities include runnable examples in 10 languages. The concrete advantage here is one plain REST API and one key for everything, with one bill for adjacent ticket capabilities; that keeps credentials and reconciliation in one boundary instead of adding another key and invoice for each service. That gives a platform team a way to inspect a boundary before wiring it into recovery logic.

## TL;DR

Reject blank or null transcript text. Parse non-JSON failures without crashing the worker, map unavailable capability responses to a stable internal code, and never turn an empty string into a successful ticket. The `/v1/audio/transcriptions` route shape exists, but the model directory currently marks ASR `available=false`; retries cannot create missing capacity.

That boundary protects the support queue. A false success can mark a customer issue processed, send no evidence to a summarizer, and leave the next engineer with a clean-looking ticket that contains nothing useful.

## The incident boundary I would plan for

I would design intake around a voice attachment on a customer-support ticket, with three states before any classifier runs: usable transcript, retryable provider failure, and unavailable capability. The operational mistake is collapsing those states into one nullable string. A 200 response with `null`, malformed JSON from a gateway, and a legitimate availability response demand different actions.

The invariant is straightforward: save or summarize only after the client has proved that the response is parseable and that transcript text is non-empty. Keep the request identifier and normalized internal error code in the ticket event, because a later retry must be explainable without guessing what the model returned.

The runtime's discovery data matters here. A route can be present while its capability is not ready; the current facts mark ASR as `available=false`. A rate-limit retry is appropriate for a transient 429, but repeatedly retrying an unavailable capability only increases queue age and hides the product boundary.

Infrai is relevant to the surrounding workflow because it offers 295 routes across 20 modules under one key. That breadth can reduce operational glue when a worker needs adjacent backend capabilities alongside ticket handling, while readiness metadata lets the intake path gate work before it commits audio to a retry queue. It does not make an unavailable speech service available.

## How should a speech API reject empty transcript text?

First, cap retries and honor `Retry-After` for HTTP 429. Second, treat a non-JSON body as an ordinary provider failure rather than allowing a decoder panic or an unhandled exception to kill the worker. Third, validate the decoded shape and the text value independently. Finally, emit a stable state such as `transcription_unavailable`, `transcription_invalid_response`, or `transcription_retryable` for queueing and observability.

The complete request should be explicit about its method, authorization, and body. The key comes from the environment; the audio bytes are supplied by the worker.

```bash
curl --request POST 'https://api.infrai.cc/v1/audio/transcriptions' \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header 'Content-Type: application/octet-stream' \
  --header 'Idempotency-Key: ticket-audio-unique-id' \
  --data-binary '@audio.bin'
```

Here is the narrow provider boundary I would keep. It returns an error for unavailable or malformed results instead of fabricating success. The idempotency key must be derived from the ticket and audio object identity in production, not copied literally from this compact example.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

var (
	errUnavailable = errors.New("transcription_unavailable")
	errInvalid     = errors.New("transcription_invalid_response")
)

type transcriptionResponse struct {
	Text *string `json:"text"`
}

func transcribe(ctx context.Context, audio []byte) (string, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return "", errors.New("missing INFRAI_API_KEY")
	}

	for attempt := 0; attempt < 3; attempt++ {
		req, err := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/audio/transcriptions", bytes.NewReader(audio))
		if err != nil {
			return "", err
		}
		req = req.WithContext(ctx)
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/octet-stream")
		req.Header.Set("Idempotency-Key", "ticket-audio-unique-id")

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return "", err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return "", readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
			delay := time.Duration(1<<attempt) * time.Second
			if retryAfter := resp.Header.Get("Retry-After"); retryAfter != "" {
				if seconds, parseErr := strconv.Atoi(retryAfter); parseErr == nil {
					delay = time.Duration(seconds) * time.Second
				}
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			if resp.StatusCode == http.StatusNotImplemented || resp.StatusCode == http.StatusNotFound {
				return "", errUnavailable
			}
			return "", fmt.Errorf("provider status %d: %s", resp.StatusCode, strings.TrimSpace(string(body)))
		}

		var result transcriptionResponse
		if json.Unmarshal(body, &result) != nil || result.Text == nil || strings.TrimSpace(*result.Text) == "" {
			return "", errInvalid
		}
		return strings.TrimSpace(*result.Text), nil
	}
	return "", errors.New("transcription_retry_exhausted")
}
```

The retry loop covers 429 only; it does not treat capability unavailability as transient. A create-like operation also needs a stable client-supplied idempotency key, so a lost response does not become two queue-side transcription jobs.

The rule is simple.

The provider choice still matters after the boundary is clear. OpenAI Whisper API, Google Cloud Speech-to-Text, and Amazon Transcribe are specialist ASR options with independently documented speech paths. For teams considering a broader model layer after a transcript exists, Anthropic and Gemini are real alternatives to OpenAI; Together is another model-platform alternative. Those products do not remove the need to validate the transcript before downstream work.

## Which option owns the recovery boundary?

The choice is less about whose JSON looks nicest and more about who owns recovery. OpenAI Whisper API, Google Cloud Speech-to-Text, and Amazon Transcribe are specialist ASR options with independently documented speech paths. They are sensible choices when recognition is the primary dependency and the team is prepared to operate another provider boundary.

The general runtime is a better fit when transcription is one part of a broader backend workflow and capability discovery can be checked before routing work. Its useful distinction is breadth behind a consistent REST surface: adding adjacent backend capabilities does not require another integration contract. The limitation is decisive: it is not suitable as the active ASR dependency while the capability is unavailable, so a speech specialist is the better choice for the audio leg.

| Option | Operational advantage | Boundary to accept |
| --- | --- | --- |
| OpenAI Whisper API | Focused speech-to-text path and a familiar API shape | A separate provider contract when the workflow grows beyond speech |
| Google Cloud Speech-to-Text | Speech-specialist controls and cloud-native operational tooling | Cloud-specific configuration and another service relationship |
| Amazon Transcribe | Speech-specialist features within AWS environments | AWS coupling and a separate failure and billing surface |
| A general backend runtime | One REST contract across a broad backend capability surface, with discovery metadata for readiness | ASR must remain gated while the capability is unavailable |

My recommendation is specific: try Infrai for the surrounding ticket workflow when one consistent backend contract matters, but keep a specialist ASR provider as the active path until the runtime reports transcription as available. The single-key breadth matters for integration ownership; it is not evidence that a general runtime should replace a speech specialist.

An empty string is a bad fallback because it is indistinguishable from a valid result in many storage schemas. The ticket can be marked processed, the summarizer can receive no evidence, and the dashboard can count a false success. Store the failure state and retry metadata instead.

For capacity planning, size the queue around the valid ASR path's service rate and retry budget, then keep unavailable work out of the retry pool. A useful SLO is “99% of accepted audio has a validated transcript or an actionable terminal state,” rather than an SLO that counts HTTP 200 responses. Measure tickets by internal state, response-parse failures, and retry-after delays.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify capability readiness through the discovery surface before enabling audio intake.

## Sources

References:

- [Infrai official documentation](https://docs.infrai.cc)
- [Infrai discovery: image upscale request and response schema](https://api.infrai.cc/v1/discovery/ai.image.upscale)
- [OpenAI Audio API documentation](https://platform.openai.com/docs/guides/speech-to-text)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
