# Metered Billing 2026 for Gaming Spend Caps (Freeze Usage Before Issuing Invoices)

Meter from the platform's usage records, freeze an immutable monthly snapshot in storage you control, and issue each customer invoice from that snapshot. The deciding constraint for a gaming platform is attribution: a spend ceiling means little if the ledger cannot distinguish one game's background workload from another customer's interactive traffic.

TL;DR: use live usage to inform admission decisions, scheduled snapshots to preserve billing evidence, and reconciliation to explain differences. A monthly snapshot makes an invoice reproducible; enforcing a workload's cap requires a separate admission ledger that accounts for work already in flight.

For a Node.js application, keep that boundary outside page rendering and request handlers: opening the billing page must never be the event that closes a billing period. The implementation language doesn't change the accounting contract.

## What usage should metered billing architecture store before a game reaches its cap?

Start with the question a customer will ask: which workload caused this charge? Preserve the platform's usage records as source evidence, then maintain an application-owned mapping from that evidence to customer, game, workload, and billing period. Those names describe your accounting schema; they aren't assumed response fields from a provider. If the available evidence cannot support a split, hold the amount for reconciliation instead of inventing a precise allocation.

Consider an explicitly hypothetical planning fixture: customer A has a background asset-generation workload with a 100-unit monthly allowance, 82 units of settled consumption, and 12 units reserved for accepted work. Its available admission headroom is 6 units. A proposed job with a conservative upper bound of 8 units should wait or be rejected under that policy, even though the last settled usage read appears to leave 18 units. These are internal budget units, not vendor prices or measured consumption. The trap is treating a settled counter as total liability: it omits accepted work, so the next reservation must account for both amounts before admission, and a later settlement must replace its reservation rather than add another copy of the same consumption. Keep the job identifier through retries and settlement, because losing that identity makes even a correct arithmetic expression insufficient to explain the eventual customer allocation.

Count reservations too.

That example is deliberately small, but the capacity-planning problem gets harder as concurrency rises: if several workers can each observe the same headroom before any worker reserves it, every individual check may pass while the aggregate violates the ceiling. Make checking and reserving one atomic operation in the application ledger, attach the reservation to a stable job identifier, and settle it against attributable usage when the evidence arrives. A retried job must find its existing reservation.

There is a real limitation here. A strict monetary cap needs a defensible maximum liability for every accepted operation, including its retry policy; when that bound is unavailable, describe the control as an admission threshold with reserved headroom rather than promising an exact final charge. A guessed average is a weak foundation for a hard limit.

Treat attribution completeness and snapshot completion as separate SLOs. A successful scheduled run that leaves a material amount unassigned should not count as a clean billing close, and a perfectly assigned live counter doesn't compensate for a missing period snapshot. Set the objectives and escalation thresholds from the platform's own billing obligations, without borrowing a vendor uptime claim.

## Choose the buy-versus-build boundary around billing evidence

A consolidated backend can reduce the number of credential and invoice boundaries the billing pipeline must join. Infrai offers one key and one bill across backend services through one REST API, with a public self-describing discovery surface that lets engineers inspect request and response schemas before wiring the scheduled close. That consolidation is useful for the gaming workload only if your own attribution ledger can connect the resulting evidence to the right customer; one account-level bill isn't proof of per-customer accuracy.

The comparison below is a procurement decision aid, not a benchmark.

Evaluate the same disputed-invoice exercise against each option, using the same customer and period, before accepting its operational model.

