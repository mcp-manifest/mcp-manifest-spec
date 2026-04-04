# mcp-manifest.json Specification v0.1

**Status:** Draft
**Author:** David H Friedel Jr
**Date:** 2026-03-29

## Abstract

`mcp-manifest.json` is a machine-readable manifest format for Model Context Protocol (MCP) servers. It describes how to install, configure, and connect to an MCP server — the information that today lives only in README files and requires manual copy-paste into client configuration.

By shipping an `mcp-manifest.json`, server authors enable MCP clients to automate discovery, installation, configuration, and connection setup.

## Motivation

The MCP ecosystem is in its "curl the install script" era. Every server is a snowflake:

- Installation varies: `npm install -g`, `dotnet tool install`, `pip install`, binary download, Docker
- Configuration varies: environment variables, CLI args, config files, hardcoded
- Transport varies: stdio, SSE, streamable HTTP
- Client wiring varies: every README has a different JSON blob to paste

The MCP protocol defines the handshake *after* connection, but is agnostic about everything upstream — how to get the server running and how to wire a client to it. This gap forces every user to manually translate README instructions into client-specific configuration.

`mcp-manifest.json` fills this gap with a single, standardized, machine-readable file.

## Spec

### File Location

The manifest MUST be named `mcp-manifest.json` and SHOULD be placed at the root of the server's repository or package.

### Schema

```json
{
  "$schema": "https://mcp-manifest.dev/schema/v0.1.json",
  "version": "0.1",

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
      "method": "string (required) — dotnet-tool | npm | pip | cargo | binary | docker",
      "package": "string (required) — package name or image",
      "source": "string (optional) — custom registry URL",
      "command": "string (required) — the command name after installation",
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
      "prompt": "string (optional) — human-readable prompt for interactive setup"
    }
  ],

  "scopes": ["string (optional) — where this server makes sense: 'global', 'project', 'both'"],

  "settings_template": {
    "description": "The exact JSON object to merge into the client's mcpServers config. Variables use ${config.key} syntax.",
    "example": {
      "command": "ironlicensing-mcp",
      "args": ["--profile", "${profile}"]
    }
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
| `binary` | Download URL template | `https://github.com/.../releases/download/v${version}/server-${os}-${arch}` |
| `docker` | Docker image | `ghcr.io/org/mcp-server:latest` |

The `source` field allows custom registries (e.g., private NuGet feeds, npm registries).

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

### Example: Complete Manifest

```json
{
  "$schema": "https://mcp-manifest.dev/schema/v0.1.json",
  "version": "0.1",
  "server": {
    "name": "ironlicensing",
    "displayName": "IronLicensing",
    "description": "Manage IronLicensing products, tiers, features, licenses, and analytics",
    "version": "1.0.0",
    "author": "IronServices",
    "homepage": "https://www.ironlicensing.com",
    "repository": "https://git.marketally.com/IronServices/ironlicensing-mcp",
    "license": "MIT",
    "icon": "https://www.ironlicensing.com/favicon.svg",
    "keywords": ["licensing", "saas", "product-management", "analytics"]
  },
  "install": [
    {
      "method": "dotnet-tool",
      "package": "IronLicensing.Mcp",
      "source": "https://git.marketally.com/api/packages/ironservices/nuget/index.json",
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
      "prompt": "Account profile (leave empty for default)"
    },
    {
      "key": "api-key",
      "description": "IronLicensing API key (sk_live_xxx) from /app/settings/api-keys",
      "type": "secret",
      "required": false,
      "env_var": "IRONLICENSING_API_KEY",
      "arg": "--api-key",
      "prompt": "API key (or configure via add_account tool after connecting)"
    },
    {
      "key": "base-url",
      "description": "IronLicensing API base URL",
      "type": "url",
      "required": false,
      "default": "http://localhost:5000",
      "env_var": "IRONLICENSING_BASE_URL",
      "arg": "--base-url",
      "prompt": "API base URL"
    }
  ],
  "scopes": ["global"],
  "settings_template": {
    "command": "ironlicensing-mcp",
    "args": ["--profile", "${profile}"]
  }
}
```

### Example: Minimal Manifest (npm, no config)

