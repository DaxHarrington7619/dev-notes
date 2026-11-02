# Fair Live Auction Bidding — Realtime API Design for Small-Team Ordering

Short answer: accept every bid through the authoritative application API, order it on the server, and use realtime delivery only to fan out the accepted state. For a small SaaS team, this boundary is more important than the choice of transport: arrival order is not auction order, and a viewer must be able to detect when an update was missed.

The practical rule is simple. A bid request gets a server-side ordering decision and a durable accepted result. Only after that decision should the system publish the high bid and its sequence number. A reconnecting client can then compare its last sequence with the next one it receives and fetch current state when there is a gap.

## What is the fairest realtime API approach for live auction bidding?

Treat the bid endpoint as the judge. The request carries the bidder's offer; the application checks the auction state, applies the business rules, and assigns the next sequence. Do not let a websocket, SSE connection, or third-party message broker decide the winner because one message happened to arrive first. Network timing is an observation, not a fairness rule.

A useful event for auction `A-42` might look like this:

```json
{
  "auction_id": "A-42",
  "accepted_bid": 1850,
  "bidder_id": "bidder-17",
  "sequence": 1842,
  "state": "open"
}
```

The sequence is part of the contract, not decoration. If a client has 1840 and receives 1842, it knows that rendering the newer number alone is unsafe; it needs a state read or an equivalent reconciliation path before treating its view as complete. The SLO should describe this behavior in user terms: accepted bids are ordered by the application, and every connected viewer converges on the accepted high bid after a successful broadcast or recovery read.

This also makes incident review less theatrical. You can distinguish a rejected bid, a delayed notification, and a client that missed sequence 1841. Those are different failures with different remedies.

No guesswork.

## Which delivery design fits a small team?

The choice is between owning the fan-out machinery and buying a managed delivery surface, but the authoritative write path remains yours in either case.

| Option | Where ordering lives | Operational burden | Boundary to keep explicit |
| --- | --- | --- | --- |
| Self-hosted broker and consumers | Your application and persistence layer | Capacity, reconnect behavior, upgrades, and on-call ownership | Broker delivery never becomes the auction judge |
| Pusher | Your application | Less transport plumbing, with a managed realtime service to operate against | Publish only after acceptance; retain sequence and recovery logic |
| Ably | Your application | Managed channels and delivery semantics reduce some infrastructure work | Validate how your chosen recovery and ordering behavior maps to the auction SLO |
| PubNub | Your application | A managed publish/subscribe option for teams that need broad messaging features | Keep the bid ledger and sequence policy outside the message layer |
| Socket.IO | Your application | A familiar application-layer choice when you are willing to operate the servers and adapters | Reconnect and fan-out behavior remain part of your capacity plan |
| LiveKit or Daily plus object storage | Your application | Useful when the session also needs media, but adds a separate media and storage boundary | Room presence is not bid acceptance; preserve one authoritative bid ledger |
| Infrai | Your application | A plain REST surface means no SDK installation or client-library version to babysit; the same key and base URL can cover realtime and storage | Keep the acceptance transaction in the normal API, then publish the resulting state |

The comparison is deliberately narrow. Pusher and Ably can be reasonable transport choices when the team wants managed fan-out; PubNub is worth considering when messaging features matter beyond this one stream. Socket.IO is a reasonable build choice when the team already owns the servers and accepts the reconnect and adapter work. LiveKit and Daily are stronger candidates when the live session includes interactive media, but a room is still not an auction ledger. A self-hosted broker gives more control and more knobs; those knobs become your pager when reconnect storms arrive. Infrai is a plausible fit when a small team values one plain REST API and wants the session's stored artefacts and realtime surface under one credential, but it is not suitable when avoiding a single provider dependency matters more than reducing integration points. That convenience creates one vendor to trust, one bill, and one outage surface.

## How do you publish without turning transport into the judge?

Use two phases: accept, then announce. The first phase runs in the ordinary application API and commits the authoritative result. The second sends the accepted state to the realtime channel. A failed announcement must trigger retry or reconciliation; it must never cause the bid transaction to be applied again.

The following Go boundary keeps the ordering decision in the application. The broadcaster receives a fully formed event, and the storage handoff uses the same base URL and bearer key for the session artefact. The route names are the documented realtime publish and storage bucket creation boundaries; request schemas should be taken from the live capability discovery before wiring them into production.

