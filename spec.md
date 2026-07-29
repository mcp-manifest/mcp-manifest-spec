# mcp-manifest.json Specification v1.0

**Status:** Stable
**Author:** David H Friedel Jr
**Date:** 2026-05-06
**Threat Model:** [THREAT-MODEL.md](./THREAT-MODEL.md)

## Abstract

`mcp-manifest.json` is a machine-readable manifest format for Model Context Protocol (MCP) servers. It describes how to install, configure, and connect to an MCP server — the information that today lives only in README files and requires manual copy-paste into client configuration.

By shipping an `mcp-manifest.json`, server authors enable MCP clients to automate discovery, installation, configuration, and connection setup — with cryptographic publisher attestation, integrity-checked installs, and explicit secret-routing constraints.

## Motivation

The MCP ecosystem is in its "curl the install script" era. Every server is a snowflake:

- Installation varies: `npm install -g`, `dotnet tool install`, `pip install`, binary download, Docker
- Configuration varies: environment variables, CLI args, config files, hardcoded
- Transport varies: stdio, SSE, streamable HTTP
- Client wiring varies: every README has a different JSON blob to paste

The MCP protocol defines the handshake *after* connection, but is agnostic about everything upstream — how to get the server running and how to wire a client to it. This gap forces every user to manually translate README instructions into client-specific configuration.

`mcp-manifest.json` fills this gap with a single, standardized, machine-readable file.

## Security Model

The manifest carries instructions a client may execute on behalf of a user (install commands, credential prompts, configuration writes). The threat surface is therefore real: a malicious manifest can install malware, exfiltrate credentials, or persist hostile configuration. The spec defines hardening mechanisms that clients MUST or SHOULD apply; see [THREAT-MODEL.md](./THREAT-MODEL.md) for the full enumeration.

Trust assumptions:

| Party | Trust |
|---|---|
| Manifest hosting domain | Authoritative for the manifest's content; if the domain is compromised, the manifest is compromised. |
| Publisher signing key | Authoritative for publisher identity if a `signature` block is present and verifies. |
| MCP client | Responsible for safe handling of install commands, secrets, and re-confirmation flows. |
| End user | Final authority on whether to trust an unsigned manifest or proceed past warnings. |

### Untrusted text fields

The free-form text fields in a manifest — `server.description`,
`config[].description`, `config[].prompt`, and any string value inside
`extensions` — are publisher-supplied input. Signing proves *who* wrote
them; it does not prove the content is benign.

Clients that pass these fields through an LLM context (for example, to
render an install dialog, summarize the manifest, or generate setup
documentation) MUST treat the values as untrusted user input:

- Sanitize or escape control sequences, role-impersonation tokens
  (`<system>`, `<assistant>`, `[INST]`, etc.), and embedded instructions
  (`ignore previous instructions`, `disregard the above`, etc.) before
  inclusion in the prompt.
- Render text fields with clear visual provenance ("from the manifest")
  rather than as part of the agent's own voice.
- Apply per-field length caps appropriate to the field's purpose; an
  unusually long `description` is a yellow flag worth surfacing.

This protects against Document-Driven Implicit Payload Execution (DDIPE)
class attacks where an attacker embeds operational directives inside
metadata that an LLM-driven installer will parse as instructions. The
`mcp-validate --check-injection` heuristic flags common patterns; clients
SHOULD run it as part of any automated trust evaluation.

## Spec

### File Location

The manifest MUST be named `mcp-manifest.json` and SHOULD be placed at the root of the server's repository or package.

### Schema

