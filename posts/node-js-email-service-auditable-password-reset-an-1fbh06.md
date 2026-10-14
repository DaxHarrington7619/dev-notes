# Node.js Email Service: Auditable Password Reset and Welcome Deliverability

TL;DR: For an e-commerce compliance notice, choose an API-first email service only after proving that a verified sending domain, suppression checks, message lookup, and bounce evidence form one auditable path. Amazon SES, Postmark, SendGrid, Mailgun, and Infrai all belong on the shortlist, but the deciding test is operational: can the team distinguish accepted, delivered, suppressed, and failed mail quickly enough to protect its delivery SLO? Infrai fits a team that wants one key and one bill across backend services and accepts polling for email feedback; it does not fit an SMTP-dependent design or a workflow that requires webhook-driven reaction.

The page arrives at 02:17: `compliance_notice_evidence_gap` has crossed its burn-rate threshold. On-call sees 43 notices whose application records say `accepted`, but whose delivery evidence has not advanced inside the agreed window. The page links to a cohort, not a vendor dashboard: sending domain, template revision, provider message identifier, first submission time, latest observed state, and suppression result. That context determines whether the response is to pause a campaign, repair evidence collection, or investigate a domain-level delivery problem.

The wrong page says only "email failures are high." It combines password resets, welcome mail, and legally significant notices, then leaves an engineer to reconstruct the blast radius while the clock runs. The useful page starts with the user obligation and works backward. A compliance stream cannot borrow the alert policy of a welcome campaign: one is evidence bound, the other is primarily a product communication, and the acceptable delays need not match. That trade-off should be visible in the ledger before it appears in a paging rule.

It pages late.

## Which email service protects password reset and welcome deliverability?

A compliance notice needs two objectives. The submission path answers whether the application handed a valid request to the provider. The evidence path answers whether subsequent delivery information was collected and attached to the business record. Treating provider acceptance as delivery makes the dashboard look healthy precisely when its evidence is weakest.

The earlier signal is a queue-age or state-age warning scoped to the compliance stream. It should fire while the evidence collector can still catch up, before the user-facing delivery SLO burns through its budget. Capacity planning starts with rates: peak notices per minute, polling interval, pages fetched per cycle, provider limits, and maximum tolerable evidence delay. If a collector can drain 600 updates per minute while a promotion creates 900 notices per minute, alert tuning cannot repair the deficit.

Polling changes the failure model. Infrai exposes email message and event information through polling rather than webhooks, enough for a basic admin panel or retry queue but limiting real-time orchestration. A five-minute collector cadence creates a five-minute observation floor even when the provider learns the outcome immediately. That delay belongs in the SLO and page threshold; hiding it inside an average produces noisy pages and misleading recovery times.

Suppression is a gate, not a postmortem field. Check it before submitting transactional mail, retain the result with the attempt, and never retry a known suppression as though it were a transient transport error. Password-reset and welcome messages benefit from the same verified-domain and template discipline, although their urgency and retry policies should remain separate from compliance notices.

## Instrument the obligation, not the SDK

The application should own a provider-neutral delivery ledger. At minimum it needs an immutable notice ID, recipient reference, purpose, template revision, sending domain, provider message ID, submission timestamp, latest state, latest evidence timestamp, attempt count, and idempotency key. Keep addresses out of metric labels; cardinality and personal data are both poor page inputs.

