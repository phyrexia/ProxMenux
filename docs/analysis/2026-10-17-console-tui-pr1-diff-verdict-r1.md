---
base_sha: 227780aba76eb3f35dc50b888b3b28facd1f0c5f
head_sha: 651272374cafb36d906d47dbc8f75ca4b2f0b76f
verdict: pending
auditor: "agy --mode plan --model gemini-3.8-flash-high (PR1 diff, round 1)"
scope: diff
kind: auditor
profile: feature
date: 2026-10-17
findings: {critical: 0, important: 4, minor: 7}
---

# Recibo de revisión — PR1 consola TUI, diff ronda 1

Veredicto `pending`: APROBAR CON CAMBIOS en ambos ejes. Adjudicación del Coordinador:

| Hallazgo | Sev. | Decisión | Razón |
|---|---|---|---|
| SC-01 `starting` vencido cae en `stale` | Important | Aceptado | `stale` exige un éxito previo; `starting` > 15 s sin boot reciente es `unavailable`. |
| SC-02 `vm["tr"]` obligatorio | Important | Aceptado, modificado | `tr` opcional con fallback a la clave; el ViewModel sigue admitiendo el callable inyectado. |
| SC-03 docs web / CHANGELOG ausentes | Minor | Rechazado | En el upstream sólo el mantenedor edita `CHANGELOG.md`, `README.md` y las docs web (historial de 60 días); P1-T11 absorbida en `AppImage/README.md` y el cuerpo del PR. |
| SC-04 sin `bold` en el valor grande | Minor | Aceptado | Contrato de §2.4. |
| SC-05 `columns.up.lower()` | Minor | Aceptado | Usar `labels.uptime`. |
| SC-06 `uptime` fuera de `LocalSampler` | Minor | Aceptado | La lectura de `/proc` pertenece a `console_metrics`. |
| RQ-01 `data["age_s"]` con `data=None` | Important | Aceptado | Acceso seguro. |
| RQ-02 enganche de `flask_server.py` sin test | Important | Aceptado | Test por AST de `_console_run_pvesh`/`_console_read_language` y de los argumentos de `ensure_publisher`. |
| RQ-03 conservar `cluster_resources_vm` | Minor | Rechazado | Ningún lector (verificado con `rg`); sin shims de compatibilidad para código eliminado. |
| RQ-04 nombres `p3-*` en fixtures | Minor | Aceptado | Nombres genéricos para el upstream. |
| RQ-05 `device` symlink en fixtures | Minor | Rechazado | En producción la raíz es `/`; los fixtures crean `device/`. |

Salida del Auditor, sin editar:

# Auditoría adversarial — Fase 5 (Diff) — PR1 Consola TUI de ProxMenux Monitor

**Repositorio base:** `MacRimi/ProxMenux` (`origin/develop`, commit `227780ab`)  
**Perfil verificado:** `feature`  
**Alcance de revisión:** PR1 (Consola TUI utilizable manualmente por `root` vía `proxmenux-console` / `AppRun --tui`, sin usuario de sistema ni drop-in de getty).

---

## 1. Veredicto: Spec-compliance (Cumplimiento de Especificación ADR 001 rev 2)

### Hallazgos de Spec-compliance