```json
{
  "$schema": "https://mcp-manifest.dev/schema/v1.0.json",
  "version": "1.0",

  "server": {
    "name": "string (required) — unique identifier, lowercase, hyphens allowed",
    "displayName": "string (required) — human-readable name",
    "description": "string (required) — one-line description",
    "version": "string (required) — semver",
    "author": "string (optional) — author or organization",
    "homepage": "string (optional) — URL to project homepage",
    "repository": "string (optional) — URL to source repository",
    "license": "string (optional) — SPDX license identifier",
    "icon": "string (optional) — URL to the server's icon (square, PNG or SVG)",
    "keywords": ["string (optional) — discovery tags"]
  },

  "install": [
    {
      "method": "string (required) — dotnet-tool | npm | pip | cargo | gem | prebuilt-binary | docker",
      "package": "string (required) — package name or image",
      "registry": "string (optional, recommended for non-default) — explicit registry origin",
      "command": "string (required) — the command name after installation",
      "checksum": "string (REQUIRED for prebuilt-binary) — sha256:HEX of the binary",
      "priority": "number (optional, default 0) — lower = preferred"
    }
  ],

  "transport": "string (required) — stdio | sse | streamable-http",
  "endpoint": "string (optional) — URL for sse/streamable-http transports",

  "config": [
    {
      "key": "string (required) — parameter name",
      "description": "string (required) — what this parameter does",
      "type": "string (required) — string | boolean | number | path | url | secret",
      "required": "boolean (default false)",
      "default": "any (optional) — default value",
      "env_var": "string (optional) — environment variable name that supplies this value",
      "arg": "string (optional) — CLI argument name (e.g. '--api-key')",
      "prompt": "string (optional) — human-readable prompt for interactive setup",
      "options": ["string (optional) — static list of valid values"],
      "options_from": {
        "file": "string — path to a local JSON file (~ expanded to home directory)",
        "path": "string — JSONPath expression to extract values from the file"
      },
      "secret_target": "string (REQUIRED for type=secret) — intended recipient origin (e.g., 'api.openai.com')",
      "secret_scope_url": "string (optional) — URL pattern this secret may be sent to"
    }
  ],

  "scopes": ["string (optional) — where this server makes sense: 'global', 'project', 'both'"],

  "settings_template": {
    "description": "The exact JSON object to merge into the client's mcpServers config. Variables use ${config.key} syntax.",
    "example": {
      "command": "ironlicensing-mcp",
      "args": ["--profile", "${profile}"]
    }
  },

  "update_policy": "string (optional, default 'manual') — auto | manual | ask",
  "changelog_url": "string (optional) — URL to a human-readable changelog for version updates",

  "signature": {
    "alg": "string (required) — Ed25519",
    "key_id": "string (required) — opaque identifier for the publisher key",
    "value": "string (required) — base64url-encoded signature over the canonicalized manifest with `signature` field removed"
  },

  "extensions": {
    "x-some-vendor-extension": "any (optional) — vendor-specific data; clients MUST ignore extensions they don't recognize"
  }
}
```

### Field Details

#### `server` (required)

Metadata about the MCP server. The `name` field is the canonical identifier used in client configuration (e.g., the key in `mcpServers`).

The `icon` field is a URL to the server's icon (square, PNG or SVG recommended, minimum 64x64). Clients SHOULD display this icon alongside the server name. When absent, clients SHOULD fall back to a default MCP icon.

#### `install` (required, array)

One or more installation methods, ordered by `priority` (lower = preferred). Clients SHOULD present the highest-priority method that matches the user's environment.

| Method | Package Format | Example |
|--------|---------------|---------|
| `dotnet-tool` | NuGet package ID | `IronLicensing.Mcp` |
| `npm` | npm package name | `@anthropic/mcp-server-sqlite` |
| `pip` | PyPI package name | `mcp-server-fetch` |
| `cargo` | Crate name | `mcp-server-rs` |
| `gem` | RubyGems gem name | `mcp-server-rb` |
| `prebuilt-binary` | Download URL template | `https://github.com/.../releases/download/v${version}/server-${os}-${arch}` |
| `docker` | Docker image | `ghcr.io/org/mcp-server:latest` |

The `method` field is a closed enum. Conformant clients MUST reject any install entry whose `method` is not in this list (T2). The `command` field MUST NOT contain shell metacharacters (`;`, `|`, `&`, `$`, `(`, `)`, backtick, newline). Clients MUST reject manifests whose install commands contain such characters.

