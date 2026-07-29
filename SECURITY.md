# Security

## Reporting a vulnerability

If you believe you have found a security vulnerability in the mcp-manifest
specification or its reference implementations, please report it privately to:

**security@ai-manifests.org**

Please do not file a public GitHub issue for security reports. We aim to
acknowledge receipt within 72 hours and to publish a coordinated disclosure
within 90 days, whichever comes first.

## Threat model

The full adversarial review for v1.0 is in [THREAT-MODEL.md](./THREAT-MODEL.md).
It enumerates seven threat classes (T1 – T7): manifest spoofing via domain
compromise, malicious install commands, configuration credential theft,
manifest bait-and-switch (update substitution), schema-extension /
field spoofing, discovery resolution race / cache poisoning, and manifest
loop / resource exhaustion via discovery.

Each spec mitigation is tagged inline in [spec.md](./spec.md) (e.g., "(T1)",
"(T2)") so reviewers can trace requirements to the threats they address.

## In scope

- Spoofing or tampering of manifest contents
- Injection or instruction-poisoning of manifest text fields
  (`server.description`, `config[].description`, `config[].prompt`) when those
  fields are passed to LLM-driven installers (see spec §Security Model)
- Atomic-resolution race conditions between discovery and install
- Bypass of signature, checksum, or schema-validation requirements
- Misuse of the `extensions` namespace by conformant validators
- Cross-manifest references that the spec forbids

## Out of scope

- TLS / DNSSEC / BGP compromise (mitigated at transport layer)
- Compromise of an honest publisher's signing key
  (general key-management hygiene)
- Social engineering of end users past warnings
- Client implementation bugs (these belong to the implementer's project)
- Real-money payment rails or post-install package-registry threats
  (npm, PyPI, etc., have their own threat surfaces)

## Related work

The agent-integration-layer attack surface (poisoned skill definitions,
malicious MCP tool descriptions, instruction injection through configuration
artifacts) is an industry-wide concern documented in 2026 by Cisco's
AI Agent Security Scanner research, Snyk's ToxicSkills audit, and the
"Supply-Chain Poisoning Attacks Against LLM Coding Agent Skill Ecosystems"
paper. mcp-manifest v1.0 mitigates the structural vectors (signing,
checksums, atomic verify/install, schema closure) but cannot inspect the
semantic intent of free-form text fields. Clients that route manifest text
through LLM contexts MUST sanitize before inclusion (see spec §Security Model
"Untrusted text fields").
