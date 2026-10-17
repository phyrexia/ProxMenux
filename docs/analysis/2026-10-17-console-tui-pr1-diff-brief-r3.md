# Auditoría adversarial — fase 5 (diff), ronda 3 acotada — PR1 consola TUI de ProxMenux Monitor

> **Modo de respuesta:** no uses ninguna herramienta, no leas ni escribas archivos. Todo el material está
> inline abajo. Escribe tu veredicto completo directamente como texto de respuesta (Markdown).

En la ronda 2 aprobaste el PR1 en `5c7a8ab0` (0/0/0). Después, la primera ejecución real de las pruebas en un
worker QA (Python 3.12, `TERM=linux`) encontró dos fallos y el primer frame real mostró tres problemas
visibles; se corrigieron en cuatro commits:

1. `79ab5888` — `console_render._header`: `re.search(r"\d+\.\d+", ...)` sin grupo de captura seguido de
   `match.group(1)` lanzaba `IndexError` con cualquier snapshot que trajera versión de PVE (la TUI salía).
   Ninguna revisión previa lo vio; lo detectaron 21 tests al ejecutarse.
2. `0b6288bd` — test de NIC físicas: un `replace` demasiado amplio también movía el contador de `eno1`
   (fallo del test, no del código).
3. `d3838d2d` — sin zram no se imprime una línea ZRAM en cero; la tira CLUSTER no afirma `NO QUORUM 0/0` sin
   datos de clúster; los paneles TOP muestran `no data` sin snapshot.
4. `731b099b` — formas compactas a 80×24 (tasas sin decimal desde 10, `TOP CPU` corto, tamaños `143M/3.2G`,
   el nombre del clúster cae antes que `n/m`, porcentajes alineados), y la clave nueva
   `runtime.consoleTui.panels.topCpuShort = "TOP CPU"`.

Resultado de QA sobre `731b099b` (qaw-p3-01): 103 tests en verde en 10 suites + `bash -n` + `py_compile` + JSON
del catálogo; `AppRun --tui --once` desde un AppDir con el layout de `build_appimage.sh` sale 0, encuentra el
catálogo y no muestra claves crudas, en modos ACS y Unicode.

Frame real 80×24 vista Nodo tras el fix (fixture de 3 nodos):

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

## Lo que se te pide

Revisión acotada a este delta: (1) ¿algún cambio introduce regresión o riesgo nuevo (en particular el helper
`_fit` de candidatos, `_decimal`/`_short_size`, la nueva rama `quorate is None` de la tira — ojo con el caso
standalone, que debe seguir mostrando `STANDALONE` —, y el `no data` de TOP)? (2) ¿Los tests nuevos prueban lo
que dicen? (3) Dado que un `IndexError` llegó hasta la ronda 2 sin ser visto, ¿ves algún otro camino en el delta
o en su contexto inmediato que pueda lanzar una excepción con datos reales y tumbar la TUI?

Dos veredictos (spec-compliance con `Perfil verificado: feature`; riesgo/calidad), Critical / Important / Minor,
`Resumen: <n> Critical, <n> Important, <n> Minor`, `APROBAR | APROBAR CON CAMBIOS | RECHAZAR`.

## Delta 5c7a8ab0..731b099b (salvo la línea minificada de en/common.json, que sólo añade `topCpuShort`)