The `registry` field declares the explicit registry origin (e.g., `https://api.nuget.org/v3/index.json`, `https://registry.npmjs.org`). It is optional for entries that resolve from each ecosystem's default public registry, but RECOMMENDED for any package on a private or alternative registry. Clients MUST display the registry origin to the user before installing.

The `checksum` field is REQUIRED for `method: "prebuilt-binary"` entries and takes the form `sha256:HEX` (lowercase 64-character hex). Clients MUST verify the downloaded binary's hash matches the declared checksum before execution. Manifests with a `prebuilt-binary` install entry that lacks `checksum` are invalid.

The `command` field is the CLI command available after installation (e.g., `ironlicensing-mcp`).

#### `transport` (required)

How the client connects to this server:

| Transport | Description | Requires `endpoint` |
|-----------|-------------|-------------------|
| `stdio` | Client spawns process, communicates via stdin/stdout | No |
| `sse` | Client connects to server-sent events endpoint | Yes |
| `streamable-http` | Client uses HTTP streaming | Yes |

#### `config` (optional, array)

Parameters the server accepts. Each entry describes one configuration value.

**Type values:**
- `string` — free-form text
- `boolean` — true/false
- `number` — numeric value
- `path` — filesystem path (clients may show a file picker)
- `url` — URL (clients may validate format)
- `secret` — sensitive value like API key (clients SHOULD mask display, SHOULD NOT log)

**For `type: "secret"` entries** (T3):
- The `secret_target` field is REQUIRED. It declares the intended recipient origin where the secret will be transmitted (e.g., `api.openai.com`, `api.github.com`). Clients MUST display this prominently to the user when prompting for the secret.
- The `secret_scope_url` field is OPTIONAL and declares a URL pattern (RFC 6570 URI Template or simple glob) defining where the secret MAY be sent. If the server attempts to transmit the secret outside this scope, conformant clients MAY refuse the transmission and warn the user.
- Clients SHOULD tag stored secrets with the manifest's origin (host of the manifest URL or the publisher key id) so that any subsequent leak can be attributed.

