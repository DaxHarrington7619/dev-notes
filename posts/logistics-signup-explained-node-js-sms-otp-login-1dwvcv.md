# Logistics Signup Explained: Node.js SMS OTP Login API Ownership

The most important choice in a Node.js SMS OTP login flow is not the sender API. It is who owns the template and the security claims embedded in it. **Short answer:** keep the canonical logistics-signup template, its version, the resend policy, and the verification policy under the same application change control; treat SMS delivery as a replaceable transport. A resend cooldown slows message creation, while a separate verification limit constrains guessing. Neither control proves that a handset received the link.

I model the failure as a bounded incident exercise: a carrier representative starts creating an account, requests a second verification link when the first text appears late, and then opens the older message. No provider outage is required. If the template says one thing about expiry while the verifier does another, or if two independently created links remain active, the operator has no clean answer to a basic question: which promise governed this signup?

One version must win. No exceptions.

The invariant is therefore about ownership, not branding: every issued challenge records the policy version and template version that described it, and verification consults application-owned state rather than message-delivery status. This advice does not make SMS an appropriate authenticator for every account. NIST SP 800-63B describes use of the public switched telephone network for out-of-band authentication as restricted, so an assurance requirement may rule out the channel before template design begins.

## How should logistics teams own a Node.js SMS OTP login API?

A verification template carries security meaning. It names the requested action, identifies the account context, presents the link, and may state an expiration interval or an instruction for an unexpected request. Copy review is therefore part of the authentication release, even if another system ultimately renders the bytes and submits them to a carrier.

The difficult boundary is change authority. An application team can own source text while allowing an external renderer to hold a deployed copy, but then deployment needs a version comparison and a rollback path. Giving a separate administrative workflow unrestricted ownership makes small copy edits fast; it also allows the message's claim to move without the verifier. I would accept slower edits here because an SMS signup message is executable policy in prose, not a campaign asset. The trade-off is concrete: the platform team gains a reviewable security release and takes on rendering tests, version reconciliation, and an extra rollback dependency.

| Ownership model | Change control | Evidence available during review | On-call consequence |
|---|---|---|---|
| Application owns and renders | Code review and application release | Source version, rendered digest, policy version | More rendering responsibility; one rollback boundary |
| Application owns source, separate system renders | Code review plus template deployment | Source and deployed versions, variables, policy version | Drift detection and coordinated rollback are required |
| Separate system owns source and rendering | Administrative publishing workflow | Template identifier and whatever history that system retains | Fast edits; security copy and verifier behavior can diverge |

This is the useful buy-versus-build split. Buying transport can remove carrier integration work from the platform backlog. It does not justify exporting authorization state, copy approval, or audit semantics, because those are coupled to the product's account model. Self-hosting transport has the opposite cost profile: more control, plus routing, delivery-event handling, capacity planning, and a larger on-call surface. Lock-in matters at the adapter, but ambiguous ownership is the sharper risk.

That is the boundary.

Capacity planning follows the same separation. Size message submission for legitimate signup attempts plus bounded retries; size verification for link opens, invalid tokens, scanners, and concentrated abuse against one destination. Those workloads do not have the same arrival pattern. A single shared counter hides that fact and makes an ordinary retry consume the wrong budget.

## A versioned challenge before a transport call

The preventative path should make four decisions before contacting any sender: whether a challenge may be issued, which prior challenge becomes invalid, which immutable template version applies, and what identifier connects subsequent evidence. The exact limits are policy inputs, not standards. For an initial test configuration, a team could exercise a 60-second resend cooldown, five sends per pseudonymous destination per hour, a 10-minute challenge lifetime, and five failed verification attempts per challenge. Those values are deliberately concrete so race tests and dashboards have thresholds; production values require an abuse model, observed traffic, and an SLO review.

Keep the controls separate. Cooldown answers “may another message be submitted now?” The rolling issuance limit answers “has this subject or network source consumed its delivery budget?” The attempt limit answers “may this active secret be tested again?” Resetting all three on resend turns a usability action into a way to refresh the guessing budget.

The following Go code is the small authorization core behind a Node.js HTTP handler. The handler can call this service through an internal typed boundary; it should not construct sender-specific payloads or select remote templates. Storage is expressed as a transactional interface because a process-local lock would fail as soon as the service has two replicas.

