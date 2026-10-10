# xtop: a macOS-first btop clone in Rust + ratatui

> Status: Planning · Owner: @s4njee · Last updated: 2026-10-10

## 1. Vision

`xtop` is a terminal resource monitor with the look and keyboard feel of
[btop](https://github.com/aristocratos/btop), built **for macOS first**. On a Mac
it should be at least as informative as Activity Monitor and fit in a terminal:

- **Apps, not just PIDs.** Group processes into the apps people recognize
  (`Google Chrome`, not 30 `Google Chrome Helper (Renderer)` rows), with total
  CPU, GPU, memory, energy and disk use for each app.
- **Services.** Show launchd daemons and agents, what each one is running, how
  much it costs, and whether it is crash-looping. Also show third-party
  background items and Homebrew services.
- **Apple Silicon numbers.** P-core and E-core clusters, GPU use, per-process
  GPU time, CPU/GPU/ANE power, thermal pressure, and memory pressure the way
  macOS defines it (app memory, wired, compressed, swap).
- **Game Performance view.** A compact view of total CPU, GPU and memory use
  (plus power, thermals and a tracked game's own usage). It is meant to sit on a
  second screen or in a small terminal while a game runs, so you can see whether
  the game is GPU-bound, CPU-bound, memory-bound or thermally throttled.

### 1.1 Goals

1. Feature parity with btop's core views (CPU, memory, network, disks, processes)
   on macOS. A good number stays a good number: only render data we trust.
2. Parity with Activity Monitor's CPU, Memory, Energy, Disk and GPU columns where
   the data can be read without root, and graceful degradation where it can't.
3. First-class **Apps**, **Services** and **Game** views that btop doesn't have.
4. Run without `sudo` by default. Optional features get better under `sudo`.
5. Low overhead: **under 1% of one core at a 1 s refresh** in Game view, and
   under 2% in the full dashboard.
6. Single static binary, Homebrew installable, signed and notarized.

### 1.2 Non-goals (for 1.0)

- Measuring a game's FPS or frame times from outside the game. This isn't
  possible without injecting into the process; see story S12.9.
- A GUI or menu-bar app.
- Mac App Store distribution (private APIs rule it out).
- Linux and Windows backends. The architecture keeps this possible (E16), but
  nothing in 1.0 depends on it.
- Per-process network accounting (stretch, S7.5).

## 2. Gap analysis: btop on macOS vs. Activity Monitor

| Area | btop on macOS today | Activity Monitor | xtop target |
|---|---|---|---|
| CPU | Per-core %, no P/E split, freq often missing | %CPU, idle wakeups, Kind (Apple/Intel) | Per-core % grouped by P/E cluster, cluster frequency, power, Kind column |
| GPU | None | % GPU, GPU Time per process, GPU history window | Total GPU use, freq, power, in-use memory, **per-process GPU %** |
| Memory | used/free/cached from a Linux-style model | Memory Used = App + Wired + Compressed, Cached Files, Swap, **Memory Pressure** | Exact AM breakdown and pressure graph; per-process *footprint* (AM's "Memory") |
| Processes | Flat or tree list of PIDs | Grouped by app in Energy tab; hierarchical view | Process list **plus** an Apps view grouped by coalition/responsible process |
| Energy | None | Energy Impact, 12 hr power, App Nap, Preventing Sleep | Energy Impact estimate, wakeups, sleep assertions, battery state |
| Services | None | None (needs `launchctl`) | launchd daemons/agents with state, PID, cost and exit status; Homebrew services |
| Thermal | Temps unreliable on Apple Silicon | None | Thermal pressure state, SoC/CPU/GPU temps, fan RPM |
| Gaming | No | No | Dedicated Game Performance HUD with session stats and recording |

## 3. Platform targets & constraints

- **OS:** macOS 13 Ventura or later. Primary test targets are macOS 15 and 26.
- **Hardware:** Apple Silicon (M1–M5 families) is first class. Intel Macs are
  best effort: CPU, memory, processes, disk and network must work; GPU, power
  and temps are "if available".
- **Privileges:** runs unprivileged. Known limits without root:
  - `proc_pidinfo(PROC_PIDTASKINFO)` and `proc_pid_rusage` fail with `EPERM` for
    other users' and root's processes. Those rows show name, PID, user and path,
    with dimmed `—` for unavailable metrics. Launching with `sudo` fills them in.
  - `sfltool dumpbtm` (the Background Task Management database) needs root.
  - Controlling services in the `system` domain needs root, so we shell out to
    `sudo` after suspending the TUI.
- **Private APIs** (IOReport, coalitions, responsibility, AGX user clients, SMC,
  IOHID sensors): wrapped behind capability checks, versioned per macOS
  release, and allowed to fail without crashing. Every private API gets a
  spike (Section 7) before we commit to it.
- **Terminal:** truecolor preferred; 256-color and 16-color fallbacks. Works in
  Terminal.app, iTerm2, Ghostty, WezTerm, Kitty, Alacritty, and inside tmux.

## 4. Architecture

### 4.1 Workspace layout

```
xtopv2/
├── Cargo.toml                 # workspace
├── crates/
│   ├── xtop-core/             # platform-agnostic: domain model, units, history
│   │                          #   ring buffers, aggregation, stats (p95…), config types
│   ├── xtop-macos/            # macOS collectors
│   │   └── src/sys/           #   ALL unsafe FFI lives here (mach, libproc, sysctl,
│   │                          #   IOKit, CoreFoundation, IOReport, SMC, objc2)
│   ├── xtop-tui/              # ratatui widgets, views, layout, theme, keymap
│   └── xtop/                  # binary: CLI, wiring, event loop, headless modes
├── fixtures/                  # recorded snapshots (JSON) for tests and --replay
├── themes/                    # built-in themes
└── docs/                      # data-source notes, accuracy notes, FAQ
```

### 4.2 Runtime model

```
          ┌────────────── collector threads ──────────────┐
          │ fast (1s):  cpu, mem, procs, gpu, power, net  │
          │ medium (2s): disks, temps, fans, battery       │──► Snapshot (Arc) ──► channel ──► UI thread
          │ slow (5–10s): services, bundle metadata, BTM   │                                  │
          └───────────────────────────────────────────────┘                                  ▼
                                   ▲                                    history store (ring buffers)
                                   │ commands (refresh now, focus pid,         │
                                   │ lite mode, signal, service action)        ▼
                                   └──────────────────────────────────── ratatui render (≤ 30 fps,
                                                                          only on change / input)
```

- **Collectors** implement a `Collector` trait: `fn sample(&mut self, ctx) ->
  Result<Partial>`. Each collector runs on its own interval and owns its
  previous-sample state for delta calculations.
- A **Scheduler** merges partial results into an immutable `Snapshot` and sends
  it as an `Arc<Snapshot>` over a `crossbeam-channel`. The UI never blocks on
  I/O or FFI.
- **History** lives in `xtop-core`: fixed-capacity ring buffers keyed by metric
  id, PID+start time, app id (bundle id) or service label. Capacity is set by
  duration, so 1 s × 300 samples is 5 minutes.
- **Commands** flow from UI to collectors: refresh now, change interval, enter
  lite mode for Game view, focus a target app. Actions such as kill, quit app
  or service control run on a separate worker so the UI stays responsive.
- **Time:** deltas use `mach_continuous_time` / `Instant`. A sample is dropped
  if its gap exceeds 3× the interval (sleep/wake, SIGSTOP), so graphs don't
  spike.
- **Availability:** every metric is `Metric<T> = Ok(T) | NeedsRoot |
  Unsupported | Error(String)`. Widgets render each state on purpose (value,
  `🔒`, `n/a`, `!`).

### 4.3 Key crates

| Purpose | Crate |
|---|---|
| TUI | `ratatui`, `crossterm` |
| Mach / libc | `mach2`, `libc` |
| CoreFoundation / IOKit | `core-foundation`, `io-kit-sys` (plus our own bindings for IOReport and SMC) |
| AppKit (NSWorkspace, NSRunningApplication) | `objc2`, `objc2-foundation`, `objc2-app-kit` |
| Plists | `plist` |
| CLI / config | `clap`, `serde`, `toml`, `directories` |
| Errors / logging | `thiserror`, `anyhow` (bin only), `tracing`, `tracing-appender` (logs go to file) |
| Concurrency | `crossbeam-channel` |
| Testing | `insta` (TUI snapshots via `ratatui::backend::TestBackend`), `proptest`, `criterion` |

Reference projects to learn from (no code copying without checking licenses):
`macmon` (IOReport power and frequency without sudo), `asitop`, `btop`'s macOS
collector, `bottom`, `sysinfo`, `nvtop` (Apple GPU support).

## 5. macOS data source reference

| Metric | Source | Root? | Notes |
|---|---|---|---|
| Per-core CPU % | `host_processor_info(PROCESSOR_CPU_LOAD_INFO)` | No | Tick deltas: user/system/idle/nice |
| P/E topology | `sysctl hw.perflevel{0,1}.{logicalcpu,name}`, `hw.nperflevels` | No | Absent on Intel |
| CPU model | `sysctl machdep.cpu.brand_string` | No | |
| Cluster freq / residency | IOReport "CPU Stats" channels plus DVFS tables from IORegistry (`pmgr`) | No | Private; macmon's approach. Spike SP1 |
| Load avg / uptime | `getloadavg`, `sysctl kern.boottime` | No | |
| VM stats | `host_statistics64(HOST_VM_INFO64)` | No | free/active/inactive/wired/speculative/compressor/purgeable/internal/external pages |
| AM memory breakdown | App = internal − purgeable; Wired; Compressed = compressor_page_count; Cached = external + purgeable; Used = App + Wired + Compressed | No | Validate against AM (S2.4) |
| Memory pressure | `sysctl kern.memorystatus_vm_pressure_level` (1/2/4), `kern.memorystatus_level` (% available) | No | |
| Swap | `sysctl vm.swapusage`; swapins/swapouts from VM stats | No | |
| Process list | `proc_listallpids` | No | |
| Process basics | `proc_pidinfo(PROC_PIDTBSDINFO)`, `proc_pidpath`, `sysctl KERN_PROCARGS2` | Partly | Args of other users' processes need root |
| Process CPU / threads | `proc_pidinfo(PROC_PIDTASKINFO)` | Own uid only | **Apple Silicon times are mach ticks**: convert with `mach_timebase_info` |
| Process memory | `proc_pid_rusage(RUSAGE_INFO_V6)`: `ri_phys_footprint` (AM "Memory"), `ri_resident_size` (Real Mem) | Own uid only | |
| Process disk I/O | `ri_diskio_bytesread/written` | Own uid only | |
| Wakeups / energy | `ri_pkg_idle_wkups`, `ri_interrupt_wkups`, `ri_billed_energy`, `ri_energy_nj` | Own uid only | Energy Impact is our own estimate (S3.5) |
| Kind (Apple/Intel) | `sysctl KERN_PROC_PID` → `p_flag & P_TRANSLATED` | No | Shows Rosetta |
| Coalition id | `proc_pidinfo(PROC_PIDCOALITIONINFO)` | ? | Private-ish. Spike SP3 |
| Responsible PID | `responsibility_get_pid_responsible_for_pid` | No | Private symbol in libsystem. Spike SP3 |
| Sleep assertions | `IOPMCopyAssertionsByProcess` | No | "Preventing Sleep" |
| Running apps | `NSWorkspace.runningApplications`, `frontmostApplication` | No | Needs run-loop pumping in a CLI. Spike SP5 |
| Bundle metadata | `Info.plist` via `CFBundle` / `plist` | No | Cached by bundle path + mtime |
| Topmost window owner | `CGWindowListCopyWindowInfo` (owner PID/name only) | No | No Screen Recording permission needed for owner fields |
| GPU utilization | IORegistry `IOAccelerator` → `PerformanceStatistics` (Device/Renderer/Tiler Utilization %, In use system memory) | No | Intel/AMD keys differ |
| Per-process GPU | IORegistry AGX `IOUserClient` children: `IOUserClientCreator` ("pid N, name") + `AppUsage[].accumulatedGPUTime` | No | Spike SP2 |
| Power (CPU/GPU/ANE/DRAM) | IOReport "Energy Model" group | No | Spike SP1 |
| System power | SMC `PSTR` | No | Key availability varies |
| Temperatures | SMC keys and/or `IOHIDEventSystemClient` temperature services | No | Per-chip sensor map. Spike SP6 |
| Fans | SMC `FNum`, `F{n}Ac`, `F{n}Mn`, `F{n}Mx` | No | Fanless models return 0 fans |
| Thermal pressure | `notify_register_check("com.apple.system.thermalpressurelevel")` / `NSProcessInfo.thermalState` | No | nominal/fair/serious/critical |
| Battery | `IOPSCopyPowerSourcesInfo`, IORegistry `AppleSmartBattery` | No | Cycle count, health, watts |
| Volumes | `getfsstat`, `statfs`, `NSURLVolumeAvailableCapacityForImportantUsageKey` | No | Hide APFS system/snapshot volumes by default |
| Disk I/O | IORegistry `IOBlockStorageDriver` → `Statistics` | No | Per physical disk |
| Network | `sysctl NET_RT_IFLIST2` (`if_data64`) | No | Avoid 32-bit `getifaddrs` counters, which wrap |
| Interface names | SystemConfiguration (`SCNetworkInterfaceCopyAll`) | No | "Wi-Fi", "Ethernet", "Thunderbolt Bridge" |
| launchd services | `launchctl print system`, `launchctl print gui/$UID`, `launchctl list`, `launchctl print-disabled` | Partly | Output isn't a stable API: fixture tests per macOS version. Spike SP4 |
| Service definitions | `/System/Library/Launch{Daemons,Agents}`, `/Library/Launch{Daemons,Agents}`, `~/Library/LaunchAgents` | No | |
| Background items (BTM) | `sfltool dumpbtm` | **Yes** | Optional enrichment |

## 6. Views & mockups

Views switch with number keys and Tab: `1` Dashboard · `2` Apps · `3` Services ·
`4` Game · `5` Processes (full screen). Inside the Dashboard, `F1`–`F6` (or
remapped keys) toggle boxes, like btop's `1`–`4`.

### 6.1 Dashboard (btop parity, plus GPU)

```
┌ cpu ── Apple M3 Pro · 6P + 6E ─────────── P 3.7GHz E 2.0GHz ── 54°C ─ 6.2W ┐┌ gpu ─ 18-core ─ 1.1GHz ─ 2.4W ┐
│⣀⣠⣤⣴⣶⣾⣿⣷⣶⣦⣤⣀⣀⣠⣤⣶⣾⣿⣿⣿⣷⣶⣤⣄⣀  total 38%                                       ││⣀⣀⣠⣤⣴⣶⣦⣤⣀⣀  util 41%           │
│ P0 ▇▇▇▇▆  71%  P1 ▃▃     22%  P2 ▅▅▅   44%   E0 ▁   4%  E1 ▂   9%  E2 ▁  3% ││ mem in use 3.2G               │
│ P3 ▂      10%  P4 ▆▆▆▆   58%  P5 ▃▃    20%   E3 ▁   2%  E4 ▁   5%  E5 ▁  2% ││ renderer 39% · tiler 12%      │
│ load 3.12 2.80 2.41 · up 4d 02:11 · procs 712 · threads 3,904               │└───────────────────────────────┘
└─────────────────────────────────────────────────────────────────────────────┘┌ mem ── 36.0G ─────────────────┐
┌ net ── en0 Wi-Fi ───────────────────────┐┌ disks ─────────────────────────┐│ Used     21.4G ██████████▌    │
│ ▼ 12.4 MB/s ⣀⣠⣤⣶⣿⣷⣤⣀   total 4.1G     ││ Macintosh HD  612G/994G ██████ ││  App     14.2G                │
│ ▲  0.8 MB/s ⣀⣀⣠⣀⣀⣀⣀⣀   total 312M     ││  R 45 MB/s  W 12 MB/s          ││  Wired    3.6G  Compr  3.6G   │
└─────────────────────────────────────────┘└────────────────────────────────┘│ Cached   11.9G  Swap 0B/0B    │
                                                                             │ Pressure ▁▁▂▂▂▃▂ 28% normal    │
┌ proc ── [f]ilter  [t]ree  [s]ort:cpu ▼  [a]pps  [e]nergy ────────────────────────────────────────────────────┘
│   PID  Name                    User   Kind   CPU%  GPU%   Memory  Thr  Wakeups  Energy                        │
│  4121  bg3                     sanjee arm64 312.4  88.1    9.71G   87     124    97.2                        │
│   612  WindowServer            _wind  arm64  14.2   9.0    1.02G   24     310    12.1                        │
│  ...                                                                                                          │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Apps view

```
┌ apps ── 23 apps · 4 menu-bar · 61 background ── [h]ide Apple  [enter] expand  [q]uit app  [Q] force quit ──┐
│   App                          Kind    CPU%   GPU%   Memory   Energy  Disk R/W     Procs  Threads  Since   │
│ ── Foreground ──                                                                                          │
│ ▼ Google Chrome 129.0          arm64   38.2    4.1    3.21G    41.0   1.2M/0.4M      23     412    3h12m │
│     Google Chrome                       6.1    0.0     410M                                              │
│     Google Chrome Helper (GPU)         12.4    4.1     890M                                              │
│     Google Chrome Helper (Renderer) ×18 19.7   —      1.90G                                              │
│ ▶ Safari 18.4 (+ WebKit ×6)    arm64   11.0    1.2    1.44G    12.3   …                                   │
│ ▶ Steam                        x86_64   4.3    0.4     812M     6.0   ← Rosetta                          │
│ ── Menu bar / accessory ──                                                                                │
│ ▶ Rectangle                    arm64    0.1    0.0      48M     0.1                                       │
│ ── Background (bundled agents) ──                                                                         │
│ ▶ Docker Desktop               arm64    9.8    0.0    4.02G     8.4                                       │
│ ── System (no bundle) ──                                                                                  │
│ ▶ kernel_task / launchd / …                                                                               │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Services view

```
┌ services ── [d]omain:all  [o]wner:3rd-party  [s]tate:any ── 412 loaded · 98 running · 3 failing ──────────┐
│ Domain    Label                              State     PID    CPU%  Memory  Exit  Restarts  Source        │
│ system    homebrew.mxcl.postgresql@16        running    830    0.1    45M     0         0   /Library/…    │
│ gui/501   com.docker.helper                  failing      -      -      -    78 ⚠      14   ~/Library/…   │
│ gui/501   com.spotify.webhelper              stopped      -      -      -     0         0   ~/Library/…   │
│ system    com.apple.mDNSResponder            running    402    0.3    12M     0         0   /System/…     │
├ detail ── com.docker.helper ──────────────────────────────────────────────────────────────────────────────┤
│ Program  /Applications/Docker.app/Contents/Library/LaunchServices/com.docker.helper                       │
│ KeepAlive: true · RunAtLoad: true · last exit 78 (EX_CONFIG) · respawned 14× in 10 min                     │
│ [k]ickstart  [b]ootout  [D]isable  [e]dit plist  [r]eveal                                                 │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.4 Game Performance view (new)

Full layout (≥ 100×14):

```
┌ xtop · GAME ── Baldur's Gate 3 · pid 4121 · arm64 · fullscreen ── session 00:42:17 ── ● REC ───────── M3 Pro ┐
│ CPU  63%  ▁▂▃▅▆▇▆▅▅▆▇▆▅▆▇▇▆▅  P 78% @3.70GHz  E 21% @2.06GHz  hottest core P2 97%   avg 58  p95 71  max 89  │
│ GPU  97%  ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇  1.40GHz · renderer 96% · tiler 41% · 6.1G in use    avg 94  p95 99  max 100 │
│ MEM  21.8 / 36.0G  ██████████▋     pressure ▂ normal · swap 0B · swap-out 0 MB/s · compressed 1.2G         │
│ PWR  23.4W  CPU 8.1 · GPU 12.3 · ANE 0.0 · DRAM 1.6       thermal NOMINAL · SoC 78°C · GPU 74°C · fan 3200 │
├ game: Baldur's Gate 3 ────────────────────────────────────────────────────────────────────────────────────┤
│ CPU 412% (4.1 cores) ▅▆▆▇▆▆  GPU 91% ▇▇▇▇▇▇  Mem 9.71G ▃▃▄▄▄▅  Threads 87  Disk R 22 MB/s                  │
├ verdict ──────────────────────────────────────────────────────────────────────────────────────────────────┤
│ ● GPU-bound (GPU ≥ 95% for 92% of last 60s; P-cluster has headroom)                        [m]ark  [r]eset │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

Compact layout (≈ 44×8, for a corner of a second display):

```
┌ GAME bg3 ─ 00:42 ─ NOMINAL ┐
│ CPU  63% ▅▆▇▆  P78 E21     │
│ GPU  97% ▇▇▇▇  1.40GHz     │
│ MEM 21.8/36G  press normal │
│ PWR 23.4W  SoC 78° fan 3.2k│
│ game C412% G91% M9.7G      │
└────────────────────────────┘
```

One-line mode (`xtop --oneline`, for tmux or SketchyBar):

```
CPU 63% · GPU 97% · MEM 21.8/36G (normal) · 23.4W · NOMINAL · bg3 C412% G91%
```

## 7. Technical spikes (do first, in M0)

Each spike is time-boxed (≤ 2 days). It produces a tiny Rust proof-of-concept
under `spikes/` plus a short write-up in `docs/spikes/NN-name.md` with: works
without root (y/n), tested macOS versions and chips, sample output, gotchas,
and a go/no-go decision.

| ID | Question | Fallback if no-go |
|---|---|---|
| SP1 | Can we read CPU/GPU/ANE power and cluster frequency through IOReport **without root** on M1–M5? | Hide power and freq; offer `sudo powermetrics --format plist` parsing as an opt-in source |
| SP2 | Is per-process GPU time readable from AGX `IOUserClient` `AppUsage` entries, and does the % match Activity Monitor? | Hide per-process GPU; keep total GPU |
| SP3 | Can we read coalition ids and/or responsible PIDs for all processes unprivileged? How well do they group helpers and XPC services? | Bundle-path + parent-chain grouping only |
| SP4 | What do `launchctl print system` / `gui/$UID` show unprivileged (PIDs, exit status)? How stable is the format across macOS 13–26? | `launchctl list` (user domain) + plist scanning; system domain only under sudo |
| SP5 | Can a CLI binary use `NSWorkspace` (running apps, frontmost app, activation policy) by pumping `CFRunLoopRunInMode` on the main thread? | `CGWindowListCopyWindowInfo` for frontmost; Info.plist `LSUIElement`/`LSBackgroundOnly` for activation policy |
| SP6 | Which SMC / IOHID temperature sensors are available per chip family, and what are their semantics? | Show thermal pressure state only |
| SP7 | Exactly which fields can we read unprivileged for root and other-user processes (`proc_pidinfo`, `proc_pid_rusage`, `KERN_PROCARGS2`, `proc_pidpath`)? | Defines the "🔒 needs root" UX |
| SP8 | Can we detect macOS Game Mode being active, and can `open --env MTL_HUD_ENABLED=1` launch a game with the Metal HUD? | Document manual steps only |

## 8. Milestones

| Milestone | Theme | Exit criteria |
|---|---|---|
| **M0 Foundation** | Workspace, CI, FFI layer, spikes SP1–SP8 | Spikes written up; `xtop --dump-json` prints a CPU+memory snapshot |
| **M1 btop-on-Mac MVP** | CPU, memory, processes, dashboard, keys | Dashboard with CPU, memory and process boxes matches `top`/AM within tolerance; filter, sort, tree, kill |
| **M2 Apps & Services** | E4, E5, E10, E11 | Apps view groups Chrome/Safari/Electron apps correctly; Services view lists, joins and controls launchd jobs |
| **M3 GPU, Power, Game** | E6, E12 | Game view usable during a real gaming session at < 1% CPU overhead; recording + summary |
| **M4 Completeness & polish** | E7, E9 remainder, E13, E14 | Net and disks boxes, themes, config menu, mouse, 24 h soak test clean |
| **M5 1.0 Release** | E15 | Signed and notarized universal binary, Homebrew tap, docs, screenshots |

## 9. Epics & stories

**Conventions**
- IDs: `E<n>` epic, `S<n>.<m>` story.
- Priority: **P0** must have for the milestone · **P1** should have · **P2** nice to have.
- Size: **S** ≤ 1 day · **M** 2–3 days · **L** ≤ 1 week. Anything bigger is split.
- Every story's Definition of Done includes the items in Section 12.

---

### E0 Foundation & architecture (M0)

**S0.1: Scaffold the Cargo workspace** · P0 · S
- Create the crates from Section 4.1 with Rust edition 2024 and a pinned MSRV in `rust-toolchain.toml`.
- Add `rustfmt.toml`, workspace clippy lints (`unsafe_op_in_unsafe_fn`, `undocumented_unsafe_blocks` in `xtop-macos`), and `deny.toml`.
- **AC:** `cargo build --workspace` succeeds on macOS; `xtop-core` also builds on Linux.

**S0.2: Set up CI** · P0 · S
- GitHub Actions: macOS arm64 runner (build, test, clippy `-D warnings`, fmt check, cargo-deny), an Intel macOS runner while one exists, and Linux `cargo check -p xtop-core`.
- **AC:** required checks on PRs; caching keeps CI under 10 min.

**S0.3: Terminal safety & logging** · P0 · S
- Use `tracing` with a file appender at `~/Library/Logs/xtop/xtop.log` (configurable with `--log-level`).
- A panic hook and SIGINT/SIGTERM handlers restore the terminal (leave the alternate screen, show the cursor, disable raw mode) before printing.
- **AC:** forcing a panic leaves a usable shell and a log entry with a backtrace.

**S0.4: CLI surface** · P0 · S
- `clap` options: `--interval <ms>`, `--view <dashboard|apps|services|game|procs>`, `--config <path>`, `--theme`, `--no-color`, `--low-color`, `--dump-json`, `--replay <file>`, `--oneline`, plus subcommands `record`, `services`, `apps` (non-interactive tables).
- **AC:** `xtop --help` documents everything; shell completions generate from clap.

**S0.5: Core domain model** · P0 · M
- Types for `Snapshot`, `CpuStats`, `MemStats`, `ProcessInfo`, `AppGroup`, `Service`, `GpuStats`, `PowerStats`, `ThermalStats`, `DiskStats`, `NetStats`, `BatteryStats`.
- Unit newtypes (`Bytes`, `Percent`, `BytesPerSec`, `Watts`, `Celsius`, `Hz`) with formatting.
- `Metric<T>` availability enum (Section 4.2). Everything is `serde`-serializable.
- **AC:** round-trip JSON tests; formatting tests for binary vs decimal units.

**S0.6: History store & statistics** · P0 · M
- Ring buffers sized by duration; per-key eviction when a process, app or service disappears (keep for N seconds so graphs fade instead of vanishing).
- Rolling stats: min, max, mean, p95, p99 (streaming, bounded memory).
- **AC:** property tests for ring buffer invariants and for percentile accuracy against a sorted reference.

**S0.7: Collector trait & scheduler** · P0 · M
- A `Collector` trait with per-collector intervals, a thread per cadence group, merging into an `Arc<Snapshot>`, and a command channel (Section 4.2).
- Each collector records its own sample duration; this feeds the self-overhead display in S14.1.
- Samples with gaps over 3× the interval are dropped (sleep/wake).
- **AC:** with a mock collector that sleeps 500 ms, the UI stays responsive; sleep/wake causes no graph spike.

**S0.8: FFI safety layer (`xtop-macos::sys`)** · P0 · L
- Safe wrappers for `sysctl` (by name and MIB), mach host calls, libproc, CF types (an RAII `CFRetained` wrapper, dictionary and number extraction helpers), IORegistry iteration, and IOService matching.
- Every `unsafe` block has a `// SAFETY:` comment. CF objects are always released.
- **AC:** unit tests run on macOS CI; `leaks --atExit -- xtop --dump-json` reports 0 leaks.

**S0.9: Fixtures & replay** · P1 · M
- `xtop --dump-json --samples N` records a series of snapshots; `--replay` drives the UI from them.
- Commit fixtures from M1 Max, M3 Pro, an Intel Mac and a fanless MacBook Air.
- **AC:** the TUI runs from a fixture on Linux CI; this backs the snapshot tests in S15.1.

**S0.10: Spikes SP1–SP8** · P0 · L (split per spike)
- See Section 7. Each spike is its own sub-story with a write-up and a go/no-go decision.

---

### E1 CPU (M1)

**S1.1: Total & per-core utilization** · P0 · S
- Tick deltas from `host_processor_info`; user, system, nice and idle split per core and total.
- **AC:** within ±3 percentage points of `top -l 2 -s 1` under a steady `yes > /dev/null` load.

**S1.2: Topology & identity** · P0 · S
- Chip name, P/E logical core counts and perf-level names (`hw.perflevelN`), mapping core index to cluster. Intel: one cluster, no P/E labels.
- **AC:** the dashboard shows `6P + 6E` on M3 Pro and `8 cores` on an Intel i9.

**S1.3: Load average, uptime, counts** · P0 · S
- `getloadavg`, `kern.boottime`, total processes and threads (summed from the process collector).

**S1.4: Cluster frequency & residency** · P1 · M · depends on SP1
- Average frequency per cluster from IOReport DVFS residency weighted by the frequency table.
- **AC:** under load, P-cluster frequency tracks `powermetrics` (with sudo, used as a reference) within 5%.

**S1.5: Cluster aggregates** · P0 · S
- P-cluster % and E-cluster % (mean of their cores), plus a hottest-core metric (the busiest core's %). Game view uses this to detect a main-thread bottleneck.

---

### E2 Memory (M1)

**S2.1: VM statistics & Activity Monitor breakdown** · P0 · M
- `host_statistics64(HOST_VM_INFO64)` × page size (`vm_kernel_page_size` / `hw.pagesize`).
- Derive Physical, Used, App, Wired, Compressed, Cached Files and Free using the formulas in Section 5.
- **AC:** each field within 2% of Activity Monitor on three machines (S2.4).

**S2.2: Memory pressure** · P0 · S
- Pressure level (normal/warn/critical → green/yellow/red) and percentage, with history for a pressure graph like Activity Monitor's.

**S2.3: Swap & paging rates** · P0 · S
- `vm.swapusage` (total/used/free) and per-second rates for pageins, pageouts, swapins, swapouts, compressions and decompressions.
- **AC:** running `memory_pressure -l warn` shows rising pressure and swap-out rates.

**S2.4: Accuracy validation procedure** · P1 · S
- `docs/accuracy/memory.md`: how to compare against Activity Monitor and `vm_stat`, with recorded results per machine.

---

### E3 Processes (M1)

**S3.1: Enumeration & static info** · P0 · M
- `proc_listallpids`; per PID: name (`proc_name`, falling back to `p_comm`), path (`proc_pidpath`), ppid, uid → user name (cached `getpwuid_r`), start time, status (running/sleeping/stopped/zombie), and args (`KERN_PROCARGS2`).
- Cache static fields keyed by `(pid, start_time)` to survive PID reuse.
- **AC:** process count matches `ps -A | wc -l` ±2; no crash when processes exit mid-read.

**S3.2: CPU % & CPU time** · P0 · M
- `PROC_PIDTASKINFO` user+system time deltas, **converted with `mach_timebase_info`** (Apple Silicon ticks are not nanoseconds), and thread count.
- %CPU uses the Activity Monitor/top convention (100% = one core). An option toggles "normalized" mode (divide by core count), like btop's Irix toggle.
- **AC:** a single-threaded busy loop reads 100% ±3; 4 threads read 400% ±10.

**S3.3: Memory metrics** · P0 · S
- From `proc_pid_rusage(RUSAGE_INFO_V6)`: footprint (default "Memory" column, like Activity Monitor) and resident size ("Real Mem"). Private/shared memory need an expensive region walk, so they appear only in the detail pane (S9.7).
- **AC:** footprint matches Activity Monitor's "Memory" column within 2% for Safari and Xcode.

**S3.4: Disk I/O per process** · P1 · S
- Byte-delta rates for read and write; cumulative totals.

**S3.5: Wakeups & Energy Impact estimate** · P1 · M
- Idle and interrupt wakeups per second, billed energy delta.
- Energy Impact estimate: a documented weighted formula of CPU time, wakeups, GPU time and disk I/O, calibrated against Activity Monitor on fixture workloads and labelled "estimate" in the UI.
- **AC:** ranking of the top 10 energy consumers agrees with Activity Monitor for ≥ 8 of 10 on a calibration workload.

**S3.6: Kind, sandbox, sleep assertions** · P1 · S
- Kind: `arm64` / `x86_64 (Rosetta)` via `P_TRANSLATED`. Sandboxed flag (`sandbox_check`). "Preventing sleep" from `IOPMCopyAssertionsByProcess`.

**S3.7: Process tree & responsibility** · P0 · S · depends on SP3
- Build a parent/child tree; attach responsible PID and coalition id when available (used by E4).

**S3.8: Unprivileged degradation** · P0 · S · depends on SP7
- `EPERM` produces a partial row with `Metric::NeedsRoot`. A header badge shows `🔒 N processes restricted — run with sudo for full data`.
- **AC:** no error spam in logs; sorting treats NeedsRoot as lowest.

**S3.9: Process actions** · P0 · M
- Send a signal (picker: TERM, KILL, INT, HUP, STOP, CONT, USR1, USR2, QUIT) with confirmation. Renice (`setpriority`), with permission errors shown inline.
- **AC:** killing a process you don't own shows "Operation not permitted — retry with sudo?" and doesn't crash.

**S3.10: Efficient refresh** · P1 · M
- Reuse buffers across ticks, call only the cheap syscalls for unchanged PIDs, read args and paths once per `(pid, start_time)`, and refresh expensive fields (sleep assertions, sandbox) on a slower cadence.
- **AC:** process collector takes under 4 ms per tick with 800 processes on M1.

---

### E4 Apps model (M2)

**S4.1: Bundle resolution** · P0 · M
- From an executable path, find the **outermost** `*.app` (so `Chrome.app/Contents/Frameworks/…/Helper (Renderer).app` maps to `Chrome.app`).
- Parse `Info.plist`: `CFBundleIdentifier`, `CFBundleDisplayName`/`CFBundleName`, version, `LSUIElement`, `LSBackgroundOnly`, `LSApplicationCategoryType`, `LSArchitecturePriority`.
- Cache by bundle path + mtime.
- **AC:** correct for Chrome, Safari, Slack (Electron), Xcode, Docker Desktop, Steam, a CrossOver bottle, and an app under `~/Applications`.

**S4.2: Running application metadata** · P1 · M · depends on SP5
- Activation policy (regular/accessory/prohibited), frontmost, hidden, launch date via `NSWorkspace`, polled on the main thread with run-loop pumping; collectors run elsewhere.
- Fallback: Info.plist keys plus `CGWindowList` for frontmost.

**S4.3: Grouping engine** · P0 · L · depends on SP3
- Assign each PID to an `AppGroup` using these strategies in order: **coalition id** → **responsible PID** → **outermost bundle** → **parent chain up to a bundled ancestor** → "System".
- Special cases: WebKit XPC (`com.apple.WebKit.*`) belongs to its responsible host app (Safari, Mail…); Electron and Chromium helpers; `wine*`/`wineserver` under CrossOver/Whisky/GPTK belong to their bottle's app. Apps launched from Terminal stay their own group, not Terminal's.
- The grouping strategy is configurable (`apps.grouping = "coalition" | "bundle"`).
- **AC:** golden tests on fixture process tables; manual check against Activity Monitor's Energy tab grouping.

**S4.4: Aggregation & history** · P0 · M
- Per app: sum of CPU%, GPU%, footprint, disk R/W rates, energy, wakeups, threads and process count; earliest launch time. History is keyed by bundle id, so it survives app restarts.
- **AC:** sum of app CPU across all groups equals total process CPU ±0.5%.

**S4.5: App categories** · P0 · S
- Sections: Foreground (Dock/regular), Menu bar/accessory, Background (bundled agents/daemons), System (no bundle). A Games tag comes from `public.app-category.*games`, Steam library paths and user pins (used by E12).

**S4.6: App actions** · P1 · M
- Quit (`NSRunningApplication.terminate`, which allows "Save changes?" dialogs), Force Quit (`forceTerminate` + SIGKILL of remaining group members, with confirmation), Hide, Activate, and Reveal in Finder (`open -R`).
- **AC:** quitting TextEdit with unsaved changes shows its save dialog; force quit removes all group PIDs.

**S4.7: "Not responding" detection** · P2 · M · spike
- Research whether hung-app state can be read. Show it if possible, otherwise drop the story.

---

### E5 Services: launchd (M2)

**S5.1: Service definition discovery** · P0 · M
- Scan the plist directories in Section 5. Parse `Label`, `Program`/`ProgramArguments`, `RunAtLoad`, `KeepAlive`, `StartInterval`, `StartCalendarInterval`, `WatchPaths`, `MachServices`, `Disabled`, `UserName`, `StandardOutPath`/`ErrorPath`.
- Owner classification: Apple (`/System/…` or `com.apple.*`), Homebrew (`homebrew.mxcl.*`), third-party.
- **AC:** parses every plist on a stock macOS 15 install with no errors; malformed plists are reported, not fatal.

**S5.2: Runtime state from launchctl** · P0 · L · depends on SP4
- Parse `launchctl print gui/$UID`, `launchctl print system` (when permitted), `launchctl list`, and `launchctl print-disabled`, giving label, PID, last exit status/reason, state (running / loaded-idle / not loaded / disabled) and domain.
- Parsers use fixture tests per macOS major version and tolerate unknown keys.
- **AC:** labels and PIDs match `launchctl list` exactly for the user domain.

**S5.3: Join services ↔ processes** · P0 · S
- PID → service map. Service rows show the CPU and memory of the service process and its descendants. Process and app detail panes show "launched by service `<label>`".

**S5.4: Classification & extra sources** · P1 · M
- Daemon vs agent, domain, Apple vs third-party vs Homebrew. Under sudo, add BTM data from `sfltool dumpbtm` (login items, background items, developer team). Optionally run `brew services list` when `brew` is on PATH.

**S5.5: Health tracking** · P1 · M
- Detect crash loops (PID changes across ticks, non-zero exit codes, respawn count within a window). Translate exit codes (`sysexits.h` names, signals).
- **AC:** a test LaunchAgent that exits 78 every 10 s is flagged "failing, 6 restarts/min".

**S5.6: Service actions** · P1 · M
- `kickstart -k`, `bootout`, `bootstrap`, `enable`/`disable`, open the plist in `$EDITOR`, and reveal in Finder. For the system domain, suspend the TUI and run `sudo launchctl …` interactively, then resume.
- Every destructive action needs confirmation, and Apple services get an extra warning.

**S5.7: Cadence & cost control** · P0 · S
- Services refresh every 5 s by default (configurable), on demand with `r`, and only while the Services view or a service detail is visible. Process joins happen every tick from cached data.

---

### E6 GPU, power, thermal, battery (M3)

**S6.1: GPU global stats** · P0 · M
- `IOAccelerator` `PerformanceStatistics`: device, renderer and tiler utilization, in-use and allocated system memory. Multi-GPU support (Intel iGPU + AMD dGPU, eGPU) shows each GPU separately.
- **AC:** within ±5 points of Activity Monitor's GPU History during a Metal benchmark.

**S6.2: Per-process GPU %** · P1 · M · depends on SP2
- Enumerate AGX user clients, parse the PID from `IOUserClientCreator`, and compute `accumulatedGPUTime` deltas divided by wall time. Feeds the process GPU% column and app aggregation.
- **AC:** a game shows 85–100% while idle apps show ~0; matches Activity Monitor's % GPU ±5.

**S6.3: IOReport bindings** · P0 · L · depends on SP1
- Bindings to `libIOReport.dylib`: copy channels by group ("Energy Model", "CPU Stats", "GPU Stats"), create subscription, sample, and delta, using RAII wrappers.
- **AC:** 24 h soak with no growth in memory or CF objects.

**S6.4: Power** · P0 · M
- CPU, GPU, ANE and DRAM watts from energy deltas; system total from SMC `PSTR` when present.
- **AC:** within 10% of `sudo powermetrics` under load.

**S6.5: GPU frequency & residency** · P1 · S
- Active frequency and idle residency from IOReport GPU Stats.

**S6.6: Temperatures** · P1 · L · depends on SP6
- Per-chip-family sensor map (config-overridable), aggregated into CPU (P/E avg/max), GPU, SoC, battery and SSD temperatures. Unknown sensors appear in a "raw sensors" detail panel.

**S6.7: Fans** · P1 · S
- RPM, min and max per fan; "fanless" label when there are no fans.

**S6.8: Thermal pressure** · P0 · S
- Nominal / fair / serious / critical with timestamps of state changes. "Throttling" badge at fair or above.

**S6.9: Battery & power source** · P1 · M
- Percent, charging/discharging, time remaining, instantaneous watts, adapter wattage, cycle count, health %, temperature. Hidden on desktops.

---

### E7 Disks & network (M4)

**S7.1: Volumes** · P0 · S
- `getfsstat`: name, mount point, filesystem, total/used/free. APFS "available for important usage" on hover/detail. System, Preboot, VM and snapshot volumes are hidden by default (toggle to show). Network volumes are marked.

**S7.2: Disk throughput** · P0 · M
- `IOBlockStorageDriver` statistics: read/write bytes/s and IOPS per physical disk, mapped to volumes where possible.

**S7.3: Network interfaces** · P0 · M
- `NET_RT_IFLIST2` 64-bit counters; friendly names from SystemConfiguration; IPv4/IPv6 addresses. Auto-selects the primary interface (default route), cycled with a key like btop. Graph scale adapts automatically.
- **AC:** no counter wrap after transferring more than 4 GiB.

**S7.4: Wi-Fi details** · P2 · S
- RSSI, noise, channel and tx rate via CoreWLAN. The SSID needs Location permission, so it's omitted gracefully.

**S7.5: Per-process network** · P2 · L · spike
- Evaluate the NetworkStatistics private framework vs. parsing `nettop -P -L 1 -J bytes_in,bytes_out`. Ship only if overhead is under 1 ms per tick.

---

### E8 TUI framework (M1, polished in M4)

**S8.1: App loop** · P0 · M
- crossterm raw mode and alternate screen; input thread; render on new snapshot or input, capped at 30 fps; resize handling; Ctrl-Z suspend and resume (`SIGTSTP`/`SIGCONT`); and a "terminal too small" message with the required size.

**S8.2: Widget kit** · P0 · L
- btop-style graphs (braille, block and TTY symbol sets; up/down mirrored for network), meters with gradients, sparklines, boxes with titles and inline hotkey hints (`[f]ilter`), and a virtualized scrollable table with sticky header, column definitions, alignment and truncation with `…`.
- **AC:** insta snapshot tests for each widget in all three symbol sets.

**S8.3: Layout engine** · P0 · M
- Toggle boxes, use presets, and switch layouts responsively by terminal size (wide, normal, narrow, compact). Box sizes are proportional with minimums.

**S8.4: Theming & color depth** · P0 · M
- Theme tokens (main_bg, box outline, titles, highlight, graph gradients per box). Detect truecolor vs 256 vs 16 (`COLORTERM`, `TERM`), downsample themes automatically, and respect `NO_COLOR`.

**S8.5: Input & keymap** · P0 · M
- Remappable keymap from config; vim keys (`hjkl`, `g/G`); mouse support (click a row to select, click a header to sort, scroll, click a box title to toggle).

**S8.6: Dialogs & overlays** · P0 · M
- Confirmation, signal picker, text input (filter), help overlay (`?`) generated from the keymap, and options menu (`o`/`Esc`).

**S8.7: Formatting** · P0 · S
- Binary (GiB) vs decimal (GB) toggle, bits vs bytes for network, compact durations (`3h12m`), and fixed-width numbers to stop columns jittering.

**S8.8: Render performance** · P1 · S
- **AC:** frame render under 2 ms at 200×60 on M1 (measured in debug overlay `Ctrl-D`).

---

### E9 Dashboard view (M1 → M4)

**S9.1: CPU box** · P0 · M
- Total graph, per-core meters grouped and labelled P/E, cluster frequency, max temperature, CPU power, load averages, uptime. Degrades cleanly on Intel and without IOReport.

**S9.2: Memory box** · P0 · M
- Activity Monitor breakdown with bars, pressure graph colored by level, swap used, and paging rate on hover/detail.

**S9.3: GPU box** · P1 · S
- Utilization graph, in-use memory, frequency and power; one row per GPU on multi-GPU Macs.

**S9.4: Network box** · P0 · S
- Download and upload graphs, current, peak and total; interface selector.

**S9.5: Disks box** · P0 · S
- Usage bars per visible volume and read/write rates.

**S9.6: Process box** · P0 · L
- Columns are configurable: PID, Name, User, Kind, CPU%, GPU%, Memory, Real Mem, Threads, Wakeups, Energy, Disk R/W, Command.
- Sort by any column (the indicator shows direction); incremental filter by name, cmdline or user (`!` negates, regex optional); tree view toggle; per-row mini CPU graph like btop.
- Selection follows the PID across re-sorts; follow mode tracks a selected process.
- **AC:** 60 fps scrolling with 1,000 rows; selection stays put when sort order changes.

**S9.7: Process detail pane** · P1 · M
- Like btop's detail view, plus path, full args, parent, responsible process, coalition/app group, Kind, sandbox, sleep assertions, open file count (`PROC_PIDLISTFDS`), CPU/memory/GPU history graphs, launching service, and private/shared memory computed on demand.

---

### E10 Apps view (M2)

**S10.1: Apps table** · P0 · M
- The categorized sections from S4.5, expand/collapse groups (`Enter`, `←/→`), and child rows of member processes. Repeated helpers collapse into `Name ×N`.
- **AC:** with Chrome running 20+ helpers, the default view shows one Chrome row.

**S10.2: App detail pane** · P1 · M
- History graphs for CPU, GPU and memory, bundle id, version, path, architecture, launch date, member processes, and related launchd services (by bundle path).

**S10.3: Sort, filter & toggles** · P0 · S
- Sort by any column; filter by name or bundle id; toggle "hide Apple apps"; toggle "show only apps using > X% CPU".

**S10.4: Wire app actions** · P1 · S
- Key bindings and confirmations for the S4.6 actions.

---

### E11 Services view (M2)

**S11.1: Services table** · P0 · M
- Filters: domain (system/gui/all), owner (Apple / third-party / Homebrew), and state (running / idle / failing / disabled / not loaded). The header shows counts.

**S11.2: Service detail pane** · P0 · M
- Plist contents rendered as YAML-like text, program and args, schedule (human-readable calendar interval), KeepAlive semantics, last exit with explanation, resource usage history, and log paths (open with `$PAGER`).

**S11.3: Wire service actions** · P1 · S
- S5.6 actions with confirmations, privilege escalation flow, and result toast.

**S11.4: "What runs at login" preset** · P2 · S
- One keypress filters to third-party agents and daemons with `RunAtLoad` or `KeepAlive`, plus BTM items. A quick audit of background cruft.

---

### E12 Game Performance view (M3)

The goal is a glanceable answer to "how hard is my Mac working while I game, and
what is the bottleneck?", with the monitor itself costing almost nothing.

**S12.1: System totals HUD** · P0 · M
- Big readouts for: **CPU total %** (with P/E cluster % and frequencies, and the hottest core), **GPU %** (frequency, renderer/tiler, in-use GPU memory), **memory used / total** with pressure state, swap used and swap-out rate, **power** (total, CPU, GPU, ANE, DRAM), **thermal state**, SoC/GPU temperatures, and fan RPM.
- Each metric has a 60 s sparkline (configurable to 5 min).
- Readout colors follow thresholds (configurable; defaults: green < 70%, yellow < 90%, red ≥ 90%; pressure and thermal use their own state colors).
- **AC:** readable at arm's length on a 1080p second monitor in the default theme (large digits rendered with a box-drawing "big font" widget when space allows).

**S12.2: Target game tracking** · P0 · M · depends on E4
- Auto-detect the target in this order:
  1. user-pinned apps;
  2. a running app tagged Games (S4.5);
  3. the frontmost fullscreen app (`CGWindowList`: window bounds equal display bounds);
  4. heuristics: Steam `steamapps/common`, CrossOver/Whisky/GPTK wine processes, Rosetta games.
- Manual picker (`g`) lists apps sorted by GPU use.
- Shows the target app group's CPU% (and as cores), GPU%, memory footprint, threads and disk read rate, each with a sparkline.
- Re-attaches automatically by bundle id when the game restarts; shows "waiting for <game>…" when it isn't running.
- **AC:** detects at least 4 of 5 test titles (one native Metal, one Rosetta, one Steam, one CrossOver/Whisky, one App Store game) without configuration.

**S12.3: Session statistics** · P0 · S · depends on S0.6
- From session start, for each headline metric: min, avg, p95, p99 and max. Also time spent at or above "fair" thermal state, peak temperature, peak swap, and time under memory pressure ≥ warn.
- `r` resets the session; the session starts automatically when the target game launches.

**S12.4: Recording & export** · P1 · M
- `R` toggles recording. Writes CSV (one row per sample: timestamp, all HUD metrics, game metrics) and a JSON summary to `~/Library/Application Support/xtop/sessions/<game>-<timestamp>/`.
- `m` adds a named marker ("boss fight", "settings: high").
- Headless mode: `xtop record --app "Baldur's Gate 3" --duration 30m` (no TUI, minimal overhead).
- `xtop report <session>` prints a summary table and compares two sessions ("high vs medium settings").
- **AC:** a 1 h recording at 1 Hz is under 2 MB; summary numbers match recomputation from the CSV.

**S12.5: Bottleneck verdict** · P1 · M
- A rolling 30–60 s window classifies the session, clearly labelled as a heuristic:
  - **GPU-bound:** GPU ≥ 95% for most of the window and P-cluster average < 70%.
  - **CPU-bound (main thread):** hottest P-core ≥ 95% while GPU < 85%.
  - **CPU-bound (multi-core):** P-cluster ≥ 90%.
  - **Memory-bound:** pressure ≥ warn or swap-out rate > 0 sustained.
  - **Thermally throttled:** thermal state ≥ fair, or P/GPU frequency drops ≥ 15% vs the session's first-minute baseline while utilization stays high.
  - **Balanced / headroom available.**
- Shows the evidence behind each verdict ("GPU ≥ 95% for 92% of last 60 s").
- **AC:** unit tests on synthetic time series for every verdict; thresholds are configurable.

**S12.6: Low-overhead "lite" collection** · P0 · M
- While Game view is active, the process collector does a full scan only every 5 s (to catch new processes) and samples just the target group and top-N GPU/CPU consumers on other ticks. Services and disks pause; render fps drops to the refresh rate.
- Shows xtop's own CPU % and memory in the footer.
- **AC:** < 1% of one core at a 1 s refresh on M1 (measured with `top -pid`), and < 0.5% at 2 s.

**S12.7: Compact & one-line layouts** · P0 · S
- Responsive Game layouts: full (≥ 100×14), medium (≥ 70×10), compact (≥ 30×7) as in Section 6.4.
- `xtop --oneline [--format '{cpu} {gpu} {mem}']` prints one line per interval for tmux status bars or SketchyBar, with `--json` for scripting.

**S12.8: Alerts** · P1 · S
- Optional terminal bell or visual flash when thermal state rises, memory pressure turns red, the game's footprint crosses a threshold, or swap-out starts. Debounced; configurable per alert.

**S12.9: FPS & Game Mode research** · P2 · M · depends on SP8
- FPS can't be measured from outside without injection. Document using the **Metal Performance HUD** (Developer settings, or `MTL_HUD_ENABLED=1`).
- If SP8 succeeds, add `xtop launch --metal-hud <App>` to launch a game with the HUD enabled, and show a "Game Mode: on" badge when detectable.
- **AC:** a decision is recorded in `docs/spikes/08-fps-gamemode.md`; any shipped feature is documented in the README's gaming section.

**S12.10: Game view docs** · P1 · S
- README section "Monitoring games": second-monitor setup, a compact-window tip for a small always-on-top terminal (e.g. iTerm2 hotkey window or a Ghostty float), interpreting verdicts, and recording workflows.

---

### E13 Config, themes, presets (M4)

**S13.1: Config file** · P0 · M
- TOML at `$XDG_CONFIG_HOME/xtop/config.toml`, falling back to `~/.config/xtop/config.toml`. Created with commented defaults on first run. Precedence: CLI > env (`XTOP_*`) > file > defaults. Invalid values produce a readable error naming the line.

**S13.2: In-app options menu** · P1 · M
- Change interval, theme, units, graph symbols, columns, grouping strategy and Game thresholds from the TUI; changes save back to the config while keeping comments (`toml_edit`).

**S13.3: Themes** · P1 · M
- Built-in themes: default (btop-like), nord, gruvbox, dracula, tokyo-night, catppuccin, solarized, monochrome. Importer for btop `.theme` files (`theme[key]="#hex"`). Custom themes go in `~/.config/xtop/themes/`.
- **AC:** imports at least 5 popular btop themes from btop's repository with no unmapped required keys.

**S13.4: Layout presets** · P1 · S
- Named presets (dashboard, minimal, apps+gpu, game-full, game-compact), cycled with `p` like btop; user-defined in config.

**S13.5: Persisted UI state** · P2 · S
- Last view, sort columns, expanded groups and pinned games, saved in `~/Library/Application Support/xtop/state.toml`, separate from the config.

---

### E14 Performance & reliability (M4)

**S14.1: Overhead budget & benchmarks** · P0 · M
- criterion benchmarks per collector; an on-screen debug overlay shows per-collector sample time, render time and xtop's own CPU and memory.
- Budgets: dashboard < 2% of one core at 1 s on M1, Game view < 1%, RSS < 40 MB with 5 min history.

**S14.2: Bounded memory** · P0 · S
- History capped by duration; dead PIDs evicted after a grace period; app and service history capped by count (LRU).

**S14.3: Edge cases** · P0 · M
- PID reuse, processes exiting mid-sample, zombies, sleep/wake gaps, time changes, hot-plugged displays and eGPUs, network interfaces appearing and disappearing, volumes mounting and unmounting, and very narrow terminals.
- **AC:** a fuzz-ish test spawns and kills 1,000 processes per second for 60 s without a panic.

**S14.4: Soak test** · P1 · S
- 24 h run with `leaks` and RSS tracking; no CF or IOKit object growth; documented procedure.

---

### E15 Quality, release, docs (M1 → M5)

**S15.1: Test strategy** · P0 · M
- Unit tests for every parser and calculator (VM math, CPU deltas, timebase conversion, launchctl parsing, grouping, verdicts).
- insta snapshot tests for each view at 3 terminal sizes using fixtures and `TestBackend`.
- macOS-only integration tests gated behind `cfg(target_os = "macos")`.

**S15.2: Accuracy harness** · P1 · M
- A script compares xtop's `--dump-json` with `top -l`, `vm_stat`, `ps`, `launchctl list` and (optionally, with sudo) `powermetrics`, then reports deviations. Results are recorded per release in `docs/accuracy/`.

**S15.3: Release pipeline** · P0 · L
- Tag-triggered: build arm64 and x86_64, `lipo` into a universal binary, codesign with Developer ID plus hardened runtime, notarize, publish a GitHub Release with checksums, and bump the Homebrew tap formula (`brew install s4njee/tap/xtop`). Also `cargo install xtop`.
- **AC:** the downloaded binary runs without a Gatekeeper prompt on a clean Mac.

**S15.4: Documentation** · P0 · M
- README: features, screenshots/GIFs (VHS tapes kept in the repo), install, keybindings, permissions and sudo explanation, the gaming guide, and an FAQ ("Why does my number differ from Activity Monitor?").
- `docs/data-sources.md` mirrors Section 5.

**S15.5: Man page & completions** · P1 · S
- Generated with `clap_mangen` and `clap_complete` and installed by the Homebrew formula.

---

### E16 Future / stretch (post-1.0)

- **S16.1** Linux backend (procfs) behind the same `Collector` traits.
- **S16.2** Remote mode: `xtop agent` streams snapshots as JSON over SSH stdio; `xtop --remote host` renders them locally.
- **S16.3** Prometheus / OpenMetrics exporter (`xtop serve --metrics :9100`).
- **S16.4** Per-process network (if S7.5 is a no-go for 1.0).
- **S16.5** Plugin hooks for custom boxes (e.g. a Docker/OrbStack container box).
- **S16.6** Session comparison charts exported as HTML from game recordings.

## 10. Risks & mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Private APIs (IOReport, AGX clients, coalitions, SMC) change between macOS releases | Features silently break | Capability probing at startup, `Metric::Unsupported` instead of panics, fixture tests per OS, beta-OS CI run each summer |
| `launchctl print` output format changes | Services view breaks | Tolerant parser, version-specific fixtures, fallback to `launchctl list` |
| Numbers disagree with Activity Monitor | Users lose trust | Accuracy harness (S15.2), documented formulas, an FAQ, "estimate" labels where applicable |
| Monitor overhead disturbs game benchmarks | Defeats the Game view's purpose | Lite mode (S12.6), self-overhead displayed, headless recording |
| Unprivileged limits hide root-process data | Incomplete process and service picture | Clear 🔒 UX, optional sudo, never require it |
| Grouping heuristics misattribute processes | Confusing Apps view | Coalition-first strategy, golden tests, user-switchable strategy |
| Intel Macs lack the same sensors | Inconsistent UX | Keep the Intel path best-effort but tested; hide unsupported boxes |

## 11. Open questions

1. Name: keep `xtop`, or pick something unclaimed on crates.io and Homebrew? (Check before M5.)
2. Should Game view auto-open when a game goes fullscreen while xtop is running? (Default: no, show a toast.)
3. Energy Impact: try to match Activity Monitor's scale exactly, or present our own clearly different "power score"?
4. Should the Services view hide Apple services by default to reduce noise? (Leaning yes, toggle with `A`.)
5. Minimum macOS: can we drop 13 to simplify APIs? (Decide after the spikes.)

## 12. Definition of Done (every story)

- Code merged with CI green (fmt, clippy `-D warnings`, tests, cargo-deny).
- New unsafe FFI has `// SAFETY:` comments and RAII-released CF/IOKit objects.
- Unit tests for logic; snapshot tests for new or changed UI.
- Graceful behavior when the data source is unavailable (no panic; renders `n/a` or 🔒).
- Overhead impact measured if the story touches a collector.
- Docs updated (README, keybindings, `docs/data-sources.md`) when user-visible.

## Appendix A: Default keybindings (draft)

| Key | Action |
|---|---|
| `1`–`5` / `Tab` | Switch view (Dashboard, Apps, Services, Game, Processes) |
| `F1`–`F6` | Toggle dashboard boxes (cpu, mem, net, disks, gpu, proc) |
| `↑↓` / `jk`, `PgUp/PgDn`, `g/G` | Navigate |
| `←→` / `hl` | Change sort column (tables), collapse/expand (Apps) |
| `Enter` | Detail pane / expand |
| `f` or `/` | Filter |
| `s` / `S` | Sort picker / reverse |
| `t` | Tree view (Processes) |
| `k` / `K` | Signal picker / SIGKILL |
| `q` / `Q` | Quit app / force quit app (Apps view); `Ctrl-C` exits xtop |
| `p` | Cycle layout preset |
| `o` / `Esc` | Options menu |
| `?` | Help |
| `+` / `-` | Change refresh interval |
| **Game view:** `g` pick game · `r` reset session · `R` record · `m` marker · `c` cycle compact/full | |
| **Services:** `k` kickstart · `b` bootout · `D` enable/disable · `e` edit plist · `A` toggle Apple services | |

## Appendix B: Suggested order of work (first 6 weeks)

1. **Week 1:** S0.1–S0.5, start spikes SP1, SP3, SP5, SP7.
2. **Week 2:** S0.6–S0.8, S1.1–S1.3, S2.1–S2.3, finish the remaining spikes.
3. **Week 3:** S3.1–S3.3, S3.7–S3.8, S8.1–S8.2.
4. **Week 4:** S9.1, S9.2, S9.6, S8.3–S8.6, S3.9. **M1 demo.**
5. **Week 5:** S4.1, S4.3–S4.5, S10.1, S10.3.
6. **Week 6:** S5.1–S5.3, S5.7, S11.1–S11.2. **M2 demo**; then E6 + E12 for M3.