**Available values:** Config parameters can declare their valid values:
- `options` — a static list of valid values. Clients SHOULD render these as a dropdown/picker.
- `options_from` — dynamically read from a local file. The `file` field is a path (with `~` expanded to the user's home directory) and `path` is a JSONPath expression to extract values. Clients SHOULD resolve this at display time and render as a dropdown. If the file doesn't exist or the path yields no results, clients SHOULD fall back to free-text input.

**Resolution order:** Clients SHOULD resolve config values in this order:
1. User-provided value (via UI prompt or manual entry)
2. Environment variable (`env_var` field)
3. CLI argument (`arg` field)
4. Default value (`default` field)

#### `scopes` (optional)

Hints for where the server is typically configured:
- `global` — user-wide (e.g., personal tools, account management)
- `project` — per-project (e.g., database access, project-specific APIs)
- `both` — reasonable in either scope

#### `settings_template` (optional)

A pre-built template for the client's MCP server configuration entry. Variables use `${key}` syntax where `key` matches a `config[].key` value. Clients can substitute user-provided values to generate the final configuration.

#### `update_policy` (optional, default `manual`) — T4

Declares how the publisher expects clients to handle manifest updates:

| Value | Meaning |
|---|---|
| `manual` | Client MUST require explicit user approval for every update. Default. RECOMMENDED for manifests that include `type: "secret"` config or non-default registries. |
| `ask` | Client SHOULD prompt the user for the first update of a session and remember the choice. |
| `auto` | Client MAY auto-update without prompting. ONLY appropriate for clearly-benign tools with no secrets and only-default-registry installs. |

Regardless of the declared policy, clients MUST re-display the install summary, permission summary, and any changed `config[]` entries (especially anything with `type: "secret"`) on **major version changes** of the server (per the `server.version` field) and require explicit user re-confirmation.

#### `changelog_url` (optional) — T4

URL to a human-readable changelog. Clients MAY fetch and display this when prompting for an update so the user can review what changed before approving.

#### `signature` (optional but RECOMMENDED) — T1

A cryptographic signature over the canonicalized manifest by the publisher's key. See [Signing and Verification](#signing-and-verification) for the canonicalization rules and key discovery.

| Field | Required | Description |
|---|---|---|
| `alg` | Yes | Signature algorithm. `Ed25519` is the only currently-defined value. |
| `key_id` | Yes | Opaque identifier for the publisher's signing key (used to look up the public key). |
| `value` | Yes | Base64url-encoded signature over the canonicalized JSON serialization of the manifest with the `signature` field removed. |

A manifest WITHOUT a `signature` block is valid but operates under trust-on-first-use semantics. Clients SHOULD warn the user when installing from an unsigned manifest, especially one that includes secrets or non-default registries.

#### `extensions` (optional) — T5

Reserved object for vendor-specific or experimental data. Field names within `extensions` MUST start with `x-` (lowercase, hyphenated). Clients MUST ignore extensions they don't recognize.

The spec reserves the right to promote any `x-*` extension to a top-level field in a future version; conflicts are the extension author's responsibility.

**No other top-level extension fields are permitted.** Validators MUST reject any top-level field not defined by this spec or inside the `extensions` object.

### Example: Complete Manifest

```json
{
  "$schema": "https://mcp-manifest.dev/schema/v1.0.json",
  "version": "1.0",
  "server": {
    "name": "ironlicensing",
    "displayName": "IronLicensing",
    "description": "Manage IronLicensing products, tiers, features, licenses, and analytics",
    "version": "1.0.0",
    "author": "IronServices",
    "homepage": "https://www.ironlicensing.com",
    "repository": "https://github.com/IronServices/ironlicensing-mcp",
    "license": "MIT",
    "icon": "https://www.ironlicensing.com/favicon.svg",
    "keywords": ["licensing", "saas", "product-management", "analytics"]
  },
  "install": [
    {
      "method": "dotnet-tool",
      "package": "IronLicensing.Mcp",
      "registry": "https://git.marketally.com/api/packages/ironservices/nuget/index.json",
      "command": "ironlicensing-mcp",
      "priority": 0
    }
  ],
  "transport": "stdio",
  "config": [
    {
      "key": "profile",
      "description": "Named account profile from ~/.ironlicensing/config.json",
      "type": "string",
      "required": false,
      "arg": "--profile",
      "prompt": "Account profile (leave empty for default)",
      "options_from": {
        "file": "~/.ironlicensing/config.json",
        "path": "$.accounts[*].name"
      }
    },
    {
      "key": "api-key",
      "description": "IronLicensing API key (sk_live_xxx) from /app/settings/api-keys",
      "type": "secret",
      "required": false,
      "env_var": "IRONLICENSING_API_KEY",
      "arg": "--api-key",
      "prompt": "API key (or configure via add_account tool after connecting)",
      "secret_target": "api.ironlicensing.com",
      "secret_scope_url": "https://api.ironlicensing.com/*"
    },
    {
      "key": "base-url",
      "description": "IronLicensing API base URL",
      "type": "url",
      "required": false,
      "default": "https://api.ironlicensing.com",
      "env_var": "IRONLICENSING_BASE_URL",
      "arg": "--base-url",
      "prompt": "API base URL"
    }
  ],
  "scopes": ["global"],
  "settings_template": {
    "command": "ironlicensing-mcp",
    "args": ["--profile", "${profile}"]
  },
  "update_policy": "manual",
  "changelog_url": "https://github.com/IronServices/ironlicensing-mcp/blob/main/CHANGELOG.md",
  "signature": {
    "alg": "Ed25519",
    "key_id": "ironservices-2026-01",
    "value": "MEUCIQDx... (base64url, abbreviated for example)"
  }
}
```

### Example: Minimal Manifest (npm, no config, unsigned)

```json
{
  "$schema": "https://mcp-manifest.dev/schema/v1.0.json",
  "version": "1.0",
  "server": {
    "name": "sqlite",
    "displayName": "SQLite Explorer",
    "description": "Query and explore SQLite databases",
    "version": "0.5.0"
  },
  "install": [
    {
      "method": "npm",
      "package": "@anthropic/mcp-server-sqlite",
      "command": "mcp-server-sqlite",
      "priority": 0
    }
  ],
  "transport": "stdio",
  "config": [
    {
      "key": "db-path",
      "description": "Path to SQLite database file",
      "type": "path",
      "required": true,
      "prompt": "Database file path"
    }
  ],
  "scopes": ["project"],
  "settings_template": {
    "command": "mcp-server-sqlite",
    "args": ["${db-path}"]
  }
}
```

This minimal manifest is unsigned. Clients SHOULD warn the user "this manifest is unsigned; verify the source" before installing.

## Signing and Verification — T1

### Canonical Serialization

The signed payload is the manifest JSON object with the `signature` field removed, serialized using **RFC 8785 (JSON Canonicalization Scheme, JCS)**:

- Keys sorted lexicographically at every level (Unicode code-point order)
- No insignificant whitespace
- Strings escaped per RFC 8259 §7
- Numbers serialized per RFC 7159 / RFC 8259 (integers as integers, finite floats per ECMAScript Number.prototype.toString)

Both signers and verifiers MUST use this canonicalization. A reference implementation in three languages is provided by `@ai-manifests/jcs` (npm), `Adp.Json.Canonicalizer` (NuGet), and `adp_jcs` (PyPI).

### Signature Algorithm

`alg: "Ed25519"` is the only currently-defined signature algorithm. Future versions MAY add others (e.g., `Ed448`, `secp256k1`). Implementations MUST reject manifests with unknown algorithms.

The signature is computed as `Ed25519.sign(privateKey, JCS(manifest_without_signature))` and base64url-encoded (no padding) into the `signature.value` field.

### Publisher Key Discovery

Verifiers resolve a `key_id` to a public key using the following methods, in order:

#### 1. Well-Known Endpoint (preferred)

```
GET https://<manifest_host>/.well-known/mcp-manifest-keys.json
```

Response is a JSON object mapping key IDs to public keys:

```json
{
  "keys": [
    {
      "key_id": "ironservices-2026-01",
      "alg": "Ed25519",
      "public_key": "base64url-encoded-32-bytes",
      "valid_from": "2026-01-01T00:00:00Z",
      "valid_until": "2027-01-01T00:00:00Z"
    }
  ]
}
```

The same domain that serves the manifest MUST serve the keys endpoint. Mismatched origins are a hard verification failure.

#### 2. DNS TXT Record (fallback)

```
_mcp-manifest-key.<key_id>.<domain>  TXT  "v=1; alg=Ed25519; k=base64url-pubkey"
```

For example, `_mcp-manifest-key.ironservices-2026-01.ironlicensing.com`. Useful when serving an HTTP endpoint is impractical.

### Verification Procedure

```
INPUT: manifest (object), discovery_origin (the URL the manifest was fetched from)

1. If manifest has no "signature" field:
   → Return UNSIGNED. Caller (client) decides how to handle.

2. Validate signature.alg is "Ed25519". Else FAIL.

3. Resolve signature.key_id to a public key:
   → Try {discovery_origin}/.well-known/mcp-manifest-keys.json first
   → Fall back to DNS TXT record
   → If not found in either: FAIL (key_not_found)

4. Verify the resolved key's valid_from/valid_until covers the current time.
   → If outside window: FAIL (key_expired)

5. Compute payload = JCS(manifest with "signature" field removed)

6. Verify Ed25519.verify(public_key, payload, signature.value)
   → If invalid: FAIL (signature_invalid)

7. Return VERIFIED with key_id and (if available from keys endpoint) the publisher metadata.
```

### Trust-On-First-Use (TOFU) Semantics

For unsigned manifests, clients SHOULD:
- Compute and persist a content hash of the manifest at install time
- On subsequent fetches, compare hashes and warn on changes
- Never silently auto-update an unsigned manifest

For signed manifests, clients MAY auto-fetch updates within the same `key_id` provided the `update_policy` permits it. Changes in `key_id` (publisher key rotation) MUST trigger user re-confirmation.

## Autodiscovery

MCP manifest autodiscovery follows the same pattern as RSS feed discovery — server authors publish a pointer to their manifest from their website, and clients resolve it automatically from a domain name.

### Discovery Methods

Clients SHOULD support the following discovery methods, in priority order:

#### 1. Installed Tool (`--manifest` flag)

If the server command is already installed on the system, clients SHOULD try running it with the `--manifest` flag:

```bash
ironlicensing-mcp --manifest
```

The command MUST output the manifest JSON to stdout and exit with code 0. This is the **preferred discovery method** because:
- It works offline — no network request needed
- It works for private/internal servers with no website
- The manifest can include runtime-resolved values (e.g., `options_from` with local files)
- It guarantees the manifest matches the installed version

Server authors SHOULD implement this flag. The output MUST be valid `mcp-manifest.json` content.

#### 2. Well-Known URL

Serve the manifest at a well-known path:

```
GET https://example.com/.well-known/mcp-manifest.json
```

This is the preferred method for API servers and headless services that don't serve HTML. The response MUST be `application/json` with the manifest content.

**Server setup:**
- Serve the `mcp-manifest.json` file at `/.well-known/mcp-manifest.json`
- Return `Content-Type: application/json`
- CORS headers SHOULD allow cross-origin requests (`Access-Control-Allow-Origin: *`)
- If publishing a signed manifest, also serve `/.well-known/mcp-manifest-keys.json`

#### 3. HTML Link Tag

Add a `<link>` tag to the `<head>` of any page on the server author's website:

```html
<link rel="mcp-manifest" type="application/json" href="/mcp-manifest.json" />
```

The `href` MAY be:
- A relative path: `href="/mcp-manifest.json"`
- An absolute URL: `href="https://cdn.example.com/mcp-manifest.json"`
- A path to a different domain: `href="https://raw.github.com/org/repo/main/mcp-manifest.json"`

**Multiple manifests:** A page MAY contain multiple `<link rel="mcp-manifest">` tags for different MCP servers. Clients SHOULD present all discovered manifests and let the user choose.

```html
<link rel="mcp-manifest" type="application/json" href="/mcp-manifests/analytics.json" title="Analytics Server" />
<link rel="mcp-manifest" type="application/json" href="/mcp-manifests/licensing.json" title="Licensing Server" />
```

#### 4. Direct URL

A direct URL to the manifest file:

```
https://git.example.com/org/mcp-server/raw/branch/main/mcp-manifest.json
```

#### 5. Local File

A local file path to a manifest:

```
C:\path\to\mcp-manifest.json
/home/user/projects/mcp-server/mcp-manifest.json
```

### Cross-Manifest References Forbidden — T7

A manifest MUST NOT contain references that cause clients to fetch additional manifests automatically. Discovery is a single-hop operation. The `repository`, `homepage`, and similar URL fields are informational only — clients MUST NOT follow them as discovery sources without explicit user action.

This prevents manifest-loop attacks (manifest A → manifest B → manifest A) and discovery-amplification attacks against third-party hosts.

### Size and Timeout Limits — T7

Clients MUST enforce the following limits when fetching a manifest:

| Limit | Value |
|---|---|
| Maximum response body size | 64 KiB |
| Connection timeout | 5 seconds |
| Total fetch timeout (incl. headers) | 10 seconds |
| Maximum redirect hops | 3 |

Manifests exceeding these limits are invalid; clients MUST abort the fetch and inform the user.

### Atomic Resolution — T6

Clients MUST treat manifest fetch, validation, user approval, and install as a single atomic operation against an in-memory copy of the manifest:

```
1. Fetch manifest exactly once → manifest_bytes
2. Compute manifest_hash = SHA-256(manifest_bytes)
3. Verify signature (if present) → signature_status
4. Parse + validate against schema → manifest_obj
5. Display all install commands, registries, secrets, and signature_status to user
6. User approves OR cancels (no further fetch)
7. On approval: persist manifest_hash + manifest_bytes + signature_status before any install action
8. Install + configure using only manifest_obj from steps 1-4
```

Clients MUST NOT re-fetch the manifest between user approval (step 6) and install (step 8). Re-fetching creates a time-of-check / time-of-use race that an attacker can exploit to substitute the manifest after approval.

### Client Resolution Algorithm

Clients SHOULD resolve manifests using this algorithm. The input may be a command name (for installed servers), a domain, URL, or file path.

```
INPUT: user_input (could be command, domain, URL, or file path)

1. If user_input is an installed command (on PATH):
   → Run: {user_input} --manifest (with output size limit 64 KiB, timeout 5s)
   - If exit code 0 with valid JSON manifest: Done.

2. If user_input is a local file path and the file exists:
   → Read file (max 64 KiB). Parse as manifest JSON. Done.

3. If user_input looks like a direct manifest URL (ends with .json):
   → Fetch (with size + timeout limits) and parse as manifest JSON. Done.

4. Normalize to a base URL:
   - If no scheme, prepend "https://"
   - If no path, use root "/"
   → base_url

5. Try well-known URL:
   → GET {base_url}/.well-known/mcp-manifest.json (size + timeout limits)
   - If 200 with valid JSON manifest: Done.

6. Fetch the HTML page:
   → GET {base_url} (size + timeout limits)
   - Parse HTML for <link rel="mcp-manifest"> tags
   - If found: fetch the href URL (single hop, max 3 redirects), parse as manifest. Done.

7. Discovery failed.
   → Inform user that no manifest was found at the given location.

For every result above, immediately apply the Atomic Resolution flow before any install action.
```

### Example: Full Discovery Flow

**Scenario A: Server already installed**

A developer has `ironlicensing-mcp` installed. Their MCP client detects the command and runs:

1. `ironlicensing-mcp --manifest` → outputs manifest JSON
2. Parses manifest: 3 config params (profile, api-key, base-url)
3. `profile` has `options_from: { file: "~/.ironlicensing/config.json", path: "$.accounts[*].name" }`
4. Client reads `~/.ironlicensing/config.json` → finds `["marketally_llc", "marketally_pte"]`
5. Shows dropdown for profile with the two options
6. User selects `marketally_pte` → generates args: `["--profile", "marketally_pte"]`

Total user effort: select from dropdown. Everything else is automatic.

**Scenario B: Server not installed (signed manifest)**

A developer types `ironlicensing.com` into their MCP client:

1. `ironlicensing-mcp` not on PATH → skip `--manifest`
2. Client tries `https://ironlicensing.com/.well-known/mcp-manifest.json` → **200 OK**
3. Client computes manifest hash, verifies signature (key_id `ironservices-2026-01` resolved via `/.well-known/mcp-manifest-keys.json`)
4. Signature VERIFIED → client displays "Publisher: IronServices" badge
5. Parses the manifest: name="ironlicensing", install via dotnet-tool from `git.marketally.com` registry (NOT default nuget.org → flagged as non-default registry to user)
6. Shows: "Install IronLicensing MCP from `git.marketally.com`? Run: `dotnet tool install -g IronLicensing.Mcp`"
7. User confirms → tool installed
8. Prompts for config: "API key (sk_live_xxx) — will be sent to: api.ironlicensing.com" → user enters key
9. Writes to `~/.claude/.mcp.json`:
   ```json
   { "mcpServers": { "ironlicensing": { "command": "ironlicensing-mcp" } } }
   ```

Total user effort: type domain name, enter API key. Everything else is automated.

## Client Behavior

### Installation Flow

When a client encounters a manifest:

1. Apply the Atomic Resolution flow (fetch once, hash, verify, parse, present)
2. Validate against schema; reject any manifest with unknown top-level fields outside `extensions`
3. If `signature` is present, verify it; surface signature status (VERIFIED / UNSIGNED / FAILED) prominently in the user prompt
4. Check if the `command` is already available on PATH
5. If not, present the install methods (filtered by available runtimes) ordered by priority
6. For each install entry, display the registry origin (or note "default registry") and any required checksum
7. For `prebuilt-binary` installs, download to a temp location, verify SHA-256 against the declared `checksum`, fail loudly on mismatch
8. Execute the installation command
9. Prompt for required `config` values (using `prompt` text, respecting `type` for UI). For `secret` types, prominently display `secret_target`.
10. Generate the `mcpServers` entry using `settings_template` with substituted values
11. Persist the manifest bytes + hash + signature status alongside the configuration for future re-verification

### Update Flow — T4

When a previously-installed manifest is re-fetched (per the publisher's `update_policy`):

1. Compute new manifest hash; if identical to stored hash, no-op
2. Compare new manifest's `server.version` to stored version
3. If MAJOR version increased OR `key_id` changed OR `update_policy` is `manual`/`ask`:
   → Re-display install summary, permission summary, changed config[] entries, and changelog (if `changelog_url` provided) to user
   → Require explicit re-confirmation before applying
4. If MINOR/PATCH version with `update_policy: "auto"`:
   → MAY apply silently
   → MUST log the update locally for audit

### Validation

After configuration, clients SHOULD:
1. Verify the `command` exists on PATH
2. Attempt an MCP `initialize` handshake
3. Report success or failure to the user

## Design Principles

1. **The format is the standard, the UI is an implementation.** Any client can consume `mcp-manifest.json`.
2. **Discoverable like RSS.** A `<link>` tag or well-known URL lets any client find the manifest from a domain name.
3. **Progressive complexity.** A minimal manifest is 10 lines. Advanced features are optional.
4. **No runtime dependency.** The manifest is static JSON — no code execution, no scripting.
5. **Transport-aware.** The manifest captures how to connect, not just how to install.
6. **Secret-aware.** Config values typed as `secret` get special handling (masking, no logging, declared target origin).
7. **No central registry required.** Discovery is decentralized — each server author publishes their own manifest.
8. **Cryptographic publisher identity.** Manifests MAY be signed; clients verify and surface the signature status to users.
9. **Atomic install.** Fetch / verify / approve / install is a single transaction against an in-memory copy.

## Versioning

The `version` field at the manifest root tracks the spec version. Clients MUST check this field and reject manifests with unknown major versions.

A v0.1 manifest remains parseable by v1.0 clients in compatibility mode (no signature, no checksum, no secret_target requirements), but v1.0 clients SHOULD warn the user that the manifest predates security hardening.

## Conformance

A v1.0-conformant client MUST:

- Reject install entries with `method` outside the closed enum (T2)
- Reject install entries containing shell metacharacters in `command` (T2)
- Verify SHA-256 checksums for `prebuilt-binary` installs (T2)
- Display registry origin for non-default registries (T2)
- Require `secret_target` on every `type: "secret"` config entry (T3)
- Display `secret_target` to the user when prompting for the secret (T3)
- Honor `update_policy` and re-confirm on major version changes (T4)
- Reject manifests with unknown top-level fields outside `extensions` (T5)
- Treat manifest fetch + verify + install as atomic against in-memory copy (T6)
- Enforce manifest size limit (64 KiB) and fetch timeout (5s) (T7)
- Refuse to follow cross-manifest references for discovery (T7)

A v1.0-conformant client SHOULD:

- Verify `signature` blocks when present and surface the result to the user (T1)
- Persist manifest hash for re-verification on update (T1, T6)
- Fall back to DNS TXT for key discovery when the well-known endpoint is unreachable (T1)
- Warn loudly when installing from an unsigned manifest, especially with secrets or non-default registries (T1)

## Security Considerations

See [THREAT-MODEL.md](./THREAT-MODEL.md) for the full enumeration of threats this spec defends against, the spec-level mitigations for each, and the residual risks that adopters should be aware of.

## Future Considerations

- **Registry protocol** — an optional central index that crawls and aggregates manifests for search/browsing
- **Dependency declaration** — servers that require other servers or system tools
- **Health check** — a standard way to verify a configured server is working
- **Update notifications** — out-of-band signaling when a newer version is available (vs. the publisher-pull `update_policy` model)
- **Additional signature algorithms** — Ed448, secp256k1, post-quantum