```go
package signup

import (
	"context"
	"crypto/rand"
	"crypto/sha256"
	"crypto/subtle"
	"encoding/base64"
	"errors"
	"fmt"
	"time"
)

var (
	ErrLimited = errors.New("signup verification limited")
	ErrInvalid = errors.New("signup verification invalid")
)

type Challenge struct {
	SubjectID       string
	Generation      uint64
	TokenDigest     [32]byte
	TemplateVersion string
	PolicyVersion   string
	ExpiresAt       time.Time
	NextSendAt      time.Time
}

type Store interface {
	// Replace atomically applies issuance limits and makes next the sole active generation.
	Replace(ctx context.Context, next Challenge, now time.Time) error
	// Consume atomically checks generation state, expiry, the attempt limit, and single use.
	Consume(ctx context.Context, subjectID string, digest [32]byte, now time.Time) error
}

type Sender interface {
	Send(ctx context.Context, destination, body, idempotencyKey string) error
}

type Service struct {
	Store   Store
	Sender  Sender
	Now     func() time.Time
	BaseURL string
}

func (s Service) Issue(
	ctx context.Context,
	subjectID, destination string,
	generation uint64,
) error {
	now := s.Now().UTC()
	raw := make([]byte, 32)
	if _, err := rand.Read(raw); err != nil {
		return fmt.Errorf("generate token: %w", err)
	}

	token := base64.RawURLEncoding.EncodeToString(raw)
	digest := sha256.Sum256([]byte(token))
	next := Challenge{
		SubjectID: subjectID, Generation: generation, TokenDigest: digest,
		TemplateVersion: "logistics-signup-v4", PolicyVersion: "signup-otp-v2",
		ExpiresAt: now.Add(10 * time.Minute), NextSendAt: now.Add(60 * time.Second),
	}
	if err := s.Store.Replace(ctx, next, now); err != nil {
		return err
	}

	link := fmt.Sprintf("%s/signup/verify?subject=%s&token=%s", s.BaseURL, subjectID, token)
	body := fmt.Sprintf("Verify your logistics account signup: %s", link)
	key := fmt.Sprintf("signup:%s:%d:%s", subjectID, generation, next.TemplateVersion)
	if err := s.Sender.Send(ctx, destination, body, key); err != nil {
		return fmt.Errorf("submit verification message: %w", err)
	}
	return nil
}

func (s Service) Verify(ctx context.Context, subjectID, token string) error {
	if subtle.ConstantTimeEq(int32(len(token)), 0) == 1 {
		return ErrInvalid
	}
	digest := sha256.Sum256([]byte(token))
	if err := s.Store.Consume(ctx, subjectID, digest, s.Now().UTC()); err != nil {
		return ErrInvalid
	}
	return nil
}
```

The illustrative path stores only a SHA-256 digest of a 32-byte random token and submits state before transport. That ordering can leave a valid generation whose message was not delivered, but the reverse ordering permits a recipient to open a credential before the verifier knows it exists. Preserve the authorization invariant. On an uncertain send result, retry the same generation with the same idempotency key; do not mint another token merely because a network deadline expired.

The store implementation carries most of the risk. `Replace` needs one conditional write or transaction covering cooldown, rolling limits, generation replacement, and the new record. `Consume` needs one atomic operation covering expiry, failed-attempt accounting, active generation, and single use. Token values must stay out of logs, traces, analytics, and callback metadata. Public failures should remain coarse enough that callers cannot distinguish an unknown account from an expired or exhausted challenge, while restricted telemetry retains the internal reason.

## Release the copy and verifier as one security change

A template release is complete only when the deployed renderer and verifier agree. Before enabling `logistics-signup-v4`, render it with representative account data, verify that the generated link targets the expected origin, check that stated expiry matches `signup-otp-v2`, and record both versions in the issuance event. If rendering happens outside the application, compare the deployed template digest or immutable version with the approved artifact. A mutable name such as `current` is not evidence.

Test sequences rather than isolated handlers. Race two resend requests at the 60-second boundary. Delay generation 41 until after generation 42 is active. Open generation 42 twice. Run five bad attempts and then present the correct token. Retry a timed-out submission with its original idempotency key. Inject latency into the limiter and confirm the chosen failure mode; silently failing open converts a dependency problem into an abuse path. Then repeat the test with most requests aimed at one depot contact, because aggregate throughput can look healthy while one transactional key queues every signup for that location. The SLO should describe the user-visible signup verification path, while operational indicators preserve its stages: issuance decisions, submission results, later delivery events when available, verification latency, stale-generation opens, and successful consumption. **Transport acceptance is not handset receipt.** A delivery event is useful evidence, but authorization must never wait for it or derive from it.

Test the hot key.

Page on sustained impairment of the signup path or exhaustion of a security dependency, not on every rejected resend. A rejection can mean the policy is working. A rise in stale-generation opens deserves investigation rather than an automatic resend storm: delayed delivery, confusing text, and an impatient client produce the same surface symptom but demand different corrections.

## Where this design stops applying

This pattern has real limitations. It is a poor fit for devices or workflows that cannot open links safely, and it should not be used where the required assurance excludes the SMS channel. A numeric one-time code is an alternative for feature phones, although it still needs expiry, single use, throttling, and protected recovery. A shared depot phone may prove possession of a number without proving which employee is acting. Higher-assurance access needs an authenticator selection that meets the assurance requirement; template discipline cannot upgrade the underlying channel.

Email fallback is a separate delivery system, not a free duplicate of SMS. Domain owners can publish DMARC policies for messages that fail identifier alignment, as specified by RFC 7489, but DMARC does not replace application challenge state or prove that a person controls an account. Keep the same application-owned security semantics across channels while versioning channel-specific copy independently.

The final decision rule is narrow: own the parts that define authorization, auditability, and the message's security promise; buy or build transport according to on-call load, required control, and lock-in tolerance. For a logistics signup link, that leaves the Node.js application in charge of challenge state and approved template versions, with the sender behind an adapter. It is a less glamorous boundary than choosing an API. It is also the boundary an incident review can defend.

## Sources

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://datatracker.ietf.org/doc/html/rfc7489
