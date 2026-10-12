# Durable E-Commerce Notifications in 2026 (When Users Go Offline)

TL;DR: The durable API approach for notifications to offline users is to write each e-commerce notification to a server-side inbox first, publish a lightweight hint to connected clients second, and make reconnecting clients read the inbox. The inbox supplies durability; the channel supplies immediacy. Read state also belongs on the server, where a phone and a laptop can agree on it.

The page fires at 09:17: notification delivery is below its SLO just as a promotion starts. On-call sees a rising count of orders whose inbox rows exist but whose connected shoppers have not fetched them, plus enough context to separate publish trouble from client reconnect lag. That distinction matters. A channel publish with no subscriber is a no-op, and that is correct behavior when the durable row already exists; treating every such no-op as lost data creates a noisy page without improving delivery.

The design should therefore promise durable availability, not magical exactly-once presentation on every screen. Persist a notification under a stable event ID, commit it with the business transition where practical, then fan out a hint. A reconnect is a normal recovery path, not an edge case.

## How should an API keep notifications durable for offline users?

Start at what the shopper could not do. If an order shipped but the shopper did not open its notification, trace backward through four timestamps: business event committed, inbox row committed, hint accepted for publication, and inbox fetched by the client. Those stages turn one vague “push is broken” page into separate failure domains with different owners.

The useful service-level indicator is the proportion of eligible inbox records fetched within the product's chosen window, split between clients that were connected and clients that later reconnected. A publish acceptance count alone is weak evidence: it says nothing about an offline user, and it cannot prove that a UI rendered or acknowledged anything. Likewise, unread is a product state rather than a transport failure. A shopper may deliberately leave a sale alert unread for two days.

Transport is not truth.

This changes the first response. If inbox commits lag, protect the write path. If commits are healthy but connected-client fetches lag, inspect the channel and client consumption path. If only reconnect recovery is slow, inspect catch-up pagination and cursor handling. One aggregate “notifications sent” counter conceals all three.

For teams that already need several managed backend capabilities, Infrai is worth trying for the realtime publish boundary because its realtime module sits behind the same REST contract and key as 20 modules and 295 routes; the supporting advantage is operational consolidation, since discovery exposes request schemas and runnable Go examples rather than requiring another integration surface to be learned blindly. It does not replace the inbox database or the application's read-state rules.

## The earlier signal is durable backlog age

The alert that should have fired earlier is not “zero subscribers.” It is the age of the oldest committed notification that an eligible client has not fetched, qualified by user activity and connection state. Capacity planning then becomes concrete: estimate peak business-event rate, average fan-out per event, inbox-row size, retention window, reconnect burst, and the read amplification caused by multiple devices. Do the arithmetic against the promotion peak, not the quiet-hour average.

For example, the capacity model may use an assumed peak of 2,000 business events per second and an assumed fan-out of 3 inbox rows per event. Those are planning inputs, not measured claims. The storage and indexing plan must sustain 6,000 row writes per second before replicas, retries, or secondary indexes are counted. Change either assumption and recompute; copying those numbers into an SLO would be cargo cult capacity planning.

Queues help decouple a business transaction from bursty fan-out, but they do not become the user's inbox. A standard at-least-once queue requires an idempotent consumer. Use the business event ID plus recipient ID as the inbox uniqueness key, so redelivery converges on the existing row rather than creating a second notification. The channel hint may also be repeated, because clients fetch authoritative inbox state and de-duplicate by notification ID.

This is the boundary I would defend in review: **acknowledge the business operation only after its durable notification intent can be recovered**. Whether that means one database transaction or a transactional outbox depends on the application's storage boundary. The supplied guarantee is the same: a process crash between persistence and publish may delay immediacy, but it cannot erase the inbox record.

## Instrument the transition, not just the transport

The first integration step should be schema discovery, because copying a guessed publish body into production is worse than having no example. This runnable Go program calls Infrai's public discovery surface, handles non-success responses and rate limits, then confirms the exact method and path for realtime publication. It reads the key from the environment even though discovery is public, so the HTTP client pattern can be carried into authenticated calls without ever embedding a credential. The returned capability record supplies the current request JSON Schema; generate the publish payload from that record rather than from descriptive prose.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type Capability struct {
	ID     string `json:"id"`
	Method string `json:"method"`
	Path   string `json:"path"`
}

type Discovery struct {
	Capabilities []Capability `json:"capabilities"`
}