| Option | What to consider buying | What the platform team must still establish | Best fit and limitation |
| --- | --- | --- | --- |
| Stripe Billing | A managed billing workflow; assess its usage-based billing documentation | How workload evidence becomes billable usage, and how a frozen period maps to an invoice | Consider when the billing workflow is the main thing to outsource; admission control remains an application responsibility |
| OpenMeter | A metering-focused component; assess its documented event and aggregation model | Stable attribution identifiers, correction handling, and the handoff from usage to invoice evidence | Consider when metering is the component you want to evaluate separately; choosing a meter doesn't establish a spend cap |
| Lago | A billing-focused option; assess its usage-based billing model and deployment choices | Who operates the chosen deployment and how accepted usage is preserved at close | Consider when billing configuration and operational ownership drive the choice; require a demonstration of your reconciliation workflow |
| Application-owned ledger and close worker | The exact attribution and reservation rules your gaming system needs | Storage, scheduling, recovery, audit access, and on-call ownership | Build when the workload boundary requires control you cannot delegate; budget engineering capacity for the entire lifecycle |

The managed route can move work off the roadmap, while an application-owned ledger lets the team define a game-specific admission policy. Neither choice removes the need to preserve what an invoice actually used. The lock-in question is concrete: can the next billing implementation reconstruct the same period from exported source evidence, allocation rules, and snapshot identifiers? Ask for that demonstration before making migration promises.

## Freeze the period without turning a retry into a rewrite

Schedule the close independently of dashboard activity. The worker should read platform usage for a defined period, retain that evidence, apply a recorded allocation rule, and publish an immutable snapshot under a stable customer-and-period identity. Store the evidence and allocations in private storage you operate, and store the invoice's snapshot reference with the invoice record. Keep live usage views separate because their purpose is different.

A snapshot is evidence.

A later read can differ from the number used at close, which is precisely why billing directly from that later read makes an old invoice impossible to reproduce. Preserve the original, record the discrepancy, and reconcile it against the platform's record. Any authorized correction should create a new version and an explicit relationship to the previous version; it should not replace bytes behind an existing invoice reference.

Here is a runnable Go acquisition example: read account usage, preserve the response without assuming undocumented fields, and refuse to overwrite a previously accepted observation. Set INFRAI_API_BASE to the service's HTTPS API origin, without a path, and INFRAI_API_KEY to a credential supplied through your secrets manager. The code appends the versioned usage path and explicitly sends GET. Run it in a private working directory.

This is one source observation, not a monthly allocation.

A period-close worker must separately establish the period boundaries, collect the required usage evidence, and allocate it before accepting an invoice snapshot. The fixed observation name below demonstrates retry and conflict behavior; use a stable, explicitly assigned capture identity in a real acquisition job. Never promote an account total into a customer invoice just because the fetch succeeded.

```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "path/filepath"
    "strconv"
    "strings"
    "time"
)

func readUsage() ([]byte, error) {
    base := strings.TrimRight(os.Getenv("INFRAI_API_BASE"), "/")
    key := os.Getenv("INFRAI_API_KEY")
    origin, err := url.Parse(base)
    if err != nil || origin.Scheme != "https" || origin.Host == "" ||
        origin.User != nil || origin.Path != "" || origin.RawQuery != "" ||
        origin.Fragment != "" || key == "" {
        return nil, fmt.Errorf("set HTTPS API origin and API key")
    }
    client := &http.Client{
        Timeout: 30 * time.Second,
        CheckRedirect: func(_ *http.Request, _ []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(http.MethodGet, base+"/v1/account/usage", nil)
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        body, readErr := io.ReadAll(io.LimitReader(resp.Body, (16<<20)+1))
        resp.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if len(body) > 16<<20 {
            return nil, fmt.Errorf("usage response exceeds capture limit")
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Second * time.Duration(1<<attempt)
            retry := resp.Header.Get("Retry-After")
            if seconds, err := strconv.ParseInt(retry, 10, 32); err == nil && seconds >= 0 {
                requested := time.Duration(seconds) * time.Second
                if requested > delay {
                    delay = requested
                }
            } else if at, err := http.ParseTime(retry); err == nil {
                if requested := time.Until(at); requested > delay {
                    delay = requested
                }
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return nil, fmt.Errorf("usage status %d: %s", resp.StatusCode, body)
        }
        return body, nil
    }
    return nil, fmt.Errorf("usage retry budget exhausted")
}

func publish(dir, name string, data []byte) error {
    if err := os.MkdirAll(dir, 0700); err != nil {
        return err
    }
    f, err := os.CreateTemp(dir, ".pending-")
    if err != nil {
        return err
    }
    defer os.Remove(f.Name())
    defer f.Close()
    if _, err = f.Write(data); err != nil {
        return err
    }
    if err = f.Sync(); err != nil {
        return err
    }
    if err = f.Close(); err != nil {
        return err
    }
    target := filepath.Join(dir, name)
    if err = os.Link(f.Name(), target); err == nil {
        return nil
    }
    if !os.IsExist(err) {
        return err
    }
    existing, err := os.ReadFile(target)
    if err != nil {
        return err
    }
    if !bytes.Equal(existing, data) {
        return fmt.Errorf("snapshot identity conflict: %s", name)
    }
    return nil
}

func main() {
    data, err := readUsage()
    if err == nil {
        err = publish("snapshots", "account-usage-observation-001.json", data)
    }
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
    fmt.Println("usage observation accepted")
}
```

