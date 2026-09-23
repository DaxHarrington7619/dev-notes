# Running Image Batch API Cancel Approach Explained — Wrong Catalogue Import Recovery

TL;DR: Treat cancellation as a durable state transition, not as a promise that every running background-removal request will stop immediately. For a marketplace recovering from a wrong catalogue import, stop new dispatches first, quarantine every output under the import's own namespace, let workers report what actually committed, and delete or retain those objects by policy. That design caps storage and cache exposure even when work crosses a process boundary and cannot be interrupted cleanly.

A bounded scenario makes the risk concrete: suppose an operator selects the wrong catalogue file and the importer accepts 40,000 product-photo rows before the mistake is noticed. This is a planning example, not a reported incident or benchmark. The dangerous response is to send an in-memory cancellation signal and declare the batch gone, because queued work, active transformations, stored derivatives, and cache entries are four different states. The invariant is stricter: after cancellation is accepted, no new item may start, and every object already produced must remain attributable to that import.

## How should an API cancel a running image batch?

The request that created a batch is only a control-plane interaction. Once workers have claimed items, terminating the caller's connection says nothing reliable about those claims, and even a cooperative worker can receive cancellation after it has written an output. Cancellation therefore has two observable outcomes: dispatch closes, while already claimed work drains to a recorded terminal state.

Cancellation is fencing.

This distinction matters more than latency during a bad catalogue import. A background-removal pipeline usually creates a source read, a decoded image, a transformed image, an object write, metadata, and perhaps a cached response. Formats differ in compression, transparency, animation, and browser support; the MDN image-format guide is a useful reminder that an output policy cannot treat every extension as interchangeable. A product cutout that requires transparency needs an explicitly supported output format rather than a renamed file.

Keep the source immutable. Put generated objects beneath a batch-scoped prefix such as `derived/<import-id>/<item-id>`, attach the import identifier to metadata, and prevent cancelled batches from publishing catalogue pointers. Cleanup can then enumerate a bounded ownership domain instead of guessing from timestamps or filenames.

Stop the fan-out first.

## The recovery boundary is storage ownership

A cancellation design should be judged by what it can prove after the control process restarts. An in-memory flag is fast but disappears with the process. A durable batch state lets every dispatcher independently reject new claims. Per-item leases avoid two workers treating the same row as theirs, while idempotent output keys make a retry converge on one location rather than create a second derivative. These are coordination properties, not vendor features.

Restarts count.

The storage layout is also the cost control. If outputs from multiple imports share an unstructured directory and cache key, responders cannot cheaply identify the wrong import's footprint, and invalidation spreads to valid products. Batch-scoped object prefixes and cache keys turn recovery into a finite set operation. Retention still needs care: immediate deletion reduces stored bytes but removes evidence useful for reconciliation; a quarantine window preserves evidence but consumes capacity. Set that window from the recovery SLO and forecast its worst-case bytes before choosing it.

For capacity planning, use an upper bound rather than an average: accepted rows multiplied by the maximum permitted derivatives per row and the maximum encoded size admitted by policy. Add source staging separately if the importer copies originals. Compression ratios observed on yesterday's catalogue are weak protection against tomorrow's unusually detailed images, so admission limits belong ahead of fan-out.

| Approach | Restart behavior | Storage and cache consequence | Operational trade-off |
|---|---|---|---|
| Process-local cancel flag | State can be lost | Orphaned outputs require discovery | Small implementation, weak recovery proof |
| Durable batch state plus cooperative workers | Dispatch remains closed after restart | Outputs remain enumerable by batch | More state transitions and reconciliation |
| Isolated worker process terminated on cancel | Active computation may stop sooner | Completed writes still need ownership and cleanup | Strong isolation, higher scheduling and on-call load |

The buy-versus-build question sits beneath those rows. A managed queue or batch service can reduce scheduler ownership, but the team must verify its cancellation, lease, retry, and retention semantics rather than infer them from a `cancel` method name. A self-hosted coordinator gives direct control over state and object naming, while transferring upgrades, capacity, and paging to the platform team. Neither choice removes the need for application-level provenance.

## A preventative Go control path

The smallest useful implementation makes the state check part of claiming work and checks it again before publication. The following sketch deliberately leaves transport and database selection behind interfaces; its contract is the important part. `Cancel` closes future dispatch durably, workers use context cancellation for cooperative local work, and committed keys are recorded for reconciliation.