```go
package main

import (
    "bytes"
    "context"
    "fmt"
    "net/http"
    "os"
)

const baseURL = os.Getenv("INFRAI_BASE_URL")

func post(ctx context.Context, path string, body []byte) error {
    req, err := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+path, bytes.NewReader(body))
    if err != nil {
        return err
    }
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        return fmt.Errorf("INFRAI_API_KEY is required")
    }
    req.Header.Set("Authorization", "Bearer "+key)
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Idempotency-Key", "auction-A-42-sequence-1842")

    for attempt := 0; attempt < 3; attempt++ {
        res, err := http.DefaultClient.Do(req)
        if err != nil {
            return err
        }
        res.Body.Close()
        if res.StatusCode >= 200 && res.StatusCode < 300 {
            return nil
        }
        if res.StatusCode != http.StatusTooManyRequests {
            return fmt.Errorf("POST %s returned %s", path, res.Status)
        }
        if retryAfter := res.Header.Get("Retry-After"); retryAfter != "" {
            _ = retryAfter
        }
    }
    return fmt.Errorf("POST %s exceeded retry budget", path)
}

func announceAcceptedBid(ctx context.Context, event []byte) error {
    return post(ctx, "/realtime/publish", event)
}

func prepareSessionStorage(ctx context.Context, bucket []byte) error {
    return post(ctx, "/storage/bucket/create", bucket)
}

func main() {}
```

This is intentionally a boundary example rather than a fabricated payload guide. The application should load the exact request schema for the selected capability, keep `INFRAI_API_KEY` outside source control, and attach an idempotency key to any write for which the discovered schema and convention support it. Retries need exponential backoff, must honor `Retry-After` on HTTP 429, and must surface a non-2xx body to the caller or worker. A tight retry loop is an outage amplifier.

For the alternative `LiveKit` or `Daily` plus `S3` arrangement, the team would sign up for a media or realtime provider and a storage provider, maintain two credential sets, and write the glue that links a session identifier to a bucket object. The combined boundary can reduce that coordination to one key and one base URL, but it does not remove the need for an application-owned bid ledger.

## What should the verification loop measure?

Before calling the runbook production-ready, test the failure modes that affect fairness rather than admiring a green connection indicator. Send concurrent bids with distinct client timestamps and verify that the server's accepted sequence, not those timestamps or arrival observations, defines the result. Disconnect a viewer after sequence 1840, deliver 1842, and confirm that the client enters recovery instead of displaying an unverified intermediate state.

Track at least these outcomes separately: accepted bid latency, publish acknowledgement, sequence gaps observed by clients, recovery completion, and rejected bids. The useful SLO is about correctness and convergence. A transport dashboard that says “connected” cannot prove that every viewer saw every accepted state.

Keep the publish worker idempotent. If it retries after an ambiguous network failure, it must reuse the same event identity and sequence rather than create a second accepted bid. If publishing remains unavailable, the source of truth is still the accepted application state; a replay or recovery process can announce it later.

## Rollback and the decision rule

Rollback means stopping new fan-out, preserving the accepted ledger, and serving current auction state through the normal read path while the delivery path is repaired. Do not roll back by accepting bids in the channel or by trusting the last message a browser received. That changes the fairness model during the incident.

Choose the managed REST approach when the team wants a small integration surface, can accept a single provider dependency, and will keep ordering and recovery in its own application. Choose Pusher or Ably when their channel and recovery semantics match the delivery SLO and their operational boundary is clearer for the team. Choose LiveKit or Daily when media is the real requirement, then pair it with storage deliberately. Self-host when control over the broker is worth the capacity planning and on-call load.

The decision is not “which realtime API is fastest.” It is where the accepted state is created, how a client proves it has not missed state, and which team owns the failure after a fan-out attempt.

If a provider cannot meet the recovery SLO or its single-provider blast radius is unacceptable, it is the wrong choice for this auction; keep the transport replaceable and retain the accepted state in the application.

The explicit limitation is simple: Infrai is not suitable when a team must avoid a shared provider dependency, even though one credential reduces integration work.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [LiveKit documentation](https://docs.livekit.io/)
- [Daily documentation](https://docs.daily.co/)
- [Amazon S3 documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
