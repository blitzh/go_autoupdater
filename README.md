# go-autoupdater

## Portable Updater (Go) — Cross-platform Self-Update Library + CLI

A small, dependency-free updater framework written in Go that supports:

- ✅ **Manifest-based updates** (HTTP `manifest.json`)
- ✅ **SHA256 verification** (integrity check)
- ✅ **Cross-platform**: Windows / Linux / macOS (incl. embedded Linux/STB)
- ✅ Works as:
  - **Library** (embed into your app/agent)
  - **Standalone CLI** (`updaterctl`)
- ✅ Service-aware update (optional):
  - Windows: **NSSM** or **SC**
  - Linux: **systemd**
  - macOS: **launchd**
  - Standalone: **noop**
- ✅ Windows-safe swapping using a separate **helper** process (`updater-helper.exe`)

> Signature verification (Ed25519) is intentionally postponed for the initial version. The architecture is designed so it can be added later without breaking the API.

---

## Table of Contents

- [Why this exists](#why-this-exists)
- [How it works](#how-it-works)
- [Repository structure](#repository-structure)
- [Manifest format](#manifest-format)
- [Quick start (CLI)](#quick-start-cli)
  - [Windows + NSSM](#windows--nssm)
  - [Windows + SC](#windows--sc)
  - [Linux + systemd](#linux--systemd)
  - [macOS + launchd](#macos--launchd)
  - [Standalone (no service)](#standalone-no-service)
- [Quick start (Library / Embedded)](#quick-start-library--embedded)
- [Build](#build)
- [Operational notes](#operational-notes)
- [Security notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [Roadmap](#roadmap)
- [License](#license)

---

## Why this exists

Self-updating is deceptively tricky because:

- On **Windows**, the running executable is **locked**, so you can't overwrite it in-place.
- Service management differs across OSes (NSSM/SC vs systemd vs launchd).
- Reliable updates require:
  - staging (`.new`)
  - swap with retries
  - backup (`.old`)
  - rollback when restart fails

This project provides a clean separation:

- **Updater engine**: fetch manifest, pick artifact, download, verify SHA256, stage update
- **Service controllers**: stop/start for different init/service managers
- **Appliers**: atomic-ish swap logic per OS (Windows uses a helper)

---

## How it works

1. `updaterctl` (or your embedded app) fetches `manifest.json`
2. Selects artifact by `GOOS` + `GOARCH`
3. Compares versions
4. Downloads the new binary into a staging file:
   - Windows: `*.new.exe`
   - Linux/macOS: `*.new`
5. Verifies SHA256 from manifest
6. Applies update:
   - **Windows**: spawns `updater-helper.exe` to stop service, swap files, start service
   - **Linux/macOS**: stops service (optional), swaps, starts service
7. Keeps backup:
   - Windows: `*.old.exe`
   - Linux/macOS: `*.old`

---

## Repository structure

```
portable-updater/
  cmd/
    updaterctl/         # CLI: check+download+verify+apply
    updater-helper/     # Windows helper: stop/swap/start (required on Windows)
  pkg/
    updater/            # core engine
    source/             # manifest sources (HTTP)
    verify/             # SHA256 verification
    apply/              # swap appliers (posix/windows)
    service/            # service controllers (nssm/sc/systemd/launchd/noop)
    util/               # utilities (download, retry rename/remove, logging)
```

---

## Manifest format

Your update server should provide a JSON manifest, for example:

### URL
`https://your-server.example.com/dldir/agent/manifest.json`

### Example manifest

```json
{
  "product": "agent",
  "channel": "stable",
  "version": "1.0.12",
  "published_at": "2025-12-24T00:00:00Z",
  "notes": "Fix reconnect + reduce log spam",
  "artifacts": [
    {
      "os": "windows",
      "arch": "amd64",
      "name": "agent.exe",
      "url": "https://your-server.example.com/dldir/agent/agent_1.0.12_windows_amd64.exe",
      "sha256": "PUT_SHA256_HERE"
    },
    {
      "os": "linux",
      "arch": "arm64",
      "name": "agent",
      "url": "https://your-server.example.com/dldir/agent/agent_1.0.12_linux_arm64",
      "sha256": "PUT_SHA256_HERE"
    }
  ]
}
```

### Rules

- `version` should follow numeric dot format like `1.2.10` (recommended)
- `sha256` **must be present** (integrity requirement)
- Each artifact must match:
  - `os` = `runtime.GOOS` (windows/linux/darwin)
  - `arch` = `runtime.GOARCH` (amd64/arm64/arm/386)
- `url` should be served over **HTTPS** (recommended)

---

## Quick start (CLI)

### 1) Build the CLI

```bash
go mod tidy
go build -o updaterctl ./cmd/updaterctl
```

On Windows:

```powershell
go build -o updaterctl.exe .\cmd\updaterctl
go build -o updater-helper.exe .\cmd\updater-helper
```

> ✅ For Windows updates, place `updater-helper.exe` **inside the same install directory** as your target executable, e.g. `C:\agent\updater-helper.exe`.

---

## Windows + NSSM

### Example
- Service name: `SampleAgent`
- Install dir: `C:\sample\agent`
- Executable: `agent.exe`
- NSSM path: `C:\tools\nssm.exe`

```powershell
updaterctl.exe `
  --manifest "https://your-server.example.com/dldir/agent/manifest.json" `
  --dir "C:\sample\agent" `
  --exe "agent.exe" `
  --current "1.0.11" `
  --service "SampleAgent" `
  --nssm "C:\tools\nssm.exe"
```

**Important**
- `updater-helper.exe` must be in `C:\sample\agent\updater-helper.exe`

---

## Windows + SC

If you do not use NSSM and rely on the built-in Windows Service Controller:

```powershell
updaterctl.exe `
  --manifest "https://your-server.example.com/dldir/agent/manifest.json" `
  --dir "C:\sample\agent" `
  --exe "agent.exe" `
  --current "1.0.11" `
  --service "SampleAgent" `
  --nssm "SC"
```

> Passing `--nssm "SC"` is the current shortcut to force SC controller.

---

## Linux + systemd

```bash
sudo ./updaterctl \
  --manifest "https://your-server.example.com/dldir/agent/manifest.json" \
  --dir "/opt/agent" \
  --exe "agent" \
  --current "1.0.11" \
  --systemd "agent.service"
```

---

## macOS + launchd

```bash
./updaterctl \
  --manifest "https://your-server.example.com/dldir/agent/manifest.json" \
  --dir "/usr/local/agent" \
  --exe "agent" \
  --current "1.0.11" \
  --launchd "com.your.agent"
```

> launchd commands can vary depending on how the job was loaded. This project uses best-effort `launchctl start/stop`.

---

## Standalone (no service)

```bash
./updaterctl \
  --manifest "https://your-server.example.com/dldir/agent/manifest.json" \
  --dir "./" \
  --exe "agent" \
  --current "1.0.11"
```

---

## Quick start (Library / Embedded)

You can embed the updater into your agent/app and trigger updates programmatically.

### Example

```go
package main

import (
  "context"
  "time"

  "github.com/blitzh/go-autoupdater/pkg/apply"
  "github.com/blitzh/go-autoupdater/pkg/service"
  "github.com/blitzh/go-autoupdater/pkg/source"
  "github.com/blitzh/go-autoupdater/pkg/updater"
  "github.com/blitzh/go-autoupdater/pkg/util"
)

func main() {
  logger := util.NewLogger("./updater.log")

  src := source.NewHTTPManifestSource("https://your-server.example.com/dldir/agent/manifest.json")

  // Standalone mode:
  ctrl := service.NoopController{}

  // Choose applier per OS:
  // - On Windows: WindowsHelperApplier requires helper exe placed in install dir
  // - On Linux/macOS: PosixApplier swaps directly
  ap := apply.PosixApplier{Retries: 40}

  u := updater.New(updater.Config{
    CurrentVersion: "1.0.11",
    InstallDir:     ".",
    ExeName:        "agent",
    Source:         src,
    Service:        ctrl,
    Applier:        ap,
    Logger:         logger,
  })

  ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
  defer cancel()

  res, err := u.Update(ctx)
  if err != nil {
    logger.Printf("update failed: %v", err)
    return
  }
  logger.Printf("result: didUpdate=%v remote=%s", res.DidUpdate, res.RemoteVersion)
}
```

### Windows embedded note

If your app runs as a Windows service, the recommended pattern is:

- your service/app triggers `updaterctl` or the updater library
- the applier spawns `updater-helper.exe` to safely stop/swap/start

---

## Build

### Requirements
- Go `1.22+` recommended

### Build all
```bash
go mod tidy
go test ./...
go build ./cmd/updaterctl
```

Windows helper:
```powershell
go build -o updater-helper.exe .\cmd\updater-helper
```

---

## Operational notes

### File naming conventions

**Windows**
- `agent.exe` (current)
- `agent.new.exe` (staging)
- `agent.old.exe` (backup)

**Linux/macOS**
- `agent` (current)
- `agent.new` (staging)
- `agent.old` (backup)

### Permissions

- Windows service updates typically require **Administrator** privileges.
- Linux systemd updates typically require `sudo` to stop/start service and write into `/opt` or `/usr/local`.

### Logging

- CLI logs to `--log` or defaults to `<installDir>/updaterctl.log`
- Helper prints to stdout/stderr; you can redirect logs via service wrapper if needed

---

## Security notes

This initial version verifies **SHA256** only.

- ✅ Protects against corrupted/partial downloads
- ❌ Does *not* protect against a compromised update server or DNS hijack

Recommended minimal safety:
- Use **HTTPS**
- Host manifest and artifacts on a controlled domain
- Restrict update URLs to trusted hosts

Planned improvement:
- Add **Ed25519** signature verification for manifest/artifacts (see [Roadmap](#roadmap)).

---

## Troubleshooting

### Windows: update fails to swap
Common causes:
- `updater-helper.exe` missing in install directory
- service still running / file lock not released
- insufficient permissions

Fix:
- Ensure `updater-helper.exe` is beside `agent.exe`
- Run the updater as Administrator
- Check logs (`updaterctl.log`) and helper output

### Linux: systemd stop/start fails
- Unit name wrong, e.g. `agent.service` vs `myagent.service`
- no sudo permissions

Fix:
- Verify with `systemctl status <unit>`
- Run with `sudo`

### No artifact found
The manifest does not contain an entry for your OS/arch.

Fix:
- Check runtime values:
  - OS: `windows/linux/darwin`
  - ARCH: `amd64/arm64/arm/386`
- Add matching artifact entry in `manifest.json`

---

## Contributing

Contributions are welcome.

### Development workflow

1. Fork the repo
2. Create a feature branch
3. Run tests:
   ```bash
   go test ./...
   ```
4. Submit PR with:
   - clear problem statement
   - minimal reproducible steps (if bug)
   - implementation notes

### Code style
- Keep core engine platform-agnostic
- Platform-specific logic should live behind build tags:
  - `//go:build windows`
  - `//go:build linux`
  - `//go:build darwin`

### What to work on
See [Roadmap](#roadmap) for planned tasks.

---

## Roadmap

- [ ] Add **Ed25519 signature verification** (manifest + artifact)
- [ ] GitHub Releases source (fetch latest release assets)
- [ ] Optional progress callbacks (download progress)
- [ ] Better launchd support (`bootstrap/bootout` workflows)
- [ ] Windows: wait for service STOPPED state (SC query) in helper
- [ ] Atomic lock file to prevent concurrent updates
- [ ] Integration examples (systemd unit, NSSM install scripts)
- [ ] Manifest generator script / CI workflow (multi-arch builds)

---

## License

This project is licensed under the **MIT License**.  
See the [LICENSE](./LICENSE) file for details.
