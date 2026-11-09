# Kubik

Kubik is a fast, cross-platform desktop cockpit for Kubernetes clusters: a
native Rust core (Tauri 2 + kube-rs) with a React interface. It connects with
your own kubeconfig, streams live resource state, and covers the daily-driver
tooling — YAML editing, logs, shells, port forwarding, Helm — without a local
kubectl.

![Kubik](src-tauri/icons/128x128.png)

## Features

| Area | Capabilities |
| --- | --- |
| Cluster connection | Multiple kubeconfig files/folders scanned recursively; context picker with per-file credentials; connect/disconnect toggle; cluster health, ping and server version in the debug panel |
| Namespaces | Multi-select namespace scope or all-namespaces, applied to every view and watch |
| Resource browsing | Full API discovery (built-ins + CRDs + aggregated APIs, probed group-by-group in parallel), live watch-driven tables with filter (`ns:` `status:` `name:`), sorting, virtualized rows and per-row context menus |
| Overview dashboard | Stat cards, workload readiness (Deployments/StatefulSets/DaemonSets/Nodes), live CPU/memory/pods capacity with history sparklines, pod distribution, warning events |
| Object inspector | Metadata, probes, volumes, conditions, related objects (pods behind a Service, PVC claims), events — all links navigate to the corresponding object |
| YAML editor | CodeMirror editor with templates, change review diff, and `kubectl edit` semantics: saves PATCH the two-way diff of the buffer (never a full replace), so untouched fields keep their live server state |
| Workload actions | Scale, rollout restart, evict (respects PodDisruptionBudgets), delete — from row context menus |
| Pod logs | Container/init-container picker, history slices, live follow, ANSI colors, wrap toggle, download |
| Pod shells | Interactive exec terminals (xterm.js) with live resize and container switching |
| Host terminal | Local shell in the dock with the active context's KUBECONFIG exported |
| Port forwarding | Pods and Services (resolved through Endpoints/EndpointSlice), auto or explicit local port, session panel |
| Helm | Releases table, revision history, manifests/values/notes viewer, repo management, install/upgrade/rollback/uninstall — via the local Helm binary (auto-detected or pinned in Settings → Tools) |
| Argo CD | Application table with dynamic columns, refresh and sync actions |
| Diagnostics | App event log, connection debug panel (ping, per-group discovery probes, live watch table), force re-discovery |
| Metrics | Pod/node CPU and memory columns and charts via metrics.k8s.io when installed |
| Settings | Kubeconfig paths, theme (dark/light/system), fonts, editor behavior, scrollback, Helm binary path |

## Getting started

### Prerequisites

- Rust 1.85+ (rustup) and Node.js 20+/npm
- A kubeconfig with at least one context (`~/.kube/config` or `KUBECONFIG`)
- Optional: a local `helm` binary for the Helm views (auto-detected from PATH,
  common install locations or the login shell; a custom path can be pinned in
  Settings → Tools)

### Run in development

```bash
npm install
npm run tauri dev
```

## Building

Local builds take the version from a single place:
`src-tauri/tauri.conf.json`. Release builds are tag-driven: pushing a
`v<version>` tag overrides the app version with the tag, so the running app
(Settings → About, title bar) and both distribution names always carry the
released version.

### macOS (local)

```bash
make mac          # or ./scripts/bundle.sh
```

Produces `Kubik.app` and `Kubik_<version>_aarch64.dmg` — a drag-to-
Applications installer with a styled background window. The script links the
Rust core serially and verifies every proc-macro dylib loads, working around a
darwin 27 linker bug (see the note in `scripts/bundle.sh`).

### Windows (local)

Windows builds run inside Docker — no Windows/MSVC tooling is installed on the
host (see `docker/windows-builder.Dockerfile`):

```bash
make windows-zip  # Kubik_<version>_x64.zip in the repo root
```

### CI

`.github/workflows/build.yml` runs on a version tag push (`v*`) and publishes
a GitHub release — in the public downloads repository only (no artifacts or
releases are kept in the private source repository):

- `Kubik_<tag-version>_aarch64.dmg` (macOS, arm64 runner, drag-to-Applications styled)
- `Kubik_<tag-version>_x64-setup.zip` (Windows: zip containing the NSIS installer)

Before building, the workflow writes the tag version into
`src-tauri/tauri.conf.json`, so the app itself reports the released version.
The macOS DMG is packed with the same drag-to-Applications styling as local
builds; if a runner has no Finder session the script falls back to a plain
DMG automatically.

Release flow:

```bash
# bump the version in src-tauri/tauri.conf.json for local builds, then
git tag v0.2.0 && git push origin v0.2.0
```

#### Private source, public builds

