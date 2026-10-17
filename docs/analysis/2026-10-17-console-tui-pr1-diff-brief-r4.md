# Auditoría adversarial — fase 5 (diff), ronda 4 acotada — PR1 consola TUI de ProxMenux Monitor

> **Modo de respuesta:** no uses ninguna herramienta, no leas ni escribas archivos. Todo el material está
> inline abajo. Escribe tu veredicto completo directamente como texto de respuesta (Markdown).

Ronda 3 (sobre `731b099b`): APROBAR CON CAMBIOS, 0C/2I/3m. Adjudicación del Coordinador:

| Hallazgo | Sev. | Decisión | Razón |
|---|---|---|---|
| 1 standalone muestra `no data` en la tira | Important | Aceptado, modificado | Con estado `ok` la tira ya omitía CLUSTER y DISK llevaba `STANDALONE`, pero con snapshot no fresco el standalone caía en `CLUSTER / no data`; ahora la condición es `is_cluster` falso en cualquier estado. |
| 2 `topCpuShort` sólo en `en` | Important | Rechazado | `Translator.__call__` en `console_tui.py` resuelve por clave: idioma → inglés → clave (`return (_catalog_value(self._load(self.language), key) or _catalog_value(self._load("en"), key) or key)`); el bot del repo rellena los demás catálogos desde `en`. Lo mismo vale para la nueva `footer.switchNode`. |
| 3 `_fit` sin candidatos | Minor | Rechazado | Todos los call sites pasan 2–3 candidatos. |
| 4 ancho de núcleos ≥ 10 | Minor | Sin cambio de código | `_top_row` alinea a la derecha la cadena completa (`_text(grid, right - len(value), y, value, ...)`), así que el `%` queda en el borde derecho con `12.0c` o `1.2c`; se añadió un test de regresión que lo fija. |
| 5 `_decimal` con negativos | Minor | Rechazado | Los valores llegan con `max(0, …)`. |
| (Coordinador) pie de la vista Clúster | Minor | Aceptado | `footer.switchNode = "Tab node"`. |

QA sobre `54f73b4a` (qaw-p3-02): 112 tests en verde en 10 suites + `bash -n` + `py_compile` + JSON del
catálogo; `AppRun --tui --once` empaquetado sale 0 con catálogo, en ACS y Unicode.

Pide: (1) cierre `cerrado/parcial/abierto` de cada hallazgo de la ronda 3 (evalúa los rechazos y el "sin cambio
de código" con la evidencia citada); (2) regresiones o riesgos nuevos en el delta; (3) dos veredictos
(spec-compliance con `Perfil verificado: feature`; riesgo/calidad) con Critical / Important / Minor sólo nuevos
o abiertos, `Resumen: <n> Critical, <n> Important, <n> Minor` y `APROBAR | APROBAR CON CAMBIOS | RECHAZAR`.

## Delta 731b099b..54f73b4a (salvo la línea minificada de en/common.json, que sólo añade `footer.switchNode`)

```diff
diff --git a/AppImage/scripts/console_render.py b/AppImage/scripts/console_render.py
index 70699eb6..9db92d1c 100644
--- a/AppImage/scripts/console_render.py
+++ b/AppImage/scripts/console_render.py
@@ -318,7 +318,7 @@ def _coverage(snapshot):
 
 def _footer(grid, vm, state, cluster_view):
     label = lambda key: _label(vm, key, grid.glyphs)
-    parts = [label("footer.quit"), label("footer.switch")]
+    parts = [label("footer.quit"), label("footer.switchNode" if cluster_view else "footer.switch")]
     if cluster_view:
         parts.append(label("footer.sort"))
     parts.extend((label("footer.refresh"), label("footer.glyphs")))
@@ -398,7 +398,7 @@ def _strip(grid, vm, y, rows, state):
     local = vm.get("local") or {}
     snapshot = vm.get("snapshot") or {}
     cluster = snapshot.get("cluster") or {}
-    standalone = not cluster.get("is_cluster", True) and state == "ok"
+    standalone = not cluster.get("is_cluster", True)
     net_width = 23 if grid.width == 80 else round(grid.width * .28)
     disk_width = (grid.width - net_width if standalone else 28) if grid.width == 80 else round(grid.width * (.72 if standalone else .34))
     cluster_x = net_width + disk_width
diff --git a/AppImage/scripts/tests/test_console_render.py b/AppImage/scripts/tests/test_console_render.py
index 56f180ec..c397d670 100644
--- a/AppImage/scripts/tests/test_console_render.py
+++ b/AppImage/scripts/tests/test_console_render.py
@@ -394,6 +394,36 @@ class ConsoleRenderTests(unittest.TestCase):
         self.assertEqual(len(edges), 1)
         self.assertIn("0.1c    5%", rows[13][:40])
 
+    def test_top_cpu_cores_and_percentages_align_for_wide_and_narrow_values(self):
+        self.vm["snapshot"]["resources"]["guests"] = [
+            {"type": "qemu", "vmid": 101, "name": "big", "node": "pve-01", "status": "running",
+             "cpu": 1.0, "maxcpu": 12, "mem": 0},
+            {"type": "lxc", "vmid": 102, "name": "small", "node": "pve-01", "status": "running",
+             "cpu": 0.1, "maxcpu": 12, "mem": 0},
+        ]
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("12.0c", rows[11][:40])
+        self.assertIn("1.2c", rows[12][:40])
+        edges = {rows[y][:40].index("%") for y in (11, 12)}
+        self.assertEqual(len(edges), 1)
+
+    def test_standalone_strip_keeps_standalone_layout_when_resources_are_stale(self):
+        self.vm["snapshot"]["cluster"]["is_cluster"] = False
+        self.vm["snapshot"]["resources"]["status"] = "stale"
+        rows = lines(render(self.vm, 80, 24))
+        self.assertNotIn("CLUSTER", rows[6])
+        self.assertIn("DISK", rows[6])
+        self.assertIn("STANDALONE", rows[6])
+
+    def test_footer_names_the_other_view_in_each_view(self):
+        self.vm["selected_view"] = "cluster"
+        footer = lines(render(self.vm, 80, 24))[-1]
+        self.assertIn("Tab node", footer)
+        self.assertNotIn("Tab cluster", footer)
+        self.vm["selected_view"] = "node"
+        footer = lines(render(self.vm, 80, 24))[-1]
+        self.assertIn("Tab cluster", footer)
+
 
 if __name__ == "__main__":
     unittest.main()
```
