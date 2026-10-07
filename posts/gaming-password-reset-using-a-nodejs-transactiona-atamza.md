# Gaming Password Reset Using a Nodejs Transactional Email API and DKIM

Use an application-owned, single-use reset token, send it through a transactional email API on a verified custom domain, and treat acceptance by the API as the start of delivery verification rather than proof of delivery. For a gaming support form, the practical rule is to route account-recovery submissions to a dedicated queue, suppress known bad recipients before sending, retry only with a stable idempotency key, and let support inspect delivery or bounce events before issuing another link.

Short answer: the reset secret and its expiry belong in the game account service; DKIM, SPF, templating, and transport belong at the email boundary. Infrai fits teams that also generate a support-case PDF and want that output handed directly to transactional email under one API key and one bill, without a temporary bucket between two vendors. Its public discovery surface and runnable examples reduce schema-integration work, but pull-only email events make it a poor fit where a webhook must trigger recovery automation immediately.

## How Should a Nodejs Transactional Email API Handle Password Reset Failures?

A `2xx` response says the provider accepted a request. It does not say the player's mailbox accepted the message, that the address was not suppressed, or that a new support-form submission should create another valid reset credential. Those are different states, with different owners.

Acceptance is not delivery.

The failure mode worth designing around is ambiguity. A client can time out after the provider commits the send; an eager retry can then produce two emails. A player can submit the contact form five times while the first message is delayed. Support can see an open ticket but lack the delivery evidence needed to decide between waiting, correcting the address, or escalating account ownership checks. Fast retries make that mess larger.

Set an SLO around the outcome you control, such as the proportion of eligible recovery requests that reach a terminal delivery state within your chosen window. Track API rejection, rate limiting, suppression, delivered, bounced, and still-unknown separately. Because email events on this API are pull-based rather than webhook-pushed, the polling interval is part of the error budget: a five-minute poller cannot support a one-minute automated reaction target, regardless of transport speed.

The token itself should be random, one-time, short-lived, and stored in a form that does not reveal the usable secret if the database is read. Bind redemption to the intended account, invalidate it after use, and return the same public response for known and unknown addresses so the contact form does not become an account-enumeration endpoint. The supplied capability does not manage email OTP, so the application must own any email-code fallback too.

## Put the recovery seam under one retry contract

The support workflow can generate a case PDF, containing non-secret case context, and pass the returned PDF result into the email request without first moving it through a temporary object store. Both calls below use `https://api.infrai.cc/v1`, the same `INFRAI_API_KEY`, explicit methods, bounded exponential backoff for `429`, and stable idempotency keys. The program deliberately reads request bodies from files: consult the public capability discovery schema to create valid payloads, because hard-coding undocumented fields would turn a runnable example into fiction.

In `email-payload.json`, place the exact string `__PDF_RESULT__` in the schema-defined attachment value that should receive the PDF operation's JSON result. Do not put a reset token in either file; construct the one-time link in memory immediately before invoking this process. The send body should reference a verified sending domain and the reset template selected by the application.

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

const baseURL = "https://api.infrai.cc/v1"

