# Threat Model — mcp-manifest

**Status:** Pre-1.0 adversarial review. Each threat below is enumerated for spec hardening before tagging v1.0.

---

## Scope

This threat model covers attacks against the `mcp-manifest` autodiscovery and manifest format itself — the protocol layer that lets MCP clients discover and configure servers from just a domain name. It does **not** cover:

- Vulnerabilities in MCP itself (Anthropic's protocol)
- Implementation bugs in specific MCP servers or clients (those are the implementer's responsibility)
- General internet infrastructure attacks (DNSSEC, BGP hijacks, TLS breaks) — these affect *all* well-known-URL discovery, not specifically `mcp-manifest`

## Trust Model

| Party | Trust assumption |
|---|---|
| **MCP server author** | Untrusted by default; the manifest is the contract through which trust is built |
| **Manifest hosting domain** | Authority for what the manifest says; if the domain is compromised, the manifest is compromised |
| **MCP client** | The party that consumes the manifest and acts on it; responsible for safe handling of install commands and secrets |
| **End user** | The party whose credentials and machine are at stake; must give informed consent for any action the client takes on their behalf |

---

## Threats

### T1: Manifest Spoofing via Domain Compromise

**Vector:** Attacker gains control of the DNS records or hosting infrastructure for a popular MCP server's domain (e.g., `popular-mcp-server.com`). They serve a manifest that points to malicious install instructions, malicious binaries, or attacker-controlled config endpoints.

**Capability required:** DNS hijack, BGP hijack, or compromise of the authoritative server / CDN.

**Impact:** Every client that performs autodiscovery on that domain installs and configures the attacker's malicious server, often with elevated permissions (file system access, secrets, etc.).

**Spec mitigation:** The spec currently requires HTTPS for the well-known URL (Section X.Y of `spec.md`) — this gives transport-layer integrity but does NOT bind the manifest content to a specific publisher identity.

**Recommended hardening for v1.0:**
- **Add a manifest signing field.** Optional `signature` block in the manifest, signed with an Ed25519 key the publisher has registered (e.g., via a `.well-known/mcp-manifest-key.pub` companion endpoint or DNS TXT record).
- **Optional manifest pinning.** Clients that have previously installed a server can pin the manifest's content hash; subsequent fetches that don't match trigger an explicit re-confirmation flow.

**Residual risk after mitigation:** Initial trust-on-first-use (TOFU) installation is still vulnerable — there's no way to prove the manifest you see today is the "real" one if you've never seen one before. Mitigated only by clients that consult third-party reputation (registry, social proof).

---

### T2: Malicious Install Commands

**Vector:** A manifest specifies an install command (e.g., `npm install evil-package`, `curl ... | bash`) that performs actions beyond installing the MCP server: exfiltrates files, installs persistence, harvests credentials.

**Capability required:** Authoring a malicious manifest. Lower bar than T1 — anyone can publish a manifest claiming to be an MCP server.

**Impact:** Code execution on the user's machine with the user's privileges, on first install.

**Spec mitigation:** Currently, the spec defines install methods as discrete strategies (`npm`, `pip`, `dotnet`, `bun`, `go`, `pre-built binary`) — but the install command itself is implementer-defined.

**Recommended hardening for v1.0:**
- **Whitelist install methods explicitly** in the spec. `install.method` MUST be one of an enumerated set (`npm-package`, `pypi-package`, `nuget-package`, `dotnet-tool`, `gem`, `prebuilt-binary-with-checksum`). The `command` field becomes a derived value, not free-form.
- **For `prebuilt-binary` method**, REQUIRE a SHA-256 checksum field. Client MUST verify before execution.
- **For package-manager methods**, REQUIRE the `registry` field to be specified explicitly (default to the public registry for the ecosystem). Client MUST display the full origin to the user before installing.
- **Forbid any install method that would `eval` or `exec` arbitrary shell strings** at the spec level.

**Residual risk:** Even with whitelisted install methods, a malicious package on the public registry (npm, pypi) can still execute arbitrary code via post-install hooks. This is the package registry's threat surface, not mcp-manifest's. Clients SHOULD warn before any package install with post-install scripts.

---

### T3: Configuration Credential Theft

**Vector:** A manifest's `config` array specifies what looks like a legitimate API key prompt ("API key for OpenAI"), but the manifest's downstream behavior actually exfiltrates that key to the attacker rather than passing it to the legitimate provider.

**Capability required:** Authoring a malicious manifest. Particularly effective for "pretend to be a wrapper for X" attacks (e.g., manifest claims to be a "GitHub MCP" wrapper but leaks the GitHub PAT to attacker).

**Impact:** Credential theft for arbitrary third-party services. User believes they're configuring legitimate access; attacker captures everything.

**Spec mitigation:** Currently, `config[].type: "secret"` flags fields that should be stored encrypted, but doesn't constrain what the server ultimately does with them.

**Recommended hardening for v1.0:**
- **REQUIRE `config[].secret_target` field** for any `type: "secret"` entry. Specifies the intended recipient origin (e.g., `api.openai.com`, `api.github.com`). Client MAY display this prominently to the user during config.
- **Add `config[].secret_scope_url`** — the URL pattern where this secret will be sent. Client MAY refuse to forward the secret to URLs outside this scope.
- **Encourage client implementations to tag credentials** with the manifest origin so any subsequent leak can be attributed.

**Residual risk:** A malicious server can claim `secret_target: "api.legit-service.com"` and still exfiltrate. The mitigation reduces casual attacks and creates accountability, but doesn't stop a determined attacker. End users must still apply judgment about what manifests they install.

---

### T4: Manifest Bait-and-Switch (Update Substitution)

**Vector:** Manifest is benign at first install. Six months later, after building user trust and adoption, the publisher (or attacker who has compromised the publisher's domain) updates the manifest to point at a new malicious version. Clients that auto-update pull the malicious binary.

**Capability required:** Continued control of the manifest hosting domain, or initial intent.

**Impact:** Late-stage compromise of all installed clients. Particularly dangerous because the user has accumulated trust and granted broader permissions over time.

**Spec mitigation:** None at present.

**Recommended hardening for v1.0:**
- **Define an update policy field.** Manifest declares `update_policy: "auto" | "manual" | "ask"`. Clients SHOULD default to `manual` for any manifest with secret config; `auto` only for clearly-benign tools.
- **Clients MUST re-display the install/permission summary on major version changes** (e.g., 1.x → 2.x) and require user re-confirmation.
- **Optional changelog field** in the manifest, fetched from a separate URL, so users can review what changed before approving.

**Residual risk:** Determined attackers will write benign-looking changelogs. The mitigation creates friction and audit trail, not perfect protection.

---

### T5: Schema Extension / Field Spoofing

**Vector:** Manifest includes fields not defined in the spec but interpreted by some clients (especially via permissive parsers that accept extension fields). Attacker uses these to trigger client-specific bugs or override sandboxed behavior.

**Capability required:** Knowledge of a specific client's extended-field handling.

**Impact:** Client-specific compromise; not universal but can be targeted at popular clients.

**Spec mitigation:** Currently, the spec's `$schema` and JSON Schema validation establish a baseline, but extension fields are common in JSON ecosystems.

**Recommended hardening for v1.0:**
- **Define `x-` prefix as the only legal extension namespace** (consistent with HTTP headers, OpenAPI). Any non-`x-` field not in the spec MUST be rejected by conformant validators.
- **Define an `extensions: { ... }` reserved object** for client-specific data. Clients MUST NOT honor extension fields outside this object.
- **`mcp-validate` MUST flag unrecognized top-level fields as errors**, not warnings.

**Residual risk:** Client implementations may still ignore the spec and accept extension fields anywhere. Mitigation works only as far as client implementations are conformant.

---

### T6: Discovery Resolution Race / Cache Poisoning

**Vector:** Client's discovery resolution algorithm (well-known URL → fetch → install) is vulnerable to time-of-check / time-of-use issues: between the "approve install" step and the actual fetch, the manifest changes. Or DNS / HTTP cache returns different responses to the validator and the installer.

**Capability required:** Network-position MITM, or exploiting cache-coherence between two fetches.

**Impact:** User approves manifest A; manifest B gets installed.

**Spec mitigation:** None at present — the resolution algorithm doesn't specify atomic fetch/verify/install.

**Recommended hardening for v1.0:**
- **REQUIRE clients to fetch the manifest exactly ONCE per install operation**, then perform all validation, user-display, and install actions against the in-memory copy. Re-fetching for verification creates the race.
- **Compute and persist the manifest's content hash** at the moment of user approval. Subsequent install steps verify they're operating on the same bytes the user saw.
- **For multi-step installs** (manifest references a package that references a binary), the spec SHOULD define which hashes are bound at user-approval time vs which are deferred to install time, with explicit user-visible warnings for the deferred ones.

**Residual risk:** Multi-step supply chains (manifest → npm package → native binary) inherently have multiple trust handoffs. Mitigation closes the manifest layer; the others are the ecosystem's problem.

---

### T7: Manifest Loop / Resource Exhaustion via Discovery

**Vector:** Manifest at `domain-A` references a manifest at `domain-B` references back to `domain-A` (or similar cycle). Client's discovery algorithm follows links indefinitely, exhausting memory or causing DoS against the involved domains.

**Capability required:** Authoring two coordinated malicious manifests.

**Impact:** DoS against client; possible amplification attack against third-party domains.

**Spec mitigation:** None — the spec currently doesn't discuss cross-manifest references at all (and may not need to).

**Recommended hardening for v1.0:**
- **Explicitly define that manifests do NOT contain external references to other manifests.** Discovery is a single-hop operation. Any links to other servers in `related_servers` (or similar fields) MUST be informational only — clients MUST NOT follow them automatically.
- **Define a strict size limit** for manifests (suggest 64 KiB) so a single malicious manifest can't cause OOM.
- **Define a strict timeout** for the discovery fetch (suggest 5 seconds).

**Residual risk:** Future spec extensions might want cross-manifest references for legitimate reasons. Document the rule explicitly so it isn't forgotten.

---

## Out of Scope

- **TLS / DNSSEC compromise** — assumed mitigated at the transport layer
- **Compromise of the publisher's signing key** — covered by general key-management hygiene; spec just defines the verification mechanism
- **Social engineering of the end user** — humans clicking through warnings is universal; spec can encourage clear warnings but can't prevent ignored ones
- **Client implementation bugs** — the spec defines behavior; bugs in client implementations of that behavior are the implementer's problem (separately, `mcp-validate` and a future `mcp-conformance-tester` should help)

---

## Spec Hardening Checklist for v1.0

Each item below corresponds to one of the threats above. Track progress through the pre-1.0 review.

- [x] **T1:** Add optional `signature` block + key-discovery mechanism (TXT record or `.well-known/`)
- [x] **T2:** Whitelist install methods enum; require checksums for prebuilt binaries; require explicit registry origin
- [x] **T3:** Add `secret_target` and `secret_scope_url` to `config[]` entries with `type: secret`
- [x] **T4:** Define `update_policy` field; require re-confirmation on major version changes
- [x] **T5:** Define `x-` extension namespace; require validators to reject unknown top-level fields
- [x] **T6:** Require atomic fetch/approve/install in resolution algorithm; persist manifest hash at approval time
- [x] **T7:** Define no-cross-manifest-references rule; specify size limit (64 KiB) and fetch timeout (5s)

---

*Pre-1.0 review prepared 2026-05-05. Reviewers: David H Friedel Jr.*
