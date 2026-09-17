# ZEN SecDB for Zed

Audit your dependency manifests for known vulnerabilities without leaving the editor.

This is the [Zed](https://zed.dev) extension for **ZEN SecDB**. As you open a
dependency manifest, it audits the declared packages against the ZEN SecDB
vulnerability database and reports known advisories inline, as diagnostics on the
exact dependency line, each linking back to the advisory.

It is a thin wrapper around the [`secdb`](https://github.com/giterlizzi/secdb-cli)
CLI: the extension launches `secdb lsp` (a Language Server that speaks JSON-RPC over
stdio) and Zed renders whatever it reports.

## What it does

- Parses the open manifest and resolves each dependency to a package URL (PURL).
- Audits the whole set against ZEN SecDB.
- Surfaces every known advisory as a diagnostic on the dependency's line, with a
  severity mapped from the advisory (error / warning / information / hint) and a
  clickable link to the advisory page.
- Re-audits as you open and edit a manifest (debounced, so it audits shortly after
  you stop typing rather than on every keystroke).
- On startup, discovers and audits every supported manifest in the workspace, so
  findings appear without opening each file (noise directories like `node_modules`,
  `vendor`, `target`, `dist` and `build` are skipped, and symlinks aren't followed).

## Requirements

The extension does **not** bundle a binary: it runs the `secdb` CLI, which must be
installed and on your `PATH`.

1. Install the `secdb` CLI — see
   [giterlizzi/secdb-cli](https://github.com/giterlizzi/secdb-cli).
2. Verify it is reachable: `secdb version`. If you start Zed from a GUI launcher,
   make sure the directory holding `secdb` (e.g. `~/.local/bin`) is on the `PATH`
   Zed inherits; launching Zed from a terminal is the simplest guarantee.
3. Optional: set `SECDB_API_KEY` in the environment Zed inherits, and/or point the
   CLI at a custom instance with `--base-url`, if your ZEN SecDB deployment
   requires it.

## Supported files

The server detects the format from the file name and audits these manifests (the
same set as `secdb audit manifest`). The extension attaches the server to the Zed
language each file is assigned, via the `languages` list in `extension.toml`:

| Manifest | Ecosystem | Zed language |
| --- | --- | --- |
| `go.mod` | Go modules | Go Mod |
| `package-lock.json` | npm | JSON |
| `yarn.lock` | npm | Plain Text |
| `requirements*.txt` | Python | Plain Text |
| `Gemfile.lock` | Ruby | Plain Text |
| `pom.xml` | Maven (Java) | XML |
| `composer.lock` | PHP (Composer) | Plain Text |

Detection is by file name, so only files named like the above are audited — opening
a bare `package.json`, for example, starts the server but produces no diagnostics
(npm findings come from `package-lock.json` / `yarn.lock`). Several of these files
are plain text in Zed (`yarn.lock`, `Gemfile.lock`, `requirements.txt`,
`composer.lock`), so the extension attaches to the `Plain Text` language; the server
simply does nothing for plain-text files it doesn't recognize.

## Installation

Until it lands in the Zed extension registry, install it as a dev extension:

1. Clone this repository.
2. In Zed, run `zed: install dev extension` from the command palette and select the
   cloned directory.

Building the extension requires the `wasm32-wasip2` Rust target; Zed installs it via
`rustup` automatically. On toolchains without `rustup`, add the target manually.

## How it works

```
editor buffer ──▶ secdb lsp ──▶ manifest parser ──▶ PURLs ──▶ ZEN SecDB audit
                                                                     │
   diagnostics on the dependency line ◀── advisory ⇄ severity ⇄ link ┘
```

The extension only tells Zed which binary to launch (`secdb lsp`) and for which
files. All the parsing and auditing happens in the CLI, so adding manifest formats
or fixing detection is done there, not here.

## ZEN SecDB MCP server (optional)

Separately from this extension, ZEN SecDB exposes a remote **MCP server** that gives
Zed's Agent panel live vulnerability tools. It is independent of the diagnostics
extension — you can use either or both. Full docs:
[secdb.nttzen.cloud/docs/integrations/mcp](https://secdb.nttzen.cloud/docs/integrations/mcp).

The tools it exposes:

| Tool | Purpose |
| --- | --- |
| `vulnerability_search` | Full-text search across CVEs, advisories, exploits and products |
| `vulnerability_info` | Complete CVE details: description, metadata, affected products |
| `vulnerability_score` | CVSS / EPSS scores with human-readable explanations |
| `epss_timeseries` | Historical EPSS probability trend for a CVE |
| `ssvc_calculator` | CISA SSVC prioritization decision for a CVE |
| `sightings_search` | Real-world exploitation evidence: PoCs, scanner plugins, mentions |
| `purl_audit` | Audit application dependencies by Package URL (PURL) |
| `linux_audit` / `linux_os` | Audit installed Linux packages / list supported distros |
| `feed_report` / `feed_report_catalog` | Run predefined analytics reports / list them |

Add it either from Zed's UI or by editing `settings.json`:

- **From the UI** — open **Settings → AI → MCP Servers** (or run `agent: open
  settings`), click **Add Server → Add Remote Server**, and enter a name (`secdb`)
  and the URL `https://secdb.nttzen.cloud/mcp`. Zed writes the entry below for you.
- **In `settings.json`**:

  ```json
  {
    "context_servers": {
      "secdb": {
        "url": "https://secdb.nttzen.cloud/mcp"
      }
    }
  }
  ```

Once connected, the server's tools are available in Zed's Agent panel — ask the
agent in natural language (e.g. *"Is pkg:npm/lodash@4.17.19 vulnerable?"*, *"Which of
my go.mod advisories are urgent by EPSS/SSVC?"*, *"Summarize CVE-2021-44228 and the
fixed version"*) and it calls the tools as needed.

Both routes need a Zed build with remote (HTTP) MCP support — on older builds, use
**Add Local Server** (or the `"command"`/`"args"` form in `settings.json`) to bridge
it locally with `npx -y mcp-remote https://secdb.nttzen.cloud/mcp`.

## Related

- **ZEN SecDB CLI** — the engine behind this extension, and a full CLI for CVE
  lookup, PURL/Linux/Docker audits, SSVC and SBOM scanning:
  [giterlizzi/secdb-cli](https://github.com/giterlizzi/secdb-cli).
- **ZEN SecDB MCP** — the remote MCP server described above
  ([docs](https://secdb.nttzen.cloud/docs/integrations/mcp)).

## License

Apache-2.0. See [LICENSE](LICENSE).
