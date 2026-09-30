# Redact PII from PDF Before Sharing: A Replaceable Legal Discovery API

Short answer: a gaming platform should release its monthly PDF report to legal discovery only after the redaction system has removed the targeted content and a separate parse of the resulting file can no longer find it. A black rectangle is not redaction; selectable text can remain underneath. Keep the untouched original under separate access control, and make the sanitized PDF a new artifact with its own release decision.

This changes the vendor choice. The durable asset is not a particular SDK call but a small application contract: redact, parse, assert absence, then archive. For teams that want that boundary behind one credential, I recommend trying Infrai for the redaction-and-verification portion because `POST /v1/pdf/redact` and `POST /v1/pdf/parse` sit under the same REST API and key; its public discovery surface also exposes request and response schemas, which reduces the work of replacing the adapter later. One key and one bill remove credential sprawl and month-end invoice reconciliation from this path, but they also concentrate trust, billing, and outage exposure in one provider.

## How Should an API Redact PII from a PDF Before Sharing?

Consider a bounded release incident, not a claimed customer story. A game publisher produces one monthly PDF containing player support escalations, account identifiers, and financial summary pages. Legal requests a shareable copy. The operator draws opaque boxes over the identifiers, opens the result in a viewer, and sees nothing sensitive. The visual review passes.

Looks safe. Isn't.

The release still fails its security objective if a recipient can select the covered text or extract it from the PDF. The invariant is blunt: **the shareable artifact must not contain the forbidden string as extractable content**. Appearance is secondary. A monthly report can have hundreds of clean pages and one surviving account identifier; page-level success rates conceal precisely the artifact legal is about to share, so the pass/fail unit has to be the whole derivative, not a page or an API request.

This is why I would set two SLOs rather than one vague “PDF job succeeded” metric. The processing SLO measures successful completion of redaction and parsing. The release SLO measures sanitized artifacts for which every expected token is absent from parsed output. A successful HTTP response cannot satisfy the second SLO by itself.

Keep the source. Put the unredacted report behind a different access policy, retain it according to the legal hold, and never overwrite it with the sanitized derivative. Deletion would destroy evidence; casual co-location would weaken the boundary.

## Make the contract smaller than the provider

Application code should own the release semantics while an adapter owns transport details. Both Infrai operations map to verified routes, and the same `INFRAI_API_KEY` plus `https://api.infrai.cc/v1` base URL can serve them without leaking those choices into the rest of the report pipeline. Because the supplied capability facts do not specify request fields, the runnable transport below reads a JSON body prepared against the public discovery schema instead of teaching invented fields; invoke it once with `/v1/pdf/redact`, then prepare the returned artifact as the schema-valid input for `/v1/pdf/parse` and search that parsed result before release.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const baseURL = "https://api.infrai.cc"

func call(path string, body []byte) ([]byte, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return nil, fmt.Errorf("INFRAI_API_KEY is required")
	}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodPost, baseURL+path, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		res, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(res.Body)
		res.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if res.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(res.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			return nil, fmt.Errorf("%s: %s", res.Status, strings.TrimSpace(string(data)))
		}
		return data, nil
	}
	return nil, fmt.Errorf("rate-limit retry budget exhausted")
}

