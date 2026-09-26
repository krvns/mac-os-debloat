### BLUF (Bottom Line Up Front)
- **Static Code Safety:** The audited codebase contains **zero** malicious primitives, dynamic code execution (`eval`/`exec`), external dependencies, or outbound network calls. Privilege escalation is restricted to tokenized `subprocess.run` calls prepended with `sudo` with `shell=False`.
- **Systemic Stability & Blast Radius:** Subsystem risk is contingent upon selection tier. The `telemetry` preset poses negligible operational risk. The `--disable-all` flag disables critical macOS daemons (`com.apple.akd`, `com.apple.campo`, `com.apple.filesystems.fskitd`, `com.apple.bridgeOSUpdateProxy`), breaking Apple ID authentication, desktop application launching on macOS 27, third-party filesystem mounts, and OS firmware updates.
- **Persistence & Reversibility:** Under System Integrity Protection (SIP), overrides written to `/var/db/com.apple.xpc.launchd/` for non-removable Apple services do not survive a cold reboot due to rootless restriction checks (`rootless.plist`). All mutations are reversible at runtime via native `launchctl` and `mdutil` CLI primitives or the bundled `--restore` snapshot.

---

### 1. Vulnerability & Threat Findings

| ID | Severity | File:Line | Threat Category | Technical Finding & Trigger Condition |
|---|---|---|---|---|
| **VUL-01** | High | [`debloat:57`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L57)<br>[`debloat:302`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L302)<br>[`debloat:420`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L420)<br>[`debloat:445`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L445)<br>[`debloat:1711-1723`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1711-L1723) | Subsystem Invalidation / Denial of Service | **Core System Subsystem Severance via `--disable-all`:** The script permits disabling critical infrastructural services: `com.apple.campo` (renders Cmd+Space and 4-finger gestures non-functional on macOS 27), `com.apple.akd` (breaks AuthKit / App Store / iCloud authentication), `com.apple.bridgeOSUpdateProxy` (aborts macOS system update installs), and `com.apple.filesystems.fskitd` (halts FSKit userspace file system drivers). Triggered via non-interactive CLI flag `--disable-all` or user selection in TUI. |
| **VUL-02** | Medium | [`debloat:911-976`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L911-L976)<br>[`debloat:1441-1442`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1441-L1442) | State Inconsistency / Fault Tolerance | **Non-Atomic Batch Mutation Pipeline:** `apply_changes` iterates sequentially across targeted labels executing blocking `launchctl disable` and `launchctl bootout` commands without asynchronous signal trapping (`SIGINT`, `SIGTERM`). An execution abort mid-batch leaves the system in a fragmented intermediate state. (Mitigated by pre-execution snapshot written to `~/.mac-os-debloat/latest.json` at line 1441). |
| **VUL-03** | Low | [`debloat:951-955`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L951-L955) | Process Management / Concurrency Race | **Signal 9 Termination Post-Bootout:** If a service process fails to terminate following `bootout`, Phase 2 issues `sudo kill -9 <pid>`. While target PIDs are filtered through direct inspection of `launchctl print <domain>` (`running_pids()`), rapid PID turnover in high-churn environments carries a residual PID recycling race condition. |
| **VUL-04** | Low | [`debloat:502`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L502)<br>[`debloat:567-594`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L567-L594)<br>[`debloat:1539-1553`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1539-L1553) | Unsanitized Input Parsing | **Arbitrary Launchd Target Injection via User Files:** `parse_labels` reads external files from `~/.mac-os-debloat/labels.txt` and `~/.mac-os-debloat/presets/*.txt`. Any valid launchd label defined in user configuration (including core daemons like `com.apple.securityd` or `com.apple.WindowServer`) will be matched against existing plists/registered jobs by `drop_absent_labels` and queued for deactivation. |
| **VUL-05** | Low | [`debloat:1058-1081`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1058-L1081) | Resource Exhaustion | **Spotlight Reindex Disk I/O Thrashing:** Toggling Spotlight enabled triggers `mdutil -a -i on` immediately followed by `mdutil -a -E`. Erasing and regenerating APFS metadata stores forces prolonged multi-threaded I/O and CPU thrashing by `mds_stores` and `mdworker` (10–30+ minutes on large storage volumes). |

---

### 2. Network & Exfiltration Audit

