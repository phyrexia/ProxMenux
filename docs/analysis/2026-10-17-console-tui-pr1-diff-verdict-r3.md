---
base_sha: 227780aba76eb3f35dc50b888b3b28facd1f0c5f
head_sha: 731b099be2b71acbd55244b9b32d429316acaee3
verdict: pending
auditor: "agy --mode plan --model gemini-3.8-flash-high (PR1 diff, round 3 scoped to 5c7a8ab0..731b099b)"
scope: diff
kind: auditor
profile: feature
date: 2026-10-17
findings: {critical: 0, important: 2, minor: 3}
---

# Recibo de revisión — PR1 consola TUI, diff ronda 3

Delta revisado: los cuatro commits que salieron de la primera QA real (crash `group(1)`, test de NIC, datos
ausentes, formas compactas a 80×24). Veredicto `pending` (APROBAR CON CAMBIOS). Adjudicación del Coordinador:

| Hallazgo | Sev. | Decisión | Razón |
|---|---|---|---|
| 1 standalone muestra `no data` en la tira | Important | Aceptado, modificado | La afirmación de que nunca se mostró STANDALONE es inexacta (con estado `ok` la tira omite CLUSTER y DISK lleva `STANDALONE`, `console_render.py:401-426`), pero con snapshot no fresco un nodo standalone sí cae en `CLUSTER / no data`: la condición pasa a `is_cluster is False` en cualquier estado. |
| 2 `topCpuShort` sólo en `en` | Important | Rechazado | `Translator.__call__` (`console_tui.py:79-81`) resuelve por clave idioma → inglés → clave; el bot del repo rellena los demás catálogos desde `en`. |
| 3 `_fit` sin candidatos | Minor | Rechazado | Todos los call sites pasan 2–3 candidatos; defensa especulativa. |
| 4 ancho de núcleos ≥ 10 | Minor | Aceptado | `12.0c` desalinea el porcentaje. |
| 5 `_decimal` con negativos | Minor | Rechazado | Los valores llegan con `max(0, …)`. |
| (Coordinador) pie de la vista Clúster decía `Tab cluster` | Minor | Aceptado | Nueva clave `footer.switchNode`. |

Salida del Auditor, sin editar:

# Auditoría adversarial — Fase 5 (diff), Ronda 3 acotada — PR1 Consola TUI

## 1. Veredictos formales

- **Spec-compliance (`Perfil verificado: feature`):** **APROBADO CON CAMBIOS**
- **Riesgo y Calidad:** **APROBADO CON CAMBIOS**
- **Resumen:** 0 Critical, 2 Important, 3 Minor
- **Decisión final:** **APROBAR CON CAMBIOS**

---

## 2. Respuestas a las tres preguntas de la auditoría

### (1) ¿Algún cambio introduce regresión o riesgo nuevo?

#### Trampa / Caso `STANDALONE` en la tira CLUSTER
> **Aclaración crítica sobre la premisa:** La pregunta advierte *"ojo con el caso standalone, que debe seguir mostrando `STANDALONE`"*.
> 
> Tras auditar el delta y la base `5c7a8ab0`: **la TUI NUNCA mostró la palabra `STANDALONE` en la tira de clúster**.
> - En `5c7a8ab0`, un nodo standalone (`cluster = {"is_cluster": False}`) evaluaba `quorate` como `None` (falsy) y mostraba `CLUSTER · NO QUORUM 0/0` con la caja vacía.
> - Tras el commit `d3838d2d`, cuando `quorate is None`, el título se reduce a `CLUSTER` y el cuerpo imprime `no data` (en rol `dim`).
> - Por tanto, **no hay regresión sobre un comportamiento previo**, pero sí existe una **ambigüedad semántica importante**: en un nodo standalone legítimo, la tira CLUSTER muestra `no data`, dando la impresión al operador de un fallo en la recolección de métricas cuando en realidad el nodo simplemente no pertenece a un clúster. En `_top`, `standalone = not cluster.get("is_cluster", True)` sí se contempla para ocultar la columna de nodo; en `_strip` se ignoró `is_cluster` y se agrupó todo bajo `quorate is None`.

