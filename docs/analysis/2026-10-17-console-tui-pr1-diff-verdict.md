---
base_sha: 227780aba76eb3f35dc50b888b3b28facd1f0c5f
head_sha: 5c7a8ab0ffd45517b834eb0969065c1fb0c30926
verdict: approved
auditor: "agy --mode plan --model gemini-3.8-flash-high (PR1 diff, round 2 scoped)"
scope: diff
kind: auditor
profile: feature
date: 2026-10-17
findings: {critical: 0, important: 0, minor: 0}
---

# Recibo de revisión — PR1 consola TUI (fase 5, diff)

## Vigente

Este recibo está **vigente** mientras `5c7a8ab0` sea el último commit de `feature/console-tui`:
`git diff --name-only 5c7a8ab0..HEAD` en esa rama debe salir vacío. Los recibos viven en la rama
`notes/console-tui` (fuera del PR upstream), así que no se cuentan a sí mismos. Cualquier commit posterior en
la rama del PR —incluido uno que corrija algo que la QA remota encuentre— lo invalida y exige otra revisión.

Pendiente aparte: las pruebas aún no se ejecutaron en un worker QA (bloqueo `web/.npmrc` en `qf-test`).

## Historia

- Ronda 1 sobre 65127237 (`2026-10-17-console-tui-pr1-diff-verdict-r1.md`): APROBAR CON CAMBIOS,
  0C/4I/7m; adjudicación: 8 aceptados (2 modificados), 3 rechazados con razón.
- Fix en 8ae1a74a + 5c7a8ab0. Ronda 2 acotada al delta (`2026-10-17-console-tui-pr1-diff-brief-r2.md`):
  APROBAR en ambos ejes, 0/0/0, los 11 hallazgos cerrados y los 3 rechazos aceptados.

## Qué este recibo NO garantiza

Integridad, no autenticidad: no prueba por sí mismo que la revisión ocurrió; la salida del Auditor va
abajo sin editar. Tampoco sustituye la ejecución de las pruebas ni la aceptación en hardware del Owner.

Salida del Auditor, ronda 2, sin editar:

# Auditoría adversarial — fase 5 (diff), ronda 2 acotada — PR1 consola TUI de ProxMenux Monitor

## 1. Seguimiento de hallazgos de la ronda 1

| Ref | Hallazgo R1 | Sev. R1 | Estado R2 | Evaluación / Detalle en una línea |
|---|---|---|---|---|
| **SC-01** | `starting` vencido cae en `stale` | Important | **cerrado** | Rama explícita en `_remote_state` retorna `"starting"` si `age <= 15` o `uptime < 120`, y `"unavailable"` si vence, sin derivar a `stale`. |
| **SC-02** | `vm["tr"]` obligatorio | Important | **cerrado** | `_label` usa `vm.get("tr")` y valida `callable()`, degradando a `full_key` sin excepción si falta el traductor. |
| **SC-03** | docs web / CHANGELOG ausentes | Minor | **cerrado** | Rechazo del Coordinador aceptado: la gobernanza upstream reserva `CHANGELOG.md`/docs web a releases del mantenedor; PR1 documenta en `AppImage/README.md`. |
| **SC-04** | sin `bold` en el valor grande | Minor | **cerrado** | Implementada superposición de atributo `("bold",)` sobre los porcentajes de CPU y memoria en la fila de títulos de caja. |
| **SC-05** | `columns.up.lower()` | Minor | **cerrado** | Sustituido por `label('labels.uptime')` directo, eliminando transformaciones de caja frágiles en internacionalización. |
| **SC-06** | `uptime` fuera de `LocalSampler` | Minor | **cerrado** | Lectura unificada en `LocalSampler.sample()` usando `self.root / "proc/uptime"`; eliminado helper redundante en `console_tui.py`. |
| **RQ-01** | `data["age_s"]` con `data=None` | Important | **cerrado** | Protegido con `age = (data or {}).get("age_s")` antes de evaluar frescura, evitando `TypeError`. |
| **RQ-02** | enganche de `flask_server.py` sin test | Important | **cerrado** | Añadido `test_console_monitor_hook.py` y registrado en la matriz de CI `.github/workflows/test-console-tui.yml`. |
| **RQ-03** | conservar `cluster_resources_vm` | Minor | **cerrado** | Rechazo del Coordinador aceptado: no existen lectores internos en `AppImage/scripts/` ni contrato de API pública externa que justifique shims. |
| **RQ-04** | nombres `p3-*` en fixtures | Minor | **cerrado** | Fixtures JSON y casos de prueba migrados a nomenclatura neutra (`pve-01`, `pve-02`, clúster `lab`). |
| **RQ-05** | symlink `device` en fixtures | Minor | **cerrado** | Rechazo del Coordinador aceptado: en pruebas basta la existencia de `(nic_dir / "device").exists()` como directorio simulado para validar el filtro. |