- **Network imports/sockets detected:** No.
  - Zero imports of `socket`, `http.client`, `urllib`, `requests`, `asyncio`, or external transport modules.
  - Verification: Standard library imports strictly limited to `curses`, `glob`, `json`, `os`, `re`, `subprocess`, `sys`, `textwrap`, `time`, `dataclasses`, `pathlib` ([`debloat:24-34`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L24-L34)).
- **Outbound traffic triggers:** None.
  - No invocations of `curl`, `wget`, `nc`, `ssh`, `dig`, or `nslookup`. (Strings containing `curl` in [`debloat:1000`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1000) and [`debloat:1737`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1737) are informational console docstrings).
- **Telemetry/Tracking mechanics:** None.
  - No hardware UUID, MAC address, telemetry beacons, serial number scraping, or remote logging channels present.

---

### 3. Execution & Privilege Audit

- **`sudo` invocation mechanics:**
  - Initial credential verification and timestamp caching executes via `sudo -v` ([`debloat:1016`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1016)).
  - Subsequent privileged operations explicitly prefix vector arrays with `"sudo"`:
    - Privilege de-escalation/service mutator: `["sudo", "launchctl", action, f"{domain}/{label}"]` ([`debloat:918`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L918)).
    - Process termination: `["sudo", "kill", "-9", str(pid)]` ([`debloat:951`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L951)).
    - Metadata index control: `["sudo", "mdutil", "-a", ...]` ([`debloat:1063`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1063), [`1069`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1069), [`1076`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L1076)).
  - Zero modifications to `/etc/sudoers` or `/etc/sudoers.d/`. No background credential persistence daemons or cached credential exfiltration.