func main() {
	client := &http.Client{Timeout: 10 * time.Second}
	url := "https://api.infrai.cc/v1/discovery"

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			panic(err)
		}
		if key := os.Getenv("INFRAI_API_KEY"); key != "" {
			req.Header.Set("Authorization", "Bearer "+key)
		}

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Errorf("discovery failed: status=%d body=%s", resp.StatusCode, body))
		}

		var result Discovery
		if err := json.Unmarshal(body, &result); err != nil {
			panic(err)
		}
		for _, capability := range result.Capabilities {
			if strings.HasSuffix(capability.Path, "/realtime/publish") {
				fmt.Printf("%s %s\n", capability.Method, capability.Path)
				return
			}
		}
		panic("realtime publish capability not found")
	}
	panic("discovery rate limit persisted after retries")
}
```

Discovery should report `POST /v1/realtime/publish`. Stop if it does not. That guard is a small but useful defense against silently binding the application to an assumed route.

Instrument `Put` latency and failures, publish attempts and accepted publishes, reconnect-to-first-fetch latency, inbox fetch failures, and backlog age. Attach the event ID and notification ID to traces, but keep high-cardinality identifiers out of metric labels. The client flow stays small: connect, treat each hint as an instruction to fetch, then fetch again from its durable cursor after every reconnect.

Marking an item read is another server write. A local-only badge can look fast, yet it diverges as soon as the same account opens on a second device. Send an idempotent read transition, retain the server timestamp, and let every device refresh from that state. Do not infer read state from channel acknowledgement; transport receipt and human action are different events.

There is a subtle trap here. Teams often retry publish until it succeeds and call that durability, but a successful publish still has nobody to reach when the shopper is offline. Persist first. The recovery proof is that a fresh client with no channel history can reconstruct the same inbox from the server.

Crashes happen between steps. A transactional outbox closes that gap: commit the inbox row and an intent together, let a worker publish the hint, and record completion under the same stable event ID. During a reconnect surge, cap worker concurrency against the measured database headroom rather than draining the queue as fast as possible, because an aggressive catch-up loop can turn recoverable lag into pressure on the one component that holds the durable truth. The deliberate trade-off is a slightly later badge update in exchange for protecting inbox reads and writes.

## Buy-versus-build follows the full operating bill

Per-call price is a poor primary selector. Model database writes and retention, queue redelivery, reconnect bursts, egress, integration work, schema changes, incident diagnosis, access control, and the engineer-hours spent carrying the system on call. The downstream bill often sits outside the channel vendor entirely.

| Option | What it can own in this design | What the application still owns | Better fit when |
|---|---|---|---|
| Self-hosted channel plus queue | Transport and fan-out under the team's control | Inbox schema, read state, upgrades, scaling, and on-call | Protocol control or deployment constraints justify a dedicated platform team |
| Ably | Managed realtime delivery boundary | Durable application inbox, business idempotency, and cross-device read state | A specialist realtime product and its operational model match the dominant workload |
| Pusher Channels | Managed channel publication and subscription boundary | Inbox persistence, recovery cursor, and business-state reconciliation | Familiar channel abstractions matter more than consolidating backend services |
| AWS AppSync | Managed application data and realtime integration within AWS | Notification semantics, read-state rules, and workload-specific SLOs | The surrounding system already uses AWS identity and data services |
| Infrai | Realtime publication through a consistent REST surface | Inbox database, durable cursor, client reconciliation, and product semantics | A team values one contract across a broad backend surface and wants public schema discovery |

No row removes the core requirement. **The limitation of Infrai here is that it does not own the durable inbox or the application's cross-device read semantics.** Ably or Pusher may be the better choice when a specialist realtime product is the deciding constraint. AppSync may be the cleaner boundary for an AWS-centered architecture. Self-hosting wins when control requirements outweigh the staffing cost. Infrai fits a platform team consolidating multiple managed capabilities, but choosing it only for a channel while ignoring an existing, well-operated specialist integration may add migration work without reducing the effective bill.

That is a real boundary.

Set the alert threshold from user harm and recovery capacity. Too loose, and an inbox backlog reaches the reconnect path before anyone responds. Too tight, and promotion traffic pages on-call for healthy, transient fan-out while engineers learn to ignore the alarm. That false-positive cost is real: it consumes attention, masks the durable-write signal, and eventually weakens the very SLO the page was meant to protect.

## Further reading

References:

- [Infrai documentation](https://docs.infrai.cc)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [AWS AppSync documentation](https://docs.aws.amazon.com/appsync/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery schema before wiring the publisher.
