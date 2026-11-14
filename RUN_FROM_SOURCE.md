# Running from Source

Local, zero-dependency execution guide for `mac-os-debloat`.

```bash
# Direct local execution
./debloat
# or:
python3 debloat
```

---

## 1. Local Setup & Security Guardrails

- **Zero External Dependencies:** Built solely on the Python standard library (`curses`, `subprocess`, `json`, `pathlib`). No `pip install` or external runtimes required.
- **Air-Gapped / Zero Network Calls:** Does not initiate network sockets or outbound requests. Executing locally from disk guarantees zero network activity and no IP/TLS metadata leakage.
- **Root Elevation:** Reads (`--status`, `--audit`, `--list`, `--dry-run`) are completely unprivileged. Writes (`--preset`, `--disable-all`, `--enable-all`, `--restore`, or TUI apply) execute privileged actions via `sudo` with explicit argument arrays (`shell=False`).

---

## 2. CLI Command Reference

```bash
./debloat                    # Launch interactive curses TUI (default)
./debloat --preset telemetry # Disable analytics, crash reporting, ads, beta enrollment
./debloat --preset balanced  # Telemetry + Siri, Apple Intelligence, iMessage, Family
./debloat --dry-run          # Preview changes for any action without mutating launchd
./debloat --status           # Summary of disabled/enabled services, active pids, reclaimable RAM
./debloat --status --json    # Machine-readable JSON status output
./debloat --audit            # Inspect embedded labels vs active macOS build compatibility
./debloat --list             # Export all known catalog labels in preset format
./debloat --restore          # Revert to snapshot state saved before the last apply
./debloat --enable-all       # Panic button: Re-enable all services and remove boot daemon
./debloat --disable-all      # Disable every catalog service (breaks iCloud, App Store, Campo)
```

---

## 3. Recommended Operational Workflow

### Step 1: Audit System Compatibility
Inspect which services exist on your specific macOS build before applying changes:
```bash
./debloat --audit
```

### Step 2: Dry-Run Verification
Preview exactly which launchd labels will be disabled or enabled:
```bash
./debloat --preset telemetry --dry-run
```

### Step 3: Apply Preset
Apply changes (prompts for `sudo` to register overrides with `launchctl`):
```bash
./debloat --preset telemetry
```

### Step 4: Verify Active State
Verify running processes and active overrides:
```bash
./debloat --status
```

---

## 4. Local Boot Daemon Management

Under macOS System Integrity Protection (SIP), launchd drops custom overrides across reboots for non-allowlisted Apple services. To persist your requested disabled state across restarts without turning SIP off, applying changes automatically manages a minimal root LaunchDaemon:

- **Daemon Label:** `io.github.techiknow.debloat`
- **Configuration Plist:** `/Library/LaunchDaemons/io.github.techiknow.debloat.plist`
- **State Directory:** `/Library/Application Support/mac-os-debloat/persist.json`
- **Logging:** Stdout and stderr are redirected to `/dev/null` (no disk logging or tracking).

### Inspecting the Daemon
```bash
# Check launchd registration status
sudo launchctl list | grep "io.github.techiknow.debloat"

# Inspect daemon state
launchctl print system/io.github.techiknow.debloat
```

### Clean Removal of the Daemon
Running `--enable-all` automatically tears down the daemon:
```bash
./debloat --enable-all
```

Or remove manually via native macOS commands:
```bash
sudo launchctl bootout system/io.github.techiknow.debloat 2>/dev/null
sudo rm -f /Library/LaunchDaemons/io.github.techiknow.debloat.plist
sudo rm -rf "/Library/Application Support/mac-os-debloat"
```

---

## 5. Rollback & Emergency Recovery

### Method 1: Automatic Snapshot Restore
Every apply operation automatically snapshots pre-mutation state to `~/.mac-os-debloat/latest.json`. To restore the system to that state:
```bash
./debloat --restore
```

### Method 2: Global Re-enable (Panic Button)
Re-enables every catalog label in all domains:
```bash
./debloat --enable-all
```

### Method 3: Granular Native macOS CLI Recovery
If a specific core subsystem experiences an unexpected XPC failure or hang, recover it directly using macOS system binaries:

- **Apple ID / AuthKit (`com.apple.akd`):**
  ```bash
  sudo launchctl enable gui/$(id -u)/com.apple.akd
  sudo launchctl bootstrap gui/$(id -u) /System/Library/LaunchAgents/com.apple.akd.plist
  ```

- **Spotlight / App Launcher on macOS 27 (`com.apple.campo`):**
  ```bash
  sudo launchctl enable gui/$(id -u)/com.apple.campo
  sudo launchctl bootstrap gui/$(id -u) /System/Library/LaunchAgents/com.apple.campo.plist
  ```

- **iCloud Drive File Sync (`com.apple.bird`):**
  ```bash
  sudo launchctl enable gui/$(id -u)/com.apple.bird
  sudo launchctl bootstrap gui/$(id -u) /System/Library/LaunchAgents/com.apple.bird.plist
  ```

- **Firmware & macOS Software Updates (`com.apple.bridgeOSUpdateProxy`):**
  ```bash
  sudo launchctl enable system/com.apple.bridgeOSUpdateProxy
  sudo launchctl bootstrap system /System/Library/LaunchDaemons/com.apple.bridgeOSUpdateProxy.plist
  ```

---

## 6. Spotlight Index Management

The utility includes full Spotlight index control:

```bash
# Check Spotlight indexing status
mdutil -s /

# Disable indexing and searching globally
sudo mdutil -a -d

# Re-enable indexing and trigger metadata rebuild
sudo mdutil -a -i on && sudo mdutil -a -E
```

---

## 7. Custom Presets & Labels

Create custom presets without modifying the source:

```bash
# 1. Export catalog template
mkdir -p ~/.mac-os-debloat/presets
./debloat --list > ~/.mac-os-debloat/presets/custom.txt

# 2. Edit the file to keep only services you want disabled

# 3. Apply custom preset
./debloat --preset custom
```

Append personal launchd labels to `~/.mac-os-debloat/labels.txt` to include them in the catalog.