- **`shell=True` or shell injection risks:** **PASS**.
  - All 11 `subprocess.run()` calls in `debloat` pass explicit list arguments (`argv` vectors) directly to the system `execve` interface with default `shell=False`.
  - Node wrapper [`bin/cli.js:11`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/bin/cli.js#L11) uses `spawnSync("python3", [script, ...process.argv.slice(2)], { stdio: "inherit" })` without shell interpolation.
  - Spliced strings (`f"{domain}/{label}"`) remain encapsulated within single vector tokens, preventing interpretation of shell control operators (`;`, `&&`, `|`, `$()`, `` ` ``).
- **Process termination mechanism (`kill -9` usage and targets):**
  - Targets are scoped dynamically by `running_pids()` ([`debloat:870-882`](file:///Users/pavkry/Documents/_projects/ai/mac-os-debloat/debloat#L870-L882)) querying `launchctl print <domain>`.
  - Extraction isolates the first field of the `services = { ... }` block matching registered services.
  - Signal 9 (`SIGKILL`) is dispatched exclusively to processes remaining resident after a deactivation cycle (`to_disable`) and failing clean exit under `launchctl bootout`.

---

### 4. macOS Subsystem Blast Radius Matrix

| Subsystem | Target Labels in Script | Potential Failure Mode / Side Effects | Recovery Mechanism |
|---|---|---|---|
| **Identity / Keychain / Auth** | `com.apple.akd`<br>`com.apple.appleaccountd`<br>`com.apple.adid`<br>`com.apple.identityservicesd`<br>`com.apple.AppSSODaemon`<br>`com.apple.AppSSOAgent`<br>`com.apple.security.keychain-circle-notification` | • `akd` deactivation induces synchronous XPC hangs/timeouts in Apple ID sign-in, System Settings, and App Store dialogs.<br>• `identityservicesd` breaks iMessage/FaceTime IDS registration and hardware key exchanges.<br>• `adid` breaks CoreADI hardware attestation.<br>• `security.keychain-circle-notification` halts iCloud Keychain synchronization. *(Local `securityd` is NOT targeted and remains functional).* | Execute in zsh:<br>`sudo launchctl enable gui/$(id -u)/com.apple.akd`<br>`sudo launchctl bootstrap gui/$(id -u) /System/Library/LaunchAgents/com.apple.akd.plist`<br>*(Or run `debloat --restore`)* |
| **Cloud / Sync** | `com.apple.cloudd`<br>`com.apple.cloudphotod`<br>`com.apple.bird`<br>`com.apple.syncdefaultsd`<br>`com.apple.icloudwebd`<br>`com.apple.SafariBookmarksSyncAgent` | • `bird` failure stops iCloud Drive syncing, document hydration, and local evictions.<br>• `cloudd` failure severs CloudKit transport for 1st- and 3rd-party apps (e.g., Bear, Things).<br>• `syncdefaultsd` breaks cross-device `NSUbiquitousKeyValueStore` syncing. | Execute in zsh:<br>`sudo launchctl enable gui/$(id -u)/com.apple.bird`<br>`sudo launchctl bootstrap gui/$(id -u) /System/Library/LaunchAgents/com.apple.bird.plist`<br>*(Or run `debloat --restore`)* |
| **GUI / App Launching / Windowing** | `com.apple.campo`<br>`com.apple.talagent`<br>`com.apple.chronod`<br>`com.apple.quicklook.ThumbnailsAgent`<br>`com.apple.filesystems.fskitd`<br>`com.apple.linkd` | • `campo`: On macOS 27, Siri AI/Campo hosts the unified Spotlight / Application Launcher UI. Disabling renders Cmd+Space and 4-finger pinch inoperative.<br>• `talagent`: App relaunch state and window layout restoration fail on reboot/login.<br>• `fskitd`: FSKit userspace daemon halts; third-party filesystem drivers (FUSE, NTFS) fail to mount.<br>• `quicklook.ThumbnailsAgent`: Finder file previews and icon thumbnails cease generating. | Execute in zsh:<br>`sudo launchctl enable gui/$(id -u)/com.apple.campo`<br>`sudo launchctl bootstrap gui/$(id -u) /System/Library/LaunchAgents/com.apple.campo.plist`<br>*(Or run `debloat --enable-all`)* |
| **Telemetry / Crash Reporting** | `com.apple.analyticsd`<br>`com.apple.ReportCrash`<br>`com.apple.spindump`<br>`com.apple.SubmitDiagInfo`<br>`com.apple.osanalytics.osanalyticshelper`<br>`com.apple.biomed`<br>`com.apple.BiomeAgent`<br>`com.apple.symptomsd-diag` | • `ReportCrash`: Unhandled exceptions (`EXC_BAD_ACCESS`, `SIGSEGV`) will not generate crash dumps in `/Library/Logs/DiagnosticReports/`, impeding post-mortem debugging.<br>• `spindump`: Hang logs and callstack sampling suppressed.<br>• Telemetry submission to Apple infrastructure is blocked.<br>*(Note: `ReportCrash` is listed in `/System/Library/Sandbox/com.apple.xpc.launchd.rootless.plist` and will persist disabled across reboots).* | Execute in zsh:<br>`sudo launchctl enable system/com.apple.ReportCrash`<br>`sudo launchctl bootstrap system /System/Library/LaunchDaemons/com.apple.ReportCrash.plist` |
| **System Updates & Firmware** | `com.apple.bridgeOSUpdateProxy`<br>`com.apple.bosreporter`<br>`com.apple.boswatcher`<br>`com.apple.SoftwareUpdateNotificationManager` | • `bridgeOSUpdateProxy`: Disabling severs the firmware update staging channel on Apple Silicon and Intel T2 Macs; macOS Software Update installation stages fail verification. | Execute in zsh:<br>`sudo launchctl enable system/com.apple.bridgeOSUpdateProxy`<br>`sudo launchctl bootstrap system /System/Library/LaunchDaemons/com.apple.bridgeOSUpdateProxy.plist` |

---

### 5. Verification Checklist (Binary Pass/Fail)

- **Zero external dependencies verified:** **PASS**
  - Confirmed: Runtime utilizes standard Python 3.12 libraries and native macOS system binaries (`launchctl`, `mdutil`, `sw_vers`, `vm_stat`, `kill`).
- **Zero network requests verified:** **PASS**
  - Confirmed: No network socket initialization, HTTP client calls, or indirect CLI network wrappers.
- **No persistence mechanisms injected:** **PASS**
  - Confirmed: Zero writes to `/Library/Launch*`, `~/Library/Launch*`, `/etc/`, shell profiles (`.zshrc`, `.bashrc`), or cron tables. Local storage writes are restricted strictly to state snapshots in `~/.mac-os-debloat/`.
- **State changes 100% reversible via standard macOS CLI:** **PASS**
  - Confirmed: Service state mutates standard launchd domain override tables (`/var/db/com.apple.xpc.launchd/`). Reversible via native `launchctl enable <domain>/<label>` and `launchctl bootstrap <domain> <plist_path>`, the built-in non-interactive rollback `debloat --restore`, or full recovery `debloat --enable-all`. Spotlight indexing is restored via `sudo mdutil -a -i on && sudo mdutil -a -E`.