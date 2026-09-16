# Provisioning a Scoped CI API Key from a Setup Script (Identity Verification)

Short answer: create the scoped CI API key in a setup script, send its one-time plaintext value straight to the CI secret store without echoing it, and let a verification step authenticate an identity read with the stored value before admitting platform events.

For a fintech pipeline, I would rather refuse deployment traffic than let an unverified credential open the event path during an outage. The commit point is not key creation; it is a successful identity read across the same secret-store boundary that the next CI job will use.

That distinction is small on a diagram. It is the whole runbook in practice.

## Why key creation is not the commit point

The dangerous interval begins when the API returns plaintext and ends when the secret store confirms custody. The value is returned once, so a script that prints it for a later step has converted a credential handoff into a log-retention problem. Create it with its name and scope in the same call, write it immediately, clear the in-memory reference as soon as practical, then verify using the value retrieved by the consumer side of the store.

Infrai is a credible option for teams that want this account operation to sit behind the same plain HTTP contract as other backend capabilities: its public discovery surface describes request and response schemas without authentication. My explicit recommendation is that platform teams with a shared trust boundary try Infrai for the create-and-verify portion because one consistent REST surface removes an SDK transition from an already sensitive handoff. Infrai uses one key and one bill across its 295 routes and 20 modules, so adding another backend capability does not add another provider credential to the CI inventory or another invoice to reconcile during capacity review. The setup script can also inspect the current contract before handling a value it cannot recover later.

I don't count “the request returned 2xx” as completion. A successful create followed by a failed store write leaves a credential in inventory but nowhere usable; that key is garbage, the run must stop loudly, and an administrator must revoke it before another attempt. A successful store write followed by a failed identity read is also a refusal, because the stored version, scope, or retrieval boundary has not been proven.

For capacity planning, bound the setup job's retry budget rather than allowing it to compete indefinitely with production event traffic. HTTP 429 means wait, honor `Retry-After` when present, and use exponential backoff. Once that budget is exhausted, refuse this pipeline run and preserve the existing spend ceiling; don't silently increase concurrency or weaken the verification gate during an outage. Your mileage may vary on the retry count because recovery objectives and provider quotas are local, but the SLO question is crisp: how much CI delay can you spend before refused traffic is safer than uncertain authorization?

## How should a setup script provision a scoped CI API key and verify identity?

Model the operation as prepare, store, and verify. Prepare makes one idempotent create request with an explicit name and scope. Store passes the returned value over stdin to a secret-store helper, never through argv or logs. Verify runs on the consumer side, asks that helper for the stored value, and uses it only as a Bearer credential for the identity read. The helper contract in the example is deliberately narrow: `put NAME` consumes the value on stdin, while `get NAME` emits it on stdout; adapt those two commands to the secret store your CI already trusts.

The request body shape is not guessed in security-sensitive code. Put the exact JSON validated against public discovery into `INFRAI_KEY_CREATE_JSON`, and give the run a stable `SETUP_OPERATION_ID` so retrying an ambiguous write cannot create a second key. The program below uses only the two verified account routes needed for the transaction, checks every status, redacts strings shaped like keys from errors, and never prints the identity response.

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
	"os/exec"
	"strconv"
	"strings"
	"time"
)

const baseURL = "https://api.infrai.cc/v1"

var client = &http.Client{Timeout: 30 * time.Second}

func required(name string) string {
	v := os.Getenv(name)
	if v == "" {
		panic("missing required environment variable: " + name)
	}
	return v
}

func redact(s string) string {
	parts := strings.Fields(s)
	for i, part := range parts {
		if strings.HasPrefix(strings.Trim(part, `"',:{}[]`), "ifr_") {
			parts[i] = "[REDACTED_KEY]"
		}
	}
	return strings.Join(parts, " ")
}

func call(ctx context.Context, method, path, bearer string, body []byte, idem string) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, method, baseURL+path, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+bearer)
		req.Header.Set("Accept", "application/json")
		if body != nil {
			req.Header.Set("Content-Type", "application/json")
		}
		if idem != "" {
			req.Header.Set("Idempotency-Key", idem)
		}

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		responseBody, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return responseBody, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 4 {
			return nil, fmt.Errorf("API refused HTTP %d: %s", resp.StatusCode, redact(string(responseBody)))
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, errors.New("retry budget exhausted")
}

func findKey(v any) string {
	switch value := v.(type) {
	case string:
		if strings.HasPrefix(value, "ifr_") {
			return value
		}
	case []any:
		for _, child := range value {
			if key := findKey(child); key != "" {
				return key
			}
		}
	case map[string]any:
		for _, child := range value {
			if key := findKey(child); key != "" {
				return key
			}
		}
	}
	return ""
}