#### Helper `_fit(width, *candidates)`
- **Comportamiento:** Devuelve el primer candidato con `len(c) <= width`; si ninguno entra, devuelve `candidates[-1]`.
- **Análisis de celdas vs `len()`:** En los 4 puntos de llamada, los candidatos solo contienen caracteres ASCII, espacios, dígitos y el separador (`·` U+00B7 o `.`). En terminales estándar, `·` tiene ancho 1 celda (`wcwidth == 1`), por lo que `len(text)` coincide exactamente con el ancho visual.
- **Degradación:** En anchos extremos donde ni el último candidato entre, el caller delega en `grid.box` o `_text`, los cuales truncan con slice seguro. No provoca desbordamiento de memoria ni excepciones.

#### `_decimal` y `_short_size`
- `_decimal(value)`: Si `float(f"{value:.1f}") >= 10`, devuelve formato sin decimal (`.0f`); si no, conserva un decimal (`.1f`). Probado con `9.94` (`9.9`) y `9.96` (`10`).
- `_short_size(value)`: Trata valores `>= 1 GiB` como `...G` y `< 1 GiB` como `...M`. Si el valor es `0`, devuelve `"0"` exacto sin unidad, permitiendo que `swap 0/8.0G` ahorre 3 celdas críticas.
- **Riesgo:** Nulo para números reales no negativos. Ver punto menor sobre valores negativos abajo.

#### `no data` en paneles TOP
- Al añadir `if state == "starting" or not snapshot:`, si `snapshot` es `None` (nodo recién arrancado o colector no listo tras 180s), ambos paneles TOP imprimen limpiamente `no data` en rol `dim` en vez de quedar en blanco o con títulos huérfanos.

---

### (2) ¿Los tests nuevos prueban lo que dicen?

**Sí, con alta fidelidad y granularidad:**
1. `test_console_metrics.py`: El fix en `replace("wlan0: 10 0 0 0 0 0 0 0 20 ", ...)` restringe el reemplazo a la línea de `wlan0`, eliminando el falso positivo donde una secuencia genérica de ceros mutaba también los contadores de `eno1`.
2. `test_rate_keeps_one_decimal_only_below_ten`: Evalúa exactamente los bordes de redondeo (`9.94 -> 9.9`, `9.96 -> 10`, `8.0 MiB/s`, `42 MiB/s`, `800 Kb/s`, `3.2 Mb/s`).
3. `test_cluster_strip_*`: Tres tests independientes para:
   - `snapshot is None` -> sin QUORUM ni 0/0, cuerpo `no data`.
   - `quorate is None` -> sin QUORUM, cuerpo `no data`.
   - `quorate is False` -> mantiene `NO QUORUM` (no lo enmascara).
4. `test_top_cpu_percentages_share_one_right_edge`: Comprueba mediante `{rows[y][:40].index("%") ...}` que la columna `%` de las filas 11, 12 y 13 comparten la misma coordenada horizontal, verificando el padding `{:>3}%`.
5. `test_memory_row_without_zram_shows_only_swap`: Valida tanto `zram=None` como `capacity_bytes=0`, asegurando que no se emite una línea ZRAM vacía.

---

### (3) ¿Existe algún otro camino en el delta o su contexto que pueda lanzar una excepción y tumbar la TUI?

Tras revisar exhaustivamente cada rama modificada y sus llamadas adyacentes:
1. **El bug de `IndexError` en `_header`:**
   `re.search(r"\d+\.\d+", version)` no definía grupos de captura `()`. Invocar `match.group(1)` garantizaba un `IndexError: no such group` en cualquier snapshot con versión válida. El cambio a `match.group(0)` (match completo) es el estándar correcto en Python.
2. **Posibles divisiones por cero:**
   - En `_short_size`: divisores constantes `1048576` y `1073741824`. Imposible división por cero.
   - En `_rate`: divisores `1000` y `1024`.
3. **Conversión de tipos en `_decimal`:**
   `float(f"{value:.1f}")` nunca lanza `ValueError` para ningún float/int soportado por el formateador. Si `value` fuera `None`, `f"{None:.1f}"` lanzaría `TypeError`, pero tanto en `_short_size` (`max(0, _number(value))`) como en `_rate` (`value` proviene de delta numérico o comparado con `< base`), el valor está sanitizado numéricamente.
4. **Desempaquetado de candidatos en `_fit`:**
   Todos los call sites pasan 2 o 3 argumentos fijos, por lo que `candidates[-1]` nunca se ejecuta sobre una tupla vacía.

