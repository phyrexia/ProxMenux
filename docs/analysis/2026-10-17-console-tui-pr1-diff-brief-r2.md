# Auditoría adversarial — fase 5 (diff), ronda 2 acotada — PR1 consola TUI de ProxMenux Monitor

> **Modo de respuesta:** no uses ninguna herramienta, no leas ni escribas archivos. Todo el material está
> inline abajo. Escribe tu veredicto completo directamente como texto de respuesta (Markdown).

En la ronda 1 revisaste el diff completo del PR1 (head 65127237) y recomendaste APROBAR CON CAMBIOS
(spec 0C/2I/4m; riesgo 0C/2I/3m). El Coordinador adjudicó así:

| Hallazgo | Sev. | Decisión | Razón |
|---|---|---|---|
| SC-01 `starting` vencido cae en `stale` | Important | Aceptado | `stale` exige un éxito previo; `starting` > 15 s fuera de la ventana de boot (uptime ≥ 120 s) pasa a `unavailable`. |
| SC-02 `vm["tr"]` obligatorio | Important | Aceptado, modificado | `tr` opcional; sin callable se muestra la clave completa. |
| SC-03 docs web / CHANGELOG ausentes | Minor | Rechazado | En el upstream sólo el mantenedor edita `CHANGELOG.md`, `README.md` y las docs web (historial de 60 días de `develop`); la documentación del PR1 va en `AppImage/README.md` y en el cuerpo del PR. |
| SC-04 sin `bold` en el valor grande | Minor | Aceptado | — |
| SC-05 `columns.up.lower()` | Minor | Aceptado | `labels.uptime`. |
| SC-06 `uptime` fuera de `LocalSampler` | Minor | Aceptado | Movido a `console_metrics`. |
| RQ-01 `data["age_s"]` con `data=None` | Important | Aceptado | — |
| RQ-02 enganche de `flask_server.py` sin test | Important | Aceptado | Nuevo `test_console_monitor_hook.py` + paso de CI. |
| RQ-03 conservar `cluster_resources_vm` | Minor | Rechazado | Ningún lector de esa clave en el repo (búsqueda en `AppImage/scripts/`); política del proyecto: sin shims de compatibilidad para código eliminado. |
| RQ-04 nombres `p3-*` en fixtures | Minor | Aceptado | `pve-01`, `pve-02`, clúster `lab`. |
| RQ-05 symlink `device` en fixtures | Minor | Rechazado | En producción la raíz es `/`; los fixtures crean `device/` explícitamente. |

## Lo que se te pide (revisión acotada al delta)

1. Para cada hallazgo de la ronda 1: `cerrado` / `parcial` / `abierto`, con una línea. Para los rechazados,
   evalúa la razón; si los sostienes, aporta evidencia concreta.
2. ¿El delta introduce regresiones o riesgos nuevos? En particular: la nueva rama de `starting` en
   `_remote_state`; el `bold` superpuesto en el título de la caja (posición calculada con `rfind`); la lectura
   de `/proc/uptime` dentro de `LocalSampler.sample()`; y si el nuevo test del enganche prueba de verdad lo
   que dice (extracción por AST).
3. Dos veredictos separados (spec-compliance con `Perfil verificado: feature`; riesgo/calidad), cada uno con
   Critical / Important / Minor sólo nuevos o aún abiertos, `Resumen: <n> Critical, <n> Important, <n> Minor`
   y `APROBAR | APROBAR CON CAMBIOS | RECHAZAR`. Después la tabla de cierre.

---

## Delta completo 65127237..5c7a8ab0