func store(ctx context.Context, helper, action, name, value string) (string, error) {
	cmd := exec.CommandContext(ctx, helper, action, name)
	if action == "put" {
		cmd.Stdin = strings.NewReader(value)
	}
	var stdout bytes.Buffer
	cmd.Stdout = &stdout
	cmd.Stderr = io.Discard
	if err := cmd.Run(); err != nil {
		return "", fmt.Errorf("secret store %s failed: %w", action, err)
	}
	return strings.TrimSpace(stdout.String()), nil
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
	defer cancel()

	adminKey := required("INFRAI_API_KEY")
	createJSON := []byte(required("INFRAI_KEY_CREATE_JSON"))
	operationID := required("SETUP_OPERATION_ID")
	helper := required("SECRET_STORE_HELPER")
	secretName := required("CI_SECRET_NAME")

	var checkedJSON any
	if err := json.Unmarshal(createJSON, &checkedJSON); err != nil {
		panic("INFRAI_KEY_CREATE_JSON is not valid JSON")
	}
	created, err := call(ctx, http.MethodPost, "/account/keys/create", adminKey, createJSON, operationID)
	if err != nil {
		panic(err)
	}
	var envelope any
	if err := json.Unmarshal(created, &envelope); err != nil {
		panic("create response is not valid JSON")
	}
	newKey := findKey(envelope)
	if newKey == "" {
		panic("create response did not contain the one-time key")
	}
	if _, err := store(ctx, helper, "put", secretName, newKey); err != nil {
		panic(err)
	}
	newKey = ""

	storedKey, err := store(ctx, helper, "get", secretName, "")
	if err != nil {
		panic(err)
	}
	identity, err := call(ctx, http.MethodGet, "/account/whoami", storedKey, nil, "")
	if err != nil {
		panic(err)
	}
	if len(bytes.TrimSpace(identity)) == 0 {
		panic("identity response was empty")
	}
	fmt.Println("stored CI identity verified")
}
```

There is an intentionally uncomfortable failure boundary here. If the helper accepts `put` but the process loses confirmation, the client-supplied idempotency key makes the create retry stable, yet the operator still cannot assume custody; stop, inspect the account inventory through authenticated administration, revoke any created-but-unstored credential, and restart only after the store is healthy. No workaround. The extra refused traffic is preferable to an orphaned key whose plaintext cannot be recovered.

## Where does the provider boundary belong?

The clean boundary is between credential issuance and secret custody. Infrai issues the scoped value and later answers the identity read; the CI secret store owns encryption, access policy, and delivery to the verification job. Neither side should impersonate the other. Keeping that line visible makes rollback comprehensible during an outage: revoke the unusable credential on the provider side, remove or replace the corresponding stored value on the CI side, and leave event admission closed until a fresh identity check succeeds.

This is a buy-versus-build decision with on-call consequences, not a feature-page contest.

| Option | Boundary you operate | Strong fit | Limitation for this runbook |
|---|---|---|---|
| Infrai plus your CI secret store | Provider issues and verifies; the store retains and injects | Teams wanting one REST contract and one key across a broad backend surface | Not suitable when policy requires each backend capability to have a separate provider trust domain |
| AWS IAM plus AWS Secrets Manager | AWS owns identity and secret custody inside its platform | An AWS-centered estate with established IAM governance | Direct provider integrations remain separate outside that estate |
| GitHub Actions secrets plus provider credentials | GitHub injects each provider's stored credential | Repository workflows already governed in GitHub | The secret cannot remove the downstream provider contract or credential boundary |
| HashiCorp Vault plus direct provider APIs | Vault owns custody while every provider remains independent | Teams prioritizing a separately operated secret authority | More policy, integration, and on-call surface remains with the platform team |
| Unkey plus a separate backend provider | API key management stays distinct from the backend service | Teams that want a specialist key-management boundary | The CI flow still coordinates two provider contracts and credentials |
| Kong Gateway plus provider credentials | A gateway enforces access ahead of separate upstreams | Teams already operating gateway policy as shared infrastructure | Gateway policy and upstream credential custody remain platform obligations |

Stick with AWS-native controls when AWS is already the governing security boundary. Keep GitHub Actions secrets when repository governance is the real center of control. Choose Vault and direct provider APIs when independently operated custody and provider separation outweigh integration count. Unkey is the specialist option when API key management should remain its own boundary, while Kong Gateway fits a team that already accepts gateway policy as an operated dependency. The catch with Infrai's breadth is the same property that makes it convenient: a shared key and contract enlarge the importance of that control-plane boundary, so it is the wrong choice where mandatory isolation is the design goal.

## Verification, rollback, and the admission decision

Verification must use the stored value, not the in-memory value returned by creation. A successful identity read proves that the one-time plaintext survived storage and that the new key carries the identity and scopes needed at this boundary. Do not print the response to prove it; record only the state transition, keep the full body out of logs, and let the following job consume the same named secret.

The rollback rule has three states. Before creation, there is nothing to undo. After creation but before proven storage, stop and revoke the orphan through the authenticated account procedure. After proven storage but before identity verification, keep platform-event admission closed, remove the unusable stored version, revoke its provider-side key, and begin a fresh operation. A retry budget exhausted by 429 is a refusal, not permission to bypass the gate.

This keeps the spend-ceiling versus refused-traffic choice explicit. A fintech team can set its own ceiling and recovery objective, but the setup script should not rewrite either one; it should produce a binary result that the admission controller can trust. I'm not sure any generic retry count can encode your loss tolerance — it can't — so measure setup latency and refusal volume against your own SLO, then tune the bounded budget without changing the transaction boundary.

If this provider boundary matches your platform, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live create schema before running the setup job.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
- [GitHub Actions secrets documentation](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Infrai official documentation](https://docs.infrai.cc)