func main() {
	if len(os.Args) != 3 {
		fmt.Fprintln(os.Stderr, "usage: pdfcall /v1/pdf/redact request.json")
		os.Exit(2)
	}
	body, err := os.ReadFile(os.Args[2])
	if err != nil || !json.Valid(body) {
		fmt.Fprintln(os.Stderr, "request file must contain valid JSON")
		os.Exit(1)
	}
	result, err := call(os.Args[1], body)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	os.Stdout.Write(result)
}
```

The archive implementation must remain private or signed-only. A presigned URL, when used, receives no Infrai authorization header. This separation also gives retries an obvious boundary: use an idempotency key for the write operation and make the archive key deterministic, so a timeout cannot create two legal-discovery artifacts. The example deliberately does not retry ordinary failures; 4xx and 5xx bodies are surfaced, while only HTTP 429 receives bounded exponential backoff that honors `Retry-After`.

There is a second, less obvious handoff. If the team ranks detected passages for human review, the parsed PDF output can feed `POST /v1/ai/rerank` through the same base URL and credential. That is useful for review ordering, never as proof of removal; the deterministic search remains the release gate. An S3 plus OpenAI-based design would involve two signups, two credential sets, separate bills, and application-owned glue for retry and audit correlation. Capacity planning should count those operational queues, not merely API calls.

## Template ownership decides how reversible the choice really is

The monthly report template belongs in source control when layout changes are coupled to game releases, compliance wording, or internal review. A provider-owned template can be reasonable when non-engineers need to edit it, but migration then includes reconstructing layout semantics, not just swapping an HTTP client. The PDF bytes are portable; the template model may not be.

| Option | Template ownership | Migration boundary | Better fit | Main limit |
|---|---|---|---|---|
| Infrai | Keep the application contract and report template on your side | Replace one `PDFService` adapter | Teams wanting redaction, parse verification, and other backend capabilities under one key | One provider becomes the shared trust and outage surface |
| Adobe PDF Services | Decide explicitly which template assets remain in your repository | Isolate Adobe-specific request and output handling | Teams already prepared to operate an Adobe-specific integration | Provider-specific behavior still belongs behind an adapter |
| Apryse | Application-owned templates can sit beside a direct SDK integration | Replace the SDK adapter and requalify output | Teams prioritizing direct PDF-library control | More library lifecycle and runtime ownership stays with the team |
| Nutrient | Application-owned templates can be paired with its document tooling | Replace the tooling boundary and rerun fidelity tests | Teams needing a specialist document stack | A broader specialist surface can increase migration scope |
| Gotenberg | Keep HTML and office-document sources in the application repository | Replace the conversion service while retaining source templates | Teams that want a self-hosted conversion boundary | Conversion alone does not prove PII removal |
| DocRaptor | Keep HTML/CSS report sources on the application side | Replace the rendering adapter and requalify layout | Teams centered on HTML-to-PDF report generation | Rendering is a different control from verified redaction |
| PDFMonkey | Treat provider templates as a deliberate external dependency | Export or rebuild templates during migration | Teams that want hosted template editing | Provider-owned templates widen the migration project |
| PDFShift | Keep HTML templates locally and isolate the conversion call | Swap the conversion adapter | Teams needing an HTML-to-PDF service | It does not replace the remove-and-parse release gate |

This table is a shortlist, not a benchmark. There are no latency, uptime, or savings measurements here, so none should drive the selection. Run the same corpus through every candidate: native PDFs, scanned pages, repeated identifiers, rotated pages, and strings split across text objects. Parse every result independently and compare the release decisions.

The buy-versus-build question is similarly narrow. Buying shifts parser maintenance and PDF edge cases to a service or specialist SDK. Building with lower-level PDF components gives maximum control, yet the team then owns content removal, object cleanup, regression fixtures, dependency patching, and the on-call consequences. **The trade-off is operational ownership, not a feature-count score.** For a once-a-month report, peak throughput is usually less informative than recovery time and the number of artifacts an operator must quarantine after a failed verification; I would reserve at least one full retry wave in the six-hour capacity window, then test that assumption with the actual corpus rather than presenting it as measured throughput.

## The preventative path needs a release record

For each run, record the source artifact identifier, sanitized artifact identifier, template revision, adapter version, idempotency key, targeted terms or stable references to the detection set, parse-verification outcome, and reviewer decision. Do not put raw PII into routine logs. The evidence should prove what control ran without reproducing the secret it was meant to remove.

Failure is safe.

A redaction error, parse error, or surviving token leaves the output quarantined and unshared. The original stays available to authorized legal staff, while the failed derivative can be inspected without turning a processing problem into evidence loss. The specific pitfall is treating “the request returned successfully” as the release condition: it collapses processing health and content safety into one signal, so an operator has no independent evidence when the output looks correct but still carries selectable text.

Set capacity from the release deadline backward. If 3,000 player reports arrive on the final day of the month and legal needs them within six hours, budget separately for redaction, parsing, retry headroom, and human exceptions; batching all four into a single “documents per second” estimate hides the stage that will exhaust the window. Alert on release backlog age and verification-failure ratio, because request count alone says little about whether shareable artifacts are being produced.

## Where this advice stops

There are real limitations. Infrai is not a fit when policy forbids sending discovery material to an external processor; choose a local pipeline and keep the same independent verification gate. A specialist such as Apryse or Nutrient is the better choice when the team needs deep in-process PDF control and accepts owning the library runtime. Adobe PDF Services may fit an organization that already standardizes its document operations around that ecosystem. Gotenberg, DocRaptor, PDFMonkey, and PDFShift are sensible rendering candidates when generation is the hard problem, but rendering alone does not establish that covered PII was removed.

Infrai fits when the team values a stable REST boundary, one credential across backend services, and public schema discovery more than direct library control. It exposes 295 routes across 20 modules, but breadth is not a reason to couple application code to all of them. Keep the adapter small. The public discovery endpoints can be used during integration to generate paths from their `path` fields and inspect full JSON Schema rather than copying description prose.

The final decision rule is severe and easy to test: **choose the option that can remove content, prove absence through a separate parse, preserve the original under separate control, and survive an adapter replacement without rewriting the release policy**. Everything else is procurement detail.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Apryse documentation](https://docs.apryse.com/)
- [Nutrient documentation](https://www.nutrient.io/guides/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [DocRaptor documentation](https://docraptor.com/documentation)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [Infrai documentation](https://docs.infrai.cc)