---

## 2. Análisis de regresiones y riesgos nuevos en el delta

### A. Nueva rama de `starting` en `_remote_state`
```python
if status == "starting":
    return ("starting" if age <= 15 or
            _number((vm.get("local") or {}).get("uptime"), 999999) < 120
            else "unavailable")
```
- **Evaluación**: La bifurcación resuelve de forma estricta la inconsistencia señalada en R1. Si el monitor publica su estado inicial (`status == "starting"`), se concede la ventana de cortesía de 15 s de antigüedad del snapshot **o** una ventana de hasta 120 s tras el arranque del host (`uptime < 120`), lo que absorbe la inicialización lenta de `pvesh` durante el arranque. Vencidas ambas condiciones, transiciona directamente a `"unavailable"` en lugar de caer por defecto en `"stale"` (el cual requiere un snapshot funcional previo). El uso de `default=999999` en `_number` ante ausencia de métricas locales previene falsos positivos en boot. No introduce regresiones.

### B. `bold` superpuesto en el título de la caja (`rfind`)
```python
cpu_percent = f"{_pct(cpu_fraction)}%"
grid.put(2 + cpu_title.rfind(cpu_percent), 1, cpu_percent, "title", ("bold",), limit=left - 1)
```
- **Evaluación**: La colocación del atributo en una segunda pasada sobreescribiendo las celdas generadas por `grid.box` es limpia dentro del modelo de búfer matricial de `Grid`. Dado que:
  1. `cpu_percent` y `memory_percent` se construyen con el sufijo literal `%`, no colisionan con prefijos ni valores intermedios (como `_gib`), y `rfind` garantiza indexar la última aparición.
  2. Las etiquetas traducidas de paneles son texto latino de ancho simple (1 columna por codepoint), por lo que el índice de caracteres coincide exactamente con la columna de pantalla.
  3. El parámetro `limit` (`left - 1` y `grid.width - 1`) previene desbordamientos si el ancho del terminal se estrechara.
  No se detectan artefactos visuales ni riesgo de desbordamiento de índice.

### C. Lectura de `/proc/uptime` dentro de `LocalSampler.sample()`
```python
try:
    uptime = max(0.0, float(_read_text(self.root / "proc/uptime").split()[0]))
except (ValueError, IndexError):
    uptime = 0.0
```
- **Evaluación**: La lectura queda encapsulada bajo `self.root`, respetando la inyección de raíz para pruebas unitarias (`test_uptime_uses_injected_root_and_falls_back_to_zero`). `_read_text` absorbe excepciones `OSError`/`FileNotFoundError` devolviendo cadena vacía (lo cual detona `IndexError` en `split()[0]`, capturado por el bloque). Formatos corruptos disparan `ValueError` y caen a `0.0`. El coste de I/O en procfs a 1 Hz es despreciable y centraliza la métrica sin fugas a `console_tui.py`.