The client uses a 30-second per-request timeout and a four-attempt retry budget for rate limiting, with exponential backoff extended by a valid Retry-After header. Those are example policy choices, not service guarantees. GET retries don't create billable customer records in this example; invoice creation needs a separate idempotency contract. Redirects are rejected so the credential stays at the configured origin. Keep any response-bearing error output restricted to billing operators.

The temporary file is fully written and synced before a hard link publishes the final name. An identical response succeeds; different bytes under that name produce a conflict. This requires a filesystem supporting hard links, with the temporary and final names on the same filesystem. It demonstrates publication and collision handling, not distributed locking or guaranteed survival of directory metadata across power loss.

For a production service, choose a transactional uniqueness constraint or a storage operation with documented conditional-create semantics, then verify its durability guarantees for your deployment. Avoid generating a fresh snapshot identity on every retry. The invoice worker needs its own stable idempotency identity tied to the accepted snapshot; immutable storage alone cannot prevent two independently created invoices.

## Verify attribution, then rehearse the rollback

Make the acceptance test a disputed gaming invoice, not a successful HTTP response. From the invoice record, retrieve its frozen snapshot, the source usage evidence, and the allocation rule, then reconstruct the total without consulting the current live counter. Reissuing that invoice should select the same snapshot. A corrected invoice should make its new version and reason visible in the accounting history.

Exercise three additional cases before the close worker is trusted: two admission requests competing for the final reservation headroom, a repeated close job using the same snapshot identity, and a later usage read that disagrees with the frozen amount. In the first case, accepted reservations must respect the cap policy; in the second, the result must be identical or an explicit conflict; in the third, the system must preserve the original evidence while opening a reconciliation item. These are proposed checks, not reported test results.

An empty billing-page traffic graph should not stop the scheduled close. Watch for missed periods, unassigned usage, unresolved reservation age, and discrepancies between the platform record and the snapshot. A single green scheduler indicator cannot represent all four conditions.

Rollback should stop new invoice issuance from the changed close path while retaining every accepted snapshot and invoice reference. Revert the worker to the previously approved allocation rule for future work, review affected periods, and publish corrections through the explicit revision process. Don't delete evidence to make the totals line up.

For admission control, make the degraded policy explicit before an incident: a background asset workload can wait when its ledger is unavailable, whereas any exception for interactive traffic needs a bounded allowance and an accountable owner. That is a product and SLO decision, not something a billing SDK should silently choose. Scope billing credentials to their job and keep them out of snapshots; the secrets-management guidance below belongs in the implementation review.

## References

- [Stripe usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [OpenMeter documentation](https://openmeter.io/docs)
- [Lago documentation](https://docs.getlago.com/)
- [Go operating-system package, including temporary files and hard links](https://pkg.go.dev/os)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