```diff
diff --git a/AppImage/scripts/console_render.py b/AppImage/scripts/console_render.py
index 4dadd567..70699eb6 100644
--- a/AppImage/scripts/console_render.py
+++ b/AppImage/scripts/console_render.py
@@ -79,6 +79,28 @@ def _size(value):
     return f"{_gib(value):.1f} GiB"
 
 
+def _decimal(value):
+    text = f"{value:.1f}"
+    return f"{value:.0f}" if float(text) >= 10 else text
+
+
+def _short_size(value):
+    value = max(0, _number(value))
+    if value == 0:
+        return "0"
+    if value < 1073741824:
+        return f"{_decimal(value / 1048576)}M"
+    return f"{_decimal(_gib(value))}G"
+
+
+def _fit(width, *candidates):
+    """Return the first candidate that fits ``width`` cells, else the last one."""
+    for text in candidates:
+        if len(text) <= width:
+            return text
+    return candidates[-1]
+
+
 def _uptime(seconds):
     seconds = max(0, int(_number(seconds)))
     days, hours = divmod(seconds // 3600, 24)
@@ -100,7 +122,7 @@ def _rate(value, bits=False):
     for unit in units:
         base = 1000 if bits else 1024
         if value < base or unit == units[-1]:
-            return f"{value:.1f} {unit}"
+            return f"{_decimal(value)} {unit}"
         value /= base
 
 
@@ -267,7 +289,7 @@ def _header(grid, vm, state, cluster_view):
     else:
         version = sanitize(node.get("pve_version") or "n/d", 40, grid.glyphs)
         match = re.search(r"\d+\.\d+", version)
-        version = match.group(1) if match else version
+        version = match.group(0) if match else version
         name = sanitize(node.get("name") or "", 20, grid.glyphs)
         info = (f"{name} {_separator(grid)} PVE {version} "
                 f"{_separator(grid)} {label('labels.uptime')} {_uptime((vm.get('local') or {}).get('uptime'))}")
@@ -341,8 +363,15 @@ def _cpu_memory(grid, vm, rows):
     zram_used = zram.get("swap_used_bytes", zram.get("used", 0))
     zram_total = zram.get("capacity_bytes", zram.get("total", 0))
     ratio = _number(zram.get("compression_ratio"))
-    _text(grid, left + 2, 4,
-          f"{label('labels.zram')} {_size(zram_used)}/{_size(zram_total)} {ratio:.1f}x {_separator(grid)} {label('labels.swap')} {_size(mem.get('swap_used'))}/{_size(mem.get('swap_total'))}", right - 4)
+    swap = f"{label('labels.swap')} {_size(mem.get('swap_used'))}/{_size(mem.get('swap_total'))}"
+    if _number(zram_total) > 0:
+        zram_part = f"{label('labels.zram')} {_size(zram_used)}/{_size(zram_total)} {ratio:.1f}x {_separator(grid)} "
+        compact_zram = f"{label('labels.zram')} {_short_size(zram_used)}/{_short_size(zram_total)} {ratio:.1f}x {_separator(grid)} "
+        compact_swap = f"{label('labels.swap')} {_short_size(mem.get('swap_used'))}/{_short_size(mem.get('swap_total'))}"
+        row = _fit(right - 4, zram_part + swap, compact_zram + compact_swap)
+    else:
+        row = swap
+    _text(grid, left + 2, 4, row, right - 4)
     history = local.get("cpu_history") or []
     history_start = 4
     history_rows = 1 if rows == 5 else 4
@@ -399,11 +428,21 @@ def _strip(grid, vm, y, rows, state):
     cluster_width = grid.width - cluster_x
     up = int(_number(cluster.get("nodes_online")))
     total = int(_number(cluster.get("nodes_total")))
-    title = f"{label('panels.cluster')} {sanitize(cluster.get('name'), 15, grid.glyphs)}"
-    title += f" {_separator(grid)} {label('labels.quorumOk') if cluster.get('quorate') else label('labels.noQuorum')} {up}/{total}"
+    quorate = cluster.get("quorate")
+    head = label("panels.cluster")
+    name = sanitize(cluster.get("name"), 15, grid.glyphs)
+    named = f"{head} {name}" if name else head
+    separator = _separator(grid)
+    if quorate is None:
+        title = _fit(cluster_width - 4, named, head)
+    else:
+        quorum_text = f"{label('labels.quorumOk') if quorate else label('labels.noQuorum')} {up}/{total}"
+        title = _fit(cluster_width - 4, f"{named} {separator} {quorum_text}",
+                     f"{head} {separator} {quorum_text}", f"{head} {separator} {up}/{total}")
     grid.box(cluster_x, y, cluster_width, rows, title)
-    if state == "starting":
-        _text(grid, cluster_x + 2, y + 1, label("status.waitingForData"), cluster_width - 4, "dim")
+    if state == "starting" or quorate is None:
+        text = label("status.waitingForData") if state == "starting" else label("status.noData")
+        _text(grid, cluster_x + 2, y + 1, text, cluster_width - 4, "dim")
         return
     members = cluster.get("members") or []
     visible = max(0, rows - 2)
@@ -437,7 +476,7 @@ def _top_row(grid, guest, x, y, width, mode, standalone):
     identity = f"{kind} {guest.get('vmid', '')} "
     name_width = max(8, min(24 if grid.width >= 160 else 13, width - (39 if not standalone else 30)))
     node = "" if standalone else sanitize(guest.get("node"), 10, grid.glyphs)
-    value = (f"{_cpu_key(guest):.1f}c  {_pct(guest.get('cpu'))}%" if mode == "cpu"
+    value = (f"{_cpu_key(guest):.1f}c  {_pct(guest.get('cpu')):>3}%" if mode == "cpu"
              else _size(guest.get("mem")))
     right = x + width - 2
     status = (sanitize(guest.get("status"), 7, grid.glyphs)
@@ -469,24 +508,28 @@ def _top(grid, vm, y, rows, state):
     standalone = not cluster.get("is_cluster", True)
     guests = _guests(snapshot)
     midpoint = grid.width // 2
-    cpu_title = label("panels.topCpu")
+    cpu_name = label("panels.topCpu")
+    cpu_short = label("panels.topCpuShort")
     memory_title = label("panels.topMemory")
+    cpu_suffix = ""
     if state == "ok":
         host_other = vm.get("host_other") or {}
         if host_other.get("cpu") is not None:
-            cpu_title += f" {_separator(grid)} {label('labels.hostOther')} {_number(host_other['cpu']):.1f}c"
+            cpu_suffix = f" {_separator(grid)} {label('labels.hostOther')} {_number(host_other['cpu']):.1f}c"
         if host_other.get("mem") is not None:
             memory_title += f" {_separator(grid)} {label('labels.hostOther')} {_size(host_other['mem'])}"
     elif snapshot and state in ("stale", "unavailable"):
         stale = _label_with(vm, "status.stale", grid.glyphs,
                             age=_age(snapshot.get("age_s")))
-        cpu_title += f" ({stale})"
+        cpu_suffix = f" ({stale})"
         memory_title += f" ({stale})"
+    cpu_title = _fit(midpoint - 4, cpu_name + cpu_suffix, cpu_short + cpu_suffix, cpu_short)
     grid.box(0, y, midpoint, rows, cpu_title)
     grid.box(midpoint, y, grid.width - midpoint, rows, memory_title)
-    if state == "starting":
+    if state == "starting" or not snapshot:
+        text = label("status.waitingForData") if state == "starting" else label("status.noData")
         for x, width in ((0, midpoint), (midpoint, grid.width - midpoint)):
-            _text(grid, x + 2, y + 1, label("status.waitingForData"), width - 4, "dim")
+            _text(grid, x + 2, y + 1, text, width - 4, "dim")
         return
     if not snapshot or not guests:
         return
diff --git a/AppImage/scripts/tests/test_console_metrics.py b/AppImage/scripts/tests/test_console_metrics.py
index cc31b5a3..28c75e95 100644
--- a/AppImage/scripts/tests/test_console_metrics.py
+++ b/AppImage/scripts/tests/test_console_metrics.py
@@ -102,8 +102,8 @@ class LocalSamplerTests(unittest.TestCase):
         sampler = self.sampler()
         sampler.sample(1.0)
         second = (self.root / "proc/net/dev").read_text()
-        second = second.replace("wlan0: 10", "wlan0: 110").replace(
-            "0 0 0 0 0 0 0 20", "0 0 0 0 0 0 0 220")
+        second = second.replace("wlan0: 10 0 0 0 0 0 0 0 20 ",
+                                "wlan0: 110 0 0 0 0 0 0 0 220 ")
         second = second.replace("eno2: 3000", "eno2: 10000")
         for nic in ("fwbr1", "fwpr1", "fwln1", "tap1", "veth1"):
             second = second.replace(f"{nic}: 1", f"{nic}: 1001")
diff --git a/AppImage/scripts/tests/test_console_render.py b/AppImage/scripts/tests/test_console_render.py
index 252d60f0..56f180ec 100644
--- a/AppImage/scripts/tests/test_console_render.py
+++ b/AppImage/scripts/tests/test_console_render.py
@@ -11,7 +11,7 @@ SCRIPTS_DIR = Path(__file__).resolve().parents[1]
 if str(SCRIPTS_DIR) not in sys.path:
     sys.path.insert(0, str(SCRIPTS_DIR))
 
-from console_render import ACS_TOKENS, render, sanitize
+from console_render import ACS_TOKENS, _rate, render, sanitize
 
 
 CATALOG = json.loads((SCRIPTS_DIR.parent / "messages" / "en" / "common.json")
@@ -83,8 +83,8 @@ class ConsoleRenderTests(unittest.TestCase):
         self.assertIn("q login", rows[23])
         self.assertIn("data 2s", rows[23])
         self.assertIn("+2 nodes", "\n".join(rows[6:10]))
-        self.assertIn("TOP CPU (host cores)", rows[10])
-        self.assertEqual(rows[10].count("(host cores)"), 1)
+        self.assertIn("host other 1.6c", rows[10][:40])
+        self.assertNotIn("(host cores)", rows[10][:40])
 
     def test_selected_tab_uses_selected_role(self):
         grid = render(self.vm, 80, 24)
@@ -277,6 +277,51 @@ class ConsoleRenderTests(unittest.TestCase):
         self.assertIn("MONITOR UNAVAILABLE", lines(grid)[0])
         self.assertEqual(grid[0][lines(grid)[0].index("MONITOR UNAVAILABLE")][1], "crit")
 
+    def test_memory_row_without_zram_shows_only_swap(self):
+        for zram in (None, {"capacity_bytes": 0, "swap_used_bytes": 0, "compression_ratio": 0}):
+            with self.subTest(zram=zram):
+                self.vm["local"]["zram"] = zram
+                row = lines(render(self.vm, 80, 24))[4]
+                self.assertNotIn("ZRAM", row)
+                self.assertIn("swap 0 MiB/8.0 GiB", row)
+
+    def test_memory_row_with_zram_keeps_zram_and_swap(self):
+        row = lines(render(self.vm, 80, 24))[4]
+        self.assertIn("ZRAM", row)
+        self.assertIn("swap", row)
+
+    def test_cluster_strip_without_data_has_no_quorum_title(self):
+        self.vm["snapshot"] = None
+        self.vm["local"]["uptime"] = 180
+        grid = render(self.vm, 80, 24)
+        rows = lines(grid)
+        self.assertIn("CLUSTER", rows[6])
+        self.assertNotIn("QUORUM", rows[6])
+        self.assertNotIn("0/0", rows[6])
+        start = rows[7].index("no data")
+        self.assertTrue(all(grid[7][x][1] == "dim" for x in range(start, start + len("no data"))))
+
+    def test_cluster_strip_quorate_unknown_has_no_quorum_title(self):
+        self.vm["snapshot"]["cluster"]["quorate"] = None
+        rows = lines(render(self.vm, 80, 24))
+        self.assertNotIn("QUORUM", rows[6])
+        self.assertIn("no data", rows[7])
+
+    def test_cluster_strip_with_quorum_false_still_shows_no_quorum(self):
+        self.vm["snapshot"]["cluster"]["quorate"] = False
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("NO QUORUM", rows[6])
+
+    def test_top_panels_without_snapshot_show_no_data(self):
+        self.vm["snapshot"] = None
+        self.vm["local"]["uptime"] = 180
+        grid = render(self.vm, 80, 24)
+        rows = lines(grid)
+        self.assertIn("no data", rows[11][:40])
+        self.assertIn("no data", rows[11][40:])
+        start = rows[11].index("no data")
+        self.assertTrue(all(grid[11][x][1] == "dim" for x in range(start, start + len("no data"))))
+
     def test_small_terminal_message(self):
         for width, height in ((79, 24), (80, 23)):
             rows = lines(render(self.vm, width, height))
@@ -313,6 +358,42 @@ class ConsoleRenderTests(unittest.TestCase):
         self.assertIn("eno1", rows[6])
         self.assertNotIn("[31m", "\n".join(rows))
 
+    def test_rate_keeps_one_decimal_only_below_ten(self):
+        self.assertEqual(_rate(400000, True), "3.2 Mb/s")
+        self.assertEqual(_rate(100000, True), "800 Kb/s")
+        self.assertEqual(_rate(44040192), "42 MiB/s")
+        self.assertEqual(_rate(8388608), "8.0 MiB/s")
+        self.assertEqual(_rate(9.94), "9.9 B/s")
+        self.assertEqual(_rate(9.96), "10 B/s")
+
+    def test_disk_and_network_rates_fit_at_80x24(self):
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("RX 3.2 Mb/s", rows[7])
+        self.assertIn("TX 800 Kb/s", rows[8])
+        self.assertIn("R 42 MiB/s  W 8.0 MiB/s", rows[7])
+
+    def test_top_cpu_title_falls_back_to_short_name_then_drops_suffix(self):
+        self.vm["tr"] = lambda key: ("a very long host other label"
+                                     if key.endswith("labels.hostOther") else tr(key))
+        rows = lines(render(self.vm, 80, 24))
+        self.assertEqual(rows[10][2:9], "TOP CPU")
+
+    def test_zram_row_uses_compact_sizes_when_full_row_does_not_fit(self):
+        row = lines(render(self.vm, 80, 24))[4]
+        self.assertIn("ZRAM 143M/3.2G 1.8x . swap 0/8.0G", row)
+        self.assertNotIn("MiB", row)
+
+    def test_cluster_strip_title_drops_name_before_counts(self):
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("CLUSTER . QUORUM OK 2/3", rows[6])
+        self.assertNotIn("lab", rows[6])
+
+    def test_top_cpu_percentages_share_one_right_edge(self):
+        rows = lines(render(self.vm, 80, 24))
+        edges = {rows[y][:40].index("%") for y in (11, 12, 13)}
+        self.assertEqual(len(edges), 1)
+        self.assertIn("0.1c    5%", rows[13][:40])
+
 
 if __name__ == "__main__":
     unittest.main()
```
