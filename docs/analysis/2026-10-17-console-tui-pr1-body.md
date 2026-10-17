## Summary

A read-only console dashboard for ProxMenux Monitor, in the spirit of bashtop, for the node's physical screen (HDMI/VGA) or any root shell. Root opens it with `proxmenux-console` (or `AppRun --tui`); `q` returns to the shell.

- **Node view (default):** CPU with history and per-core bars on wide screens, memory with ARC, ZRAM and swap, physical NIC and whole-disk throughput, temperature and load, a cluster summary, and the guests using the most CPU (in host cores) and the most memory, plus the share of host load not attributable to guests.
- **Cluster view (`Tab`):** one compact row per node (offline first) and a guest table sortable by CPU or memory (`c`/`m`). Totals are shown only with quorum and full coverage; otherwise it shows `coverage k/n`.
- **Local metrics come from `/proc` and `/sys`** every second, so the screen is useful even while the Monitor is starting or stopped. Cluster, guest and storage data come from a snapshot the Monitor writes atomically to `/run/proxmenux-monitor/console.json` (root-only in this PR), only while a console is open.
- **No second `pvesh`:** the existing `cluster/resources` cache now fetches all resource types once and the VM view filters `qemu`/`lxc`, so the snapshot reuses it. Existing callers get the same entries in the same order.
- **Linux console friendly:** works from 80x24 up to full HD sizes, uses ACS line drawing by default with `unicode` and `ascii` glyph sets (`a` cycles them), 8 colours, a full repaint every 30 s so kernel messages do not linger, and guest names/tags are reduced to printable ASCII before drawing.
- The Monitor never depends on it: the publisher start is wrapped so any failure is ignored, and the TUI does not run external commands.
- Strings live under `runtime.consoleTui` in `AppImage/messages/en/common.json` for the translation bot.

Node view at 80x24 with sample data:

```
 PROXMENUX MONITOR    [Node] Cluster  pve-01 · PVE 9.0 · up 7d 04h     07:56:36
┌─CPU  70%────────────────────────────────┐┌─MEMORY  16.0/32 GiB  50%──────────┐
│ ███████████████████████████░░░░░░░░░░░░ ││ ████████████████░░░░░░░░░░░░░░░░░ │
│ load 0.42 0.51 0.48 · 4 thr · 48° C     ││ ARC 2.0 GiB · avail 16.0 GiB      │
│ ░░█░█                                   ││ ZRAM 143M/3.2G 1.8x · swap 0/8.0G │
└─────────────────────────────────────────┘└───────────────────────────────────┘
┌─NET eno1 2.5G───────┐┌─DISK─────────────────────┐┌─CLUSTER · QUORUM OK 2/3───┐
│ RX 3.2 Mb/s         ││ R 42 MiB/s  W 8.0 MiB/s  ││ pve-01* 7% 50% online     │
│ TX 800 Kb/s         ││ local-zfs 40%            ││ +2 nodes                  │
└─────────────────────┘└──────────────────────────┘└───────────────────────────┘
┌─TOP CPU · host other 1.6c────────────┐┌─TOP MEMORY · host other 8.0 GiB──────┐
│ VM 101 plex        pve-01 1.2c   60% ││ VM 103 db             pve-02 6.0 GiB │
│ CT 205 nginx       pve-02 0.3c   30% ││ VM 101 plex           pve-01 4.0 GiB │
│ VM 103 db          pve-02 0.1c    5% ││ VM 110 win    pve-01 unknown 2.0 GiB │
```

A follow-up PR would add an optional start on `tty1` at boot (once per boot through a `getty@tty1` drop-in, as an unprivileged user, falling back to the normal login on exit or on any error). I kept it out of this one so the boot path can be reviewed on its own. I did not touch the web docs or the changelog; happy to add a docs page under Settings if you want one.

## Verification

- On a remote Ubuntu 24.04 test worker (Python 3.12, `TERM=linux`), head `54f73b4a`: 112 tests passed across the new suites (metrics, snapshot, renderer, cluster resources cache, Monitor hook, console command wiring, and a pty smoke test that runs the TUI under `python3 -I` from another directory), plus the existing `test_cluster_guest_storage_ownership`, `test_lxc_vm_ip_inventory` and `test_zram_metrics`. Bash syntax checks, Python compilation and catalog JSON validation passed.
- `AppRun --tui --once` from an AppDir with the same layout `build_appimage.sh` produces exits 0, finds the packaged catalogs and draws correctly in the ACS and Unicode glyph sets.
- The full AppImage build on that worker reached and passed the console module copy step, then stopped because the worker could not download the hardware `.deb` packages (`ipmitool`, `libfreeipmi17`), the same limitation as before this change.
- New CI workflow `test-console-tui.yml` runs the suites on Python 3.11 and 3.13.