No se detectan nuevos caminos de excepción no controlada que puedan abortar el renderizado.

---

## 3. Hallazgos detallados

### [Important]

#### 1. Semántica de nodo standalone en `_strip` vs `quorate is None`
- **Ubicación:** `AppImage/scripts/console_render.py:428-445`
- **Descripción:** En Proxmox VE sin clúster (nodo standalone), `cluster.get("is_cluster")` es `False` y `quorate` no existe (`None`). Al entrar en la rama `if quorate is None:`, la tira muestra título `CLUSTER` y texto `no data` en gris (`dim`).
- **Impacto:** Confunde al usuario mostrando `no data` (que sugiere un error de recopilación de métricas) en lugar de reflejar el estado real de la máquina.
- **Acción recomendada:** Si `not cluster.get("is_cluster", True)`, renderizar título `label("panels.cluster")` (o `label("labels.standalone")` si existe) y en el cuerpo un texto informativo como `"standalone"` o `"single node"`, diferenciándolo del `no data` cuando el snapshot no trae información del clúster.

#### 2. Clave i18n `topCpuShort` añadida únicamente a `en/common.json`
- **Ubicación:** `AppImage/messages/en/common.json` vs `es/common.json`
- **Descripción:** El delta añade `runtime.consoleTui.panels.topCpuShort = "TOP CPU"` en el catálogo inglés. Si ProxMenux se ejecuta en español (`es`) y el resolutor de traducciones no tiene fallback estricto a inglés por clave, `label("panels.topCpuShort")` devolverá la clave cruda (`"panels.topCpuShort"`, 19 caracteres).
- **Impacto:** Si la clave cruda entra como fallback en `_fit`, ocupará 19 celdas en el título del panel TOP CPU en terminales en español.
- **Acción recomendada:** Añadir `"topCpuShort": "TOP CPU"` a `AppImage/messages/es/common.json` (y demás catálogos activos) antes de empaquetar la release.

---

### [Minor]

#### 3. `_fit` vulnerable a invocaciones sin candidatos
- **Ubicación:** `AppImage/scripts/console_render.py:95-100`
- **Descripción:** Si en futuras refactorizaciones `_fit(width, *candidates)` se invoca con `*candidates` vacío, `candidates[-1]` lanzará `IndexError`.
- **Acción recomendada:** Firma más robusta: `def _fit(width, default, *candidates):` o salvaguarda `return candidates[-1] if candidates else ""` por programación defensiva.

#### 4. Alineación horizontal de cores >= 10 en `_top_row`
- **Ubicación:** `AppImage/scripts/console_render.py:476-480`
- **Descripción:** `value = f"{_cpu_key(guest):.1f}c  {_pct(guest.get('cpu')):>3}%"`. El porcentaje tiene ancho 3 (`:>3%`), pero los cores (`{_cpu_key(guest):.1f}c`) no tienen ancho mínimo reservado. Si un guest consume `12.0c` (5 caracteres) y otro `1.2c` (4 caracteres), el porcentaje del primero se desplazará una posición a la derecha.
- **Acción recomendada:** Utilizar `f"{_cpu_key(guest):>4.1f}c  {_pct(guest.get('cpu')):>3}%"` para fijar la columna de cores hasta 99.9c.

#### 5. `_decimal` no contempla valores negativos
- **Ubicación:** `AppImage/scripts/console_render.py:82-85`
- **Descripción:** `return f"{value:.0f}" if float(text) >= 10 else text`. Si `value` fuera `-15.0`, `float("-15.0") >= 10` es `False`, devolviendo `"-15.0"` con decimal, mientras que `+15.0` devuelve `"15"`.
- **Acción recomendada:** Usar `abs(float(text)) >= 10` para consistencia matemática en el helper.

---

## 4. Conclusión

El delta resuelve de manera sólida el `IndexError` inadvertido del regex, corrige el flake en tests de métricas y logra un empaquetado de texto estético y limpio en 80×24 sin truncamientos abruptos. Los dos puntos importantes señalados (reflejar la clave en el catálogo `es` y dar un mensaje apropiado a nodos standalone en lugar de `no data`) son ajustes de acabado que no comprometen la estabilidad del ejecutable. 

Se dictamina **APROBAR CON CAMBIOS** para incorporar las traducciones y el tratamiento de standalone.
