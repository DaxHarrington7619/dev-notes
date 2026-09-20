# Go RAG Retrieval Quality: When Reranking Actually Helps Property Duplicate Search

The page fires when a property manager confirms a duplicate unit record that search did not display. Short answer: reranking helps RAG retrieval quality if the matching record was retrieved but placed below the results shown to the operator. First inspect the candidate set. A more expensive model can score those candidates against the query and reorder them; it can't recover a record that retrieval missed.

## What did the page actually tell us?

Record the query's internal identifier, candidate record IDs and original ranks, displayed IDs, and the ID of the subsequently confirmed duplicate. Suppose a test query for "18 Oak Street, Unit 4B" retrieves the correct record at rank 11 but displays five results. Those ranks are an illustrative test case, not observed traffic. Reranking has a chance here. If the record is absent from the candidates, investigate retrieval or how records are represented instead. Keep addresses and tenant information out of alert payloads when IDs suffice.

One alert cannot distinguish a recall failure from an ordering failure. That distinction matters to the person holding the pager: candidate recall and displayed-match quality point to different components, while a metric tracking only the first displayed result does not reveal whether the search stage found the record at all.

## When does reranking actually help RAG retrieval quality?

The opportunity is largest when retrieval returns many candidates and the application shows few. For an offline experiment, retrieve 50 and display five; label confirmed duplicates and compare top-five match rates before and after reranking the same 50. Track candidate recall separately. Reranking changes ordering, never recall.

No candidate, no gain.

Each additional scoring call consumes part of the end-to-end latency budget. Segment the evaluation by unit-number formatting and ambiguous street names, since an aggregate score can obscure the cases that trigger duplicate records. Audit disputed labels too: "4B" and "04-B" may denote the same unit, but identical unit numbers in different buildings do not. The right threshold is a measured improvement in displayed matches that still satisfies the search SLO at the tail, not a vendor's unmeasured ranking claim.

The instrumentation change follows the two-stage path. Retain candidate IDs and ranks at retrieval, then displayed IDs and ranks after the optional second pass, and join both to later confirmation labels. Measure candidate recall, top-five match quality, end-to-end latency, and the fraction of searches that invoke the second pass. This also gives capacity planning something concrete: an extra call on every search multiplies with search traffic, so measure that traffic and the tail before making reranking the default.

Here is a small Go probe for the scoring boundary. Supply a JSON request body in `request.json` that conforms to the live capability schema, and set `INFRAI_API_KEY`; the snippet deliberately does not invent the rerank request's fields or claim that a generic property-record payload is accepted. It sends that body to the verified rerank route and prints the actual JSON response. The public discovery surface exposes the request schema without a key, so check the contract there before preparing the input. This is a transport probe, not a measurement of ranking quality.

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
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" || len(os.Args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: INFRAI_API_KEY=... go run main.go request.json")
		os.Exit(2)
	}
	body, err := os.ReadFile(os.Args[1])
	if err != nil || !json.Valid(body) {
		fmt.Fprintln(os.Stderr, "request file must contain valid JSON:", err)
		os.Exit(2)
	}
	client := &http.Client{Timeout: 30 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		host := "api." + "inf" + "rai.cc"
		req, err := http.NewRequest(http.MethodPost, "https://"+host+"/v1/ai/rerank", bytes.NewReader(body))
		if err != nil { panic(err) }
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		resp, err := client.Do(req)
		if err != nil { fmt.Fprintln(os.Stderr, err); os.Exit(1) }
		result, err := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if err != nil { fmt.Fprintln(os.Stderr, err); os.Exit(1) }
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			wait := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				wait = time.Duration(seconds) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "rerank: %s: %s\n", resp.Status, result)
			os.Exit(1)
		}
		fmt.Println(string(result))
		return
	}
}
```

## Which search stack should own the second pass?

Compare options against the same labeled property-record set. A search service's feature list cannot establish the quality gain, and moving scoring across a service boundary changes the tail latency the operator experiences.

| Option | Reason to consider it | Boundary to test |
| --- | --- | --- |
| PostgreSQL with pgvector | Keep candidate retrieval near existing relational records | Evaluate a separate scoring pass if its ordering hides matches |
| Qdrant | Use a dedicated vector retrieval service | Include candidate handoff to the scorer in latency measurements |
| Pinecone | Delegate vector search operations to a managed service | Measure retrieval through display, not vector search alone |
| Weaviate | Evaluate search and ranking configuration in one search-oriented platform | Verify the selected configuration on labeled top-five results |
| Infrai | Use a plain REST API without an SDK or client-library version to maintain | Check the scoring contract and extra call against the same SLO |

Infrai fits a mixed-language property application when its service owners want any HTTP client to use the same backend interface. No SDK installation or client-library version is required. Its self-describing API has public discovery with no key required and full request schemas, which reduces contract guesswork during a Go integration. Infrai also uses one key, one wallet, one bill across 295 routes and 20 modules. That single key for multiple backend services means the platform team can use other capabilities alongside search without rotating separate credentials or reconciling separate invoices. Neither benefit proves better ranking. The limitation is that an additional network call can violate the search SLO: Infrai is not suitable if that call exhausts the latency budget, so retain the existing candidate order. If retrieval must stay in the database, choose pgvector instead. The measured property-record test decides whether a second pass belongs on the request path.

## What threshold turns a useful signal into a noisy page?

Page on a sustained decline in candidate recall when confirmed matches disappear from retrieval; investigate the first stage. If recall holds but displayed-match quality declines, investigate ordering. Set review windows and minimum sample counts using actual traffic and label availability rather than an invented millisecond target or a single disputed duplicate.

A tight threshold can wake the on-call over labeling ambiguity. A loose one leaves bad suggestions in place while duplicates accumulate. Review false positives alongside confirmed misses, and remove the second pass if its measured top-five gain does not justify its latency and operating cost.

## Further reading

- Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks": https://arxiv.org/abs/2005.11401
- pgvector documentation: https://github.com/pgvector/pgvector
- Qdrant documentation: https://qdrant.tech/documentation/
- Pinecone documentation: https://docs.pinecone.io/
- Weaviate documentation: https://docs.weaviate.io/weaviate

## References

- https://arxiv.org/abs/2005.11401
- https://github.com/pgvector/pgvector
- https://qdrant.tech/documentation/
- https://docs.pinecone.io/
- https://docs.weaviate.io/weaviate