#### [Important] SC-01: Desviación en la máquina de estados ante `resources.status == "starting"` con edad > 15 s
- **Archivo y línea:** [AppImage/scripts/console_render.py:204-216](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/console_render.py#L204-L216)
- **Evidencia:**
  En `_remote_state(vm)`:
  ```python
  if status == "starting" and age <= 15:
      return "starting"
  if not snapshot:
      return "starting" if _number((vm.get("local") or {}).get("uptime"), 999999) < 120 else "unavailable"
  if age > 60 or status in ("unavailable", "error"):
      return "unavailable"
  if status == "stale" or age > 10:
      return "stale"
  return "ok" if status == "ok" else "unavailable"
  ```
  Si el publicador arrancó y escribió el snapshot inicial (`status == "starting"`), pero `pvesh` tarda o falla y transcurren más de 15 segundos (`age > 15` y `age <= 60`), la condición `status == "starting" and age <= 15` es falsa. Como `snapshot` no está vacío, pasa al chequeo `status == "stale" or age > 10`, que evalúa a `True` (dado que `age > 10`). Devuelve `"stale"`.
  Esto contradice el ADR 001 §2.2 / §2.3: `stale` significa *"hubo éxito antes pero la última falló (se conservan datos)"*. En `starting` nunca hubo éxito previo ni datos. Al devolver `"stale"`, la cabecera pinta como si hubiera datos y los paneles TOP muestran cajas vacías con título `TOP CPU (host cores) (16s old)` en lugar de degradar a `MONITOR UNAVAILABLE` (`crit`).
- **Corrección propuesta:**
  Tratar explícitamente el estado `starting` expirado antes del chequeo de `stale`:
  ```python
  if status == "starting":
      return "starting" if age <= 15 else "unavailable"
  ```

---

#### [Important] SC-02: `console_render` exige obligatoriamente un callable `vm["tr"]`, rompiendo el contrato de `ViewModel` plano
- **Archivo y línea:** [AppImage/scripts/console_render.py:173-175](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/console_render.py#L173-L175)
- **Evidencia:**
  El ADR 001 §2.4 especifica: *"console_render recibe un ViewModel plano (dicts/listas de tipos básicos)... Sin curses: puro, determinista, testeable."*
  Sin embargo, en `_label`:
  ```python
  def _label(vm, key, glyphs):
      result = vm["tr"]("runtime.consoleTui." + key)
      return str(result) if glyphs == "unicode" else sanitize(result, 200, glyphs)
  ```
  Se indexa `vm["tr"]` directamente como callable. Un fixture puramente JSON (como [fixtures/console/viewmodel_cluster.json](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/tests/fixtures/console/viewmodel_cluster.json)) no puede contener funciones. Si se pasa un `ViewModel` deserializado desde JSON a `render()`, lanza `KeyError: 'tr'`.
- **Corrección propuesta:**
  Permitir que `vm.get("tr")` sea opcional con un fallback determinista:
  ```python
  def _label(vm, key, glyphs):
      tr = vm.get("tr")
      result = tr("runtime.consoleTui." + key) if callable(tr) else key.split(".")[-1]
      return str(result) if glyphs == "unicode" else sanitize(result, 200, glyphs)
  ```

---

#### [Minor] SC-03: Documentación web y CHANGELOG omitidos respecto al plan de PR1
- **Archivo y línea:** Diff global (ausencia de `web/` y `CHANGELOG.md`)
- **Evidencia:**
  El ADR 001 §2.9 asigna a PR1 la tarea **P1-T11** (*"Documentación de uso manual: página web (JSON + page.tsx), CHANGELOG.md, README.md"*). El diff solo incluye [AppImage/README.md](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/README.md), omitiendo `web/messages/en/docs/settings/console-monitor.json`, `web/app/[locale]/docs/settings/console-monitor/page.tsx`, `CHANGELOG.md` y el `README.md` de la raíz del repo.
- **Corrección propuesta:**
  Incorporar los archivos de documentación web y changelog en PR1, o registrar explícitamente en el commit de cierre de PR1 que la documentación de Next.js se unifica en PR2.

---

#### [Minor] SC-04: Omisión del atributo `A_BOLD` en el valor numérico destacado de CPU y MEM
- **Archivo y línea:** [AppImage/scripts/console_render.py:317-326](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/console_render.py#L317-L326)
- **Evidencia:**
  ADR 001 §2.4 señala: *"`A_BOLD` sólo en el valor grande de CPU/MEM; nunca es el único portador de un significado"*. En `_cpu_memory()`, los títulos de las cajas se pasan sin atributos adicionales `("bold",)`. No introduce regresión visual grave (fuentes VGA de 512 glifos lo ignoran), pero difiere del contrato de estilo.
- **Corrección propuesta:**
  Si se desea mantener la fidelidad a la maqueta, pintar el porcentaje grande de CPU/MEM con atributo `("bold",)` cuando el ancho de celda lo permita.

---

#### [Minor] SC-05: Inconsistencia en etiqueta de Uptime en cabecera
- **Archivo y línea:** [AppImage/scripts/console_render.py:272](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/console_render.py#L272)
- **Evidencia:**
  En la cabecera de la vista Node se genera:
  `f"{label('columns.up').lower()} {_uptime((vm.get('local') or {}).get('uptime'))}"`
  El subárbol i18n provee la clave específica `labels.uptime` (`"up"`). Usar `columns.up` (`"UP"`) y forzar `.lower()` en Python puede generar anomalías en idiomas donde el encabezado de columna sea una sigla o término no transformable limpiamente a minúscula.
- **Corrección propuesta:**
  Cambiar `label('columns.up').lower()` por `label('labels.uptime')`.

---

#### [Minor] SC-06: `LocalSampler` no suministra `uptime`, delegándolo a `console_tui`
- **Archivo y línea:** [AppImage/scripts/console_metrics.py:270-272](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/console_metrics.py#L270-L272) y [AppImage/scripts/console_tui.py:238](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/console_tui.py#L238)
- **Evidencia:**
  `LocalSampler.sample()` no lee `/proc/uptime`. Es `console_tui._run()` quien llama a `_uptime(args.proc_root)` y lo inyecta en `local["uptime"]`. Si `LocalSampler` se utiliza de forma aislada, el diccionario `local` carece de `uptime`.
- **Corrección propuesta:**
  Mover la lectura de `/proc/uptime` a `LocalSampler.sample()`, donde ya se tiene acceso a `self.root`.

---

Resumen: 0 Critical, 2 Important, 4 Minor  
Recomendación: **APROBAR CON CAMBIOS**

---

## 2. Veredicto: Riesgo y Calidad de Código (Risk & Code-Quality)

### Hallazgos de Riesgo y Calidad

#### [Important] RQ-01: Indexación directa de `data["age_s"]` en chequeo de frescura
- **Archivo y línea:** [AppImage/scripts/console_tui.py:273-274](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/console_tui.py#L273-L274)
- **Evidencia:**
  En las líneas 267–274:
  ```python
  else:
      data = None
      language = _configuration_language()
  translator.set_language(language)
  resources = (data or {}).get("resources") or {}
  node = (data or {}).get("node") or {}
  fresh = (resources.get("status") == "ok" and data["age_s"] is not None
           and data["age_s"] <= 10)
  ```
  Si `data` es `None`, `data["age_s"]` no falla en la implementación actual únicamente gracias a que `resources.get("status") == "ok"` es `False` y se produce un cortocircuito booleano. Sin embargo, en las dos líneas previas el desarrollador utilizó `(data or {})`. Si en un refactor futuro se altera el orden de los operandos en `fresh` o se evalúa `data["age_s"]` independientemente, provocará un `TypeError: 'NoneType' object is not subscriptable` en el bucle principal de la TUI.
- **Corrección propuesta:**
  Utilizar acceso seguro:
  ```python
  age = (data or {}).get("age_s")
  fresh = (resources.get("status") == "ok" and age is not None and age <= 10)
  ```

---

#### [Important] RQ-02: `test_cluster_resources_cache.py` aísla funciones por AST y no cubre la inicialización del publicador en `flask_server.py`
- **Archivo y línea:** [AppImage/scripts/tests/test_cluster_resources_cache.py:17-33](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/tests/test_cluster_resources_cache.py#L17-L33) y [AppImage/scripts/flask_server.py:5923-5952](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/flask_server.py#L5923-L5952)
- **Evidencia:**
  `test_cluster_resources_cache.py` utiliza `ast.parse` para extraer exclusivamente `get_cached_pvesh_cluster_resources` y `get_cached_pvesh_cluster_resources_vm`. El bloque añadido en `flask_server.py:5923-5952` (`_console_run_pvesh`, `_console_read_language`, y la llamada `ensure_publisher(...)`) no es importado ni ejecutado en ninguna prueba automatizada de la CI (`.github/workflows/test-console-tui.yml`). Si bien está cubierto por un `try...except Exception: pass` que evita tumbar el Monitor, un error de firma en los callables no se detectaría en CI.
- **Corrección propuesta:**
  Añadir un test unitario dedicado que valide sintáctica y semánticamente `_console_run_pvesh` y `_console_read_language` con mocks de `subprocess.run` y `open`.

---

#### [Minor] RQ-03: Sustitución de `'cluster_resources_vm'` en `_pvesh_cache` sin retención de compatibilidad
- **Archivo y línea:** [AppImage/scripts/flask_server.py:1759-1763](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/flask_server.py#L1759-L1763)
- **Evidencia:**
  Se modificó `_pvesh_cache` eliminando la clave `'cluster_resources_vm'` y sustituyéndola por `'cluster_resources'`. Aunque se mantuvo `'cluster_resources_vm_time'` como marca de invalidación, cualquier código externo o prueba que leyera directamente `_pvesh_cache['cluster_resources_vm']` recibiría un `KeyError`.
- **Corrección propuesta:**
  Mantener `'cluster_resources_vm': None` en `_pvesh_cache` por retrocompatibilidad defensiva, o documentarlo explícitamente en el docstring del módulo.

---

#### [Minor] RQ-04: Nombres de nodos reales de infraestructura homelab en fixtures de pruebas
- **Archivo y línea:** [fixtures/console/pvesh_cluster_status.json:2-3](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/tests/fixtures/console/pvesh_cluster_status.json#L2-L3) y [fixtures/console/pvesh_resources_full.json:1-5](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/tests/fixtures/console/pvesh_resources_full.json#L1-L5)
- **Evidencia:**
  Los fixtures contienen referencias directas al clúster `p3` y a los nodos `p3-01`, `p3-02`. Aunque las IPs han sido debidamente neutralizadas con `.invalid`, para el mantenedor upstream (`MacRimi/ProxMenux`) es más limpio usar nomenclaturas genéricas (`pve-01`, `pve-02`, clúster `pve-cluster`).
- **Corrección propuesta:**
  Renombrar los identificadores de nodos en los fixtures de prueba a `node-01` / `node-02`.

---

#### [Minor] RQ-05: Verificación de symlink en `sysfs` dependiente de resolución relativa
- **Archivo y línea:** [AppImage/scripts/console_metrics.py:133](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/console_metrics.py#L133) y [:153](file:///Users/gggp/Code/proxmenux/ProxMenux/.claude/worktrees/console-tui/AppImage/scripts/console_metrics.py#L153)
- **Evidencia:**
  En `_network()` y `_disk()` se usa `(device / "device").exists()`. En Linux, `/sys/class/net/<nic>/device` es un symlink relativo hacia `../../../devices/...`. `Path.exists()` resuelve el destino del enlace. Si `proc_root` no es `/` (por ejemplo, en un contenedor de prueba o fixture con chroot parcial), el enlace puede quedar roto si no se recrea la jerarquía completa de `sys/devices`, haciendo que la interfaz física sea ignorada.
- **Corrección propuesta:**
  Para entornos inyectados/simulados, documentar o contemplar `(device / "device").is_symlink()` o asegurar la presencia del árbol de dispositivos en los fixtures.

---

Resumen: 0 Critical, 2 Important, 3 Minor  
Recomendación: **APROBAR CON CAMBIOS**

---

## 3. Respuestas a las Preguntas Explícitas

### 1. ¿Los cuatro módulos (`console_metrics`, `console_snapshot`, `console_render`, `console_tui`) encajan entre sí sin errores de nombre/firma?
**Sí, encajan operativamente**, pero con una asimetría en las claves del `ViewModel`:
- **Flujo de datos verificado:**
  1. `console_tui._run` instancia `LocalSampler(proc_root=args.proc_root)` y llama a `sample(time.monotonic())`.
  2. Inyecta `local["uptime"] = _uptime(args.proc_root)`.
  3. Llama a `console_snapshot.load_snapshot(args.snapshot, now)`. Retorna instancia `Snapshot` con `.data` validado.
  4. Calcula `fresh` y llama a `console_metrics.host_other(local, guests, node.get("name"), fresh)`.
  5. Empaqueta el `view_model` y llama a `console_render.render(view_model, width, height)`.
  6. Recibe la `Grid` y la pasa a `_paint(stdscr, grid, pairs)`, que traduce tokens ACS y atributos curses.
- **Discrepancias en claves del `ViewModel`:**
  - **`lang`**: La TUI pone `"lang": translator.language`, pero `console_render` nunca lee `vm["lang"]` porque obtiene los textos directamente de `vm["tr"]`.
  - **`uptime`**: `console_render` lee `vm["local"]["uptime"]`, pero `LocalSampler.sample()` no lo genera; depende exclusivamente de que `console_tui` lo añada antes del renderizado.
  - **`tr`**: `console_render` exige `vm["tr"]` (callable), lo cual excede la definición de `ViewModel` como diccionario plano JSON-serializable (ver hallazgo SC-02).
  - **`cached`**: `LocalSampler` genera `local["mem"]["cached"]`, pero `console_render` no lo dibuja en pantalla (en la vista 80×24 solo muestra `arc`, `available`, `zram` y `swap`).

### 2. En el AppImage empaquetado, ¿arranca `AppRun --tui` y encuentra los catálogos e imports?
**Sí, arranca correctamente y localiza tanto los módulos como los catálogos:**
1. **Imports bajo `python3 -I`:** El flag `-I` activa `-P`, impidiendo que CPython anteponga el directorio del script a `sys.path`. `console_tui.py` implementa en sus líneas 5–7:
   ```python
   _HERE = os.path.dirname(os.path.realpath(__file__))
   if _HERE not in sys.path:
       sys.path.insert(0, _HERE)
   ```
   Esto garantiza la resolución de `console_metrics`, `console_render`, `console_snapshot` y `zram_metrics`, todos copiados a `<AppDir>/usr/bin/` por `build_appimage.sh`.
2. **Catálogos i18n sin `APPDIR` exportado:** `AppRun` define `APPDIR="$(dirname "$(readlink -f "${0}")")"` como variable bash local sin `export`. En `console_tui.py:48`:
   ```python
   appdir = Path(os.environ.get("APPDIR") or module.parents[2])
   ```
   Como `console_tui.py` se ubica en `<AppDir>/usr/bin/console_tui.py`, `module.parents[2]` evalúa exactamente al directorio raíz del AppImage (`<AppDir>`). Por tanto, la búsqueda en `appdir / "usr/share/proxmenux/messages"` localiza los catálogos empaquetados sin depender de variables de entorno.

### 3. ¿La generalización de la caché puede cambiar algo observable para los consumidores web existentes?
**No, no produce cambios observables ni regresiones para los consumidores web:**
1. **Contenido de los datos:** En PVE (`PVE/API2/Cluster.pm`), `resources` recopila internamente la totalidad del estado RRD y de configuración del clúster; el parámetro `--type vm` solo actuaba como filtro posterior en Perl. Los diccionarios resultantes de `qemu` y `lxc` son idénticos.
2. **Filtrado:** `get_cached_pvesh_cluster_resources_vm()` filtra por `r.get('type') in ('qemu', 'lxc')` preservando el orden relativo original.
3. **Inmutabilidad y concurrencia:** Al devolver una comprensión de lista, los consumidores web reciben una lista nueva. Si un consumidor ordena o modifica la lista devuelta, no contamina la caché compartida ni afecta al publicador de la consola.
4. **Invalidación:** Se conserva el TTL de 5 s, el lock single-flight `_pvesh_cluster_resources_lock` y la política ante timeout (devolver la última caché válida o lista vacía).

### 4. ¿Algún test pasa en verde sin probar realmente el comportamiento (tautologías, mocks que reemplazan lo probado)?
**No se identifican tautologías en los tests implementados**, pero existe un falso positivo de cobertura:
- En `test_cluster_resources_cache.py`, los tests prueban rigurosamente el comportamiento de single-flight, TTL, expiración de timestamp de invalidación y filtrado.
- No obstante, para evitar dependencias del entorno Proxmox/Flask en la suite unitaria, el test compila por AST únicamente las funciones de caché aisladas. En consecuencia, el bloque de arranque real del publicador en `flask_server.py:5923-5952` (`_console_run_pvesh` y `ensure_publisher`) queda sin ejecución en la CI (aunque está protegido por `try...except`).

### 5. ¿Hay algo en el diff que el mantenedor upstream rechazaría por estilo o alcance?
1. **Nombres de nodos en fixtures:** Las referencias a `p3`, `p3-01` y `p3-02` corresponden a la infraestructura privada del autor. Deben renombrarse a `node-01`, `node-02` y `pve-cluster`.
2. **Alcance de documentación:** El ADR comprometía para PR1 la documentación web en Next.js (`web/`) y la actualización de `CHANGELOG.md`. Upstream observará la disparidad entre la documentación del AppImage y la web del proyecto si no se entrega completa.
3. **Convenciones de estilo:** Por lo demás, el código respeta escrupulosamente los estándares del repositorio upstream (Python stdlib sin librerías de terceros pesadas, curses puro sin dependencias externas, captura de errores en demonios para no interferir con el servicio principal).