```diff
diff --git a/.github/workflows/test-console-tui.yml b/.github/workflows/test-console-tui.yml
index 60b26351..a3e0fdf0 100644
--- a/.github/workflows/test-console-tui.yml
+++ b/.github/workflows/test-console-tui.yml
@@ -13,6 +13,7 @@ on:
       - 'AppImage/scripts/tests/test_console_snapshot.py'
       - 'AppImage/scripts/tests/test_console_render.py'
       - 'AppImage/scripts/tests/test_cluster_resources_cache.py'
+      - 'AppImage/scripts/tests/test_console_monitor_hook.py'
       - 'AppImage/scripts/tests/test_cluster_guest_storage_ownership.py'
       - 'AppImage/scripts/tests/test_lxc_vm_ip_inventory.py'
       - 'AppImage/scripts/tests/fixtures/console/**'
@@ -28,6 +29,7 @@ on:
       - 'AppImage/scripts/tests/test_console_snapshot.py'
       - 'AppImage/scripts/tests/test_console_render.py'
       - 'AppImage/scripts/tests/test_cluster_resources_cache.py'
+      - 'AppImage/scripts/tests/test_console_monitor_hook.py'
       - 'AppImage/scripts/tests/test_cluster_guest_storage_ownership.py'
       - 'AppImage/scripts/tests/test_lxc_vm_ip_inventory.py'
       - 'AppImage/scripts/tests/fixtures/console/**'
@@ -61,6 +63,8 @@ jobs:
         run: python -I -B -m unittest discover -s AppImage/scripts/tests -p test_console_render.py -v
       - name: Test cluster resources cache
         run: python -I -B -m unittest discover -s AppImage/scripts/tests -p test_cluster_resources_cache.py -v
+      - name: Test the Monitor hook that starts the console publisher
+        run: python -I -B -m unittest discover -s AppImage/scripts/tests -p test_console_monitor_hook.py -v
       - name: Test console installer
         run: python -I -B -m unittest discover -s tests -p test_console_installer.py -v
       - name: Test cluster guest storage ownership guard
diff --git a/AppImage/scripts/console_metrics.py b/AppImage/scripts/console_metrics.py
index 060acdc7..6a435e13 100644
--- a/AppImage/scripts/console_metrics.py
+++ b/AppImage/scripts/console_metrics.py
@@ -224,6 +224,10 @@ class LocalSampler:
         return storages
 
     def sample(self, now):
+        try:
+            uptime = max(0.0, float(_read_text(self.root / "proc/uptime").split()[0]))
+        except (ValueError, IndexError):
+            uptime = 0.0
         cpu = _cpu_counters(self.root)
         net = _network(self.root)
         disk = _disk(self.root)
@@ -251,6 +255,7 @@ class LocalSampler:
         if len(loads) < 3:
             load1 = load5 = load15 = 0.0
         result = {
+            "uptime": uptime,
             "cpu_total": cpu_total,
             "cpu_history": list(self.cpu_history),
             "per_core": per_core,
diff --git a/AppImage/scripts/console_render.py b/AppImage/scripts/console_render.py
index 9aa7fd38..4dadd567 100644
--- a/AppImage/scripts/console_render.py
+++ b/AppImage/scripts/console_render.py
@@ -170,7 +170,9 @@ class Grid(list):
 
 
 def _label(vm, key, glyphs):
-    result = vm["tr"]("runtime.consoleTui." + key)
+    full_key = "runtime.consoleTui." + key
+    translator = vm.get("tr")
+    result = translator(full_key) if callable(translator) else full_key
     return str(result) if glyphs == "unicode" else sanitize(result, 200, glyphs)
 
 
@@ -200,8 +202,10 @@ def _remote_state(vm):
     resources = snapshot.get("resources") or {}
     age = _number(snapshot.get("age_s"))
     status = resources.get("status")
-    if status == "starting" and age <= 15:
-        return "starting"
+    if status == "starting":
+        return ("starting" if age <= 15 or
+                _number((vm.get("local") or {}).get("uptime"), 999999) < 120
+                else "unavailable")
     if not snapshot:
         return "starting" if _number((vm.get("local") or {}).get("uptime"), 999999) < 120 else "unavailable"
     if age > 60 or status in ("unavailable", "error"):
@@ -266,7 +270,7 @@ def _header(grid, vm, state, cluster_view):
         version = match.group(1) if match else version
         name = sanitize(node.get("name") or "", 20, grid.glyphs)
         info = (f"{name} {_separator(grid)} PVE {version} "
-                f"{_separator(grid)} {label('columns.up').lower()} {_uptime((vm.get('local') or {}).get('uptime'))}")
+                f"{_separator(grid)} {label('labels.uptime')} {_uptime((vm.get('local') or {}).get('uptime'))}")
         role = "title"
         if state == "ok" and not cluster.get("is_cluster", True):
             info = f"{name} {_separator(grid)} {label('labels.standalone')}"
@@ -315,9 +319,16 @@ def _cpu_memory(grid, vm, rows):
     memory_fraction = _number(mem.get("used")) / max(1, _number(mem.get("total")))
     left = 43 if grid.width == 80 else (grid.width + 1) // 2
     right = grid.width - left
-    grid.box(0, 1, left, rows, f"{label('panels.cpu')}  {_pct(cpu_fraction)}%")
-    grid.box(left, 1, right, rows,
-             f"{label('panels.memory')}  {_gib(mem.get('used')):.1f}/{_gib(mem.get('total')):.0f} GiB  {_pct(memory_fraction)}%")
+    cpu_title = f"{label('panels.cpu')}  {_pct(cpu_fraction)}%"
+    memory_title = (f"{label('panels.memory')}  {_gib(mem.get('used')):.1f}/"
+                    f"{_gib(mem.get('total')):.0f} GiB  {_pct(memory_fraction)}%")
+    grid.box(0, 1, left, rows, cpu_title)
+    grid.box(left, 1, right, rows, memory_title)
+    cpu_percent = f"{_pct(cpu_fraction)}%"
+    memory_percent = f"{_pct(memory_fraction)}%"
+    grid.put(2 + cpu_title.rfind(cpu_percent), 1, cpu_percent, "title", ("bold",), limit=left - 1)
+    grid.put(left + 2 + memory_title.rfind(memory_percent), 1,
+             memory_percent, "title", ("bold",), limit=grid.width - 1)
     grid.bar(2, 2, left - 4, cpu_fraction)
     grid.bar(left + 2, 2, right - 4, memory_fraction)
     load = " ".join(f"{_number(local.get('load' + str(i))):.2f}" for i in (1, 5, 15))
diff --git a/AppImage/scripts/console_tui.py b/AppImage/scripts/console_tui.py
index 3d94f282..db054360 100644
--- a/AppImage/scripts/console_tui.py
+++ b/AppImage/scripts/console_tui.py
@@ -90,14 +90,6 @@ def _configuration_language():
         return "en"
 
 
-def _uptime(proc_root):
-    try:
-        with (Path(proc_root) / "proc/uptime").open(encoding="ascii") as stream:
-            return max(0.0, float(stream.read().split()[0]))
-    except (OSError, UnicodeError, ValueError, IndexError):
-        return 0.0
-
-
 class GuestRates:
     """Derive guest byte rates only across consecutive valid observations."""
 
@@ -234,7 +226,6 @@ def _run(stdscr, args):
     while not _stop_requested:
         now = time.time()
         local = sampler.sample(time.monotonic())
-        local["uptime"] = _uptime(args.proc_root)
         try:
             console_snapshot.touch_lease(args.state_dir, now)
         except (OSError, ValueError):
@@ -267,8 +258,9 @@ def _run(stdscr, args):
         translator.set_language(language)
         resources = (data or {}).get("resources") or {}
         node = (data or {}).get("node") or {}
-        fresh = (resources.get("status") == "ok" and data["age_s"] is not None
-                 and data["age_s"] <= 10)
+        age = (data or {}).get("age_s")
+        fresh = (resources.get("status") == "ok" and age is not None
+                 and age <= 10)
         other = console_metrics.host_other(local, guests, node.get("name"), fresh)
         view_model = {"local": local, "snapshot": data, "host_other": other,
                       "glyphs": glyphs, "lang": translator.language, "tr": translator,
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_cluster_resources.json b/AppImage/scripts/tests/fixtures/console/pvesh_cluster_resources.json
index e9ff04a9..c519cb04 100644
--- a/AppImage/scripts/tests/fixtures/console/pvesh_cluster_resources.json
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_cluster_resources.json
@@ -1,8 +1,8 @@
 [
-  {"type":"node","node":"p3-01","status":"online","cpu":0.07,"maxcpu":16,"mem":1717986918,"maxmem":34359738368,"disk":21474836480,"maxdisk":53687091200,"uptime":612000,"level":"","cgroup-mode":2},
-  {"type":"node","node":"p3-02","status":"offline"},
-  {"type":"qemu","vmid":101,"name":"plex","node":"p3-01","status":"running","cpu":0.6,"maxcpu":2,"mem":4294967296,"maxmem":8589934592,"disk":0,"maxdisk":34359738368,"uptime":3600,"netin":123456789,"netout":98765432,"diskwrite":0,"tags":"media;prod","template":0},
-  {"type":"lxc","vmid":102,"name":"nginx","node":"p3-02","status":"unknown","tags":"edge;\u001b[31mred\nline","template":0},
-  {"type":"storage","id":"storage/p3-01/local-zfs","storage":"local-zfs","node":"p3-01","plugintype":"zfspool","shared":0,"status":"available","disk":21474836480,"maxdisk":53687091200,"content":"images,rootdir"},
+  {"type":"node","node":"pve-01","status":"online","cpu":0.07,"maxcpu":16,"mem":1717986918,"maxmem":34359738368,"disk":21474836480,"maxdisk":53687091200,"uptime":612000,"level":"","cgroup-mode":2},
+  {"type":"node","node":"pve-02","status":"offline"},
+  {"type":"qemu","vmid":101,"name":"plex","node":"pve-01","status":"running","cpu":0.6,"maxcpu":2,"mem":4294967296,"maxmem":8589934592,"disk":0,"maxdisk":34359738368,"uptime":3600,"netin":123456789,"netout":98765432,"diskwrite":0,"tags":"media;prod","template":0},
+  {"type":"lxc","vmid":102,"name":"nginx","node":"pve-02","status":"unknown","tags":"edge;\u001b[31mred\nline","template":0},
+  {"type":"storage","id":"storage/pve-01/local-zfs","storage":"local-zfs","node":"pve-01","plugintype":"zfspool","shared":0,"status":"available","disk":21474836480,"maxdisk":53687091200,"content":"images,rootdir"},
   {"type":"pool","pool":"unused"}
 ]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_cluster_status.json b/AppImage/scripts/tests/fixtures/console/pvesh_cluster_status.json
index 3b11f0fd..7b00d300 100644
--- a/AppImage/scripts/tests/fixtures/console/pvesh_cluster_status.json
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_cluster_status.json
@@ -1,5 +1,5 @@
 [
-  {"type":"cluster","name":"p3","quorate":1,"nodes":2,"version":7},
-  {"type":"node","name":"p3-01","nodeid":1,"online":1,"local":1,"ip":"node-one.invalid","level":""},
-  {"type":"node","name":"p3-02","nodeid":2,"online":0,"local":0,"ip":"node-two.invalid","level":""}
+  {"type":"cluster","name":"lab","quorate":1,"nodes":2,"version":7},
+  {"type":"node","name":"pve-01","nodeid":1,"online":1,"local":1,"ip":"node-one.invalid","level":""},
+  {"type":"node","name":"pve-02","nodeid":2,"online":0,"local":0,"ip":"node-two.invalid","level":""}
 ]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_no_quorum_status.json b/AppImage/scripts/tests/fixtures/console/pvesh_no_quorum_status.json
index 67cbaf74..2b60230f 100644
--- a/AppImage/scripts/tests/fixtures/console/pvesh_no_quorum_status.json
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_no_quorum_status.json
@@ -1,5 +1,5 @@
 [
-  {"type":"cluster","name":"p3","quorate":0,"nodes":2,"version":8},
-  {"type":"node","name":"p3-01","nodeid":1,"online":1,"local":1,"ip":"node-one.invalid","level":""},
-  {"type":"node","name":"p3-02","nodeid":2,"online":0,"local":0,"ip":"node-two.invalid","level":""}
+  {"type":"cluster","name":"lab","quorate":0,"nodes":2,"version":8},
+  {"type":"node","name":"pve-01","nodeid":1,"online":1,"local":1,"ip":"node-one.invalid","level":""},
+  {"type":"node","name":"pve-02","nodeid":2,"online":0,"local":0,"ip":"node-two.invalid","level":""}
 ]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_resources_full.json b/AppImage/scripts/tests/fixtures/console/pvesh_resources_full.json
index c354ac8d..54d5eb7c 100644
--- a/AppImage/scripts/tests/fixtures/console/pvesh_resources_full.json
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_resources_full.json
@@ -1,10 +1,10 @@
 [
-  {"id": "node/p3-01", "type": "node", "node": "p3-01", "status": "online", "cpu": 0.07, "maxcpu": 16},
-  {"id": "qemu/101", "type": "qemu", "vmid": 101, "name": "plex", "node": "p3-01", "status": "running", "cpu": 0.6, "maxcpu": 2, "mem": 4294967296, "maxmem": 8589934592, "tags": "media;prod"},
-  {"id": "storage/p3-01/local-zfs", "type": "storage", "storage": "local-zfs", "node": "p3-01", "status": "available"},
-  {"id": "lxc/205", "type": "lxc", "vmid": 205, "name": "nginx", "node": "p3-02", "status": "running", "cpu": 0.3, "maxcpu": 1, "mem": 858993459, "maxmem": 2147483648},
-  {"id": "node/p3-02", "type": "node", "node": "p3-02", "status": "online", "cpu": 0.09, "maxcpu": 16},
-  {"id": "qemu/103", "type": "qemu", "vmid": 103, "name": "db", "node": "p3-02", "status": "stopped", "cpu": 0, "maxcpu": 4, "mem": 0, "maxmem": 8589934592},
+  {"id": "node/pve-01", "type": "node", "node": "pve-01", "status": "online", "cpu": 0.07, "maxcpu": 16},
+  {"id": "qemu/101", "type": "qemu", "vmid": 101, "name": "plex", "node": "pve-01", "status": "running", "cpu": 0.6, "maxcpu": 2, "mem": 4294967296, "maxmem": 8589934592, "tags": "media;prod"},
+  {"id": "storage/pve-01/local-zfs", "type": "storage", "storage": "local-zfs", "node": "pve-01", "status": "available"},
+  {"id": "lxc/205", "type": "lxc", "vmid": 205, "name": "nginx", "node": "pve-02", "status": "running", "cpu": 0.3, "maxcpu": 1, "mem": 858993459, "maxmem": 2147483648},
+  {"id": "node/pve-02", "type": "node", "node": "pve-02", "status": "online", "cpu": 0.09, "maxcpu": 16},
+  {"id": "qemu/103", "type": "qemu", "vmid": 103, "name": "db", "node": "pve-02", "status": "stopped", "cpu": 0, "maxcpu": 4, "mem": 0, "maxmem": 8589934592},
   {"id": "pool/media", "type": "pool", "pool": "media"},
   {"id": "sdn/vnet/media", "type": "sdn", "sdn": "media"}
 ]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_resources_vm.json b/AppImage/scripts/tests/fixtures/console/pvesh_resources_vm.json
index b8dbcd29..9b749098 100644
--- a/AppImage/scripts/tests/fixtures/console/pvesh_resources_vm.json
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_resources_vm.json
@@ -1,5 +1,5 @@
 [
-  {"id": "qemu/101", "type": "qemu", "vmid": 101, "name": "plex", "node": "p3-01", "status": "running", "cpu": 0.6, "maxcpu": 2, "mem": 4294967296, "maxmem": 8589934592, "tags": "media;prod"},
-  {"id": "lxc/205", "type": "lxc", "vmid": 205, "name": "nginx", "node": "p3-02", "status": "running", "cpu": 0.3, "maxcpu": 1, "mem": 858993459, "maxmem": 2147483648},
-  {"id": "qemu/103", "type": "qemu", "vmid": 103, "name": "db", "node": "p3-02", "status": "stopped", "cpu": 0, "maxcpu": 4, "mem": 0, "maxmem": 8589934592}
+  {"id": "qemu/101", "type": "qemu", "vmid": 101, "name": "plex", "node": "pve-01", "status": "running", "cpu": 0.6, "maxcpu": 2, "mem": 4294967296, "maxmem": 8589934592, "tags": "media;prod"},
+  {"id": "lxc/205", "type": "lxc", "vmid": 205, "name": "nginx", "node": "pve-02", "status": "running", "cpu": 0.3, "maxcpu": 1, "mem": 858993459, "maxmem": 2147483648},
+  {"id": "qemu/103", "type": "qemu", "vmid": 103, "name": "db", "node": "pve-02", "status": "stopped", "cpu": 0, "maxcpu": 4, "mem": 0, "maxmem": 8589934592}
 ]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_standalone_status.json b/AppImage/scripts/tests/fixtures/console/pvesh_standalone_status.json
index 249e6347..ed2bec3e 100644
--- a/AppImage/scripts/tests/fixtures/console/pvesh_standalone_status.json
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_standalone_status.json
@@ -1,3 +1,3 @@
 [
-  {"type":"node","name":"p3-01","nodeid":1,"online":1,"local":1,"ip":"node-one.invalid","level":""}
+  {"type":"node","name":"pve-01","nodeid":1,"online":1,"local":1,"ip":"node-one.invalid","level":""}
 ]
diff --git a/AppImage/scripts/tests/fixtures/console/viewmodel_cluster.json b/AppImage/scripts/tests/fixtures/console/viewmodel_cluster.json
index 104337b2..1f425940 100644
--- a/AppImage/scripts/tests/fixtures/console/viewmodel_cluster.json
+++ b/AppImage/scripts/tests/fixtures/console/viewmodel_cluster.json
@@ -25,19 +25,19 @@
     "schema": 1,
     "age_s": 2,
     "language": "en",
-    "node": {"name": "p3-01", "pve_version": "pve-manager/9.0.3/abc"},
-    "cluster": {"status": "ok", "is_cluster": true, "name": "p3", "quorate": true, "nodes_total": 3, "nodes_online": 2, "members": [{"name": "p3-01", "online": true, "local": true}, {"name": "p3-02", "online": true, "local": false}, {"name": "p3-03", "online": false, "local": false}]},
+    "node": {"name": "pve-01", "pve_version": "pve-manager/9.0.3/abc"},
+    "cluster": {"status": "ok", "is_cluster": true, "name": "lab", "quorate": true, "nodes_total": 3, "nodes_online": 2, "members": [{"name": "pve-01", "online": true, "local": true}, {"name": "pve-02", "online": true, "local": false}, {"name": "pve-03", "online": false, "local": false}]},
     "resources": {
       "status": "ok",
-      "nodes": [{"name": "p3-01", "status": "online", "cpu": 0.07, "maxcpu": 4, "mem": 17179869184, "maxmem": 34359738368, "disk": 21474836480, "maxdisk": 53687091200, "uptime": 619200}, {"name": "p3-02", "status": "online", "cpu": 0.09, "maxcpu": 8, "mem": 19327352832, "maxmem": 34359738368, "disk": 21474836480, "maxdisk": 53687091200, "uptime": 619200}],
+      "nodes": [{"name": "pve-01", "status": "online", "cpu": 0.07, "maxcpu": 4, "mem": 17179869184, "maxmem": 34359738368, "disk": 21474836480, "maxdisk": 53687091200, "uptime": 619200}, {"name": "pve-02", "status": "online", "cpu": 0.09, "maxcpu": 8, "mem": 19327352832, "maxmem": 34359738368, "disk": 21474836480, "maxdisk": 53687091200, "uptime": 619200}],
       "guests": [
-        {"vmid": 101, "type": "qemu", "name": "plex", "node": "p3-01", "status": "running", "cpu": 0.6, "maxcpu": 2, "mem": 4294967296, "maxmem": 8589934592, "uptime": 3600, "template": false, "tags": ["media"]},
-        {"vmid": 103, "type": "qemu", "name": "db", "node": "p3-02", "status": "running", "cpu": 0.05, "maxcpu": 2, "mem": 6442450944, "maxmem": 8589934592, "uptime": 7200, "template": false},
-        {"vmid": 205, "type": "lxc", "name": "nginx", "node": "p3-02", "status": "running", "cpu": 0.3, "maxcpu": 1, "mem": 858993459, "maxmem": 2147483648, "uptime": 1800, "template": false},
-        {"vmid": 110, "type": "qemu", "name": "win", "node": "p3-01", "status": "unknown", "cpu": 0, "maxcpu": 4, "mem": 2147483648, "maxmem": 4294967296, "uptime": 0, "template": false},
-        {"vmid": 999, "type": "lxc", "name": "template", "node": "p3-01", "status": "stopped", "cpu": 0.9, "maxcpu": 1, "mem": 4294967296, "template": true}
+        {"vmid": 101, "type": "qemu", "name": "plex", "node": "pve-01", "status": "running", "cpu": 0.6, "maxcpu": 2, "mem": 4294967296, "maxmem": 8589934592, "uptime": 3600, "template": false, "tags": ["media"]},
+        {"vmid": 103, "type": "qemu", "name": "db", "node": "pve-02", "status": "running", "cpu": 0.05, "maxcpu": 2, "mem": 6442450944, "maxmem": 8589934592, "uptime": 7200, "template": false},
+        {"vmid": 205, "type": "lxc", "name": "nginx", "node": "pve-02", "status": "running", "cpu": 0.3, "maxcpu": 1, "mem": 858993459, "maxmem": 2147483648, "uptime": 1800, "template": false},
+        {"vmid": 110, "type": "qemu", "name": "win", "node": "pve-01", "status": "unknown", "cpu": 0, "maxcpu": 4, "mem": 2147483648, "maxmem": 4294967296, "uptime": 0, "template": false},
+        {"vmid": 999, "type": "lxc", "name": "template", "node": "pve-01", "status": "stopped", "cpu": 0.9, "maxcpu": 1, "mem": 4294967296, "template": true}
       ],
-      "storages": [{"storage": "local-zfs", "node": "p3-01", "disk": 21474836480, "maxdisk": 53687091200, "status": "available"}]
+      "storages": [{"storage": "local-zfs", "node": "pve-01", "disk": 21474836480, "maxdisk": 53687091200, "status": "available"}]
     }
   }
 }
diff --git a/AppImage/scripts/tests/test_console_metrics.py b/AppImage/scripts/tests/test_console_metrics.py
index 7cd09dde..cc31b5a3 100644
--- a/AppImage/scripts/tests/test_console_metrics.py
+++ b/AppImage/scripts/tests/test_console_metrics.py
@@ -22,6 +22,7 @@ class LocalSamplerTests(unittest.TestCase):
         temporary = tempfile.TemporaryDirectory()
         self.addCleanup(temporary.cleanup)
         self.root = Path(temporary.name)
+        self.write("proc/uptime", "60.5 12.0\n")
         self.write("proc/loadavg", "0.25 0.50 0.75 1/100 1\n")
         self.write("proc/meminfo", (
             "MemTotal: 10000 kB\nMemFree: 1000 kB\nMemAvailable: 4000 kB\n"
@@ -61,6 +62,7 @@ class LocalSamplerTests(unittest.TestCase):
         self.assertEqual(first["cpu_history"], [])
         self.assertEqual(first["net"]["rx_bps"], 0)
         self.assertEqual(first["disk"]["read_bps"], 0)
+        self.assertEqual(first["uptime"], 60.5)
 
         self.second_counters()
         second = sampler.sample(11.0)
@@ -75,6 +77,14 @@ class LocalSamplerTests(unittest.TestCase):
         })
         self.assertEqual(second["disk"], {"read_bps": 4096, "write_bps": 6144})
 
+    def test_uptime_uses_injected_root_and_falls_back_to_zero(self):
+        sampler = self.sampler()
+        self.assertEqual(sampler.sample(1.0)["uptime"], 60.5)
+        self.write("proc/uptime", "invalid\n")
+        self.assertEqual(sampler.sample(2.0)["uptime"], 0.0)
+        (self.root / "proc/uptime").unlink()
+        self.assertEqual(sampler.sample(3.0)["uptime"], 0.0)
+
     def test_only_physical_nics_with_device_are_counted(self):
         for nic in ("fwbr1", "fwpr1", "fwln1", "tap1", "veth1", "wlan0"):
             self.write(f"sys/class/net/{nic}/speed", "1000\n")
@@ -197,19 +207,19 @@ class HostOtherTests(unittest.TestCase):
     def test_arithmetic_filters_local_running_guests(self):
         local = {"cpu_total": 0.5, "threads": 8, "mem": {"used": 10_000, "arc": 1000}}
         guests = [
-            {"node": "p3-01", "status": "running", "cpu": 0.5, "maxcpu": 2, "mem": 3000},
-            {"node": "p3-01", "status": "stopped", "cpu": 0.5, "maxcpu": 2, "mem": 2000},
-            {"node": "p3-02", "status": "running", "cpu": 0.5, "maxcpu": 2, "mem": 4000},
+            {"node": "pve-01", "status": "running", "cpu": 0.5, "maxcpu": 2, "mem": 3000},
+            {"node": "pve-01", "status": "stopped", "cpu": 0.5, "maxcpu": 2, "mem": 2000},
+            {"node": "pve-02", "status": "running", "cpu": 0.5, "maxcpu": 2, "mem": 4000},
         ]
-        self.assertEqual(host_other(local, guests, "p3-01", snapshot_fresh=True),
+        self.assertEqual(host_other(local, guests, "pve-01", snapshot_fresh=True),
                          {"cpu": 3.0, "mem": 4000})
 
     def test_clamps_and_omits_stale_snapshot(self):
         local = {"cpu_total": 0.1, "threads": 2, "mem": {"used": 1000, "arc": 500}}
-        guests = [{"node": "p3-01", "status": "running", "cpu": 1, "maxcpu": 2, "mem": 2000}]
-        self.assertEqual(host_other(local, guests, "p3-01", snapshot_fresh=True),
+        guests = [{"node": "pve-01", "status": "running", "cpu": 1, "maxcpu": 2, "mem": 2000}]
+        self.assertEqual(host_other(local, guests, "pve-01", snapshot_fresh=True),
                          {"cpu": 0.0, "mem": 0})
-        self.assertEqual(host_other(local, guests, "p3-01", snapshot_fresh=False),
+        self.assertEqual(host_other(local, guests, "pve-01", snapshot_fresh=False),
                          {"cpu": None, "mem": None})
 
 
diff --git a/AppImage/scripts/tests/test_console_monitor_hook.py b/AppImage/scripts/tests/test_console_monitor_hook.py
new file mode 100644
index 00000000..1013ecbb
--- /dev/null
+++ b/AppImage/scripts/tests/test_console_monitor_hook.py
@@ -0,0 +1,87 @@
+"""Check the Monitor console publisher glue without importing Flask."""
+
+import ast
+import json
+import subprocess
+import tempfile
+import unittest
+from pathlib import Path
+from types import SimpleNamespace
+from unittest.mock import Mock
+
+
+SERVER_SOURCE = Path(__file__).resolve().parents[1] / "flask_server.py"
+
+
+def load_server_functions(*names):
+    tree = ast.parse(SERVER_SOURCE.read_text(encoding="utf-8"))
+    functions = {node.name: node for node in tree.body
+                 if isinstance(node, ast.FunctionDef) and node.name in names}
+    missing = set(names) - functions.keys()
+    if missing:
+        raise AssertionError(f"Missing production functions: {sorted(missing)}")
+    module = ast.Module(body=[functions[name] for name in names], type_ignores=[])
+    ast.fix_missing_locations(module)
+    namespace = {"subprocess": subprocess, "json": json, "open": open}
+    exec(compile(module, str(SERVER_SOURCE), "exec"), namespace)
+    return namespace
+
+
+class ConsoleMonitorHookTests(unittest.TestCase):
+    def test_run_pvesh_uses_exact_command_and_parses_list(self):
+        namespace = load_server_functions("_console_run_pvesh")
+        expected = [{"type": "node", "name": "pve-01"}]
+        run = Mock(return_value=SimpleNamespace(returncode=0, stdout=json.dumps(expected)))
+        namespace["subprocess"] = SimpleNamespace(run=run)
+        self.assertEqual(namespace["_console_run_pvesh"](), expected)
+        run.assert_called_once_with(
+            ["pvesh", "get", "/cluster/status", "--output-format", "json"],
+            capture_output=True, text=True, timeout=10,
+        )
+
+    def test_run_pvesh_raises_on_nonzero_exit(self):
+        namespace = load_server_functions("_console_run_pvesh")
+        namespace["subprocess"] = SimpleNamespace(
+            run=Mock(return_value=SimpleNamespace(returncode=1, stdout="[]")))
+        with self.assertRaises(OSError):
+            namespace["_console_run_pvesh"]()
+
+    def test_read_language_handles_config_and_bad_documents(self):
+        namespace = load_server_functions("_console_read_language")
+        with tempfile.TemporaryDirectory() as directory:
+            config = Path(directory) / "config.json"
+            namespace["open"] = lambda _path, **kwargs: config.open(**kwargs)
+            self.assertIsNone(namespace["_console_read_language"]())
+            for document, expected in (({"language": "es"}, "es"),
+                                       ("invalid JSON", None),
+                                       (["en"], None)):
+                config.write_text(json.dumps(document) if not isinstance(document, str)
+                                  else document, encoding="utf-8")
+                self.assertEqual(namespace["_console_read_language"](), expected)
+
+    def test_publisher_hook_is_module_level_and_guarded(self):
+        tree = ast.parse(SERVER_SOURCE.read_text(encoding="utf-8"))
+        expected_args = ["get_proxmox_node_name", "get_cached_pvesh_cluster_resources",
+                         "_console_run_pvesh", "_console_read_language"]
+        matches = []
+        for node in tree.body:
+            if not isinstance(node, ast.Try):
+                continue
+            calls = [child for statement in node.body for child in ast.walk(statement)
+                     if isinstance(child, ast.Call)
+                     and isinstance(child.func, ast.Attribute)
+                     and child.func.attr == "ensure_publisher"]
+            for call in calls:
+                matches.append((node, call))
+        self.assertEqual(len(matches), 1)
+        guard, call = matches[0]
+        self.assertTrue(any(isinstance(handler.type, ast.Name)
+                            and handler.type.id == "Exception" for handler in guard.handlers))
+        self.assertEqual([argument.id for argument in call.args], expected_args)
+        keywords = {keyword.arg: keyword.value for keyword in call.keywords}
+        self.assertIsInstance(keywords["get_pve_version"], ast.Name)
+        self.assertEqual(keywords["get_pve_version"].id, "get_proxmox_version")
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/AppImage/scripts/tests/test_console_render.py b/AppImage/scripts/tests/test_console_render.py
index 723c9248..252d60f0 100644
--- a/AppImage/scripts/tests/test_console_render.py
+++ b/AppImage/scripts/tests/test_console_render.py
@@ -48,6 +48,25 @@ class ConsoleRenderTests(unittest.TestCase):
                     self.assertTrue(all(len(row) == width for row in grid))
                     self.assertTrue(all(len(cell) == 3 for row in grid for cell in row))
 
+    def test_json_only_viewmodel_renders_80x24_without_translator(self):
+        self.vm.pop("tr")
+        grid = render(self.vm, 80, 24)
+        self.assertEqual(len(grid), 24)
+        self.assertTrue(all(len(row) == 80 for row in grid))
+
+    def test_cpu_and_memory_title_percentages_are_bold(self):
+        grid = render(self.vm, 80, 24)
+        title = lines(grid)[1]
+        for value in ("70%", "50%"):
+            start = title.index(value)
+            self.assertTrue(all("bold" in grid[1][x][2]
+                                for x in range(start, start + len(value))))
+        self.assertNotIn("bold", grid[1][title.index("CPU")][2])
+
+    def test_node_header_uses_uptime_label(self):
+        self.vm["tr"] = lambda key: "uptime-label" if key.endswith("labels.uptime") else tr(key)
+        self.assertIn("uptime-label", lines(render(self.vm, 128, 48))[0])
+
     def test_node_80x24_panel_positions_and_collapse(self):
         rows = lines(render(self.vm, 80, 24))
         self.assertIn("[Node] Cluster", rows[0])
@@ -122,7 +141,7 @@ class ConsoleRenderTests(unittest.TestCase):
     def test_cluster_offline_first_and_coverage_suppresses_totals(self):
         self.vm["selected_view"] = "cluster"
         rows = lines(render(self.vm, 80, 24))
-        self.assertIn("p3-03", rows[2])
+        self.assertIn("pve-03", rows[2])
         self.assertIn("offline", rows[2])
         self.assertIn("coverage 2/3", rows[0])
         self.assertNotIn("64 GiB", rows[0])
@@ -146,7 +165,7 @@ class ConsoleRenderTests(unittest.TestCase):
         self.assertIn(".", lines(grid)[3])
         cluster = self.vm["snapshot"]["cluster"]
         for index in range(5):
-            cluster["members"].append({"name": f"p3-{index + 10}", "online": False})
+            cluster["members"].append({"name": f"pve-{index + 10}", "online": False})
         cluster["nodes_total"] = len(cluster["members"])
         guests = self.vm["snapshot"]["resources"]["guests"]
         for index in range(13):
@@ -205,7 +224,7 @@ class ConsoleRenderTests(unittest.TestCase):
         self.vm["selected_view"] = "cluster"
         self.vm["snapshot"]["cluster"]["members"] = []
         rows = lines(render(self.vm, 80, 24))
-        self.assertIn("p3-01", rows[2])
+        self.assertIn("pve-01", rows[2])
 
     def test_no_quorum_and_stale_roles(self):
         self.vm["selected_view"] = "cluster"
@@ -231,6 +250,22 @@ class ConsoleRenderTests(unittest.TestCase):
         self.assertIn("MONITOR UNAVAILABLE", lines(grid)[0])
         self.assertEqual(grid[0][lines(grid)[0].index("MONITOR UNAVAILABLE")][1], "crit")
 
+    def test_old_starting_snapshot_after_boot_is_unavailable(self):
+        self.vm["snapshot"]["resources"]["status"] = "starting"
+        self.vm["snapshot"]["age_s"] = 16
+        self.vm["local"]["uptime"] = 600
+        grid = render(self.vm, 80, 24)
+        self.assertIn("MONITOR UNAVAILABLE", lines(grid)[0])
+        self.assertEqual(grid[0][lines(grid)[0].index("MONITOR UNAVAILABLE")][1], "crit")
+
+    def test_old_starting_snapshot_during_boot_stays_warn(self):
+        self.vm["snapshot"]["resources"]["status"] = "starting"
+        self.vm["snapshot"]["age_s"] = 16
+        self.vm["local"]["uptime"] = 60
+        grid = render(self.vm, 80, 24)
+        self.assertIn("Monitor starting", lines(grid)[0])
+        self.assertEqual(grid[0][lines(grid)[0].index("Monitor starting")][1], "warn")
+
     def test_absent_snapshot_uses_local_uptime_for_starting(self):
         self.vm["snapshot"] = None
         self.vm["local"]["uptime"] = 80
@@ -271,10 +306,10 @@ class ConsoleRenderTests(unittest.TestCase):
         self.assertIn(grid[2][2][0], ("█", "░"))
 
     def test_snapshot_and_nic_names_are_sanitized(self):
-        self.vm["snapshot"]["node"]["name"] = "p3\x1b[31m-01"
+        self.vm["snapshot"]["node"]["name"] = "pve\x1b[31m-01"
         self.vm["local"]["net"]["nic"] = "eno\n1"
         rows = lines(render(self.vm, 80, 24))
-        self.assertIn("p3-01", rows[0])
+        self.assertIn("pve-01", rows[0])
         self.assertIn("eno1", rows[6])
         self.assertNotIn("[31m", "\n".join(rows))
 
diff --git a/AppImage/scripts/tests/test_console_snapshot.py b/AppImage/scripts/tests/test_console_snapshot.py
index 0397784e..f124c841 100644
--- a/AppImage/scripts/tests/test_console_snapshot.py
+++ b/AppImage/scripts/tests/test_console_snapshot.py
@@ -37,7 +37,7 @@ class ConsoleSnapshotTests(unittest.TestCase):
 
     def publisher(self, resources=None, status=None, version=None):
         return console_snapshot.Publisher(
-            get_node_name=lambda: "p3-01",
+            get_node_name=lambda: "pve-01",
             get_resources=lambda: self.resources if resources is None else resources(),
             run_pvesh=lambda: self.status if status is None else status(),
             read_language=lambda: "es",
@@ -57,7 +57,7 @@ class ConsoleSnapshotTests(unittest.TestCase):
     def test_normalizes_complete_resources_and_missing_fields(self):
         result = console_snapshot.normalize_resources(self.resources)
         self.assertEqual(set(result), {"nodes", "guests", "storages", "truncated"})
-        self.assertEqual(result["nodes"][0]["name"], "p3-01")
+        self.assertEqual(result["nodes"][0]["name"], "pve-01")
         self.assertEqual(result["nodes"][0]["cgroup_mode"], 2)
         self.assertIsNone(result["nodes"][1]["cpu"])
         self.assertEqual(result["guests"][0]["tags"], ["media", "prod"])
@@ -100,7 +100,7 @@ class ConsoleSnapshotTests(unittest.TestCase):
     def test_initial_snapshot_and_idempotent_ensure(self):
         with mock.patch.object(console_snapshot.threading, "Thread") as thread:
             first = console_snapshot.ensure_publisher(
-                lambda: "p3-01", lambda: self.resources, lambda: self.status,
+                lambda: "pve-01", lambda: self.resources, lambda: self.status,
                 lambda: "es", path=self.path, state_dir=self.state_dir,
                 clock=lambda: 100.0, version="9.9.9",
             )
@@ -113,7 +113,7 @@ class ConsoleSnapshotTests(unittest.TestCase):
         data = json.loads(self.path.read_text(encoding="utf-8"))
         self.assertEqual((data["cluster"]["status"], data["resources"]["status"]),
                          ("starting", "starting"))
-        self.assertEqual(data["node"]["name"], "p3-01")
+        self.assertEqual(data["node"]["name"], "pve-01")
         self.assertEqual(data["generator"]["version"], "9.9.9")
 
     def test_unwritable_snapshot_path_does_not_break_ensure_publisher(self):
@@ -121,7 +121,7 @@ class ConsoleSnapshotTests(unittest.TestCase):
         blocked.write_text("", encoding="utf-8")
         with mock.patch.object(console_snapshot.threading, "Thread") as thread:
             publisher = console_snapshot.ensure_publisher(
-                lambda: "p3-01", lambda: self.resources, lambda: self.status,
+                lambda: "pve-01", lambda: self.resources, lambda: self.status,
                 lambda: "es", path=blocked / "console.json", state_dir=self.state_dir,
                 clock=lambda: 100.0,
             )
@@ -274,7 +274,7 @@ class ConsoleSnapshotTests(unittest.TestCase):
             calls.append(1)
             return "pve-manager/9.0.3"
         publisher = console_snapshot.Publisher(
-            lambda: "p3-01", lambda: self.resources, lambda: self.status,
+            lambda: "pve-01", lambda: self.resources, lambda: self.status,
             lambda: "en", path=self.path, state_dir=self.state_dir,
             get_pve_version=version,
         )
diff --git a/AppImage/scripts/tests/test_console_tui_smoke.py b/AppImage/scripts/tests/test_console_tui_smoke.py
index f8ad44eb..cc2062f3 100644
--- a/AppImage/scripts/tests/test_console_tui_smoke.py
+++ b/AppImage/scripts/tests/test_console_tui_smoke.py
@@ -23,7 +23,7 @@ sys.path.insert(0, str(SCRIPTS))
 import console_tui  # noqa: E402
 
 
-def guest(vmid=101, node="p3-01", uptime=30, **counters):
+def guest(vmid=101, node="pve-01", uptime=30, **counters):
     return {"type": "qemu", "vmid": vmid, "node": node, "uptime": uptime,
             "netin": counters.get("netin", 100),
             "netout": counters.get("netout", 200),
@@ -43,9 +43,9 @@ class GuestRateTests(unittest.TestCase):
                          {"net_rx_bps": 10, "net_tx_bps": 4,
                           "disk_read_bps": 8, "disk_write_bps": 8})
         self.assertNotIn("net_rx_bps", rates.update([guest(netin=999)], 15)[0])
-        self.assertNotIn("net_rx_bps", rates.update([guest(node="p3-02")], 20)[0])
-        self.assertNotIn("net_rx_bps", rates.update([guest(node="p3-02", uptime=1)], 25)[0])
-        self.assertNotIn("net_rx_bps", rates.update([guest(node="p3-02", uptime=2,
+        self.assertNotIn("net_rx_bps", rates.update([guest(node="pve-02")], 20)[0])
+        self.assertNotIn("net_rx_bps", rates.update([guest(node="pve-02", uptime=1)], 25)[0])
+        self.assertNotIn("net_rx_bps", rates.update([guest(node="pve-02", uptime=2,
                                                             netin=1)], 30)[0])
 
     def test_missing_counter_resets_its_rate(self):
@@ -129,7 +129,7 @@ class TerminalSmokeTests(unittest.TestCase):
     def write_snapshot(self):
         document = {"schema": 1, "generator": {"name": "monitor", "version": "1"},
                     "observed_at": time.time(), "publish_interval_s": 5,
-                    "language": "en", "node": {"status": "ok", "name": "p3-01"},
+                    "language": "en", "node": {"status": "ok", "name": "pve-01"},
                     "cluster": {"status": "ok", "members": [], "is_cluster": False},
                     "resources": {"status": "ok", "observed_at": time.time(),
                                   "nodes": [], "guests": [], "storages": []}}
```