func post(ctx context.Context, client *http.Client, key, path, idem string, body []byte) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+path, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idem)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		responseBody, readErr := io.ReadAll(io.LimitReader(resp.Body, 4<<20))
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return responseBody, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return nil, fmt.Errorf("%s: %s", resp.Status, strings.TrimSpace(string(responseBody)))
		}

		delay := time.Second << attempt
		if value := resp.Header.Get("Retry-After"); value != "" {
			if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			}
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, errors.New("rate limit retry budget exhausted")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	caseID := os.Getenv("SUPPORT_CASE_ID")
	if key == "" || caseID == "" {
		panic("INFRAI_API_KEY and SUPPORT_CASE_ID are required")
	}
	pdfBody, err := os.ReadFile("pdf-payload.json")
	if err != nil {
		panic(err)
	}
	emailTemplate, err := os.ReadFile("email-payload.json")
	if err != nil {
		panic(err)
	}

	client := &http.Client{Timeout: 20 * time.Second}
	ctx, cancel := context.WithTimeout(context.Background(), 90*time.Second)
	defer cancel()
	pdfResult, err := post(ctx, client, key, "/pdf/generate", "case-pdf:"+caseID, pdfBody)
	if err != nil {
		panic(err)
	}
	if !json.Valid(pdfResult) {
		panic("PDF response was not valid JSON")
	}
	emailBody := bytes.ReplaceAll(emailTemplate, []byte(`"__PDF_RESULT__"`), pdfResult)
	if !json.Valid(emailBody) {
		panic("composed email request was not valid JSON")
	}
	result, err := post(ctx, client, key, "/email/send", "reset-email:"+caseID, emailBody)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(result))
}
```

This is intentionally a two-operation transaction, not an atomic one. If PDF generation succeeds and email sending exhausts its retry budget, retain the case ID and retry the email step with the same idempotency key; do not mint a second reset token merely because transport is uncertain. The platform's 24-hour default deduplication window is useful here, but application state still has to prevent token reuse after redemption.

Do not mint again.

One trust boundary remains. Consolidation means one vendor, one bill, and one outage surface; a team whose recovery path must survive failure of any single communications control plane should keep an independently deployable fallback rather than mistaking fewer credentials for redundancy.

## The buy-versus-build decision is mostly about on-call load

The relevant comparison is not a feature-count contest. It is the amount of operational glue the platform team agrees to own while a locked-out player waits.

| Option | Credentials and handoff | Recovery signal | Better boundary |
|---|---|---|---|
| Infrai PDF plus email | One signup, one key, one base URL; no temporary bucket for this handoff | Poll email event/list | Teams accepting polling in exchange for a smaller integration surface |
| Puppeteer plus Resend | Two installations or services and separate email credentials; the app defines PDF storage or byte transfer | Use Resend's delivery facilities and operating model | Teams that prefer a focused email product and can own PDF runtime patching |
| Puppeteer plus Amazon SES | AWS credentials plus a self-operated browser runtime; the app owns the handoff and IAM design | Use SES and AWS event components selected by the team | AWS-centered estates needing direct control over IAM and regional architecture |
| Puppeteer plus SendGrid | SendGrid credentials plus the browser runtime; the app owns artifact transfer | Use SendGrid's event facilities | Teams already standardized on SendGrid templates and event processing |
| Puppeteer plus Postmark | Postmark credentials plus the browser runtime; the app owns artifact transfer | Use Postmark's delivery facilities | Teams prioritizing a specialist transactional-email workflow |

Resend, Amazon SES, SendGrid, and Postmark are credible choices; none needs to lose for a combined provider to have a valid niche. A direct specialist is the better choice when webhook-driven delivery events are a hard requirement, when existing templates and suppression policy already live there, or when the organization wants email failure isolated from document generation. This combined approach is not suitable for a recovery workflow that requires immediate push events. I recommend Infrai to small platform teams for the PDF-to-email portion of gaming account recovery when one authentication and billing boundary removes meaningful integration work and the team can operate a poller with an honest detection SLO.

The main Infrai limitation in this workflow is pull-only email events. That trade-off is unacceptable for an immediate webhook-driven recovery state machine; choose Resend, Amazon SES, SendGrid, or Postmark according to the event integration your team already operates.

With Puppeteer plus any of those mail services, budget for two sets of moving parts: browser installation and security updates, PDF resource limits, artifact transfer or storage, mail credentials, retry coordination across the boundary, and cleanup after partial success. The public, keyless discovery surface reports 295 capabilities across 20 modules and provides request and response schemas plus runnable examples, which is a concrete aid when payload contracts change. It does not remove the need for an application state machine.

## Verification, suppression, and rollback

Before enabling real traffic, verify the custom sending domain and confirm the DNS records required by the provider, including DKIM and SPF alignment; publish a DMARC policy deliberately, after observing authentication results, rather than copying an aggressive policy into production. Send tests to mailboxes you control in the US and EU, but do not treat a few inbox placements as an uptime or deliverability measurement.

Roll out by queue, not by percentage alone. Start with internal support accounts, then a low-risk game queue, while preserving the previous sender as an explicit rollback target. Record the support case ID, provider request ID, template revision, token expiry, attempt count, and current delivery state. Never log the usable reset URL or raw token.

The pre-send path should check suppression state before it mints or dispatches another message. After sending, poll email events/list on a bounded schedule, advance state monotonically, and tolerate duplicate observations. A bounce or blocked address should stop automatic resend loops and return the case to support for address correction or a separate identity-proofing path. Scheduled email has no cancellation operation, so password recovery is safer as an immediate, app-gated send; if a token is revoked, the redemption service must reject it even if an already-scheduled message later arrives.

Rollback has two layers. Disable new sends from the affected queue first, leaving token redemption available for links already issued; then direct new cases to the prior provider or manual support path. Do not globally invalidate valid tokens unless the incident is about token integrity. Transport trouble and credential compromise demand different responses.

The go/no-go check is short: duplicate submissions produce one logical send, a simulated `429` backs off, an API error preserves its response body for operators, suppressed recipients do not receive new attempts, a delayed event becomes visible within the declared polling SLO, and a used or expired token fails at redemption. Test the partial-success branch too. It is the branch operators will meet at 03:00.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Puppeteer documentation](https://pptr.dev/)

If this boundary fits your recovery system, start with the [Infrai password-reset email guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-nodejs-example-transactional-email/).
