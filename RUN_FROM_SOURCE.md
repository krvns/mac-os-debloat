
# Running from source

```bash
./debloat
# or: python3 debloat
```

## Commands

Runs the interactive TUI by default. Non-interactive flags for scripting and quick recovery:

```bash
debloat                    # interactive TUI (default)
debloat --preset telemetry # disable analytics, crash reports, ads, beta enrollment (47)
debloat --preset balanced  # telemetry + Siri, Apple Intelligence, iMessage, Family (184)
debloat --list             # print every label, in preset file format
debloat --status           # per-domain disabled/enabled, how many run, how many ignore the override, free RAM
debloat --audit            # list any embedded labels not present on your macOS build
debloat --disable-all      # disable every label, no exceptions (prompts sudo)
debloat --enable-all       # re-enable everything — the panic button
debloat --restore          # revert to the state before your last apply
debloat --dry-run          # with --preset/--disable-all/--enable-all: preview only
debloat --status --json    # machine-readable status
```

An apply is two `sudo launchctl` calls per label per domain, so `--disable-all` runs several hundred of them; it prints a `disabling 137/296 com.apple.…` line in place while it works, on the command line and in the TUI alike. Piped output gets none of that.

An apply prompts for your sudo password on the TUI's own bottom line — the TUI never drops back to your shell, and the progress line and the result land in the same place. `--status`, `--audit`, `--list`, and `--dry-run` need no sudo — reading launchd state is unprivileged. Every apply first snapshots your current state to `~/.mac-os-debloat/latest.json`, so `--restore` always brings you back. If anything feels off, `debloat --enable-all` turns it all back on.