This runnable Go poller retrieves email events without assuming an undocumented response schema. It explicitly authenticates, checks every status, honors `Retry-After` on HTTP 429, and otherwise uses bounded exponential backoff. Set `EMAIL_API_BASE_URL` to the approved API base in deployment; keeping that operational value outside the source also prevents a copied example from silently targeting production. The adapter that parses the returned JSON should preserve the raw evidence before normalizing it into the ledger.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	baseURL := strings.TrimRight(os.Getenv("EMAIL_API_BASE_URL"), "/")
	if baseURL == "" {
		panic("EMAIL_API_BASE_URL is required")
	}

	client := &http.Client{Timeout: 15 * time.Second}
	backoff := time.Second
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet,
			baseURL+"/email/event/list", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}

		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Println(string(body))
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			panic(fmt.Sprintf("event poll failed: status=%d body=%s",
				resp.StatusCode, body))
		}

		wait := backoff
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
			wait = time.Duration(seconds) * time.Second
		}
		time.Sleep(wait)
		backoff *= 2
	}
	panic("event poll remained rate limited after 5 attempts")
}
```

Keep separate counters for submission rejection, suppression, evidence age, hard bounce, and temporary failure. One blended success rate destroys the diagnosis needed during a page. Store raw provider evidence alongside the normalized state, with access controls and a retention policy reviewed by compliance. Normalization makes alerts portable; the raw record makes an audit defensible.

Transport is bought. Accountability is not.

## Provider choice is an on-call decision

Feature checklists reward breadth. An SRE review should instead ask who operates each missing piece and how evidence leaves the provider boundary. This table is a procurement test plan, not a claim that every row has already passed. Require each vendor to demonstrate the checks against the actual account, region, and sending domain.

| Option | Operational reason to shortlist | Boundary to validate | Best fit |
|---|---|---|---|
| Amazon SES | Direct candidate for teams already operating in AWS | Domain setup, suppression behavior, event export, and the assembled on-call path | Teams willing to compose delivery evidence from AWS services |
| Postmark | Focused transactional-email candidate | Domain isolation, bounce evidence, retention, and API behavior with a real notice cohort | Teams preferring a dedicated transactional-mail product |
| SendGrid | Broad email API candidate | Suppression ownership, event evidence, template governance, and account-specific limits | Teams willing to govern a larger email surface |
| Mailgun | API-oriented email candidate | Event retrieval, suppression semantics, regional requirements, and domain controls | Teams comfortable owning an internal evidence adapter |
| Infrai | One REST API, one key, and one bill reduce credential and invoice sprawl | Polling feedback, no SMTP relay, and application-owned email OTP | Teams standardizing backend calls on HTTP that can budget for polling delay |

No vendor wins by name. Run a bake-off on a dedicated test domain using password-reset, welcome, and compliance templates; seed valid recipients, suppressed recipients, and controlled failures; then compare state completeness, evidence latency observed by the collector, retry safety, exportability, and operator effort. Do not publish a deliverability percentage from a tiny or unrepresentative cohort. Infrai's public discovery describes 295 routes across 20 modules, but breadth does not answer the delivery-evidence question; the email workflow still has to pass the same cohort test.

The buy-versus-build boundary matters. Buying transport still leaves the application responsible for the business ledger, notice identity, legal-purpose record, SLO, and audit access. Building SMTP transport expands the on-call surface into reputation, feedback processing, and relay operations. For an API-first Node.js service that does not require SMTP, buying transport and building the thin evidence layer is the defensible split.

## Boundaries that change the answer

The principal limitation is direct: Infrai is not suitable for this design if SMTP relay is mandatory, webhook delivery is a hard real-time requirement, or managed email OTP is part of the specification. Choose a provider that demonstrates those required controls instead. Email feedback here must be polled. Email OTP generation and validation remain application work, and scheduled email should not be designed around later cancellation because that control is unavailable. Domestic Tencent email support is pending, so it cannot serve as evidence for a China-specific compliance decision.

Those are architecture constraints. A password-reset flow also has a different security boundary from a welcome message: a mail API can carry the reset link, but token creation, expiry, single use, and account protections remain with the application. Likewise, a compliance delivery record proves what the system observed; it does not prove that a recipient read or legally acknowledged a notice. Legal counsel must define the required evidence.

A consolidated API reduces key sprawl and month-end invoice reconciliation, yet consolidation increases dependency concentration. Record provider identifiers and raw events behind an internal schema, rehearse export, and keep the adapter narrow. That is the lock-in control that matters during an incident.

## Tune the page without buying silence

Start the evidence-gap alert as a ticket or warning while collecting a representative baseline, then page on sustained budget consumption for the compliance stream. Use a minimum cohort so one delayed record does not wake someone during low volume, but retain a separate age ceiling so low traffic cannot hide one legally important notice forever. Multi-window burn-rate alerts are useful when the SLO and traffic volume support them; an absolute oldest-record alarm is clearer for sparse obligations.

Thresholds have a carrying cost. Set the collector-lag warning too close to its normal polling cadence and every routine cycle becomes an incident. Set it beyond the business deadline and the page becomes an obituary. Review page count, actionable-page ratio, missed obligations, and evidence-recovery time, split by stream and domain. Change one threshold at a time and retain the before-and-after alert history.

Noise compounds.

False positives consume the finite on-call attention needed for a real domain or suppression failure. Silence is worse. The balanced design pages on an obligation at risk, routes capacity warnings earlier, and keeps vendor telemetry as evidence rather than as the definition of success.

## References

- Amazon SES Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Postmark developer documentation: https://postmarkapp.com/developer
- SendGrid email API documentation: https://www.twilio.com/docs/sendgrid/api-reference
- Mailgun documentation: https://documentation.mailgun.com/docs/mailgun/
- Google SRE Workbook, alerting on SLOs: https://sre.google/workbook/alerting-on-slos/
- RFC 3463, enhanced mail system status codes: https://www.rfc-editor.org/rfc/rfc3463