The source repository can stay private while the builds stay publicly
downloadable in the public repository
[`akopyanlabs/kubik`](https://github.com/akopyanlabs/kubik). CI publishes
every release there when the `RELEASES_TOKEN` secret is set.

One-time setup:

1. Create a fine-grained personal access token with **Contents: Read and
   write** limited to the `akopyanlabs/kubik` repository.
2. Add it to the private source repository's secrets as `RELEASES_TOKEN`.

Without the secret the pipeline is green and just skips the public publish.
The CI also copies this README into the public repository on every release,
so the download page always shows current documentation.

#### macOS signing and notarization (optional)

Without an Apple Developer certificate the macOS build is ad-hoc signed and
Gatekeeper asks for a one-time confirmation on first launch (see below).
To ship a fully signed and notarized DMG, add these repository secrets —
the pipeline picks them up automatically and skips the steps when absent:

| Secret | Purpose |
| --- | --- |
| `MACOS_CERTIFICATE_P12` | base64 of the Developer ID Application certificate (.p12) |
| `MACOS_CERTIFICATE_PASSWORD` | password of the .p12 |
| `MACOS_KEYCHAIN_PASSWORD` | any strong password for the temporary CI keychain |
| `MACOS_NOTARIZATION_APPLE_ID` | Apple ID of the developer account |
| `MACOS_NOTARIZATION_PASSWORD` | app-specific password for notarytool |
| `MACOS_NOTARIZATION_TEAM_ID` | Team ID |

Create the base64 on a machine that has the certificate:
`base64 -i DeveloperID.p12 | pbcopy`.

### macOS Gatekeeper

Unsigned (ad-hoc) builds downloaded from the internet get quarantined and the
first launch is blocked with "cannot verify the developer". Any of these
unblocks it once:

- Right-click Kubik in Applications → Open → Open
- System Settings → Privacy & Security → scroll to "Open Anyway"
- `xattr -dr com.apple.quarantine /Applications/Kubik.app`

A permanent fix requires a paid Apple Developer account: put the secrets from
the table above into the repository and the CI will notarize every release.

## Architecture

```
src/                    React + TypeScript interface
├── api/ipc.ts          typed Tauri command wrappers + push channels
├── state/              zustand stores: cluster, tabs, watches, dock,
│                       detailPanel, settings, appLog
├── components/         TitleBar, Sidebar, Dock, TabStrip, ResourceTable,
│                       DetailPanel, OverviewView, YamlEditorView, LogsView,
│                       TerminalView, HelmView, ArgoCDView, ForwardsPanel,
│                       CommandPalette, SettingsModal, DebugModal, toast
└── utils/              format (age/bytes/cpu), ANSI colors, YAML helpers

src-tauri/src/          Rust core
├── contracts.rs        IPC data contracts (single source of truth)
├── kubeconfig.rs       kubeconfig resolution + bounded folder scanning
├── cluster.rs          per-context client pool (mtime-aware cache)
├── discovery.rs        resilient parallel API discovery + curated catalog
├── resources.rs        list/get (paginated, cache-served) + row projection
├── watch.rs            watch hub: per-namespace streams, re-list on error/410
├── events.rs           core/v1 event feeds
├── metrics.rs          metrics.k8s.io reader (quantity parsing included)
├── logs.rs             log slices + follow streams
├── exec.rs             exec session hub (stdin/stdout/resize multiplexing)
├── forward.rs          TCP port forwarding (per-connection forwarders)
├── mutations.rs        kubectl-edit-style PATCH, server-side apply, delete,
│                       scale, rollout restart, eviction
├── helm.rs             Helm CLI wrapper with binary auto-detection
├── argocd.rs           Argo CD refresh/sync
├── host_terminal.rs    local terminal sessions
├── fonts.rs            installed monospace font discovery
└── lib.rs              Tauri commands, app state, lifecycle cleanup
```

### IPC surface

`get_settings`, `set_kubeconfig_paths`, `get_kube_contexts`,
`set_current_context`, `ping_cluster`, `get_namespaces`, `get_api_resources`,
`api_debug`, `list_resources`, `list_raw_resources`, `get_resource`,
`watch_resources`/`stop_watch` (channel), `get_events`, `get_metrics`,
`get_pod_log_slice`, `start_log_stream` (channel),
`start_exec`/`exec_write`/`exec_resize`/`exec_close` (channel),
`start_forward`/`stop_forward`/`list_forwards`, `apply_yaml`, `edit_yaml`,
`delete_resource`, `evict_pods`, `scale_workload`, `rollout_restart`,
`argocd_refresh`/`argocd_sync`, `helm_*` (check/list/history/get/repos/search/
ops + binary pinning), `start_host_terminal`/`host_terminal_*` (channel),
`list_monospace_fonts`, `open_url`.

All commands return typed errors (`{code, message, status}`) with codes
`forbidden | not_found | conflict | timeout | unavailable | invalid | internal`.

### Design notes

- **Watches** keep one stream per namespace, emit `upsert`/`delete` rows over a
  Tauri channel, re-list on stream end or error with exponential backoff, and
  fully resync after `410 Gone`.
- **Listing** is served from the apiserver watch cache (`resourceVersion=0`)
  with server-side pagination (500/page) and a quorum fallback; the client
  negotiates gzip, which matters a lot over VPN links.
- **YAML edits** follow `kubectl edit`: the saved buffer is diffed against the
  buffer as opened and PATCHed (`application/merge-patch+json`) with the
  identity preconditions kubectl enforces — untouched fields keep their live
  server state, so racing controllers are not clobbered. Server-side apply is
  used for the multi-document New Resource flow.
- **Workload status** distinguishes a mid-rollout state (`· progressing`) from
  fully ready and failed: `readyReplicas` alone is not green while the old pod
  is still running.
- **Exec sessions** multiplex stdin/stdout/resize in a single owning task;
  closing the tab drops the session and aborts the remote process.
- **Port forwards** bind a local listener and spawn a dedicated forwarder per
  accepted TCP connection, so concurrent connections keep working.
- **Discovery** probes each API group concurrently and independently — one
  broken aggregated API no longer blanks the resource tree (see the debug
  panel for per-group status).

## Attribution

Kubik's visual identity (name and icon) follows the Kubik desktop Kubernetes
IDE; this is an independent implementation written from scratch, communicating
with clusters exclusively through the Kubernetes API.

## License

Apache-2.0
