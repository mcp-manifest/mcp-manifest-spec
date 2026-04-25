# mcp-manifest specification

_A specification by MarketAlly_

A machine-readable manifest format for [MCP](https://modelcontextprotocol.io) servers — enabling autodiscovery, one-click installation, and zero-config client setup.

## The Problem

Every MCP server is a snowflake installation. The protocol defines what happens *after* you connect, but is silent about everything before: how to install, how to configure, and how to wire your client.

## The Solution

Ship an `mcp-manifest.json` with your MCP server:

```json
{
  "$schema": "https://mcp-manifest.dev/schema/v0.1.json",
  "version": "0.1",
  "server": {
    "name": "my-server",
    "displayName": "My Server",
    "description": "What it does",
    "version": "1.0.0"
  },
  "install": [{ "method": "npm", "package": "my-mcp-server", "command": "my-mcp-server", "priority": 0 }],
  "transport": "stdio",
  "config": [
    { "key": "api-key", "type": "secret", "required": true, "env_var": "MY_API_KEY", "prompt": "API key" }
  ]
}
```

Enable autodiscovery by adding one line to your website:

```html
<link rel="mcp-manifest" type="application/json" href="/.well-known/mcp-manifest.json" />
```

Then any MCP client can set up your server from just your domain name.

## Contents

| File | Description |
|------|-------------|
| [spec.md](spec.md) | Full specification (v0.1) |
| [schema/v0.1.json](schema/v0.1.json) | JSON Schema for validation |
| [examples/](examples/) | Reference manifests |

## Autodiscovery

Clients discover manifests automatically via:

1. **Well-known URL** — `https://example.com/.well-known/mcp-manifest.json`
2. **HTML link tag** — `<link rel="mcp-manifest" href="/mcp-manifest.json" />`

Same pattern as RSS feeds and OpenID. No central registry required.

## Quick Start

### For MCP server authors

1. Create `mcp-manifest.json` in your repo root ([examples](examples/))
2. Serve it at `/.well-known/mcp-manifest.json` on your website
3. Add `<link rel="mcp-manifest">` to your HTML `<head>`

### For MCP client developers

1. Accept a domain/URL from the user
2. Resolve the manifest using the [discovery algorithm](spec.md#client-resolution-algorithm)
3. Walk the user through install + config using the manifest fields

## Examples

- [Minimal](examples/minimal.json) — bare minimum manifest (10 lines)
- [SQLite](examples/sqlite.json) — npm install, file path config, project scope
- [GitHub](examples/github.json) — npm install, secret token, global scope
- [IronLicensing](examples/ironlicensing.json) — dotnet tool, custom registry, multi-account profiles

## Badge

Show that your MCP server supports manifest autodiscovery:

```markdown
[![MCP Manifest](https://mcp-manifest.dev/media/mcp-manifest-badge-light.svg)](https://mcp-manifest.dev)
```

[![MCP Manifest](media/mcp-manifest-badge-light.svg)](https://mcp-manifest.dev)

## Media Kit

Assets for referencing the spec in docs, READMEs, and blog posts:

| File | Description |
|------|-------------|
| [mcp-manifest-icon.svg](media/mcp-manifest-icon.svg) | Source icon (SVG) |
| [mcp-manifest-icon-256.png](media/mcp-manifest-icon-256.png) | Raster icon (256px PNG) |
| [mcp-manifest-badge-light.svg](media/mcp-manifest-badge-light.svg) | "MCP manifest ready" badge (light) |
| [mcp-manifest-badge-dark.svg](media/mcp-manifest-badge-dark.svg) | "MCP manifest ready" badge (dark) |
| [favicon.ico](media/favicon.ico) | Favicon (16–256px, transparent corners) |

## Status

**v0.1 — Draft.** The spec is functional and has a reference implementation. Feedback welcome via issues.

## Licensing Model

The mcp-manifest specification text is released under the [Community Specification 
License 1.0 (Patent and Copyright)](https://github.com/CommunitySpecification/1.0).
Reference implementations, schemas, and tooling are released under the Apache 2.0 License.
The specification is fully implementable without reliance on any specific implementation.

Contributions to this specification are accepted under the same Community 
Specification License 1.0 terms. By submitting a contribution, you agree to 
the copyright and patent grants defined therein.

## Document Metadata
- Author: David H Friedel Jr.
- Copyright: © 2026 David H Friedel Jr.
- First Published: 2026-03-20
- Affiliated with: MarketAlly (https://marketally.ai)
- **Specification**: mcp-manifest  
- **Version**: v0.1 (Draft)  
- **Status**: Public Draft  

David H Friedel Jr. is the current copyright holder of this specification. 
The MarketAlly group includes MarketAlly LLC (USA), MarketAlly Pte Ltd 
(Singapore), and MarketAlly OÜ (Estonia). This specification is being 
prepared for assignment to a MarketAlly entity; until that assignment is 
in place, copyright remains with the individual author.

mcp-manifest wraps installation, configuration, and discovery metadata 
around servers built on Anthropic's Model Context Protocol (MCP). MCP itself 
is Anthropic's protocol; the mcp-manifest format is a separate specification 
that complements it.

"mcp-manifest", "ADP", "ACB", "ADJ", and "Agent Registry" are trademarks of 
MarketAlly. The specifications are openly licensed; the names are not.
