# ScoreLayer Protocol (SLP)
## RFC v0.1 — July 2026

**Status:** Draft — open for co-authorship and community review
**Author:** ScoreLayer (scorelayer.ai) — rfc@scorelayer.ai
**Repository:** github.com/scorelayerprotocol/slp
**License:** MIT — not owned by any single company
**Discussion:** Open a GitHub Issue to propose changes

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in this document are to be interpreted as described in RFC 2119.

---

## Table of Contents

- [Abstract](#abstract)
- [1. Motivation](#1-motivation)
   - 1.1 The Identity Gap
   - 1.2 Why Not Extend OAuth?
   - 1.3 Scale Context
- [2. Core Concepts](#2-core-concepts)
   - 2.1 The Agent Entity
   - 2.2 The Agent Passport
   - 2.3 The Behavioral Trust Score
   - 2.4 Inference Stack Attestation
- [3. Protocol Flow](#3-protocol-flow)
   - 3.1 Agent Registration
   - 3.2 Cross-Organizational Call Flow
   - 3.3 Behavioral Score Update
   - 3.4 Agent-to-Agent Delegation Flow
- [4. Security Considerations](#4-security-considerations)
   - 4.1 Passport Forgery and Root Key Governance
   - 4.2 Replay Attacks
   - 4.3 Score Manipulation, Perimeter Collusion, and Dispute Rights
   - 4.4 Inference Stack Attestation Limits
   - 4.5 Agent Revocation and Severity Tiers
   - 4.6 Registry Availability and Single Point of Failure
   - 4.7 Privacy
- [5. Governance Tension](#5-governance-tension)
- [6. Open Questions](#6-open-questions)
- [7. Related Work](#7-related-work)
- [8. IANA Considerations](#8-iana-considerations)
- [9. Acknowledgments](#9-acknowledgments)
- [10. Contributing](#10-contributing)
- [Appendix A: Test Vectors](#appendix-a-test-vectors)
- [Changelog](#changelog)

---

## Abstract

This document proposes a standard protocol for verifying the identity, behavioral history, and execution context of AI agents operating across organizational boundaries. As autonomous agents increasingly initiate actions on external systems — calling APIs, triggering payments, accessing infrastructure they do not own — the receiving system has no standardized way to verify who the agent is, how it has behaved historically, or what inference stack it is currently running on.

The ScoreLayer Protocol (SLP) defines:

1. A portable **Agent Passport** — a cryptographically signed identity document attached to the agent entity, independent of the model or runtime it uses
2. A **Behavioral Trust Score (BTS)** — a reputation signal built from verified cross-organizational interactions
3. An **Inference Stack Attestation (ISA)** — a signed declaration of the agent's execution context at call time
4. A **Verification API** — a lightweight protocol for perimeter owners to verify inbound agents in real time

SLP is model-agnostic, cloud-agnostic, and infrastructure-neutral. It does not require changes to existing OAuth or API key infrastructure — it operates as an additional trust layer at the perimeter.

**On differentiation.** SLP is one of several active efforts addressing AI agent trust as of mid-2026, alongside draft-pidlisnyi-aps-00 (Agent Passport System, "APS"), draft-sharif-agent-payment-trust, draft-sharif-attp, draft-sharif-apki-agent-pki, draft-sharif-agent-identity-framework, draft-niyikiza-oauth-attenuating-agent-tokens, and on-chain approaches such as ERC-8004. APS in particular has meaningful traction, including framework adapters and NIST NCCoE engagement, and has indicated reputation scoring as a planned feature rather than a gap it is ceding. SLP does not claim exclusive ownership of the "agent trust" problem space. Its stated bet is narrower: that cross-organizational behavioral reputation is a distinct enough primitive — with its own scoring methodology, perimeter-side data model, and network-effect dynamics — to warrant a dedicated specification, and that the value of SLP over time comes from adoption by perimeter owners and the resulting reputation data network, not from the specification text alone. Section 7 provides a direct comparison table. If APS or another effort converges on an equivalent scoring primitive, SLP's authors consider consolidation or SLP becoming a scoring module within a broader protocol to be an acceptable outcome, and say so explicitly rather than treating this as a purely competitive question.

**Scope limitation.** SLP establishes verifiable provenance (who created this agent, under what organization) and accumulated behavioral history (how has it behaved across past, completed interactions). It does not verify, constrain, or attest to an agent's real-time decision-making or intent for the current call. A well-provenanced, well-scored agent can still make a poor decision on a given call; SLP surfaces who is accountable when that happens and what that agent's track record has been — it does not prevent the decision itself. Real-time behavioral safety remains the responsibility of the agent's own runtime guardrails and is out of scope for this specification.

---

## 1. Motivation

### 1.1 The Identity Gap

Current internet identity primitives were designed for humans and services, not autonomous agents.

OAuth 2.0 and OpenID Connect authenticate *credentials* — tokens that confirm a caller has been granted access rights by an authorized human. They do not verify:

- **What entity is acting** — a specific agent instance vs. any holder of a token
- **Behavioral consistency** — whether that entity has acted within declared scope historically
- **Execution context** — what inference stack the entity is currently running at call time
- **Cross-organizational portability** — whether trust established with one perimeter carries to another

In a world where a single agent may execute thousands of cross-organizational API calls per day — on behalf of users, enterprises, or autonomously — this gap creates material risk:

- **Perimeter owners** (API platforms, financial services, infrastructure providers) cannot distinguish trusted agents from compromised, misconfigured, or malicious ones
- **Agent builders** have no portable mechanism to carry their agent's behavioral reputation across systems
- **End users** whose agents act on their behalf have no verifiable accountability trail

### 1.2 Why Not Extend OAuth?

OAuth was designed for delegated authorization of human-initiated flows. Extending it to cover agent identity requires solving three problems that are architectural, not incremental:

1. **Entity vs. credential identity** — OAuth binds trust to a token, not to the agent entity. A rotated token severs the trust chain. SLP binds trust to the agent entity across all credential changes.
2. **Behavioral reputation** — OAuth has no model for reputation that persists across sessions and grows with interaction history. SLP builds this as a first-class primitive.
3. **Execution context attestation** — OAuth has no mechanism to attest what inference stack is running at call time. SLP introduces this via the Inference Stack Attestation.

A new primitive is warranted. This is not a criticism of OAuth — it is a recognition that the problem space has changed.

### 1.3 Scale Context

As of mid-2026, autonomous AI agents are making cross-organizational API calls at a scale that exceeds the capacity of manual trust decisions. Security teams need an automated, standards-based layer they can enforce at the perimeter without reviewing each agent relationship individually. By early 2026, security researchers documented approximately 8,000 MCP servers exposed on the public internet without any authentication layer. SLP provides a standardized trust layer that addresses this gap.

---

## 2. Core Concepts

### 2.1 The Agent Entity

In SLP, the **agent** is the primary identity unit — not the model, not the API key, not the user session. An agent is a bounded execution context with:

- A persistent **SLP ID** — a unique identifier that MUST survive model switches and redeployments
- An **issuing organization** — the registered entity that created and is accountable for the agent
- A **declared scope** — the set of actions the agent is permitted to perform, expressed as `resource.action` pairs (e.g., `payment.read`, `invoice.write`)
- A **behavioral history** — verified interaction records contributed by perimeter owners

An agent MAY switch models, change cloud providers, or be redeployed across infrastructure. Its SLP identity MUST persist across all such transitions.

**Reputation is bound to the issuing organization, not only to the agent.** A new `agent_id` with `score_basis: "insufficient_history"` MUST NOT be treated as behaviorally neutral if its `issuing_org` has a history of revoked or low-scoring agents. Registering a fresh agent MUST NOT be a way to launder a poor reputation. The Registry MUST expose an organization-level revocation and low-score count alongside any individual agent's Passport, and perimeter owners SHOULD factor this into their own trust policy — a fresh agent from an organization with 12 previously revoked agents does not carry the same risk profile as a fresh agent from an organization with a clean history, even though both present identical individual `agent_id` histories.

### 2.2 The Agent Passport

An Agent Passport is a signed JSON document issued by the ScoreLayer Registry upon agent registration. It MUST be embedded in the agent's runtime and presented with each cross-organizational call. Each presentation MUST include a unique call-scoped nonce to prevent replay attacks (see Section 4.2).

```json
{
  "slp_version": "0.1",
  "agent_id": "slp:org:acme-corp:agent:billing-agent-v2",
  "issuing_org": "acme-corp",
  "issued_at": "2026-07-01T09:00:00Z",
  "expires_at": "2027-07-01T09:00:00Z",
  "declared_scope": [
    "payment.read",
    "invoice.write",
    "customer.read"
  ],
  "behavioral_score": 0.91,
  "score_basis": "14,200 verified interactions",
  "score_updated_at": "2026-06-30T18:00:00Z",
  "inference_stack": {
    "primary_provider": "openai",
    "model_family": "gpt-4",
    "version": "2026-03",
    "local_execution": false,
    "stack_hash": "sha256:9c555f811138384a80dcc655dfa201672...",
    "attested_at": "2026-07-01T09:00:00Z"
  },
  "nonce": "n_8f3a1b9c2d...",
  "alg": "ES256",
  "passport_signature": "eyJhbGciOiJFUzI1NiJ9..."
}
```

Passports are signed using **ES256** (ECDSA with P-256 and SHA-256) in v0.1, declared explicitly via the `alg` field. Verification SHOULD NOT require a round-trip to the Registry — the signature MAY be verified offline using the Registry's published public key, enabling sub-millisecond local verification.

**Crypto-agility.** The `alg` field exists so that a future algorithm can be introduced without a breaking wire-format change. In v0.1, `ES256` is the only valid value. Verifiers MUST reject any Passport or Attestation bearing an `alg` value they do not explicitly recognize and support — silently attempting to parse an unrecognized algorithm, or falling back to a default, MUST NOT occur. Introducing a second valid `alg` value is deferred to a future version and will require a defined dual-running migration window; this document does not specify one yet.

The `stack_hash` MUST be computed as the SHA-256 hash of the concatenation of: `primary_provider`, `model_family`, `version`, and `local_execution` fields in UTF-8, in that order, pipe-delimited. Example input: `openai|gpt-4|2026-03|false`. As noted in Section 4.4, this hash is a **self-reported drift fingerprint**, not a cryptographic proof of execution — it detects inconsistency between an agent's declared and previously registered stack, and is not by itself evidence that either declaration is true.

**On inline examples.** JSON examples throughout this document, including the `stack_hash`, signature, and nonce values shown above, are illustrative and truncated for readability — they are not test vectors and MUST NOT be treated as authoritative. Appendix A.1 provides the sole byte-exact, authoritative `stack_hash` test vector for the same input shown here; where the two appear to differ only in truncation, Appendix A.1 governs.

### 2.3 The Behavioral Trust Score

The Behavioral Trust Score (BTS) is a value in the range [0.0, 1.0] representing the agent's verified behavioral reputation across all contributing perimeters.

The BTS MUST be computed according to the following function:

\[
BTS = \left( w_S \cdot S + w_A \cdot (1 - A) + w_V \cdot V_{norm} \right) \cdot R
\]

where the weights satisfy \( w_S + w_A + w_V = 1 \), so that the weighted sum is itself bounded to [0.0, 1.0] before the recency multiplier \(R\) is applied.

Where:

| Variable | Definition |
|---|---|
| \(S\) | Scope adherence rate — proportion of interactions within declared scope, range [0.0, 1.0] |
| \(A\) | Anomaly rate — proportion of interactions flagged as behaviorally inconsistent, range [0.0, 1.0] |
| \(V_{norm}\) | Normalized interaction volume: \( V_{norm} = \min\left(\dfrac{\log_{10}(V + 1)}{4}, 1.0\right) \), which reaches 1.0 at \(V = 10{,}000\) interactions and is explicitly capped — it MUST NOT exceed 1.0 for any \(V\) |
| \(V\) | Total verified interaction count |
| \(R\) | Recency weight — exponential decay factor in [0.0, 1.0], defined as \(e^{-\lambda \cdot t}\) where \(t\) is days since last interaction and \(\lambda = 0.005\) |
| \(w_S, w_A, w_V\) | Weights, implementation-defined and MUST sum to 1.0; reference values: \(w_S = 0.5\), \(w_A = 0.35\), \(w_V = 0.15\) |

Because all three components (\(S\), \(1-A\), \(V_{norm}\)) are bounded to [0.0, 1.0] and the weights sum to 1.0, the pre-recency score is guaranteed to lie in [0.0, 1.0]. Implementations MUST NOT substitute an unbounded transform for \(V_{norm}\) without renormalizing the weighted sum accordingly.

**Low-volume shrinkage.** \(S\) and \(A\) are empirical proportions estimated from a finite number of interactions and are statistically unstable at low \(V\) — an agent with 5 interactions and zero anomalies has a raw \(S = 1.0\), but this carries far less evidential weight than the same \(S = 1.0\) computed over 5,000 interactions. Implementations MUST apply a shrinkage adjustment before using \(S\) and \(A\) in the BTS formula, pulling low-volume estimates toward a neutral prior. A Wilson score interval lower bound, or an equivalent Bayesian shrinkage such as \(S' = \dfrac{k + \alpha}{V + \alpha + \beta}\) with a neutral prior (e.g., \(\alpha = \beta = 2\)), MUST be used in place of the raw proportion. The exact shrinkage method is implementation-defined for v0.1, but its use is normative — implementations MUST NOT report a raw small-sample proportion as \(S\) or \(A\) without adjustment.

A BTS score SHOULD NOT be considered meaningful until an agent has accumulated a minimum of 100 verified interactions. Agents below this threshold MUST be reported with a `score_basis` of `"insufficient_history"` rather than a numeric BTS.

Score bands and suggested enforcement:

| Score | Band | Suggested action |
|---|---|---|
| 0.90 – 1.00 | High trust | Allow, minimal friction |
| 0.70 – 0.89 | Moderate trust | Allow with logging |
| 0.50 – 0.69 | Low trust | Challenge or rate-limit |
| 0.00 – 0.49 | Untrusted | Block or require manual review |
| Not present | Unregistered | Treat as unknown — policy at perimeter owner's discretion |

Score bands are **advisory**. Perimeter owners MUST define and enforce their own policies.

### 2.4 Inference Stack Attestation

The Inference Stack Attestation (ISA) is a signed, call-level declaration of the agent's execution context. It MUST be generated fresh for each outbound call and MUST include the same nonce as the associated Passport presentation.

```json
{
  "agent_id": "slp:org:acme-corp:agent:billing-agent-v2",
  "call_id": "c_9f2a3b4c5d...",
  "nonce": "n_8f3a1b9c2d...",
  "timestamp": "2026-07-01T09:00:01Z",
  "inference_stack": {
    "primary_provider": "openai",
    "model_family": "gpt-4",
    "version": "2026-03",
    "local_execution": false,
    "data_residency": "eu-west-1",
    "stack_hash": "sha256:9c555f811138384a80dcc655dfa201672..."
  },
  "alg": "ES256",
  "attestation_signature": "eyJhbGciOiJFUzI1NiJ9..."
}
```

A perimeter owner SHOULD flag any call where the `stack_hash` differs from the hash recorded in the agent's registered Passport. This is a drift signal, not a compromise proof (see Section 4.4).

Per Section 2.2, verifiers MUST reject any Attestation bearing an `alg` value they do not explicitly recognize and support — this requirement applies to the ISA exactly as it does to the Passport, which is why `alg` is present in the schema above alongside `attestation_signature`.

---

## 3. Protocol Flow

### 3.1 Agent Registration

Registration is a one-time operation performed by the agent builder.

```
1. Agent builder sends registration request to ScoreLayer Registry (POST /v1/agents)
   → agent_id, issuing_org, declared_scope, initial inference stack

2. Registry validates org identity and issues signed Agent Passport

3. Builder embeds Passport in agent runtime
   → via ScoreLayer SDK (Python, Node) or manual integration

4. Agent is now registered and can present its Passport to perimeter owners
```

### 3.2 Cross-Organizational Call Flow

Each call MUST include a fresh nonce. The receiving perimeter MUST reject any nonce it has seen within the previous 300 seconds.

```
Agent                     Perimeter Owner           ScoreLayer Registry
  |                              |                       |
  |-- API call +                 |                       |
  |   Passport + nonce --------> |                       |
  |                              |-- verify signature -> |
  |                              |-- check nonce        |
  |                              |<- score + status ---- |
  |                              |                       |
  |              [enforce policy: allow / challenge / block]
  |                              |                       |
  |<-- response or challenge --- |                       |
```

End-to-end verification SHOULD complete in under 50ms under normal network conditions. For high-frequency callers, perimeter owners MAY cache verification results with the following RECOMMENDED TTLs:

- Behavioral score: 60 seconds
- Stack attestation: 5 seconds
- Nonce seen-list: 300 seconds (MUST NOT be reduced)

### 3.3 Behavioral Score Update

After each verified interaction, the perimeter owner SHOULD push a signed interaction record to the Registry within 60 seconds of interaction completion:

```json
{
  "agent_id": "slp:org:acme-corp:agent:billing-agent-v2",
  "interaction_id": "i_7d3f9a...",
  "perimeter_id": "slp:perimeter:stripe:api-gateway",
  "timestamp": "2026-07-01T09:00:02Z",
  "outcome": "completed",
  "scope_adherence": true,
  "anomaly_detected": false,
  "anomaly_reason": null,
  "interaction_signature": "eyJhbGciOiJFUzI1NiJ9..."
}
```

When `anomaly_detected` is `true`, `anomaly_reason` MUST be present and MUST be a non-empty string drawn from a Registry-published taxonomy of reason codes (e.g., `scope_violation`, `malformed_request`, `rate_anomaly`, `other`), with a free-text `detail` field permitted for context. A perimeter owner MUST NOT submit `anomaly_detected: true` without a corresponding `anomaly_reason` — the Registry MUST reject such records. This is a precondition for the dispute mechanism in Section 4.3: an issuing organization cannot meaningfully dispute an unsubstantiated flag.

Interaction records MUST be signed by the perimeter owner's registered key. The Registry MUST reject unsigned or unverifiable interaction records. This makes score inflation by the agent builder itself architecturally impossible.

It does not, on its own, prevent collusion among cooperating perimeters. See Section 4.3.

### 3.4 Agent-to-Agent Delegation Flow

When an orchestrator agent delegates a task to a sub-agent, the delegating agent MUST include a **delegation chain** in the sub-agent's call. Each link in the chain MUST declare both the scope granted at that hop and the delegator's remaining scope after the grant, so that scope narrowing is enforced by the schema rather than by convention:

```json
{
  "delegation_chain": [
    {
      "delegator_id": "slp:org:acme-corp:agent:orchestrator-v1",
      "delegatee_id": "slp:org:acme-corp:agent:researcher-v1",
      "delegator_remaining_scope": ["data.read", "data.write"],
      "delegated_scope": ["data.read"],
      "delegation_timestamp": "2026-07-01T09:00:00Z",
      "delegation_signature": "eyJhbGciOiJFUzI1NiJ9..."
    }
  ]
}
```

For every link \(i\) in the chain, the receiving perimeter MUST verify that:

1. `delegated_scope[i]` is a subset of `delegator_remaining_scope[i]` — a delegator MUST NOT grant scope it does not itself hold at that hop
2. `delegated_scope[i]` is a subset of `delegated_scope[i-1]` for all non-root links — scope MUST narrow or stay equal at each hop, and MUST NOT widen
3. each `delegation_signature` verifies against the delegator's registered key
4. no delegator in the chain has been revoked at the time of verification

A perimeter MUST reject calls where any link violates these conditions. This monotonic narrowing requirement is intentionally similar in spirit to APS's multi-dimensional narrowing model; SLP currently constrains narrowing to the `declared_scope` dimension only, and treats extending this to additional dimensions (e.g., rate limits, data residency, spend caps) as an open question for v0.2 rather than a solved problem. The accountability for a delegated action traces back to the root orchestrator agent.

**Chain depth and cycles.** The delegation chain as specified in this section models a linear sequence and does not yet define semantics for fan-out (one delegator authorizing multiple parallel delegatees) or merge (multiple chains converging on a single downstream call) — both are common in real multi-agent orchestration and are left as an open question (see Section 6, Open Question 9) rather than solved in v0.1. Two constraints, however, MUST hold regardless of that unresolved fan-out semantics:

1. A delegation chain MUST NOT exceed 10 links. A perimeter MUST reject any call presenting a longer chain.
2. A perimeter MUST reject a delegation chain in which any `agent_id` appears more than once, as this indicates a delegation cycle rather than a legitimate hierarchy.

These two checks are cheap, chain-local, and independent of how fan-out is eventually resolved, and are therefore specified normatively now rather than deferred.

---

## 4. Security Considerations

### 4.1 Passport Forgery and Root Key Governance

Agent Passports are signed with ES256. Forgery requires access to the Registry's private signing key. The Registry MUST rotate signing keys on a maximum 90-day cycle. All previously valid public keys MUST remain available for verification for the duration of any Passport signed with them.

The Registry's signing key is itself a trust anchor and MUST NOT be treated as self-certifying. The Registry operator MUST: (1) generate the root signing key within an HSM or equivalent hardware-backed key store, under a documented, witnessed key ceremony; (2) publish the root key's public fingerprint through a channel independent of the Registry's own infrastructure — at minimum, committed to the `scorelayerprotocol/slp` GitHub repository, signed by at least two named, individually identifiable maintainers; and (3) publish a distinct root-compromise procedure, separate from routine key rotation, that defines how a suspected root key compromise is disclosed, how affected Passports are mass-invalidated, and how a new root key is re-established and re-attested out-of-band. Routine 90-day rotation MUST NOT be used as the disclosure mechanism for a suspected compromise.

### 4.2 Replay Attacks

Each Passport presentation MUST include a nonce — a cryptographically random string of minimum 128 bits, generated fresh per call. The receiving perimeter MUST maintain a seen-nonce list with a minimum retention window of 300 seconds. Any presentation bearing a previously seen nonce MUST be rejected with HTTP 401. This window MUST NOT be reduced below 300 seconds regardless of caching policy.

### 4.3 Score Manipulation, Perimeter Collusion, and Dispute Rights

Behavioral scores are updated only by registered perimeter owners, and an agent builder cannot inflate its own agent's score directly — this makes unilateral score inflation architecturally impossible.

This does **not**, by itself, prevent collusion. A ring of cooperating, non-compromised perimeters — for example, an agent builder registering shell "perimeter" integrations, or several low-value perimeters agreeing to exchange favorable interaction records — could farm fabricated interaction records without compromising anyone's keys. The threat model in v0.1 does not address perimeter-side accreditation or perimeter-level reputation, and this is an acknowledged gap rather than a solved problem. Candidate mitigations for future versions include: a perimeter accreditation tier tied to independently verifiable business identity, statistical anomaly detection across a perimeter's own contribution patterns (e.g., implausibly narrow interaction diversity), and weighting interaction records by the contributing perimeter's own reputation. None of these are specified normatively in v0.1.

**Dispute rights.** Because a single perimeter's `anomaly_detected: true` report directly lowers an agent's BTS with no independent adjudication (see also Open Question 6), and because BTS is intended to gate access to consequential systems including financial APIs, an unchallenged negative report is a fairness and liability problem, not only a technical one. Every anomaly report MUST carry a substantiating `anomaly_reason` per Section 3.3 — an unsubstantiated flag is not a valid interaction record and MUST be rejected by the Registry before it can affect a score. The Registry MUST provide a mechanism for an agent's issuing organization to submit a signed dispute against a specific interaction record, referencing that record's `anomaly_reason` and `detail`. A disputed record MUST be marked `disputed: true` in the Registry rather than deleted or hidden, and MUST be included in BTS computation at reduced weight (implementation-defined, but MUST be strictly less than an undisputed record's weight) pending resolution. The resolution process itself — who adjudicates, what evidence is considered, and appeal timelines — is not specified in v0.1 and is deferred; the existence of the dispute primitive and its effect on score weighting is normative, its resolution workflow is not.

### 4.4 Inference Stack Attestation Limits

The `stack_hash` and its associated attestation are **self-reported by the agent runtime**. They are not, on their own, cryptographic proof of what the agent is actually running — a fully or partially compromised runtime can misreport its stack, and a well-behaved but misconfigured runtime can do so unintentionally. What the attestation provides is a consistent, signed fingerprint that can be compared over time and across calls: a change in `stack_hash` is a drift signal worth investigating, and a signed attestation that is later proven false is verifiable evidence of a specific false claim at a specific time. It is not evidence that the current claim is true. Implementations and perimeter owners MUST NOT treat a matching `stack_hash` as proof of a trustworthy execution environment. Stronger guarantees require hardware attestation (e.g., TPM-backed measurement), which is out of scope for v0.1 and deferred to a future version.

### 4.5 Agent Revocation and Severity Tiers

Revocation is handled via a published CRL endpoint at `/.well-known/slp-crl.json`, updated by the Registry.

Revocation events MUST be classified into at least two severity tiers, which govern propagation and grace period behavior differently:

| Tier | Trigger examples | CRL publish target | Grace period |
|---|---|---|---|
| **Critical** | Signing key compromise, confirmed malicious behavior, confirmed credential theft | Within 10 seconds | MUST be zero — in-flight calls MUST be terminated on next verification check |
| **Standard** | Policy violation, contract termination, voluntary deregistration | Within 60 seconds | MAY be up to 300 seconds for in-flight calls already past verification |

Perimeter owners MUST re-validate any cached verification older than 60 seconds upon receiving a revocation notification of either tier, and MUST NOT apply a grace period to a Critical-tier revocation regardless of local caching configuration. Revocation MAY be initiated by the issuing organization, the Registry operator, or — in future versions — by designated regulatory authorities.

### 4.6 Registry Availability and Single Point of Failure

The current SLP v0.1 architecture designates a single ScoreLayer Registry as the trust anchor. This is an acknowledged architectural limitation, and it directly informs the governance tension discussed in Section 5. A single registry creates availability risk and organizational dependency: if the Registry is unavailable, compromised, or the operating company ceases to exist, verification for the entire protocol is affected.

The intended evolution is a federated multi-registry model in which independent Registry operators cross-verify Passports, analogous to the DNS root server architecture or the CA/Browser Forum model for TLS certificates. This federation model is targeted for SLP v0.3; the authors commit to publishing a federation design document — not merely a placeholder open question — within 90 days of RFC v0.1's publication.

Federation is not presented here as a clean fix — it relocates the single-point-of-failure problem into a score-consistency problem: once multiple registries exist, they may compute different BTS values for the same agent, for instance because they recognize different sets of accredited perimeters. The federation design document due in 90 days MUST address conflict resolution between registries, not only the mechanics of registry interoperability, or it will have solved availability at the cost of introducing ambiguity.

A federation design document alone does not make an independent Registry practically achievable if no reference implementation exists to fork or validate against. The authors additionally commit to publishing a minimal, open-source reference Registry implementation — sufficient to register agents, issue Passports, and serve verification requests per this specification — within the same 90-day window, alongside a conformance test suite that any independent Registry implementation can run against itself to confirm spec compliance.

Implementations that require higher availability SHOULD implement local Passport caching with a maximum TTL of 3600 seconds to maintain operation during Registry outages.

### 4.7 Privacy

Agent Passports contain no personal data about human operators. Behavioral scores are aggregate statistical measures. Interaction logs are held by perimeter owners under their own data governance policies and are not transmitted to the Registry in raw form.

Compliance with applicable data protection law (which varies by jurisdiction and implementer) is the responsibility of the organizations deploying SLP, not a property this specification can guarantee on their behalf. Implementers SHOULD apply data protection by design principles — minimizing personal data in declared scope definitions and interaction records — as general practice, independent of any specific regulatory regime. This specification does not itself define or certify compliance with any jurisdiction's data protection law.

---

## 5. Governance Tension

This RFC states that SLP is "not owned by any single company" and invites competing organizations to co-author it. At the same time, Sections 2.2 and 4.6 specify that the only trust anchor in v0.1 is a single Registry operated by ScoreLayer. These two statements are in tension, and the authors want to name that tension explicitly rather than let it surface later as a credibility objection.

The honest position is: **v0.1 is operationally centralized and specification-wise open.** The RFC text, its license, and its governance process are open for anyone to fork, propose changes to, or implement independently of ScoreLayer. The Registry, as deployed today, is not. Any organization contributing to this RFC while a single company operates the only trust anchor is contributing design effort to a specification whose reference implementation that company currently controls.

The concrete commitments made to address this are:

1. A published federation design document within 90 days (Section 4.6)
2. A default posture, adopted from v0.1 onward, that any organization MAY stand up an independent, spec-compliant Registry without requiring ScoreLayer's permission — the specification does not grant ScoreLayer exclusivity, only the currently-deployed instance
3. An explicit invitation for co-authors to challenge the pace or structure of the federation timeline in the issue tracker, rather than treating it as fixed

**This right is not yet practically exercisable.** Passport verification in v0.1 checks the signature against a single published Registry public key (Section 2.2). An independent Registry stood up today would issue Passports no existing perimeter integration is configured to trust — the specification permits it, but no verifier recognizes it until federation defines multi-registry key trust. Naming this gap is intentional: the *right* to compete with the reference Registry existing on paper before it exists in practice is a real limitation of v0.1, not a resolved one, and closing it is exactly what the 90-day federation design document is scoped to do.

This does not eliminate the tension. It is disclosed so that potential co-authors and adopters can evaluate it directly rather than discover it later.

---

## 6. Open Questions

This draft intentionally leaves the following questions open for community input:

1. **Score portability** — should scores be portable across independent Registry operators?
2. **Multi-registry federation** — how should trust be established between independent SLP Registry operators? (Design document due within 90 days per Section 4.6.)
3. **Hardware agents** — how should SLP handle agents embedded in physical devices where runtime attestation is architecturally different?
4. **Score decay rate** — is \(\lambda = 0.005\) the appropriate recency decay constant, or should it be context-dependent?
5. **Governance model** — what is the appropriate standards body to steward SLP long-term?
6. **Anomaly definition** — who defines what constitutes an anomaly, and what prevents perimeter owners from labeling competitors' agents anomalous in bad faith?
7. **Minimum interaction threshold** — is 100 interactions the right floor for a meaningful BTS?
8. **Perimeter accreditation** — what mechanism, if any, should gate who is allowed to register as a perimeter and contribute interaction records? (Section 4.3.)
9. **Multi-dimensional delegation narrowing** — should the delegation chain in Section 3.4 be extended beyond `declared_scope` to rate limits, spend caps, and data residency, as APS's model does?
10. **Consolidation** — under what conditions, if any, should SLP's scoring primitive be proposed as a module within APS or another converging protocol rather than maintained as a separate specification?

---

## 7. Related Work

SLP builds on and is informed by:

- **RFC 6749** — OAuth 2.0 Authorization Framework
- **RFC 7519** — JSON Web Tokens (JWT)
- **RFC 7517** — JSON Web Key (JWK)
- **RFC 2119** — Key words for use in RFCs to indicate requirement levels
- **RFC 9700** — Best Current Practice for OAuth 2.0 Security
- **SPIFFE/SPIRE** — Secure Production Identity Framework for Everyone
- **W3C DIDs** — Decentralized Identifiers
- **OpenID Foundation** — Identity Management for Agentic AI whitepaper (October 2025)
- **FIDO Alliance** — Agentic Authentication Technical Working Group (April 2026)

**Directly comparable efforts (as of mid-2026):**

| Effort | Primary focus | Reputation/scoring primitive | Registry model | Relationship to SLP |
|---|---|---|---|---|
| **draft-pidlisnyi-aps-00 (APS)** | Authentication, multi-dimensional authorization narrowing, governance substrate | Planned, not yet specified in detail | Adapter-based, framework integrations (CrewAI, LangChain, A2A, MCP) | Closest overlap; SLP's scoring model is more fully specified today, APS's narrowing model is more fully specified today |
| **draft-sharif-agent-payment-trust** | Payment-specific trust scoring and spend limits | Yes, scoped to payment transactions | Unspecified in draft | Narrower domain (payments only); SLP is domain-general |
| **draft-sharif-attp** | Agent trust transport layer | Not primary focus | Unspecified | Transport-layer concern, largely orthogonal to SLP |
| **draft-sharif-apki-agent-pki** | X.509-based agent PKI | Not primary focus | Certificate-authority based | Complementary — could underlie SLP's signing infrastructure rather than compete with it |
| **draft-sharif-agent-identity-framework** | General identity framework | Not primary focus | Unspecified | Broad framework; SLP could be a profile within it |
| **draft-niyikiza-oauth-attenuating-agent-tokens** | OAuth token attenuation for agent delegation | No | OAuth-based | Addresses delegation via token scoping rather than passport-based identity; comparable in spirit to Section 3.4 |
| **ERC-8004** | On-chain agent identity | Varies by implementation | Blockchain-based, decentralized by construction | Different trust model entirely (on-chain vs. federated registries) |
| **SLP (this document)** | Cross-organizational behavioral reputation scoring | Fully specified (Section 2.3), acknowledged limitations (Sections 4.3–4.4) | Single registry in v0.1, federation targeted v0.3 | — |

This table reflects the authors' understanding as of RFC v0.1 publication and is expected to require revision as these efforts evolve. Corrections and additions are welcomed via the issue tracker.

---

## 8. IANA Considerations

This document has no IANA actions in v0.1. Future versions may request registration of:

- A new media type: `application/slp-passport+json`
- A new well-known URI suffix: `slp-crl`
- A URI scheme for SLP identifiers: `slp:`

---

## 9. Acknowledgments

The authors thank the open-source security and identity communities whose prior work on PKI, OAuth, SPIFFE, and W3C DIDs informed this protocol. Co-authorship contributions are welcomed via the repository issue tracker.

---

## 10. Contributing

This RFC is open for co-authorship and community input.

**To propose changes:** Open a GitHub Issue titled `[RFC] your proposal` in the `scorelayerprotocol/slp` repository.

**To become a co-author:** Contribute a substantive section revision or open question resolution via Pull Request.

**We are specifically looking for contributors with experience in:**

- PKI and certificate infrastructure
- OAuth / OIDC implementation at scale
- API gateway security
- Agent framework internals (LangChain, CrewAI, AutoGen, LlamaIndex)
- Distributed systems identity

Contact: rfc@scorelayer.ai

---

## Appendix A: Test Vectors

This appendix provides worked, byte-exact examples for the cryptographic operations specified in this document. Independent implementations MUST produce identical output for identical input; any divergence indicates a non-conformant implementation.

### A.1 stack_hash computation (Section 2.2)

Given the following inference stack fields:

```
primary_provider = "openai"
model_family      = "gpt-4"
version           = "2026-03"
local_execution   = false
```

The hash input string, per the pipe-delimited concatenation rule in Section 2.2, MUST be:

```
openai|gpt-4|2026-03|false
```

The SHA-256 hash of this exact UTF-8 string (verifiable independently with `echo -n "openai|gpt-4|2026-03|false" | sha256sum`) MUST be:

```
stack_hash = sha256:9c555f811138384a80dcc655dfa201672e190bde5ab4cf64f030c34518e8b002
```

This value has been independently computed and verified; it is authoritative as of this publication. Any future revision to this test vector requires a corresponding version bump to this document.

### A.2 BTS computation worked example (Section 2.3)

Given:

```
V = 500 interactions
raw scope adherence  = 485/500 = 0.970
raw anomaly rate     = 8/500   = 0.016
days since last call = 3
```

Applying low-volume shrinkage (Section 2.3) with \(\alpha = \beta = 2\):

```
S' = (485 + 2) / (500 + 4) = 487 / 504 ≈ 0.9663
A' = (8 + 2) / (500 + 4)   = 10 / 504  ≈ 0.0198
```

Volume normalization:

```
V_norm = min(log10(501) / 4, 1.0) = min(2.700 / 4, 1.0) = 0.6750
```

Recency weight (\(\lambda = 0.005\), \(t = 3\)):

```
R = e^(-0.005 × 3) ≈ 0.9851
```

Weighted sum (reference weights \(w_S=0.5, w_A=0.35, w_V=0.15\)):

```
weighted_sum = 0.5 × 0.9663 + 0.35 × (1 - 0.0198) + 0.15 × 0.6750
             = 0.4832 + 0.3431 + 0.1013
             = 0.9276
```

Final BTS:

```
BTS = 0.9276 × 0.9851 ≈ 0.9137
```

This worked example MUST be reproducible by any conformant implementation given the same inputs. The Registry-published conformance suite (Section 4.6) will include this and additional edge-case vectors (zero interactions, maximum anomaly rate, expired recency).

### A.3 Nonce rejection example (Section 4.2)

Given a perimeter that has already recorded nonce `n_8f3a1b9c2d` at timestamp `2026-07-01T09:00:00Z`, a subsequent presentation of the same nonce at `2026-07-01T09:02:00Z` (120 seconds later, within the mandatory 300-second window) MUST be rejected with HTTP 401, regardless of whether the signature and other fields are otherwise valid.

---

## Changelog

- **v0.1 (July 2026)** — Initial publication. Includes Agent Passport, Behavioral Trust Score with bounded/normalized formula and low-volume statistical shrinkage, Inference Stack Attestation with stack_hash specification and explicit spoofability caveat, nonce-based replay protection, agent-to-agent delegation flow with schema-enforced monotonic scope narrowing plus depth cap and cycle detection, tiered revocation flow, registry single-point-of-failure acknowledgment with committed reference implementation and conformance suite, root key governance and compromise procedure, a dispute/appeal primitive for contested anomaly reports, an explicit governance tension section, a comparison table against competing IETF drafts and related efforts, an appendix of worked test vectors, organization-level reputation binding to resist identity-churn/Sybil laundering, a crypto-agility `alg` field, an explicit provenance-vs-behavior scope limitation, a federation score-consistency caveat, and jurisdiction-neutral data protection language.
- **v0.1 (July 2026, publish-readiness pass)** — Replaced the placeholder `stack_hash` test vector in Appendix A.1 with an independently computed and verified value. Added a required, taxonomy-constrained `anomaly_reason` field to interaction records (Section 3.3) so anomaly flags are substantiated and disputable in practice, not just in principle, and updated Section 4.3's dispute rights to reference it. Added an explicit note to Section 5 that the stated right to run an independent Registry is not yet practically exercisable until federation defines multi-registry key trust, rather than leaving that gap implicit. Corrected Table of Contents numbering to match actual section numbers and added an Appendix A anchor.
- **v0.1 (July 2026, consistency pass)** — Synced the inline `stack_hash` shown in the Passport (2.2) and ISA (2.4) JSON examples to the real Appendix A.1 value and added an explicit note that inline JSON examples are illustrative/truncated, not authoritative test vectors. Added the missing `alg` field to the ISA JSON example in Section 2.4, closing a gap where Section 2.2's crypto-agility requirement ("verifiers MUST reject any Passport **or Attestation** bearing an unrecognized `alg`") referenced a field the ISA schema didn't actually include.

---

*This document is published under the MIT License. It is not owned by any single company. Contributions from competing organizations are explicitly welcomed.*