### D. Verificación del nuevo test de enganche (`test_console_monitor_hook.py`)
- **Evaluación**: La extracción mediante `ast` es técnicamente sólida y está plenamente justificada para no importar el entorno completo de Flask/servidor en la suite de pruebas de consola:
  1. **Comportamiento dinámico**: Para las funciones auxiliares (`_console_run_pvesh` y `_console_read_language`), el test compila el AST y **ejecuta realmente su bytecode** en un espacio de nombres con mocks (`subprocess`, `open`), verificando parámetros reales de invocación (`["pvesh", "get", "/cluster/status", "--output-format", "json"]`, `timeout=10`), control de código de retorno y parseo de configuraciones válidas e inválidas.
  2. **Verificación estática del enganche**: Inspecciona el árbol sintáctico del módulo `flask_server.py` confirmando que la llamada a `ensure_publisher` ocurre a nivel de módulo, se encuentra obligatoriamente protegida dentro de un bloque `try ... except Exception:`, y enlaza exactamente los símbolos requeridos (`get_proxmox_node_name`, `get_cached_pvesh_cluster_resources`, `_console_run_pvesh`, `_console_read_language`, y `get_pve_version=get_proxmox_version`).
  El test verifica de manera determinista y real la integración sin riesgo de regresión silenciosa.

---

## 3. Veredictos

### Veredicto 1: Conformidad con la especificación (Spec-Compliance)
- **Perfil verificado:** `feature`
- **Hallazgos nuevos o abiertos:**
  - Critical: 0
  - Important: 0
  - Minor: 0
- **Resumen:** 0 Critical, 0 Important, 0 Minor
- **Decisión:** **APROBAR**

### Veredicto 2: Riesgo y calidad de código
- **Hallazgos nuevos o abiertos:**
  - Critical: 0
  - Important: 0
  - Minor: 0
- **Resumen:** 0 Critical, 0 Important, 0 Minor
- **Decisión:** **APROBAR**

---

## 4. Tabla de cierre

| Ref | Categoría | Severidad | Estado R1 | Estado R2 | Resolución final |
|---|---|---|---|---|---|
| **SC-01** | Spec | Important | Abierto | **Cerrado** | Semántica de `starting` corregida con ventana de boot y fallback estricto a `unavailable`. |
| **SC-02** | Spec | Important | Abierto | **Cerrado** | `tr` desacoplado y opcional en viewmodel; no falla ante exportaciones JSON puras. |
| **SC-03** | Spec | Minor | Rechazado | **Cerrado** | Justificación upstream aceptada; documentación de PR1 centralizada en `AppImage/README.md`. |
| **SC-04** | Calidad | Minor | Abierto | **Cerrado** | Porcentajes destacados con `bold` en la cabecera de las cajas de métricas. |
| **SC-05** | Calidad | Minor | Abierto | **Cerrado** | Clave de traducción `labels.uptime` utilizada directamente sin manipulación de casing. |
| **SC-06** | Calidad | Minor | Abierto | **Cerrado** | Métrica de `uptime` absorbida en el ciclo normal de muestreo de `LocalSampler`. |
| **RQ-01** | Calidad | Important | Abierto | **Cerrado** | Acceso seguro a `age_s` previniendo excepciones ante viewmodels nulos. |
| **RQ-02** | Calidad | Important | Abierto | **Cerrado** | Test de enganche y publisher añadido en suite unitaria y cableado en CI. |
| **RQ-03** | Calidad | Minor | Rechazado | **Cerrado** | Rechazo aceptado; código muerto eliminado sin deuda técnica de compatibilidad innecesaria. |
| **RQ-04** | Higiene | Minor | Abierto | **Cerrado** | Fixtures saneados de nombres de infraestructura de laboratorio privado. |
| **RQ-05** | Calidad | Minor | Rechazado | **Cerrado** | Rechazo aceptado; directorio mock suficiente para verificar el filtro de interfaz física. |

**Conclusión:** El delta `65127237..5c7a8ab0` subsana con precisión todos los hallazgos aceptados sin introducir regresiones funcionales ni degradación de cobertura. El PR1 se encuentra listo para integración.