```go
package batch

import (
    "context"
    "errors"
    "fmt"
)

var ErrCancelled = errors.New("batch cancelled")

type Store interface {
    MarkCancelled(ctx context.Context, batchID string) error
    IsCancelled(ctx context.Context, batchID string) (bool, error)
    RecordCommitted(ctx context.Context, batchID, itemID, key string) error
}

type Transformer interface {
    RemoveBackground(ctx context.Context, sourceKey, destinationKey string) error
}

type Worker struct {
    state Store
    images Transformer
}

func (w Worker) Cancel(ctx context.Context, batchID string) error {
    return w.state.MarkCancelled(ctx, batchID)
}

func (w Worker) Process(ctx context.Context, batchID, itemID, sourceKey string) error {
    cancelled, err := w.state.IsCancelled(ctx, batchID)
    if err != nil {
        return fmt.Errorf("read batch state: %w", err)
    }
    if cancelled {
        return ErrCancelled
    }

    destinationKey := fmt.Sprintf("derived/%s/%s.png", batchID, itemID)
    if err := w.images.RemoveBackground(ctx, sourceKey, destinationKey); err != nil {
        return fmt.Errorf("transform image: %w", err)
    }

    cancelled, err = w.state.IsCancelled(ctx, batchID)
    if err != nil {
        return fmt.Errorf("recheck batch state: %w", err)
    }
    if cancelled {
        // The reconciler owns deletion; never publish this key to the catalogue.
        return ErrCancelled
    }
    if err := w.state.RecordCommitted(ctx, batchID, itemID, destinationKey); err != nil {
        return fmt.Errorf("record committed output: %w", err)
    }
    return nil
}
```

There is an unavoidable race between the second state check and the commit record. Production code closes it with a transactional claim/publication state machine or an outbox in the same consistency boundary as catalogue publication. Object storage may sit outside that transaction, which is why deterministic keys and a reconciler remain necessary. Do not hide this race behind retries; retries make an ambiguous write more frequent unless the destination and state transition are idempotent.

Writes can win.

The cancellation endpoint should return after the durable state transition, not after every worker exits. Expose drain progress separately: claimed, succeeded before cancellation, quarantined after cancellation, failed, and still leased. Alert on a batch whose drain exceeds the recovery objective, on dispatch after the cancellation timestamp, and on a mismatch between recorded objects and objects found under the batch prefix.

## Comparing the operational choices

Choose cooperative cancellation when transformations respond to context and a short amount of post-cancel computation is acceptable. It uses fewer isolation boundaries and is easier to capacity-plan, but its SLO must allow active items to finish or notice cancellation. Choose process isolation when third-party or native image code can hang, memory containment matters, or the recovery objective requires forceful termination; the price is a more complicated scheduler and a larger surface for duplicate claims.

A queue purge alone is insufficient. It can remove waiting messages, yet it cannot account for leased items or objects written just before the purge. Likewise, deleting every derivative immediately is unsafe if valid and invalid imports share keys, and broad cache invalidation can turn a catalogue correction into a traffic spike. The defensible sequence is durable cancel, dispatch fencing, drain observation, reconciliation by import identifier, catalogue correction, and scoped cache invalidation.

Test that sequence with failure injection. Cancel before the first claim, during image decoding, after the object write but before the commit record, and while the coordinator restarts. Then retry the same item and run reconciliation twice. The expected result is boring: no publication from the cancelled import, one deterministic output location per item, and a cleanup operation that is safe to repeat.

Run it twice.

## When this pattern is too heavy

A synchronous tool that transforms one local image and writes one local file does not need a durable batch state machine; context cancellation plus removal of a temporary file may be enough. The same is true when generated data is disposable, cannot be published, and the entire operation fits inside one transactional boundary.

A marketplace catalogue import is different because it crosses durable queues, image processing, object storage, metadata, and caches. Once those boundaries exist, cancellation is a recovery workflow. Define its SLO in terms of halted dispatch and reconciled ownership, keep every derivative attributable to the import that created it, and make storage cleanup repeatable. That is the approach that limits both customer-visible contamination and an unbounded cache bill.

## Sources

- MDN, Image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
