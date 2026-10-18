# Local notes: fix for "BlueZ timed out" (issue #640) + autostart

> Working document written by an AI (Mistral Vibe, Mistral AI) on 2026-10-18,
> at the request of the repository owner. Carried in this fork to
> document the setup: it describes the local configuration (systemd user
> service) in addition to the code patch. The code patch itself is committed
> as `6becfa74` (`feat(linux): auto-attach profile when device becomes available`).

## 1. Diagnosis (summary)

Two distinct failure modes at startup, observed in `journalctl` across 8 boots:

1. **BlueZ timeout** (issue #640): repeated `le-connection-abort-by-local`,
   then `Gave up after 5 attempts. Timed out` logged from
   `GalaxyBudsClient.Platform.Linux/BluetoothService.cs`.
   Cause: the earbuds are not accepting an LE connection (in the case,
   asleep, or connected to the phone — Buds4 Pro have no standard Bluetooth
   multipoint). This is NOT "BlueZ not ready": bluetoothd answers every
   attempt; what fails is the radio-level BlueZ→earbuds connection.
   Retrying more often does not help while the earbuds are unreachable.

2. **Residual error**: once the app gave up, it stopped listening to BlueZ.
   When the earbuds later became available (connected via GNOME, or put in
   the ears), the app never noticed: the SMEP profile was never attached,
   the error stayed on screen, and a manual click on "Connect" worked
   instantly (the existing already-connected shortcut).

There was also a crash at login (tray icon / GNOME AppIndicator extension,
Avalonia exception in `DBusTrayIconImpl.CreateTrayIcon`), worked around at
the system level — see §3. Not fixed in the application code to date.

## 2. Code patch (commit 6becfa74)

File: `GalaxyBudsClient.Platform.Linux/BluetoothService.cs` (+55 lines).

- New `RecoveryWatcher` (one-shot subscription to the BlueZ device's
  `Connected` D-Bus property).
- Armed on both give-up paths of `ConnectAsync`: the 5-attempt basic
  connection loop, and the timeout while waiting for the `Connected`
  property.
- Trigger: `Connected=true` (desktop-initiated connection, or the earbuds
  auto-connecting when worn) → automatically re-run `ConnectAsync`, which
  uses the existing already-connected shortcut to attach the SMEP profile
  in ~1 s.
- Guards: cleared whenever a new connect cycle starts (a manual click
  supersedes it); not armed if the post-connection watchdog is already
  active (the existing single-subscription invariant is preserved);
  reentrancy lock against overlapping recovery attempts.

### Honest limitations
- **This is a band-aid** on the symptom of issue #640, not a fix of the
  root causes: the kernel-side `le-connection-abort-by-local` failures and
  the ~50 s BlueZ per-attempt timeout are unchanged.
- **Event-driven only**: if the earbuds become reachable but nothing
  initiates a connection, no retry happens.
- **Watcher cost**: one persistent D-Bus subscription (the
  `PropertiesChanged` signal) while the app is in recovery state.
  No polling, no timers: the impact on laptop battery is negligible,
  but the subscription exists and costs a minimum.
- Behavior unchanged on Windows/macOS (Linux backend only).

### Tests performed (CachyOS, GNOME 51, BlueZ/PipeWire, Buds4 Pro)
1. Earbuds unreachable → 5 failures → watcher armed (log line
   `Retrying automatically once the device becomes available`).
2. Earbuds left on the phone → total silence, no retry loop.
3. Manual connection via GNOME (without touching the app) →
   `Device became available. Retrying connection...` then
   `Connected to profile` ~1 s later, app fully functional.
4. Concurrent double manual connection → absorbed by the pre-existing
   guards (`Already connecting`), no crash.

## 3. systemd service (autostart workaround)

File: `~/.config/systemd/user/galaxybudsclient.service`
(replaces the `~/.config/autostart/galaxybudsclient.desktop` entry, removed).

Important points of the local configuration:

- `WantedBy=graphical-session.target`: starts on graphical login.
- `ExecStartPre=/usr/bin/sleep 25` (currently reduced to 1 for testing):
  lets bluetoothd/wireplumber and the AppIndicator extension initialize.
  **CachyOS pitfall**: `/usr/lib/systemd/user.conf.d/00-timeout.conf` sets
  `DefaultTimeoutStartSec=15s` for user units → without the explicit
  `TimeoutStartSec=45s` in the unit, the sleep 25 is killed before it
  finishes and the service fails in a loop ("start-pre operation timed out").
- `Restart=on-failure` + `RestartSec=10`: relaunches the app if it crashes
  (tray icon race at login); by then the extension is ready and the
  relaunch succeeds.
- `StartLimitIntervalSec=120` / `StartLimitBurst=4` (in the `[Unit]`
  section): definite stop after 4 failures within 2 minutes (restarts
  spaced more than 10 s apart bypass the default start limit).
- `ExecStart` currently points to the locally built test binary:
  `/path/to/GalaxyBudsClient/bin_linux64/GalaxyBudsClient`
  (to be switched back to `/usr/bin/galaxybudsclient` once an official
  release/package includes the fix).

### Local build used
```bash
dotnet publish -f net10.0 -r linux-x64 -o bin_linux64 -c Release \
  -p:PublishSingleFile=true --self-contained true \
  GalaxyBudsClient/GalaxyBudsClient.csproj
```
Note: `global.json` pins SDK 10.0.300; with the installed SDK 10.0.113,
the version must be adjusted temporarily and the file restored afterwards
(the 2026-10-18 build was done this way; `global.json` is back to its
committed state).

## 4. For commenting on issue #640

Useful points to mention there:
- The error comes from the retry loop in `BluetoothService.cs`; the attempts
  fail at the radio level (`le-connection-abort-by-local`), not because
  BlueZ is unavailable.
- After the give-up, the app listens to no BlueZ signal: a later connection
  of the earbuds is never detected, hence the error that "stays".
- Proposed patch (commit 6becfa74, Linux backend): one-shot watcher on
  `Connected` + automatic retry. Hardware-tested (3 scenarios above).
- Uncovered leads: retry on a "device found during scan" event instead of
  `Connected=true` only; lightweight periodic state polling
  (`GetConnectedAsync`).

## 5. Rollback

- Code: `git revert 6becfa74` (or
  `git checkout 754b6fdf -- GalaxyBudsClient.Platform.Linux/BluetoothService.cs`
  before committing the revert).
- Service: restore `ExecStart=/usr/bin/galaxybudsclient /StartMinimized`,
  put `ExecStartPre=/usr/bin/sleep 25` back if desired, then
  `systemctl --user daemon-reload && systemctl --user restart galaxybudsclient.service`.
- Original setup: recreate `~/.config/autostart/galaxybudsclient.desktop`
  with `Exec=/usr/bin/galaxybudsclient /StartMinimized`, then
  `systemctl --user disable --now galaxybudsclient.service`.