```json
{
  "$schema": "https://mcp-manifest.dev/schema/v0.1.json",
  "version": "0.1",
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

## Autodiscovery

MCP manifest autodiscovery follows the same pattern as RSS feed discovery — server authors publish a pointer to their manifest from their website, and clients resolve it automatically from a domain name.

### Discovery Methods

Clients SHOULD support the following discovery methods, in priority order:

#### 1. Well-Known URL

Serve the manifest at a well-known path:

```
GET https://example.com/.well-known/mcp-manifest.json
```

This is the preferred method for API servers and headless services that don't serve HTML. The response MUST be `application/json` with the manifest content.

**Server setup:**
- Serve the `mcp-manifest.json` file at `/.well-known/mcp-manifest.json`
- Return `Content-Type: application/json`
- CORS headers SHOULD allow cross-origin requests (`Access-Control-Allow-Origin: *`)

#### 2. HTML Link Tag

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

#### 3. Direct URL

A direct URL to the manifest file:

```
https://git.example.com/org/mcp-server/raw/branch/main/mcp-manifest.json
```

#### 4. Local File

A local file path to a manifest:

```
C:\path\to\mcp-manifest.json
/home/user/projects/mcp-server/mcp-manifest.json
```

#### 5. Installed Tool

An installed tool that outputs its manifest:

```bash
ironlicensing-mcp --manifest
```

### Client Resolution Algorithm

When a user provides input (e.g., typing "ironlicensing.com" into an "Add MCP Server" field), clients SHOULD resolve the manifest using this algorithm:

```
INPUT: user_input (could be domain, URL, or file path)

1. If user_input is a local file path and the file exists:
   → Parse as manifest JSON. Done.

2. If user_input looks like a direct manifest URL (ends with .json):
   → Fetch and parse as manifest JSON. Done.

3. Normalize to a base URL:
   - If no scheme, prepend "https://"
   - If no path, use root "/"
   → base_url

4. Try well-known URL:
   → GET {base_url}/.well-known/mcp-manifest.json
   - If 200 with valid JSON manifest: Done.

5. Fetch the HTML page:
   → GET {base_url}
   - Parse HTML for <link rel="mcp-manifest"> tags
   - If found: fetch the href URL, parse as manifest. Done.

6. Discovery failed.
   → Inform user that no manifest was found at the given location.
```

### Example: Full Discovery Flow

A developer wants to add the IronLicensing MCP server. They type `ironlicensing.com` into their MCP client:

1. Client tries `https://ironlicensing.com/.well-known/mcp-manifest.json` → **200 OK**
2. Parses the manifest: name="ironlicensing", install via dotnet-tool
3. Checks if `ironlicensing-mcp` is on PATH → not found
4. Shows: "Install IronLicensing MCP? Run: `dotnet tool install -g IronLicensing.Mcp`"
5. User confirms → tool installed
6. Prompts for config: "API key (sk_live_xxx)" → user enters key
7. Writes to `~/.claude/settings.json`:
   ```json
   { "mcpServers": { "ironlicensing": { "command": "ironlicensing-mcp" } } }
   ```
8. Verifies MCP handshake → success

Total user effort: type domain name, enter API key. Everything else is automated.

## Client Behavior

### Installation Flow

When a client encounters a manifest:

1. Check if the `command` is already available on PATH
2. If not, present the install methods (filtered by available runtimes) ordered by priority
3. Execute the installation command
4. Prompt for required `config` values (using `prompt` text, respecting `type` for UI)
5. Generate the `mcpServers` entry using `settings_template` with substituted values
6. Write to the appropriate settings file based on `scopes`

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
6. **Secret-aware.** Config values typed as `secret` get special handling (masking, no logging).
7. **No central registry required.** Discovery is decentralized — each server author publishes their own manifest.

## Versioning

The `version` field at the manifest root tracks the spec version. Clients SHOULD check this field and gracefully handle unknown versions.

## Future Considerations

- **Registry protocol** — an optional central index that crawls and aggregates manifests for search/browsing
- **Dependency declaration** — servers that require other servers or system tools
- **Health check** — a standard way to verify a configured server is working
- **Update notifications** — signaling when a newer version is available
- **Signed manifests** — cryptographic signatures for manifest authenticity
