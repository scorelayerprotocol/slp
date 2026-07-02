# ScoreLayer Protocol (SLP)

**A trust layer for AI agents operating across organizational boundaries.**

When an AI agent calls an API, makes a payment, or accesses infrastructure it doesn't own, the system on the receiving end has no standard way to answer three basic questions: *Who is this agent? How has it behaved before? What is it actually running right now?*

SLP is an open specification that answers those questions.

---

## The problem, in one sentence

OAuth tells you a caller has a valid token. It doesn't tell you whether that agent has a track record of staying in scope, whether it's been flagged for anomalous behavior elsewhere, or what model/inference stack is generating its calls at this exact moment — and as of 2026, agents are making cross-organizational API calls at a volume no human can review one relationship at a time.

## What SLP defines

| Component | What it does |
|---|---|
| **Agent Passport** | A signed identity document bound to the agent itself — survives model switches, redeployments, and infrastructure changes |
| **Behavioral Trust Score (BTS)** | A reputation score built from verified interactions across every organization the agent has dealt with |
| **Inference Stack Attestation (ISA)** | A signed, per-call declaration of what model/runtime the agent is actually using |
| **Verification API** | A lightweight way for any perimeter (API gateway, payment processor, infra provider) to check an inbound agent in real time |

SLP is model-agnostic and doesn't require replacing your existing OAuth or API key setup — it sits alongside it as an additional trust check at the perimeter.

SLP also treats a few predictable failure modes as first-class design problems rather than afterthoughts: an agent flagged as anomalous by one perimeter can be formally disputed rather than silently penalized, reputation is bound to the issuing organization (not just the agent) so a bad actor can't launder a poor track record by re-registering, and revocation is severity-tiered so a compromised key doesn't get the same lenient grace period as routine deregistration.

## Status

**Draft — RFC v0.1, open for co-authorship.** This is not a finished standard; it's a starting point for the industry to argue over, improve, and eventually converge on. See the [full RFC](./SLP-RFC-v0_1.md) for the complete specification, including known limitations, open questions, and an honest section on the current governance tension (Section 5) between "this spec is open" and "the reference registry today is run by one company." A federation design document and an open-source reference Registry implementation are committed within 90 days of this RFC's publication (Section 4.6) — the single-operator state described above is meant to be temporary, not the intended end state.

SLP is one of several active efforts in this space (see Section 7 of the RFC for a direct comparison with APS, ERC-8004, and others). It does not claim to be the only answer — it's a bet that cross-organizational behavioral reputation deserves its own specification.

## Read the RFC

The full spec, including the Agent Passport schema, the BTS formula, delegation flow, security considerations, and worked test vectors, is here:

**[SLP-RFC-v0_1.md](./SLP-RFC-v0_1.md)**

## Contributing

This is explicitly open for outside co-authorship — including from people at competing efforts.

- **Propose a change:** open a GitHub Issue titled `[RFC] your proposal`
- **Become a co-author:** submit a Pull Request with a substantive section revision or a resolution to one of the [10 open questions](./SLP-RFC-v0_1.md#6-open-questions)
- **Areas we specifically want help with:** PKI/certificate infrastructure, OAuth/OIDC at scale, API gateway security, agent framework internals (LangChain, CrewAI, AutoGen, LlamaIndex), distributed systems identity

Contact: rfc@scorelayer.ai

## License

MIT. Not owned by any single company — see Section 5 of the RFC for what that does and doesn't mean in practice today.
