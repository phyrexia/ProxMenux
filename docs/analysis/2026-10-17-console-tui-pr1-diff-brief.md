# Auditoría adversarial — fase 5 (diff) — PR1 consola TUI de ProxMenux Monitor

> **Modo de respuesta:** no uses ninguna herramienta, no leas ni escribas archivos. Todo el material está
> inline abajo. Escribe tu veredicto completo directamente como texto de respuesta (Markdown).

Eres el Auditor (solo lectura, independiente de quien implementó). Revisa el **diff completo del PR1**
contra el ADR 001 (revisión 2, `accepted` tras dos rondas de auditoría de spec). PR1 = TUI de consola
utilizable manualmente por root (`proxmenux-console` / `AppRun --tui`), sin usuario de sistema ni drop-in
de getty (eso es PR2, fuera de este diff). Repositorio upstream: `MacRimi/ProxMenux` (rama base `develop`,
`227780ab`); el PR se abre desde un fork. Perfil declarado: `feature`.

Contexto de verificación: las pruebas **todavía no se han ejecutado** (se ejecutan en workers QA remotos,
pendiente). Por eso tu revisión debe ser especialmente atenta a errores que un test no ejecutado no habría
detectado: imports, nombres, firmas entre módulos, rutas empaquetadas, casos borde y tests que no prueban
lo que dicen.

## Lo que se te pide

Dos veredictos separados:
1. **Spec-compliance**: ¿el diff implementa lo que el ADR fija para PR1 (§2.2–2.4, §2.6–2.9 en su parte PR1,
   §8)? Incluye la línea `Perfil verificado: feature`. Señala desviaciones del contrato (`console.json`
   schema 1, ViewModel, teclas, estados `starting/ok/stale/unavailable/error`, lease, permisos, `host other`,
   TOP con `mem > 0` y `cpu×maxcpu > 0`, glifos, saneamiento, repintado 30 s, `sys.path` bajo `-I`).
2. **Riesgo / code-quality**: bugs, regresiones en el Monitor existente (en particular la caché
   `get_cached_pvesh_cluster_resources*` y el arranque del publicador en `flask_server.py`, que no debe poder
   tumbar el Monitor), seguridad (cadenas no confiables, rutas, permisos de `/run`, lease), robustez de la
   TUI en consola Linux (`TERM=linux`, ACS, locale), corrección de los tests y de la CI, estilo frente al repo.

Cada veredicto con hallazgos Critical / Important / Minor, cada uno con archivo y línea del diff, evidencia y
corrección propuesta; una línea `Resumen: <n> Critical, <n> Important, <n> Minor` y una recomendación
`APROBAR | APROBAR CON CAMBIOS | RECHAZAR`.

Preguntas explícitas:
1. ¿Los cuatro módulos (`console_metrics`, `console_snapshot`, `console_render`, `console_tui`) encajan entre sí
   sin errores de nombre/firma? Recorre el camino de datos de `console_tui._run` → `LocalSampler.sample` →
   `load_snapshot` → `host_other` → `render` → `_paint` y señala cualquier clave del ViewModel que el render
   lea y la TUI no ponga, o viceversa.
2. En el AppImage empaquetado (`<AppDir>/usr/bin/console_*.py`, catálogos en
   `<AppDir>/usr/share/proxmenux/messages`, `AppRun` no exporta `APPDIR`), ¿arranca `AppRun --tui` y encuentra
   los catálogos e imports?
3. ¿La generalización de la caché puede cambiar algo observable para los consumidores web existentes?
4. ¿Algún test pasa en verde sin probar realmente el comportamiento (tautologías, mocks que reemplazan lo
   probado)?
5. ¿Hay algo en el diff que el mantenedor upstream rechazaría por estilo o alcance (referencias internas,
   comentarios, archivos fuera de convención)?

---

## (A) ADR 001 — texto íntegro (revisión 2, accepted)

# ADR 001 — Consola TUI de ProxMenux Monitor en tty1 (HDMI/VGA)

- Estado: **accepted** (2026-10-17; auditoría adversarial aprobada en ronda 2 — recibo `.notes/docs/analysis/2026-10-17-console-tui-redteam-verdict.md`)
- Fecha: 2026-10-17
- Revisión: 2 (2026-10-17, tras auditoría ronda 1; registro de adjudicación en §8)
- Autor del diseño: Designer (Fable). Implementación autorizada por el Owner el 2026-10-16 17:48.
- Entrega: **dos PRs upstream secuenciales** (decisión del Coordinador tras la ronda 1): PR1 = TUI utilizable manualmente por root (`proxmenux-console`), PR2 = autoarranque en tty1. Detalle en §2.9.
- Grill: `.notes/docs/specs/2026-10-17-console-tui-grill.md` (requisitos confirmados, restricciones, preguntas abiertas con defaults).
- Base: `origin/develop` 227780ab, worktree `feature/console-tui`.

Convenciones del documento: rutas, identificadores, claves y fragmentos de código en inglés tal como irán al repo; texto en español. Las cifras y rutas citadas del repo fueron verificadas en el worktree el 2026-10-17.

## 1. Contexto

ProxMenux Monitor ya recolecta casi todo lo que la consola necesita: `pvesh get /cluster/resources` cacheado 5 s con single-flight (`AppImage/scripts/flask_server.py:1766`, `:2759-2800`), nombre del nodo por `/etc/pve/local` (`:122-154`), hilos de muestreo daemon (`_ensure_net_rate_sampler`, `:5891-5897`; `_ensure_vm_disk_refresher`, `:2499-2508`) y un lector de zram inyectable (`AppImage/scripts/zram_metrics.py`). Corre como root bajo `proxmenux-monitor.service` con `Restart=on-failure` (`install_proxmenux.sh:625-640`).

La salida de video del nodo la gobierna `getty@tty1.service` (systemd 252 en PVE 8, 257.13 en PVE 9) con `Restart=always`, `RestartSec=0`, `Type=idle`, `TTYVTDisallocate=yes` y `UnsetEnvironment=LANG LANGUAGE LC_*`. El `ExecStart` del fabricante es `-/sbin/agetty -o '-- \\u' --noreset --noclear - ${TERM}` en 257 y `-/sbin/agetty -o '-p -- \\u' --noclear - $TERM` en 252.

El host aporta `python3` (3.11/3.13) con `_curses` en `libpython3.x-stdlib` y el terminfo `linux` en `ncurses-base`; `AppRun` usa ese intérprete y sólo añade `dist-packages` empaquetados. La consola renderiza con fuentes de 256/512 glifos (CP437 del kernel o fontsets de console-setup) que carecen de braille y de bloques parciales; en modo texto VGA una fuente de 512 glifos deja 8 colores.

El Owner validó las maquetas: la vista **A** (nodo primero, selector Node/Cluster, con listas de los guests que más CPU y memoria consumen) es la principal; la vista **B** (clúster, una fila compacta por nodo) es la secundaria; la variante densa C queda descartada. La TUI debe arrancar sola, salir al login al cerrarla y poder relanzarse.

## 2. Decisión

### 2.1 Resumen

Una TUI propia en Python `curses` (stdlib del host, sin dependencias nuevas) empaquetada en el AppImage del Monitor y lanzada con `AppRun --tui`. Lee **métricas locales directamente de `/proc` y `/sys`** cada 1 s, y lee de un **snapshot JSON** (`/run/proxmenux-monitor/console.json`) publicado por el servicio Monitor lo que exige `pvesh` (clúster, guests, almacenamiento PVE, versión); el publicador reutiliza la **única** caché single-flight de `pvesh get /cluster/resources` del Monitor (generalizada a todos los tipos), de modo que dashboard y consola comparten una llamada cada 5 s. **PR1**: el comando `proxmenux-console` la ejecuta como el root que lo invoca (sólo lectura; el invocante ya es root autenticado). **PR2**: en `tty1` la lanza **una vez por arranque** un wrapper root del `getty@tty1` (drop-in de dos líneas), como **usuario de sistema sin privilegios**; al salir, el wrapper hace un reset del VT (`ESC c`) y ejecuta el `agetty` del fabricante. Todo es sólo lectura y toda la configuración de arranque es root-owned.

### 2.2 Modelo de procesos y privilegios

| Pieza | Corre como | Qué hace | Qué no hace |
|---|---|---|---|
| **Publicador** (`console_snapshot.py`, hilo daemon dentro de `flask_server.py`; PR1) | root (proceso del Monitor) | Al arrancar escribe de inmediato un snapshot con `resources.status = "starting"`. Después, sólo si existe un *lease* reciente, cada 5 s toma `/cluster/resources` (todos los tipos) de la caché single-flight compartida con el dashboard (`get_cached_pvesh_cluster_resources()`, §2.7) y cada 10 s llama `pvesh get /cluster/status --output-format json`; normaliza y escribe `console.json` de forma atómica. En cuanto aparece un lease, publica sin esperar al siguiente tick. | No expone endpoints HTTP, no escribe credenciales, no toca `/etc/pve`, no lanza un segundo `pvesh` de recursos. |
| **TUI** (`console_tui.py` + `console_metrics.py` + `console_render.py`; PR1) | PR1: el root que invoca `proxmenux-console`. PR2 en tty1: `proxmenux-console` (system user, `nologin`, sin home, sin sudo, no en `www-data`, **no en `tty`**) | Lee `/proc`, `/sys` y `console.json`; renueva el lease; pinta con curses sobre los fd 0/1/2 heredados; responde al teclado. | No ejecuta comandos externos (ni `pvesh`, ni `sensors`, ni `systemctl`), no escribe fuera de `/run/proxmenux-console/client/`, no importa `flask_server.py`. |
| **Wrapper getty** (`/usr/local/sbin/proxmenux-console-getty`; PR2) | root (contexto de `getty@tty1`) | Decide por boot si lanza la TUI, la lanza con `setpriv`, resetea el VT y después hace `exec` de `agetty`. | No hace `chvt`, no cambia permisos del tty, no reinicia nada. |
| **Lanzador manual** (`/usr/local/bin/proxmenux-console`) | quien lo invoque | PR1: exige root y ejecuta `AppRun --tui` directamente. PR2: si existe el usuario `proxmenux-console`, root baja a él con `setpriv`; un miembro del grupo ejecuta directo; otro usuario recibe un mensaje. Vuelve al shell al salir. | No toca el marcador por boot. |

**Por qué el PR1 corre como root sin `setpriv`.** El invocante ya es root en una sesión autenticada (SSH u otra TTY); la TUI es sólo lectura, no ejecuta comandos ni escribe fuera de `/run/proxmenux-console/client/`, y su entrada es el teclado del mismo root. Crear el usuario de sistema sólo tiene sentido cuando la TUI se muestra **antes** del login (PR2). En PR1 `console.json` queda `0600 root:root` en un directorio `0700`; el publicador ya incorpora la lógica de grupo (si `grp.getgrnam('proxmenux-console')` existe → `0750`/`0640` con ese gid), latente hasta que PR2 cree el grupo.

**Local desde `/proc` y no desde el snapshot — justificación.** (a) El Owner mira la consola justo después de un reinicio, cuando el Monitor arranca, se actualiza o está en gracia (`startup_grace.py`: 180–300 s); la consola debe mostrar el nodo aunque el Monitor esté parado o desactivado desde `config_menu.sh:403-440`. (b) Cadencia de 1 s sin IPC ni carga extra en el proceso gevent del Monitor. (c) Todo lo necesario es legible sin privilegios: `/proc/stat`, `/proc/meminfo`, `/proc/loadavg`, `/proc/uptime`, `/proc/net/dev`, `/proc/diskstats`, `/sys/class/net/*/{operstate,speed,device}`, `/sys/class/hwmon/*/{name,temp*_input,temp*_label}`, `/sys/class/thermal/thermal_zone*/temp`, `/proc/spl/kstat/zfs/arcstats`, `/sys/block/zram*/{disksize,mm_stat,comp_algorithm}` y `/proc/swaps` (se reutiliza `zram_metrics.read_zram_metrics`). (d) Es testeable con fixtures por inyección de rutas, como ya hace `test_zram_metrics.py`.

**Comportamiento ante fallos.**

| Situación | Local (A, columna izquierda y strip) | Clúster / guests / storage PVE |
|---|---|---|
| **Arranque del nodo** (ventana de boot) | En vivo desde el primer frame | Estado `STARTING` (rol `warn`, no `crit`): se muestra `Monitor starting…` en la cabecera y los paneles remotos en `dim` con `waiting for data`. La TUI lo decide sin privilegios y sin subprocesos: (i) `console.json` ausente **y** `/proc/uptime` < 120 s → `STARTING`; (ii) `console.json` presente con `resources.status = "starting"` (el publicador lo escribe nada más arrancar) y edad ≤ 15 s → `STARTING`; (iii) `console.json` ausente y uptime ≥ 120 s → `MONITOR UNAVAILABLE` (`crit`). El publicador emite el primer snapshot real en cuanto ve el lease (sin esperar al tick de 5 s), así que el tiempo típico hasta datos de clúster es ≤ 2 s de sondeo + 1 `pvesh`. No se consulta `systemctl`: la TUI no ejecuta comandos, y el estado `starting` del propio snapshot distingue "Monitor arrancado pero sin primer `pvesh`" de "Monitor caído", cosa que `ActiveState` no sabe. |
| Monitor parado o deshabilitado (fuera de la ventana de boot) | En vivo | `console.json` ausente o con `observed_at` > 60 s → cabecera `MONITOR UNAVAILABLE` en rojo; los últimos valores quedan atenuados con su edad (`data 3m old`); si nunca hubo archivo y uptime ≥ 120 s, paneles con `no data` |
| `pvesh` falla o tarda > 10 s | En vivo | `resources.status = "stale"` con `error`; tras 3 fallos seguidos `"error"`; valores previos atenuados con edad |
| Sin quorum | En vivo | `cluster.quorate = false` → `NO QUORUM` rojo; nodos `online=false` en rojo; guests de nodos offline aparecen con `status` tal cual lo da PVE (normalmente `unknown`) y sin métricas; **no se calcula total de clúster**, se muestra `coverage 1/2 nodes` |
| Nodo remoto inalcanzable con quorum | En vivo | Fila del nodo `offline`; agregados sólo de los online y marcados con cobertura |
| Standalone (sin `corosync.conf`) | En vivo | `cluster.is_cluster = false` → cabecera `STANDALONE`; la pestaña **Cluster** muestra la vista B con un solo nodo y todos los guests (sigue siendo útil como tabla) |
| Terminal < 80×24 | — | Sólo el mensaje `Terminal too small (80x24)`; `q` sigue funcionando |

### 2.3 Contrato de `console.json`

- Ruta: `/run/proxmenux-monitor/console.json`. Directorio `/run/proxmenux-monitor` creado por el publicador: si el grupo `proxmenux-console` existe (PR2), `root:proxmenux-console 0750` y archivo `0640 root:proxmenux-console`; si no (PR1), `0700` y `0600 root:root`. La comprobación del grupo se repite en cada escritura (barata) para que la instalación de PR2 surta efecto sin reiniciar el Monitor.
- Escritura atómica: `tempfile.NamedTemporaryFile(dir=same_dir, delete=False)` → `json.dump` → `os.fchmod` → `os.fchown` (si hay grupo) → `os.replace`. `/run` es tmpfs: sin `fsync`.
- Arranque: `ensure_publisher()` escribe de inmediato (antes de cualquier lease) un snapshot con `cluster.status = resources.status = "starting"`, listas vacías y `node` rellenado; así la TUI distingue "Monitor arrancando" de "Monitor caído" (§2.2).
- Intervalo: 5 s mientras `/run/proxmenux-console/client/lease` tenga `mtime` < 15 s; en reposo, un `os.lstat()` cada 2 s y nada más (`stat.S_ISREG` obligatorio: un symlink o un directorio en su lugar cuenta como "sin lease"). Al detectar un lease nuevo, publica **inmediatamente** y después cada 5 s. Un lease renovado sin fin por el usuario sin privilegios sólo mantiene el régimen normal (1 llamada compartida / 5 s); se acepta.
- Tamaño: ≤ 1 MiB; `guests` se trunca a 2000 (orden: `cpu*maxcpu` desc, luego `mem` desc) y se marca `truncated: true`.
- Codificación: UTF-8, `ensure_ascii=False`; las cadenas se sanean en el **lector** (la TUI), nunca se confía en el archivo.

Esquema (`schema: 1`). Tipos JSON; `number|null` cuando PVE puede omitir el campo.

```jsonc
{
  "schema": 1,
  "generator": { "name": "proxmenux-monitor", "version": "1.2.6.2-beta", "pid": 1234 },
  "observed_at": 1760700000.123,        // time.time() del publicador en el momento de escribir
  "publish_interval_s": 5,
  "language": "es",                     // .language de /usr/local/share/proxmenux/config.json, "en" si falta
  "node": {
    "status": "ok",                     // ok | error
    "observed_at": 1760700000.1,
    "name": "p3-01",                    // get_proxmox_node_name() (symlink /etc/pve/local, fallback hostname)
    "pve_version": "pve-manager/9.0.3/…", // salida de `pveversion`, cacheada 300 s; null si falla
    "error": null
  },
  "cluster": {
    "status": "ok",                     // ok | stale | unavailable | error
    "observed_at": 1760699995.0,        // instante de la última llamada exitosa a /cluster/status
    "is_cluster": true,                 // existe /etc/pve/corosync.conf y /cluster/status trae type=cluster
    "name": "p3",                       // [cluster].name; null en standalone
    "quorate": true,                    // [cluster].quorate; null si desconocido
    "nodes_total": 2,                   // [cluster].nodes (incluye offline)
    "nodes_online": 2,                  // cuenta de members.online
    "config_version": 7,                // [cluster].version
    "members": [
      { "name": "p3-01", "nodeid": 1, "online": true, "local": true,  "ip": "…", "level": "" },
      { "name": "p3-02", "nodeid": 2, "online": true, "local": false, "ip": "…", "level": "" }
    ],
    "error": null
  },
  "resources": {
    "status": "ok",                     // ok | stale | unavailable | error
    "observed_at": 1760700000.0,        // instante de la última llamada exitosa a /cluster/resources
    "truncated": false,
    "nodes": [                          // type == "node"
      { "name": "p3-01", "status": "online", "cpu": 0.07, "maxcpu": 16,
        "mem": 1717986918, "maxmem": 34359738368,        // bytes; en ZFS `mem` incluye ARC
        "disk": 21474836480, "maxdisk": 53687091200,     // rootfs, bytes
        "uptime": 612000, "level": "", "cgroup_mode": 2 }
    ],
    "guests": [                         // type in ("qemu","lxc")
      { "vmid": 101, "type": "qemu", "name": "plex", "node": "p3-01", "status": "running",
        "cpu": 0.60, "maxcpu": 2,                        // cpu = fracción de maxcpu (0..1)
        "mem": 4294967296, "maxmem": 8589934592,         // bytes
        "disk": 0, "maxdisk": 34359738368,               // bytes (disk suele ser 0 sin guest agent)
        "uptime": 3600,
        "netin": 123456789, "netout": 98765432,          // bytes acumulados desde el arranque del guest
        "diskread": 0, "diskwrite": 0,                   // ídem; null si PVE no lo da
        "tags": ["media", "prod"],                       // split de `tags` por ';'
        "hastate": null, "template": false, "lock": null }
    ],
    "storages": [                       // type == "storage"
      { "id": "storage/p3-01/local-zfs", "storage": "local-zfs", "node": "p3-01", "plugintype": "zfspool",
        "shared": false, "status": "available", "disk": 21474836480, "maxdisk": 53687091200,
        "content": ["images", "rootdir"] }
    ],
    "error": null
  }
}
```

Semántica de `status` por fuente: `starting` = el publicador arrancó y aún no ha intentado ninguna llamada (snapshot inicial; listas vacías); `ok` = última llamada exitosa dentro de 2× su intervalo; `stale` = hubo éxito antes pero la última falló (se conservan datos); `unavailable` = se intentó y nunca hubo éxito desde que arrancó el publicador (listas vacías); `error` = 3 fallos consecutivos (listas conservadas, `error` con el mensaje recortado a 200 caracteres). `observed_at` del documento es el instante de escritura; los `observed_at` por fuente son los de su último éxito, de modo que la TUI muestre la edad real del dato remoto y no la del archivo.

Tasas por guest (red, disco) **no** viajan en el snapshot: la TUI las deriva entre dos snapshots consecutivos como `(counter_now - counter_prev) / (observed_at_now - observed_at_prev)` sólo si el guest sigue en el mismo nodo, `uptime` no decreció y el contador no decreció; en otro caso reinicia la base. Se muestran únicamente a ≥ 160 columnas.

Lector (`console_snapshot.load_snapshot(path, now) -> Snapshot`): valida `schema == 1`, tipos y rangos (negativos → `null`), limita longitudes (`name` ≤ 64, `tags` ≤ 16 × 32), y devuelve además `age_s = now - observed_at` (negativo → 0). JSON inválido → `Snapshot.unavailable(reason)`; la TUI nunca cae por el archivo.

### 2.4 Especificación de pantalla

#### Modelo de vista

`console_render` recibe un `ViewModel` plano (dicts/listas de tipos básicos) con: `local` (cpu_total, cpu_history[≤60], per_core[], load1/5/15, threads, temp_c|None, mem{total,used,available,cached,arc,swap_used,swap_total}, zram{…}|None, net{rx_bps,tx_bps,nic,speed_mbps}, disk{read_bps,write_bps}, storages_local[]), `snapshot` (tal cual §2.3 más `age_s`), `glyphs` (`acs|unicode|ascii`), `lang`, `now`, `selected_view` (`node|cluster`), `sort` (`cpu|mem`). Devuelve una `Grid` de `height × width` celdas `(char, color_role, attrs)`. Sin curses: puro, determinista, testeable.

#### Juego de glifos

| Elemento | `acs` (default en consola) | `unicode` (`GLYPHS=unicode`) | `ascii` |
|---|---|---|---|
| Caja | `ACS_HLINE VLINE ULCORNER URCORNER LLCORNER LRCORNER LTEE RTEE TTEE BTEE PLUS` (terminfo `linux` los mapea a la G1 del kernel; es lo que ya usa `dialog` en estos hosts) | U+2500 2502 250C 2510 2514 2518 251C 2524 252C 2534 253C | `- \| + + + + + + + + +` |
| Barra llena / vacía | `ACS_BLOCK` (█) / `ACS_CKBOARD` (▒) | U+2588 / U+2591 | `#` / `.` |
| Historial de CPU | columnas de `ACS_BLOCK`; alto 2 filas (3 niveles) a < 40 líneas, 4 filas (5 niveles) a ≥ 40 | añade U+2584 (▄) para doblar la resolución vertical | `#`/`.` por filas |
| Separador | `ACS_BULLET` (·) | U+00B7 | `.` |
| Grado | `o` tras el número (`48 C`) | `°` (U+00B0, en CP437 y Lat15) | `C` |

Verificado: los 11 puntos de caja y 2588/2591/2592 están en `linux.set` de console-setup y en CP437; 2584/2580/00B0/00B7 están en CP437 y `useful.set` (no garantizados en todos los fontsets, por eso sólo en modo `unicode`). Nada de braille ni U+2581–2587. `a` alterna `acs`→`ascii`→`unicode`.

#### Colores (8 + atributos)

| Rol | Pair (fg sobre fondo por defecto) | Uso |
|---|---|---|
| `title` | cyan | títulos de panel, nombre del nodo, encabezados de tabla |
| `text` | default | valores |
| `dim` | default + `A_DIM` | datos stale, ayuda |
| `ok` | green | barras < 70 %, `online`, `running`, `QUORUM OK` |
| `warn` | yellow | barras 70–90 %, `stale`, `unknown` |
| `crit` | red | barras ≥ 90 %, `offline`, `NO QUORUM`, `MONITOR UNAVAILABLE` |
| `selected` | default + `A_REVERSE` | pestaña activa, línea de cabecera |

`A_BOLD` sólo en el valor grande de CPU/MEM; nunca es el único portador de un significado (fuentes de 512 glifos lo pierden en modo texto).

#### Vista A — Node (principal)

80×24 (filas 0–23):

```
 PROXMENUX MONITOR  [Node] Cluster   p3-01 · PVE 9.0 · up 7d 04h          12:34:56
┌ CPU  7% ───────────────────────────────┐┌ MEMORY  1.6/32 GiB  5% ─────────────┐
│ ███▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒ ││ ██▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒ │
│ load 0.42 0.51 0.48 · 16 thr · 48 C     ││ ARC 6.1 GiB · avail 29 GiB            │
│ ▒▒█▒▒▒█▒▒▒▒▒█▒▒▒█▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒ ││ ZRAM 0.14/3.2 GiB 1.8x · swap 0/8 GiB│
└─────────────────────────────────────────┘└───────────────────────────────────────┘
┌ NET eno1 2.5G ─────┐┌ DISK ────────────────────┐┌ CLUSTER p3 · QUORUM 2/2 ──────┐
│ RX 3.2 Mb/s        ││ R 42 MiB/s  W 8 MiB/s    ││ p3-01*  7%  5% online         │
│ TX 0.8 Mb/s        ││ local-zfs ████▒▒▒▒▒ 39%  ││ p3-02   9%  6% online         │
└────────────────────┘└──────────────────────────┘└───────────────────────────────┘
┌ TOP CPU (host cores) ─────────────────┐┌ TOP MEMORY ───────────────────────────┐
│ VM 101 plex         p3-01  1.2c  60%  ││ VM 103 db           p3-02   6.0 GiB   │
│ CT 205 nginx        p3-02  0.3c  30%  ││ VM 101 plex         p3-01   4.0 GiB   │
│ VM 110 win          p3-01  0.2c   5%  ││ CT 205 nginx        p3-02   0.8 GiB   │
│ …                                      ││ …                                     │
└───────────────────────────────────────┘└───────────────────────────────────────┘
 q login · Tab cluster · r refresh · a glyphs · Ctrl+Alt+F2 other TTY     data 2s
```

Reparto vertical a 24 filas: cabecera 1; CPU/MEMORY 5 (3 interiores: barra, texto, historial de 1 fila); strip NET/DISK/CLUSTER 4 (2 interiores); TOP 12 (10 filas de guests); pie 1. Regla general: `top_rows = height - 1 - cpu_mem_rows - strip_rows - 1`, con `top_rows ≥ 5` (3 guests) garantizado.

Crecimiento con la altura (`height ≥ 40`): CPU/MEMORY pasan a 8 filas (historial de 4 filas, 5 niveles); con `width ≥ 120` el panel CPU muestra además barras por núcleo en columnas (`CPU0 3% ██▒▒`, hasta 64 hilos; más allá, se agrupan de 2 en 2 y se indica `x2`). El strip pasa a 5 filas (storage local lista hasta 3 entradas; CLUSTER hasta 3 nodos y `+N nodes`). TOP ocupa el resto.

Crecimiento con la anchura: a `width ≥ 160` las listas TOP añaden columnas `UP` (uptime) y, con dos snapshots, `NET` (`rx/tx KiB/s`) y `DISK` (`r/w`); los nombres de guest reciben hasta 24 celdas. A 240×67: CPU/MEMORY 8 filas, strip 5, TOP 52 filas (50 guests por lista — en la práctica todos los guests de un homelab).

Colapso cuando falta espacio (de primero a último en desaparecer): (1) historial de CPU reducido a 1 fila; (2) barras por núcleo; (3) tercera entrada de storage y nodos extra del strip (quedan `+N`); (4) columnas extra de TOP; (5) panel CLUSTER del strip en standalone se sustituye por `STANDALONE` dentro de DISK ensanchado; (6) nombres truncados con `…` hasta un mínimo de 8 celdas. Cabecera, pie, CPU, MEMORY y las dos listas TOP (≥ 3 filas) nunca desaparecen por encima de 80×24.

Semántica de las listas TOP: se excluyen plantillas (`template = true`). **TOP CPU** incluye guests con `cpu × maxcpu > 0`, ordena por ese producto (núcleos de host consumidos) y muestra `1.2c` y `60 %` (de lo asignado). **TOP MEMORY** incluye todo guest con `mem > 0` **sea cual sea su `status`** (en `/cluster/resources` una VM pausada sigue como `running`, y un guest en migración o con `lock` retiene memoria), ordena por `mem` y muestra GiB con una decimal (`MiB` si < 1 GiB); un `status` distinto de `running` se muestra en la fila. En clúster se listan guests de todos los nodos con la columna `node`; en standalone la columna `node` se omite y el espacio va al nombre. Si el snapshot está stale, las listas se pintan en `dim` y el título lleva `(2m old)`.

**Indicador de consumo del host fuera de los guests** (a petición de la auditoría: un nodo saturado por `vzdump`, un scrub de ZFS o la replicación mostraría barras rojas y listas TOP tranquilas). Se calcula localmente y se muestra en el título de cada lista TOP, sólo para el nodo local: `TOP CPU (host cores) · host other 0.4c` donde `host other = cpu_total_local × threads − Σ(cpu × maxcpu) de los guests running del nodo local`, acotado a ≥ 0; y `TOP MEMORY · host other 2.1 GiB` donde `host other = mem.used − mem.arc − Σ mem de los guests del nodo local`, acotado a ≥ 0 (el ARC ya se muestra aparte en MEMORY). Si no hay snapshot fresco, el indicador se omite (no se puede restar lo que no se conoce). Una **lista de procesos del host queda fuera de alcance**: el Owner pidió VM/CT; leer `/proc/*/stat` de ~1000 PIDs cada segundo es la parte cara de bashtop; y mostrar nombres de procesos root antes del login amplía la exposición. Si el indicador demuestra que hace falta, será otro ADR (tecla `p` reservada).

Cabecera: título, pestañas (`[Node]` en `selected`), nombre del nodo, `PVE x.y` (del snapshot, `n/d` si no hay), uptime local, reloj local a la derecha. Durante la ventana de boot (§2.2) la mitad derecha muestra `Monitor starting…` en `warn`; si el snapshot está en estado crítico, `MONITOR UNAVAILABLE (3m)` en `crit`.

**Repintado completo periódico.** `tty1` es destino de `console=tty0`: cualquier `printk` (udev, USB, ZFS, red) escribe directamente en el VT y curses, que sólo repinta diferencias, no lo borraría. El bucle de la TUI fuerza `stdscr.clearok(True)` + `refresh()` **cada 30 s**, además de tras `KEY_RESIZE` y con `r`. Coste: un frame completo de 240×67 ≈ 16 k celdas, del orden de milisegundos cada medio minuto; la ventana máxima de basura visible es 30 s, por debajo del tiempo que tarda un operador en acercarse a la pantalla. No se cambia el `loglevel` del kernel ni se redirige la consola: la TUI no altera estado del sistema.

Pie: teclas en el idioma configurado; a la derecha `data Ns` (edad del snapshot) o `no data`.

#### Vista B — Cluster

80×24:

```
 PROXMENUX MONITOR  Node [Cluster]   p3 · QUORUM OK 2/2 · 64 GiB · CPU 8%  12:34:56
 NODE      STATUS   CPU  ████▒▒▒▒   MEM             ████▒▒▒▒   ROOT  GUESTS   UP
 p3-01*    online    7%  █▒▒▒▒▒▒▒   1.6/32 GiB      █▒▒▒▒▒▒▒   39%   4/5     7d
 p3-02     online    9%  █▒▒▒▒▒▒▒   1.8/32 GiB      █▒▒▒▒▒▒▒   44%   3/3     7d
─ GUESTS  running 7 · stopped 2 · sort: cpu (c/m) ────────────────────────────────
 TYPE  ID   NAME               NODE    STATUS    CPU          MEM
 VM    101  plex               p3-01   running   1.2c  60%    4.0 GiB
 CT    205  nginx              p3-02   running   0.3c  30%    0.8 GiB
 …
 q login · Tab node · c/m sort · r refresh · a glyphs                     data 2s
```

Filas: cabecera 1; tabla de nodos `2 + n_nodes` (máx. `height/3`, resto `+N nodes`); separador 1; tabla de guests el resto; pie 1. Orden de nodos: offline primero, luego por nombre. Guests: `running` primero ordenados por `cpu × maxcpu` (`c`) o `mem` (`m`), después `stopped` en `dim`; los que no quepan, `+N guests`. Columna `ROOT` = `disk/maxdisk` del nodo; `GUESTS` = `running/total` del nodo. A `width ≥ 160`: columnas `CORES` (`maxcpu`), `UPTIME` del guest, `HA`, `TAGS` (hasta 3, saneados). Cabecera: `QUORUM OK n/m` (`ok`) o `NO QUORUM n/m` (`crit`); el total de RAM y la media de CPU **sólo** cuando `quorate` y todos los nodos online tienen datos frescos; en otro caso `coverage k/m`.

#### Teclas

| Tecla | Acción |
|---|---|
| `q`, `Q`, `Ctrl+C` (SIGINT), SIGTERM, SIGHUP | salir limpiamente por el mismo camino: los handlers sólo activan una bandera; el bucle sale, `curses.wrapper` ejecuta `endwin()` (restaura termios, emite `rmacs` → SI, `sgr0`, `cnorm` según terminfo `linux`) y el proceso termina con código 0. SIGKILL/OOM no se pueden interceptar: de ese caso responde el wrapper (§2.5 paso 9) |
| `Tab`, `←`/`→`, `1`/`2` | alternar Node / Cluster |
| `r` | releer snapshot ya, renovar lease, limpiar y repintar (`clearok`) |
| `a` | ciclar glifos `acs → ascii → unicode` |
| `c` / `m` | orden de la tabla de guests (vista B) |
| `KEY_RESIZE` | recalcular layout |

Sin `Esc` (retardo de curses), sin ratón.

#### Saneamiento de cadenas no confiables

`console_render.sanitize(text, max_cells)`: `NFKD` → descartar marcas combinantes → conservar sólo `0x20–0x7E` (ASCII imprimible), sustituir cualquier otro carácter por `?`, colapsar espacios repetidos, truncar a `max_cells - 1` + `…`/`~` según glifos. Se aplica a `name`, `tags`, `node`, `storage`, `pve_version`, `error`, y a todo lo que venga del snapshot o de `/sys/class/net/*`. Las claves del catálogo i18n pasan por el mismo filtro cuando `glyphs != unicode` (translitera `č → c`).

### 2.5 Ciclo de vida en tty1

#### Artefactos (todos root-owned, 0644 salvo ejecutables 0755)

| Ruta instalada | Origen en el repo | PR | Rol |
|---|---|---|---|
| `/usr/local/bin/proxmenux-console` | `scripts/console/proxmenux-console` | PR1 (PR2 lo amplía) | lanzador manual |
| `/etc/systemd/system/getty@tty1.service.d/proxmenux-console.conf` | generado por `console_lifecycle.sh` | PR2 | drop-in |
| `/usr/local/sbin/proxmenux-console-getty` | `scripts/console/proxmenux-console-getty` | PR2 | wrapper del getty (fuera de `$BASE_DIR` a propósito) |
| `/usr/local/share/proxmenux/scripts/console/console_lifecycle.sh` | `scripts/console/console_lifecycle.sh` (llega con el `cp -r ./scripts/*` del instalador, `install_proxmenux.sh:877`) | PR2 | biblioteca bash con `install_console_tui`, `remove_console_tui`, `console_tty1_idle`, `console_autostart_set`; la cargan los instaladores desde el clon (`$TEMP_DIR/scripts/console/console_lifecycle.sh`) y `config_menu.sh` desde `$LOCAL_SCRIPTS` (como ya hace con `global/pmx_journal.sh`, `config_menu.sh:64-65`), **antes** del `rm -rf "$BASE_DIR"`. Si falta en `$LOCAL_SCRIPTS`, no hay consola instalada por esta versión y el menú omite el paso con un aviso |
| `/etc/proxmenux/console.conf` | generado si no existe | PR2 | `AUTOSTART=yes|no`, `GLYPHS=auto|unicode|ascii`, `PRIVACY=names|ids`, `CONSOLE_USER=proxmenux-console` |
| `/etc/tmpfiles.d/proxmenux-console.conf` | generado | PR2 | `d /run/proxmenux-console 0755 root root -` y `d /run/proxmenux-console/client 0755 proxmenux-console proxmenux-console -` |
| usuario/grupo `proxmenux-console` | `adduser --system --group --no-create-home --shell /usr/sbin/nologin proxmenux-console` | PR2 | identidad de la TUI. **No** se añade al grupo `tty`: la TUI usa sólo los fd 0/1/2 heredados de systemd (`StandardInput=tty`), `initscr()` de ncurses llama `newterm(name, stdout, stdin)` sin abrir `/dev/tty` fuera de `#ifdef __CYGWIN__` (`ncurses/base/lib_initscr.c`), y si algún código abriera `/dev/tty`, ese nodo es `0666 root:tty` (udev `50-udev-default.rules`: `KERNEL=="tty", GROUP="tty", MODE="0666"`) y funciona para cualquier proceso con terminal de control. Pertenecer a `tty` daría escritura sobre **todas** las terminales (`tty[0-9]*` y pts son `0620 root:tty`), es decir, inyección de texto en sesiones ajenas: un privilegio que la TUI no necesita |

#### Drop-in exacto

```ini
# Managed by ProxMenux Monitor console (installer). Removed by its uninstaller.
[Service]
ExecStart=
ExecStart=-/usr/local/sbin/proxmenux-console-getty %I
```

La línea vacía reinicia la lista de `ExecStart` (systemd.service(5)); el prefijo `-` conserva la semántica del fabricante. `TERM` llega del entorno que systemd fija para unidades con `TTYPath` en VT (`linux`). No se añade nada más: el resto de la unidad (`Type=idle`, `Restart=always`, `TTYVHangup`, `TTYVTDisallocate`, `UnsetEnvironment`) queda intacto.

#### Pasos del wrapper (`proxmenux-console-getty <tty>`)

1. `set -u`; definir `run_login() { exec /sbin/agetty -o '-- \u' --noreset --noclear - "${TERM:-linux}"; }` — idéntico al `ExecStart` de systemd 257 (Debian 13 / PVE 9) y compatible con util-linux de Debian 12 (`-p` de la 252 sólo propaga un entorno que `UnsetEnvironment` ya vació). **Toda rama termina en `run_login`.**
2. Variables con override por entorno para pruebas: `PMX_CONSOLE_CONF`, `PMX_CONSOLE_STATE_DIR`, `PMX_CONSOLE_LAUNCHER` (default `/usr/local/share/proxmenux/monitor-app/AppRun`), `PMX_CONSOLE_SETPRIV` (default `setpriv`).
3. Cargar `console.conf` sólo si existe, pertenece a uid 0 y no es escribible por grupo/otros (`stat -c '%u %a'`); si no cumple, ignorarlo y seguir con defaults.
4. `[ "$AUTOSTART" = yes ] || run_login`.
5. Marcador `${STATE_DIR}/autostart.${TTY}`: si existe → `run_login`. Si no, `mkdir -p "$STATE_DIR"` y crear el marcador **antes** de lanzar nada (guarda contra bucles si la TUI muere en el acto).
6. Comprobaciones: lanzador ejecutable, `id "$CONSOLE_USER"` válido, `python3 -c 'import curses'` del host → si algo falla, registrar en `${STATE_DIR}/last-error.log` y `run_login`.
7. `install -d -o "$CONSOLE_USER" -g "$CONSOLE_USER" -m 0755 "${STATE_DIR}/client"`.
8. `printf '\033[H\033[2J'`; ejecutar (sin `exec`): `setpriv --reuid="$CONSOLE_USER" --regid="$CONSOLE_USER" --init-groups --reset-env --no-new-privs env TERM="${TERM:-linux}" LANG=C.UTF-8 HOME=/nonexistent PMX_CONSOLE_STATE_DIR="$STATE_DIR" "$LAUNCHER" --tui --autostart 2>"${STATE_DIR}/client/tui.log"`. Timeout: ninguno (la TUI puede quedar horas); `SIGTERM` de `systemctl stop getty@tty1` llega al grupo de control y mata ambos.
9. Tras **cualquier** salida de la TUI (normal, traceback, SIGKILL, OOM): `stty sane <&0 2>/dev/null || true; printf '\033c'`; `run_login`. Los dos pasos actúan sobre capas distintas y ambos son necesarios: `stty sane` repara la disciplina de línea del kernel (termios: `ICANON`, `ECHO`, `ISIG`… que curses deja en modo raw si muere), y `ESC c` (RIS) repara el emulador del VT. Según `reset_terminal()` del kernel (`drivers/tty/vt/vt.c`, v6.14), RIS pone `Gx_charset[0]=LAT1_MAP`, `Gx_charset[1]=GRAF_MAP`, `charset=0` (equivale a SI: sale del G1 gráfico que deja `ACS_*`), `vc_utf=default_utf8`, `vc_deccm=global_cursor_default` (cursor visible), `vc_cursor_type=cur_default`, `default_attr()` (atributos y colores por defecto), tabuladores, `vt_reset_keyboard()` (modos de teclado y LEDs, **no** el keymap) y limpia la pantalla (`csi_J`). No llama a `set_palette` ni a `con_font_set`: la fuente y la paleta cargadas por console-setup se conservan, así que RIS no introduce regresión. Por eso la secuencia `\033[0m\017\033[?25h` propuesta en la auditoría es redundante con RIS (SGR0, SI y DECTCEM quedan cubiertos por `default_attr`, `charset=0` y `vc_deccm`) y se omite: una sola secuencia atómica de 2 bytes no tiene "a medio camino". Limitación conocida: si el administrador arrancó con `vt.default_utf8=0`, RIS saca al VT del modo UTF-8 (Debian y PVE compilan `default_utf8=1`). Se evita `tput`: no hace falta terminfo en el camino crítico del login.

El wrapper no cambia dueño ni permisos del tty: la TUI hereda los fd 0/1/2 ya abiertos por systemd (`StandardInput=tty`), y curses trabaja sobre ellos; no necesita abrir `/dev/tty1` ni pertenecer al grupo `tty` (ver tabla de artefactos).

#### Qué hace cada salida

| Evento | Resultado |
|---|---|
| `q` / `Ctrl+C` / SIGTERM / SIGHUP | TUI termina 0 vía `endwin()` → wrapper: `stty sane` + `ESC c` → `agetty` → `login:` en < 2 s |
| Traceback, SIGKILL u OOM de la TUI | stderr en `tui.log`; el VT puede quedar en G1 y sin cursor → el wrapper lo repara (`stty sane` + `ESC c`) → `login:` legible y con cursor |
| Logout de una sesión en tty1 | `Restart=always` relanza el wrapper → marcador presente → `agetty` directo (sin TUI) |
| `systemctl restart getty@tty1` | ídem (marcador persiste hasta el reinicio) |
| Reinicio del nodo | `/run` se vacía → TUI de nuevo una vez |
| `AUTOSTART=no` | `agetty` directo en todos los casos |
| `/dev/tty0` ausente (sin VT) | `getty@tty1` no arranca por su propia condición; nada que hacer |

#### Relanzamiento manual

`proxmenux-console` desde cualquier shell (SSH, otra TTY, la propia tty1 ya logueada). **PR1**: exige `id -u` = 0 (mensaje claro si no) y hace `exec /usr/local/share/proxmenux/monitor-app/AppRun --tui "$@"` como root; al salir devuelve el shell. **PR2**: si existe el usuario `proxmenux-console`, root baja a él con la misma línea `setpriv` del wrapper (sin `--autostart`); un miembro del grupo ejecuta `AppRun --tui` directamente; cualquier otro recibe un mensaje. No toca el marcador. Para volver a verla en tty1 sin reiniciar: `rm /run/proxmenux-console/autostart.tty1 && systemctl restart getty@tty1` (documentado; sólo si nadie está logueado en tty1).

#### Menú de ProxMenux (`scripts/menus/config_menu.sh`)

Junto a las entradas del Monitor (`:586-602`), cuando `console.conf` existe: `Console monitor (HDMI/VGA): launch now`, `Console monitor: enable/disable autostart on tty1` (edita `AUTOSTART` en `console.conf`) y `Console monitor: status` (muestra `AUTOSTART`, si el marcador existe, última línea de `tui.log`). Todo con `translate "..."`.

#### Instalación / actualización / desinstalación

- **PR1 — instalar/desinstalar el comando manual**: los instaladores añaden `install -m 0755 "$TEMP_DIR/scripts/console/proxmenux-console" /usr/local/bin/proxmenux-console` tras copiar `scripts/`; `uninstall_proxmenux_monitor` y `uninstall_proxmenu` añaden `rm -f /usr/local/bin/proxmenux-console`. Nada más del sistema cambia en PR1.
- **PR2 — instalar** (`install_proxmenux.sh` y `install_proxmenux_beta.sh`, tras `create_monitor_service` / la actualización, con `source "$TEMP_DIR/scripts/console/console_lifecycle.sh"`): función `install_console_tui`: crear usuario, copiar wrapper y lanzador a `/usr/local/{sbin,bin}`, escribir tmpfiles y `systemd-tmpfiles --create` del archivo, escribir drop-in, `daemon-reload`; escribir `console.conf` **sólo si no existe** (pregunta Q1 del grill con default Sí; `AUTOSTART=no` si el usuario rehúsa). Si `AUTOSTART=yes` y tty1 está ocioso (`loginctl list-sessions --no-legend` sin sesión en `tty1` **y** `ps -o comm= -t tty1` es exactamente `agetty`) → `systemctl restart getty@tty1.service` para mostrarla ya; si no, se mostrará en el próximo arranque y se informa. Nunca `chvt`.
- **Actualizar**: misma función (idempotente): rescribe wrapper, lanzador, tmpfiles y drop-in; **no toca `console.conf`** (respeta un `AUTOSTART=no` explícito); no reinicia getty si ya hay marcador o sesión.
- **Desinstalar Monitor** (`uninstall_proxmenux_monitor`, `config_menu.sh:95-137`) y **desinstalar ProxMenux** (`uninstall_proxmenu`, `:787`, antes del `rm -rf "$BASE_DIR"` de `:821`): `config_menu.sh` carga `$LOCAL_SCRIPTS/console/console_lifecycle.sh` si existe y llama a `remove_console_tui` (la **misma** función que usan los instaladores: una sola fuente de verdad, sin duplicar); en este orden, borrar drop-in → `daemon-reload` → borrar wrapper, lanzador, tmpfiles, `console.conf`, `/run/proxmenux-console` → `deluser --system proxmenux-console` (si existe y no tiene procesos; si los tiene, se deja y se informa). Si tty1 ejecuta en este momento sólo la TUI (sin sesión), `systemctl restart getty@tty1` para devolver el getty del fabricante; si hay sesión, se deja para el próximo respawn. Sólo se borran rutas propias enumeradas; nunca `getty@.service` ni otros drop-ins.
- **Interacción con otras consolas**: `serial-getty@*`, `tty2–6`, `autovt@`, SSH y `rescue.target` no se tocan. El drop-in es por instancia (`getty@tty1.service.d`), así que `getty@tty2` etc. siguen con el fabricante.

### 2.6 i18n

- **TUI (Python)**: claves bajo `runtime.consoleTui.*` en `AppImage/messages/en/common.json`; el bot `build-i18n-messages.yml` rellena `de es fr it pt sk sv`. Los catálogos ya se empaquetan en `usr/share/proxmenux/messages/<locale>/common.json` (`build_appimage.sh:178-181`). La TUI resuelve `APPDIR/usr/share/proxmenux/messages/<lang>/common.json` → `en` → clave, con la misma lógica de `notification_templates._catalog_value` (`:52-73`) reimplementada en 20 líneas dentro de `console_tui.py` (no se importa `notification_templates` para no arrastrar `urllib` ni la caché de notificaciones). Idioma: `snapshot.language` si hay snapshot; si no, `.language` de `/usr/local/share/proxmenux/config.json` (misma fuente que `load_language`, `scripts/utils.sh:228-236`); si no, `en`. Textos pasan por `sanitize` cuando los glifos no son `unicode`.
- Claves iniciales (T6 puede avanzar en paralelo): `title`, `tabs.node`, `tabs.cluster`, `panels.cpu`, `panels.memory`, `panels.network`, `panels.disk`, `panels.cluster`, `panels.topCpu`, `panels.topMemory`, `panels.nodes`, `panels.guests`, `labels.load`, `labels.threads`, `labels.temp`, `labels.arc`, `labels.available`, `labels.zram`, `labels.swap`, `labels.rx`, `labels.tx`, `labels.read`, `labels.write`, `labels.used`, `labels.uptime`, `labels.hostCores`, `labels.quorumOk`, `labels.noQuorum`, `labels.standalone`, `labels.coverage`, `labels.online`, `labels.offline`, `labels.unknown`, `labels.running`, `labels.stopped`, `labels.sort`, `labels.moreNodes`, `labels.moreGuests`, `labels.hostOther`, `status.data`, `status.noData`, `status.stale`, `status.starting`, `status.waitingForData`, `status.monitorUnavailable`, `status.terminalTooSmall`, `columns.type`, `columns.id`, `columns.name`, `columns.node`, `columns.status`, `columns.cpu`, `columns.mem`, `columns.root`, `columns.guests`, `columns.up`, `footer.quit`, `footer.switch`, `footer.refresh`, `footer.glyphs`, `footer.otherTty`, `footer.sort`.
- **Bash** (instaladores, wrapper, menú): `translate "..."` como el resto del repo; el bot de `lang/*.json` las recoge de `scripts/**/*.sh` y de los instaladores. El wrapper imprime lo mínimo (nada en el camino feliz).
- **Docs web**: `web/messages/en/docs/settings/console-monitor.json` + `web/app/[locale]/docs/settings/console-monitor/page.tsx`, siguiendo `docs/settings/proxmenux-monitor`; traducción por `build-web-docs-i18n.yml`.

### 2.7 Empaquetado

- `AppImage/scripts/AppRun`: nueva rama antes del arranque de Flask: `if [[ "$1" == "--tui" ]]; then exec python3 -I "${APPDIR}/usr/bin/console_tui.py" "${@:2}"; fi` — sin los `echo` de banner (ensuciarían la consola) y con `-I` (modo aislado). Las ramas `--translate` y por defecto no cambian.
- **`sys.path` bajo `-I`**: `-I` implica `-E -s -P`, y `-P` "no antepone el directorio del script" a `sys.path` (docs de CPython 3.11+, `using/cmdline`), así que `import console_render` fallaría con `ModuleNotFoundError`. `console_tui.py` debe empezar **exactamente** así, antes de cualquier import hermano:
  ```python
  import os
  import sys
  _HERE = os.path.dirname(os.path.realpath(__file__))
  if _HERE not in sys.path:
      sys.path.insert(0, _HERE)
  import console_metrics, console_render, console_snapshot, zram_metrics  # noqa: E402
  ```
  `realpath` resuelve el symlink de `AppRun`/`usr/bin`; `-I` sigue ignorando `PYTHONPATH`, el cwd y el site del usuario. Riesgo de *import hijack*: nulo mientras `monitor-app/` sea root-owned sin escritura para otros (lo extrae el instalador como root, `extract_appimage_to_runtime_dir`); el smoke test de §2.8 lo ejecuta desde otro cwd para probarlo.
- `AppImage/scripts/build_appimage.sh:128-148`: añadir `cp` de `console_tui.py`, `console_render.py`, `console_metrics.py`, `console_snapshot.py` a `$APP_DIR/usr/bin/` (sin `2>/dev/null ||` : son obligatorios, como `zram_metrics.py` en `:129`).
- Python/curses: nada que empaquetar; el host aporta `_curses` (verificado en `libpython3.11-stdlib` y `libpython3.13-stdlib`) y `ncurses-base` el terminfo `linux`. El wrapper comprueba `import curses` antes de lanzar (paso 6).
- **Caché compartida de `/cluster/resources`** (`flask_server.py`, commit propio PR1-T3). Verificado en `PVE/API2/Cluster.pm` (pve-manager, HEAD): el handler lee **siempre** `get_nodelist`, `get_members`, `rrd_dump`, `get_vmlist`, el estado HA y `static-info`, y el parámetro `type` sólo gobierna qué bloques de entradas se construyen (`if (!$param->{type} || $param->{type} eq 'vm')` …); las entradas de guests salen de `PVE::API2Tools::extract_vm_stats($vmid, $data, $rrd)`, que no recibe `type`. Es decir, una llamada sin `--type` cuesta lo mismo que `--type vm` en lo caro (IPC con pmxcfs/RRD) y devuelve dicts de guest **idénticos** más las entradas `node`, `storage`, `pool`, `sdn`. Decisión: la función existente `get_cached_pvesh_cluster_resources_vm()` (`:2759-2797`) pasa a ser un filtro `[r for r in get_cached_pvesh_cluster_resources() if r.get('type') in ('qemu', 'lxc')]` sobre una nueva `get_cached_pvesh_cluster_resources()` que ejecuta `pvesh get /cluster/resources --output-format json` con el mismo lock single-flight, el mismo `_PVESH_CACHE_TTL = 5`, el mismo `timeout=10` y la misma política de fallo (devuelve la última caché o `[]`). Los 13 call sites internos (`:2445, 3693, 6815, 12087, 12417, 13091, 15427, 15638, 15706, 15844, 15896, 16053, 16107`) y los externos (`health_monitor.py:6204, 6350`, `notification_events.py:2828`) **no cambian** de nombre ni de contrato; los tests que parchean `get_cached_pvesh_cluster_resources_vm` (`test_cluster_guest_storage_ownership.py`, `test_lxc_vm_ip_inventory.py`) siguen válidos. Prueba de caracterización (`AppImage/scripts/tests/test_cluster_resources_cache.py`): con `subprocess.run` parcheado para devolver un JSON completo grabado (nodos + guests + storages), `get_cached_pvesh_cluster_resources_vm()` devuelve exactamente el JSON `--type vm` grabado del mismo clúster (igualdad de listas tras ordenar por `id`), una sola invocación de `pvesh` por TTL aunque se llamen ambas funciones, y en fallo se conserva la caché previa. `_get_proxmox_storage_uncached` (`:5652`, `--type storage`, TTL propio) queda fuera: sería un segundo cambio de comportamiento.
- `flask_server.py` (commit propio PR1-T7): `import console_snapshot` y `console_snapshot.ensure_publisher(get_node_name=get_proxmox_node_name, get_resources=get_cached_pvesh_cluster_resources, run_pvesh=<callable sólo para /cluster/status>, read_language=<callable>)` al cargar el módulo, junto a `_ensure_net_rate_sampler()` (`:5900` aprox.). Con dashboard y TUI abiertos sigue habiendo **una** llamada de recursos cada 5 s.

### 2.8 Estrategia de pruebas (sólo en workers QA remotos)

Las pruebas nunca se ejecutan en el Mac: corren en los P3 habilitados o en `qf-infra` mediante `qf-test up --generic --auto --allow-qf-infra` → `exec` → `artifacts` → `down`, en un contenedor Debian 13 con `python3`, `bash`, `util-linux`, `jq`. CI upstream: nuevo `.github/workflows/test-console-tui.yml` con el mismo patrón que `test-lvm-disk-policy.yml` (`python -I -B -m unittest discover -s <dir> -p <file> -v`, Python 3.11 y 3.13 en matriz).

| Capa | Archivo de prueba | Qué prueba (fixtures en `AppImage/scripts/tests/fixtures/console/`) |
|---|---|---|
| Métricas locales | `AppImage/scripts/tests/test_console_metrics.py` | Dos muestras de `/proc/stat`, `/proc/net/dev`, `/proc/diskstats` con `dt=1.0` → porcentajes y tasas exactas; filtro de NIC físicas (sólo con `device/`, excluye `lo`, `vmbr*`, `fwbr*`, `fwpr*`, `fwln*`, `tap*`, `veth*`, bonds); `meminfo` con y sin ARC (no doble conteo); `hwmon` `k10temp`/`coretemp`/sin sensor → `None`; zram vía `zram_metrics` con rutas inyectadas; `statvfs` simulado. |
| Snapshot | `AppImage/scripts/tests/test_console_snapshot.py` | Normalización desde JSON de `pvesh` grabados: clúster 2 nodos, standalone, sin quorum, nodo offline, campos ausentes (`diskread` null), `tags` con `;` y caracteres de control, 2500 guests → `truncated`; estados `starting/ok/stale/unavailable/error` (snapshot inicial inmediato en `ensure_publisher`, luego tras 0/1/3 fallos); escritura atómica (archivo temporal + `replace`) con permisos `0600`/`0700` sin grupo y `0640`/`0750` cuando `grp.getgrnam` (parcheado) devuelve el grupo; lease: sin lease no se invoca `get_resources` (callable contado), al aparecer el lease se publica en el mismo ciclo, un symlink o directorio en lugar del lease cuenta como ausente (`lstat` + `S_ISREG`); lector rechaza `schema != 1` y JSON corrupto sin excepción; `age_s` con reloj atrasado → 0. |
| Caché compartida | `AppImage/scripts/tests/test_cluster_resources_cache.py` | Caracterización (§2.7): `get_cached_pvesh_cluster_resources_vm()` sobre el JSON completo grabado == JSON `--type vm` grabado; un solo `pvesh` por TTL para ambas funciones; fallo conserva caché; los tests existentes que parchean `get_cached_pvesh_cluster_resources_vm` siguen pasando. |
| Render | `AppImage/scripts/tests/test_console_render.py` | `render(view_model, 80, 24, glyphs)` y `(128, 48)`, `(240, 67)`: dimensiones exactas, ninguna fila supera `width`, celdas clave (cabecera, títulos, filas TOP ordenadas por `cpu×maxcpu` y `mem`, `+N nodes`), TOP MEMORY incluye `mem > 0` con cualquier `status` y excluye plantillas, indicador `host other` correcto y acotado a ≥ 0 (y ausente sin snapshot fresco), estado `STARTING` en `warn` vs `MONITOR UNAVAILABLE` en `crit` según uptime/estado, colapso a 80×24 (historial 1 fila, sin per-core), modo `ascii` sin bytes > 0x7E, `sanitize` elimina `\x1b[31m`, `\n`, `č → c`, trunca a celdas; vista B: offline primero, totales sólo con quorum y cobertura completa; `Terminal too small` a 79×24 y 80×23. |
| Wrapper getty (PR2) | `tests/test_console_getty_wrapper.py` | Bash con `PATH` restringido (patrón `tests/test_tui_count_messages.py:51-54`), `PMX_CONSOLE_*` apuntando a un fake root; `agetty`, `setpriv`, `python3`, `stty`, `id`, `install` falsos que registran argv y **stdout capturado byte a byte**: sin marcador → lanza TUI y luego `agetty` con argv exacto `-o '-- \u' --noreset --noclear - linux`; marcador presente → sólo `agetty`; `AUTOSTART=no` → sólo `agetty`; conf escribible por otros → ignorada; lanzador que sale 1 o que muere por señal 9 → `agetty` igual; marcador creado antes de invocar el lanzador (orden en el log); **los bytes `\x1b` `c` aparecen en stdout después de la salida del lanzador y antes de la entrada de `agetty` en el log, y `stty sane` se invoca en el mismo intervalo**; `TERM` ausente → `linux`. |
| Biblioteca de ciclo de vida (PR2) | `tests/test_console_lifecycle.py` | `source scripts/console/console_lifecycle.sh` en bash con `systemctl`, `loginctl`, `ps`, `adduser`, `deluser`, `systemd-tmpfiles`, `install` falsos y prefijo de rutas por variable: drop-in con contenido exacto; `console.conf` no se sobrescribe si existe; reinicio de `getty@tty1` sólo con tty1 ocioso; desinstalación borra sólo las rutas propias y en el orden drop-in → reload → resto; idempotencia (dos ejecuciones = mismo árbol); el mismo archivo se carga desde un árbol tipo `$TEMP_DIR` y desde uno tipo `$LOCAL_SCRIPTS`. |
| Instaladores y menú (PR1 y PR2) | `tests/test_console_installer.py` | Extracción acotada (helper `function()` de `test_tui_count_messages.py`) de los bloques que copian/borran `proxmenux-console` (PR1) y de las llamadas a `install_console_tui`/`remove_console_tui` (PR2) en ambos instaladores y en `config_menu.sh`; en `uninstall_proxmenu` la llamada a `remove_console_tui` precede al `rm -rf "$BASE_DIR"` (orden verificado en el texto extraído). |
| Smoke TUI | `AppImage/scripts/tests/test_console_tui_smoke.py` | `pty.fork()` con `TERM=linux` y **cwd = un directorio temporal distinto del de los módulos**, ejecutando `python3 -I <ruta absoluta>/console_tui.py --once --snapshot <fixture> --proc-root <fixture>`: pinta un frame y sale 0 (prueba el arreglo de `sys.path`); `q` enviado por el pty termina el proceso < 2 s; SIGTERM al proceso → sale 0 y el pty recibe la secuencia de `endwin` (`\x1b[?25h` o `\x0f` presentes); sin snapshot y uptime simulado < 120 s → sale 0 mostrando `Monitor starting`; sin snapshot y uptime ≥ 120 s → `MONITOR UNAVAILABLE`. Se ejecuta sólo en el worker QA (necesita pty y curses reales). |

**Aceptación en hardware (Owner, P3 con HDMI)**, tras instalar el AppImage de la rama en un nodo de prueba:

0. (PR1, antes de PR2) Por SSH como root: `proxmenux-console` abre la TUI, `q` devuelve el shell; con el dashboard web abierto a la vez, `ps` muestra a lo sumo un `pvesh get /cluster/resources` cada 5 s.
1. Reiniciar: en los primeros segundos la cabecera muestra `Monitor starting…` (ámbar, no rojo) con CPU/RAM locales ya en vivo; en ≤ 30 s aparecen clúster y guests. Los glifos de caja y barras se ven correctos (si no, `a` y anotar la fuente activa: `cat /etc/default/console-setup`).
2. `q` → `login:` en < 2 s; iniciar sesión, `exit` → vuelve `login:`, no la TUI.
2b. Desde SSH, `kill -9` al `python3` de la TUI en tty1 → en tty1 aparece `login:` con cursor visible y el texto tecleado se lee en caracteres normales (no gráficos).
2c. Provocar un `printk` (conectar un USB) con la TUI abierta → el texto del kernel desaparece en ≤ 30 s sin tocar nada.
3. `proxmenux-console` desde SSH y desde tty1 logueada → se abre y al salir vuelve el shell.
4. `Ctrl+Alt+F2` → getty normal en tty2; `Ctrl+Alt+F1` → la TUI sigue (o el login si ya se salió).
5. Pestaña Cluster muestra ambos P3 online y los guests; `c`/`m` cambian el orden.
6. `systemctl stop proxmenux-monitor` → en ≤ 60 s la cabecera marca `MONITOR UNAVAILABLE`, CPU/RAM/red locales siguen en vivo; `start` → vuelve a `data Ns`.
7. Desconectar la red del nodo → `NO QUORUM` en rojo con cobertura `1/2`; reconectar → `QUORUM OK 2/2`.
8. Poner `AUTOSTART=no`, reinstalar/actualizar → sigue en `no`; reiniciar → sólo `login:`.
9. Desinstalar Monitor desde el menú → reiniciar → `login:` del fabricante en tty1; `systemctl cat getty@tty1` sin drop-in de ProxMenux.
10. `top` en el nodo durante 5 min con la TUI abierta: `python3` de la TUI ≤ 2 % CPU, ≤ 40 MiB RSS; `journalctl -u proxmenux-monitor` sin errores del publicador.

### 2.9 Descomposición de la implementación (DAG) — dos PRs

Un commit por tarea; propiedad de archivos sin solapes dentro de cada PR; **nivel**: *principal* (lógica compleja, contratos) o *light*. Ninguna tarea edita archivos de otra; cuando dos tareas tocan el mismo archivo (`flask_server.py`, `proxmenux-console`), son secuenciales con dependencia explícita y cada una posee una región distinta. Lo que cruce fronteras se resuelve en este ADR, no en el código.

#### PR1 — `feature/console-tui`: TUI utilizable manualmente por root (sin usuario de sistema, sin drop-in de getty)

| # | Tarea | Archivos propios | Depende de | Pruebas que la cierran | Nivel |
|---|---|---|---|---|---|
| P1-T1 | Lectores locales `/proc`/`/sys` con rutas inyectables, tasas, historial de 60, cálculo de `host other` (CPU y memoria) a partir del snapshot | `AppImage/scripts/console_metrics.py`, `AppImage/scripts/tests/test_console_metrics.py`, `AppImage/scripts/tests/fixtures/console/proc_*` | — | `test_console_metrics.py` | principal |
| P1-T2 | Contrato del snapshot: normalizador, estados `starting/ok/stale/unavailable/error`, snapshot inicial inmediato, escritura atómica con permisos según exista el grupo, lease con `lstat`+`S_ISREG` y publicación inmediata al aparecer, `ensure_publisher(get_node_name, get_resources, run_pvesh, read_language)`, `load_snapshot(...)` | `AppImage/scripts/console_snapshot.py`, `AppImage/scripts/tests/test_console_snapshot.py`, `AppImage/scripts/tests/fixtures/console/pvesh_*.json` | — | `test_console_snapshot.py` | principal |
| P1-T3 | Caché compartida: nueva `get_cached_pvesh_cluster_resources()` (todos los tipos) y `get_cached_pvesh_cluster_resources_vm()` como filtro, misma TTL/lock/timeout/política de fallo; sin cambios en los 14 call sites ni en los consumidores externos | `AppImage/scripts/flask_server.py` (sólo la región `_pvesh_cache`/`get_cached_pvesh_cluster_resources*`, `:1758-1800` y `:2759-2797`), `AppImage/scripts/tests/test_cluster_resources_cache.py`, `AppImage/scripts/tests/fixtures/console/pvesh_resources_full.json` y `pvesh_resources_vm.json` | — | `test_cluster_resources_cache.py` + los tests existentes `test_cluster_guest_storage_ownership.py` y `test_lxc_vm_ip_inventory.py` en verde | principal |
| P1-T4 | Motor de render puro: `Grid`, paneles, glifos, roles de color, vistas A y B, colapso, estado `STARTING`, títulos TOP con `host other`, regla `mem > 0`, `sanitize`, textos vía `tr(key)` | `AppImage/scripts/console_render.py`, `AppImage/scripts/tests/test_console_render.py`, `AppImage/scripts/tests/fixtures/console/viewmodel_*.json` | — (`ViewModel` fijado en §2.4) | `test_console_render.py` | principal |
| P1-T5 | Entrada `console_tui.py`: cabecera `sys.path` (§2.7), argumentos (`--once`, `--snapshot`, `--proc-root`, `--glyphs`, `--autostart` reservado), bucle de 1 s, repintado completo cada 30 s, `KEY_RESIZE`, teclas, decisión `STARTING` por `/proc/uptime`, carga de snapshot y edad, tasas por guest, lease, i18n, señales → `endwin` | `AppImage/scripts/console_tui.py`, `AppImage/scripts/tests/test_console_tui_smoke.py` | P1-T1, P1-T2, P1-T4 | `test_console_tui_smoke.py` (pty, cwd distinto, QA remoto) | principal |
| P1-T6 | Catálogo i18n `runtime.consoleTui.*` en inglés (lista de §2.6 más `status.starting`, `labels.hostOther`) | `AppImage/messages/en/common.json` | — | carga JSON + prueba en P1-T5 de que toda clave usada existe en `en` | light |
| P1-T7 | Arranque del publicador en el Monitor: `import console_snapshot` + `ensure_publisher(...)` junto a `_ensure_net_rate_sampler()`, callables de nodo/recursos/`cluster/status`/idioma | `AppImage/scripts/flask_server.py` (sólo el bloque nuevo junto a `:5900`) | P1-T2, P1-T3 | `test_console_snapshot.py`; verificación manual en QA (`journalctl`, `ls -l /run/proxmenux-monitor`) | light |
| P1-T8 | Empaquetado: rama `--tui` en `AppRun`, `cp` de los cuatro módulos en `build_appimage.sh`, nota en `AppImage/README.md` | `AppImage/scripts/AppRun`, `AppImage/scripts/build_appimage.sh`, `AppImage/README.md` | P1-T5 (nombres) | `bash -n`; build en QA remoto y `AppRun --tui --once` sobre fixture | light |
| P1-T9 | Comando manual root-only `scripts/console/proxmenux-console` (PR1: `exec AppRun --tui`), copia a `/usr/local/bin` en ambos instaladores y borrado en `uninstall_proxmenux_monitor`/`uninstall_proxmenu` | `scripts/console/proxmenux-console`, `install_proxmenux.sh` (una línea `install`), `install_proxmenux_beta.sh` (ídem), `scripts/menus/config_menu.sh` (dos líneas `rm -f`), `tests/test_console_installer.py` | P1-T8 (ruta de `AppRun`) | `tests/test_console_installer.py` (bloques extraídos con fakes) | light |
| P1-T10 | CI: `test-console-tui.yml` (matriz 3.11/3.13; lanes de P1-T1..T4; el smoke de pty queda para QA remoto) | `.github/workflows/test-console-tui.yml` | P1-T1..T4 (nombres) | workflow en verde | light |
| P1-T11 | Documentación de uso manual: página web (JSON + `page.tsx`), `CHANGELOG.md`, `README.md` | `web/messages/en/docs/settings/console-monitor.json`, `web/app/[locale]/docs/settings/console-monitor/page.tsx`, `CHANGELOG.md`, `README.md` | P1-T5, P1-T9 | enlaces y `npm run build` de `web/` en QA remoto | light |

Orden: P1-T1 ∥ P1-T2 ∥ P1-T3 ∥ P1-T4 ∥ P1-T6 → P1-T5, P1-T7 → P1-T8 → P1-T9 → P1-T10 → P1-T11. PR1 se abre contra `develop`; PR2 parte de `develop` una vez mergeado PR1.

#### PR2 — `feature/console-tui-autostart`: autoarranque en tty1

| # | Tarea | Archivos propios | Depende de | Pruebas que la cierran | Nivel |
|---|---|---|---|---|---|
| P2-T1 | Wrapper del getty (§2.5 pasos 1–9, incluido `stty sane` + `ESC c`) | `scripts/console/proxmenux-console-getty`, `tests/test_console_getty_wrapper.py` | PR1 mergeado | `test_console_getty_wrapper.py` (incluye la aserción de bytes de reset antes de `agetty`) | principal |
| P2-T2 | Biblioteca de ciclo de vida: `install_console_tui`, `remove_console_tui`, `console_tty1_idle`, `console_autostart_set`, pregunta de autoarranque (Q1, default Sí), creación/borrado del usuario sin grupo `tty` | `scripts/console/console_lifecycle.sh`, `tests/test_console_lifecycle.py` | P2-T1 (rutas) | `test_console_lifecycle.py` | principal |
| P2-T3 | Instaladores: `source "$TEMP_DIR/scripts/console/console_lifecycle.sh"` + llamada a `install_console_tui` tras crear/actualizar el servicio Monitor, en ambos | `install_proxmenux.sh`, `install_proxmenux_beta.sh` | P2-T2 | extensión de `tests/test_console_installer.py` (bloques extraídos) | light |
| P2-T4 | Lanzador: política root→`setpriv` al usuario si existe, miembro del grupo directo, otros mensaje | `scripts/console/proxmenux-console`, `tests/test_console_command.py` | P2-T2 (nombre de usuario) | `tests/test_console_command.py` (bash con `id`/`setpriv` falsos) | light |
| P2-T5 | Menú: `source "$LOCAL_SCRIPTS/console/console_lifecycle.sh"` si existe; entradas lanzar / autoarranque / estado; `remove_console_tui` en `uninstall_proxmenux_monitor` y antes del `rm -rf "$BASE_DIR"` en `uninstall_proxmenu` | `scripts/menus/config_menu.sh` | P2-T2 | extensión de `tests/test_console_installer.py` (orden `remove_console_tui` → `rm -rf` verificado en el texto) | light |
| P2-T6 | CI: añadir las lanes de P2-T1, P2-T2, P2-T4 al workflow | `.github/workflows/test-console-tui.yml` | P2-T1, P2-T2, P2-T4 | workflow en verde | light |
| P2-T7 | Documentación del autoarranque (página web, `CHANGELOG.md`, `README.md`) | `web/messages/en/docs/settings/console-monitor.json`, `web/app/[locale]/docs/settings/console-monitor/page.tsx`, `CHANGELOG.md`, `README.md` | P2-T3, P2-T5 | enlaces y `npm run build` de `web/` en QA remoto | light |

Orden: P2-T1 → P2-T2 → P2-T3 ∥ P2-T4 ∥ P2-T5 → P2-T6 → P2-T7. Integración, build del AppImage, ejecución de las lanes en el worker QA y la auditoría adversarial (Gemini vía `agy`, lectura) las hace el Coordinador en cada PR; la aceptación en hardware (§2.8) la hace el Owner: pasos 0 para PR1 y 1–10 para PR2.

## 3. Alternativas rechazadas

| Alternativa | Por qué se rechaza (medible) |
|---|---|
| **btop/bashtop instalado y lanzado en tty1** | No muestra guests ni clúster (requisito del Owner); btop añade un paquete de ~2 MB y C++ sin integración con `pvesh`; su modo TTY sigue sin saber de quorum. |
| **TUI como `ExecStartPre=` del `getty@tty1`** | `TimeoutStartSec` (90 s por defecto) mataría la TUI y marcaría la unidad fallida; con `infinity`, la unidad queda en `activating` durante horas y `systemctl is-system-running` reporta `starting` mientras la TUI está abierta. Dos efectos secundarios observables frente a cero del wrapper. |
| **Unidad propia `proxmenux-console.service` con `Conflicts=getty@tty1`** | Ambas unidades son `Wants` de targets; systemd resuelve el conflicto en la transacción de arranque de forma no determinista (puede descartar el getty), y `systemd-logind` puede lanzar `autovt@tty1` sobre la VT. El drop-in mantiene al getty como único dueño de tty1. |
| **TUI como `ExecStart` sin marcador por boot** | `Restart=always` + `RestartSec=0`: al pulsar `q` la TUI reaparece en < 1 s; nunca hay login (lo detectó ya el diseño previo). |
| **Autologin de root + TUI en el shell** | Un `Ctrl+C` o un fallo deja un shell root sin contraseña en la pantalla física. Inaceptable. |
| **TUI como root para llamar `pvesh` directamente** | `pvesh` exige root; un proceso root con teclado y strings no confiables antes del login amplía la superficie de ataque. El Monitor ya es root y ya cachea `pvesh`: publicarlo cuesta un hilo. |
| **TUI leyendo la API HTTP del Monitor (`/api/system`, `/api/node/metrics`)** | Requieren JWT; habría que guardar una credencial administradora legible por el usuario de la consola o abrir endpoints sin autenticación. El archivo en `/run` con permisos de grupo no expone nada por red. |
| **Todo desde el snapshot, incluidas CPU/RAM locales** | Con el Monitor parado (toggle del menú), actualizándose o en gracia de arranque (180–300 s) la consola quedaría vacía; además duplicaría la cadencia de 1 s dentro del proceso gevent. |
| **Publicar el snapshot siempre, haya o no TUI** | Un `pvesh` cada 5 s son ~0,3–0,8 s de CPU de Perl por llamada en estos nodos (6–16 % de un núcleo) para nadie. El lease lo reduce a 0 cuando no hay consumidor. |
| **Textual / Rich / un binario Go o Rust** | Dependencias nuevas que hoy no están en el AppImage (Textual ≈ 10 MB con sus deps) o una toolchain adicional; `curses` ya está en el host sin empaquetar nada. |
| **Gráficas braille / bloques parciales como btop** | CP437 (font del kernel) y los fontsets de console-setup (`Lat15.256`, `Uni2.512`, `linux.set`, `useful.set`) no tienen braille ni U+2581–2587: se verían como cuadros. Se usa `ACS_BLOCK` por filas. |
| **Depender de 16 colores** | En modo texto VGA una fuente de 512 glifos deja 8 colores (`console-setup(5)`); el diseño con 8 + atributos funciona en ambos casos. |
| **Wrapper dentro de `$BASE_DIR`** | `uninstall_proxmenu` hace `rm -rf "$BASE_DIR"` (`config_menu.sh:821`); un `ExecStart` apuntando a un archivo borrado devolvería 203/EXEC en bucle y agotaría `StartLimitBurst` dejando tty1 **sin login**. En `/usr/local/sbin` sobrevive y, sin TUI, sigue ejecutando `agetty`. |
| **Reproducir el `ExecStart` del fabricante leyéndolo del unit en tiempo de ejecución** | Requiere parsear especificadores (`%I`, `${TERM}`, `\\u`) y comillas en bash en el camino crítico del login. Hardcodear la línea de la 257 (compatible con 252) es más simple y está cubierto por prueba. |
| **`chvt` o reinicio incondicional de `getty@tty1` al instalar** | Mataría la sesión de quien instala desde la consola física. Sólo se reinicia con tty1 ocioso. |
| **pytest** | El repo usa `unittest` puro y runners `-I -B` en todas las lanes; introducir pytest cambiaría la convención de CI por una sola función. |
| **Añadir el usuario de la TUI al grupo `tty`** (propuesto en la auditoría ronda 1) | No hace falta (`initscr()` usa `stdout`/`stdin` heredados; `/dev/tty` es `0666`) y concede escritura sobre todas las terminales `0620 root:tty` del sistema: inyección de texto en sesiones ajenas. Ver §8 R1-4. |
| **Consultar `systemctl show -p ActiveState proxmenux-monitor` desde la TUI para la ventana de boot** | Rompe la regla "la TUI no ejecuta comandos" y `active` no distingue "Flask arrancado sin primer `pvesh`" de "con datos"; el estado `starting` del propio snapshot más `/proc/uptime` lo resuelven sin subprocesos. Ver §8 R1-5. |
| **Lista de procesos del host (tecla `p`) en la vista A** | Fuera del pedido del Owner (VM/CT); leer `/proc/*/stat` de ~1000 PIDs cada segundo es la parte cara de bashtop; expone nombres de procesos root antes del login. Se cubre la necesidad real con el indicador `host other` (§2.4). |
| **Secuencia de reset ampliada `\033[0m\017\033[?25h\033c`** | `ESC c` ya restablece atributos, G0/G1 y cursor en `reset_terminal()` del kernel; los tres prefijos son redundantes y sólo añaden bytes al camino crítico del login. Ver §8 R1-1. |
| **Publicación continua del snapshot (sin lease) con intervalo largo** | Reevaluado en ronda 1 (pregunta 8): un `pvesh` permanente aunque nadie mire sigue costando 6–16 % de un núcleo en N100/P3; la ventana inicial se resuelve con el snapshot `starting` inmediato y la publicación instantánea al aparecer el lease, no con más llamadas. |
| **`AUTOSTART=no` por defecto en la beta** (sugerido en la auditoría) | El Owner pidió autoarranque; se mantiene la pregunta en el instalador con default Sí (Q1) y, para el riesgo upstream, la división en dos PRs. |

## 4. Consecuencias

Positivas:
- Dos PRs: PR1 no toca el arranque del sistema y ya entrega valor (TUI por SSH/TTY como root); PR2 añade tty1 sobre una base ya mergeada y probada.
- Dashboard y consola comparten una sola llamada `pvesh` de recursos cada 5 s (la caché generalizada también simplifica futuros consumidores de nodos/storages).
- La pantalla física del nodo pasa a ser útil sin login y sin añadir dependencias al AppImage.
- Separación limpia: el Monitor sigue siendo el único proceso root que habla con `pvesh`; la TUI es un lector sin privilegios y sin red.
- Coste nulo cuando nadie mira (lease) y acotado cuando sí (1 `pvesh`/5 s, ≤ 2 % CPU de la TUI).
- El login nunca depende de la TUI: toda rama del wrapper termina en `agetty`, y el wrapper sobrevive a la desinstalación de `$BASE_DIR`.
- Render puro y lectores inyectables → pruebas deterministas en CI y en QA remoto sin hardware.

Negativas / deudas asumidas:
- Datos de guests visibles para quien tenga acceso físico antes del login (mitigación: `PRIVACY=ids`; pregunta Q2).
- Las filas de nodos remotos en la vista B se limitan a lo que da `/cluster/resources` (sin red, zram ni temperatura remotas).
- Un cambio upstream en el `ExecStart` del `getty@` (p. ej. nuevos flags de agetty) requeriría actualizar la constante del wrapper; la prueba `test_console_getty_wrapper.py` y una nota en el wrapper lo hacen visible.
- `deluser` en la desinstalación puede dejar el usuario si hay procesos; se informa y no se fuerza.
- El idioma de la consola translitera a ASCII los caracteres que la fuente no tiene (p. ej. eslovaco).
- Nueva superficie de instalación (drop-in, usuario, tmpfiles, dos binarios) en PR2; la biblioteca `console_lifecycle.sh` evita duplicarla entre `install_proxmenux.sh`, `install_proxmenux_beta.sh` y `config_menu.sh`, pero añade un archivo que debe existir en `$LOCAL_SCRIPTS` para que el menú desinstale.
- En PR1 la TUI corre como root cuando se invoca manualmente; es sólo lectura y el invocante ya es root, pero hasta PR2 no hay separación de privilegios.
- `get_cached_pvesh_cluster_resources_vm()` deja de ser la función que llama a `pvesh`: quien lea el código debe seguir una indirección más (documentada en el docstring).

## 5. Qué haría revisar esta decisión

- Que Proxmox o Debian cambien `getty@.service` de forma que `ExecStart=` ya no sea sobreescribible por drop-in o que `agetty` cambie sus argumentos obligatorios → revisar el wrapper o pasar a `agetty --login-program`.
- Que el Monitor deje de correr como root o deje de usar `pvesh` (p. ej. migre a un token de API) → el publicador seguiría al Monitor, el contrato del snapshot no cambia.
- Que el Owner quiera acciones (reiniciar un guest, apagar el nodo) desde la consola → exigiría autenticación en la TUI y un modelo de privilegios distinto; es otro ADR.
- Que aparezca una necesidad de histórico > 60 s o gráficas por nodo remoto → requeriría `rrddata` (coste N llamadas) o persistencia; otro ADR.
- Que la validación en hardware muestre glifos ACS rotos en la fuente real de los P3 → promover `ascii` a default o investigar `setfont` dentro del wrapper (se evitó a propósito por tocar el estado global de la consola).
- Que `python3`/`_curses` dejen de venir en la stdlib de Debian (improbable) → empaquetar `_curses` en el AppImage por ABI como ya se hace con gevent.

## 6. Qué se conserva y qué se corrige del diseño previo (Astra, 2026-10-14)

Se conserva: TUI propia en `curses` dentro del AppImage con `AppRun --tui`; snapshot atómico en `/run/proxmenux-monitor/console.json` con versión de esquema, `observed_at` y estado por fuente; wrapper del `getty@tty1` con marcador por boot root-owned y usuario dedicado sin privilegios; relanzamiento manual; sólo lectura; sanitización de nombres/tags; rollback que retira sólo lo propio; la observación de que `netin/netout` son contadores acumulados de guests.

Se corrige o se precisa:
1. **Métricas locales**: el diseño previo las hacía depender del Monitor con psutil como respaldo "si Monitor tarda"; aquí la TUI las lee **siempre** de `/proc`/`/sys` (sin psutil, sin gevent) y el snapshot sólo trae lo que requiere `pvesh`. Motivo: disponibilidad tras reinicio y presupuesto.
2. **Coste del publicador**: el diseño previo publicaba de continuo; se añade el **lease** para que sin TUI no haya llamadas a `pvesh`.
3. **`rrddata` por nodo remoto cada ~30 s**: eliminado (N llamadas extra, promedios históricos que confunden); la vista B del Owner pide nodos resumidos y basta `/cluster/resources`.
4. **Ubicación del wrapper**: fuera de `$BASE_DIR` por el `rm -rf` de la desinstalación completa (riesgo de tty1 sin login, no contemplado antes).
5. **Nombre del módulo bash propuesto (`scripts/utilities/proxmenux_console_getty.sh`)**: pasa a `scripts/console/` con instalación en `/usr/local/sbin`; `utilities/` contiene herramientas invocadas desde el menú, no infraestructura de arranque.
6. **Consola "Nunca bloquea el teclado esperando `pvesh`"**: queda garantizado por construcción (la TUI no ejecuta `pvesh`), no por timeouts.
7. **Compacto 80×24 "y caracteres compatibles"**: se concreta el juego de glifos con fuentes verificadas (CP437, `linux.set`, `useful.set`) y tres modos; braille y bloques parciales quedan expresamente fuera.
8. **Activación inmediata al instalar**: se define la condición exacta (sin sesión en tty1 y `agetty` ocioso) y se descarta `chvt`.
9. **Vista**: ya no es la cabecera/gráficas/tabla genérica: son las vistas A y B validadas por el Owner, con las listas top CPU/MEM por núcleos de host.

## 7. Fuentes

Ver §9 del grill (`.notes/docs/specs/2026-10-17-console-tui-grill.md`). Añadidas en la revisión 2: `drivers/tty/vt/vt.c` y `drivers/tty/vt/keyboard.c` (Linux v6.14, kernel.org: `reset_terminal()`, `vt_reset_keyboard()`); `ncurses/base/lib_initscr.c` (mirror oficial: `newterm(name, stdout, stdin)`, `/dev/tty` sólo bajo `__CYGWIN__`); systemd `rules.d/50-udev-default.rules.in` v257 (modos de `tty`, `tty[0-9]*`, `ptmx`); pve-manager `PVE/API2/Cluster.pm` y `PVE/API2Tools.pm` (handler `resources`, `extract_vm_stats`); docs de CPython 3.13 `using/cmdline` (`-I`, `-P`); `setpriv(1)` (opciones usadas). Las cifras del repo citadas aquí (`flask_server.py:1766, 2759-2800, 5891-5897, 2499-2508, 122-154`; `install_proxmenux.sh:528, 608, 646, 625-640`; `install_proxmenux_beta.sh:415, 524`; `config_menu.sh:95-137, 403-440, 578-612, 821`; `utils.sh:228-277`; `build_appimage.sh:128-148, 178-181`; `notification_templates.py:25-73`) se verificaron en el worktree el 2026-10-17.

## 8. Registro de adjudicación — auditoría ronda 1

Auditor: Gemini 3.8 Flash vía `agy` (lectura), recibo `.notes/docs/analysis/2026-10-17-console-tui-redteam-verdict-r1.md`. Veredicto: APROBAR CON CAMBIOS en ambos ejes (spec 0C/2I/1m; riesgo 1C/3I/2m). Cada hallazgo se adjudicó contra fuentes primarias o el repo; nada se aceptó por autoridad.

| # | Eje | Sev. | Hallazgo | Decisión | Razón / evidencia | Dónde se corrige |
|---|---|---|---|---|---|---|
| R1-1 | riesgo | Critical | VT corrupto (G1 activo, cursor oculto) tras salida abrupta de la TUI; `stty sane` no basta | **Aceptado, modificado** | Correcto en el fondo: `stty sane` sólo toca termios y `agetty --noreset --noclear` no repara el emulador. Verificado en `reset_terminal()` de `drivers/tty/vt/vt.c` (v6.14) que `ESC c` restablece `Gx_charset[0]=LAT1_MAP`, `Gx_charset[1]=GRAF_MAP`, `charset=0`, `default_attr()`, `vc_deccm=global_cursor_default`, `vc_cursor_type`, tabuladores y `vt_reset_keyboard()` (modos/LEDs, no el keymap), y **no** llama a `set_palette` ni `con_font_set` (fuente y paleta de console-setup se conservan). Por eso se adopta `stty sane` + `ESC c` y se omite el prefijo `\033[0m\017\033[?25h` propuesto: redundante con RIS (SGR0 = `default_attr`, SI = `charset=0`, DECTCEM = `vc_deccm`), y una secuencia de 2 bytes no tiene fallo "a medio camino". Limitación anotada: `vc_utf=default_utf8` (Debian/PVE compilan `1`). Se añade la conducta de la TUI (handlers → bandera → `endwin`) y una aserción de bytes en el test del wrapper. | §2.5 paso 9, tabla "Qué hace cada salida", §2.4 tabla de teclas, §2.8 (wrapper, smoke, aceptación 2b) |
| R1-2 | riesgo | Important | Doble `pvesh` cada 5 s (dashboard + publicador) | **Aceptado** | Verificado en `PVE/API2/Cluster.pm` (pve-manager HEAD): `rrd_dump`, `get_vmlist`, `get_nodelist`, `get_members` y el estado HA se leen siempre; `type` sólo gobierna qué bloques de entradas se construyen; las entradas de guest salen de `API2Tools::extract_vm_stats($vmid,$data,$rrd)` sin depender de `type`. Una llamada sin `--type` cuesta lo mismo en lo caro y devuelve guests idénticos. Se generaliza la caché existente y la función `_vm` pasa a ser un filtro; 13 call sites internos + 3 externos no cambian; prueba de caracterización con JSON grabados. Tarea propia P1-T3 (un commit). | §2.1, §2.2 tabla, §2.7, §2.8 fila "Caché compartida", §2.9 P1-T3 |
| R1-3 | riesgo | Important | `ModuleNotFoundError` bajo `python3 -I` (`-P` quita el directorio del script) | **Aceptado** | Confirmado en docs de CPython (`using/cmdline`: `-I` implica `-E -P -s`; `-P` "don't prepend the script's directory"). Se especifican las líneas exactas con `os.path.realpath(__file__)` y un smoke test ejecutado desde otro cwd con `-I`. Riesgo de hijack nulo mientras `monitor-app/` sea root-owned (lo extrae el instalador como root). | §2.7 (`sys.path`), §2.8 smoke, §2.9 P1-T5 |
| R1-4 | riesgo | Important | Añadir `proxmenux-console` al grupo `tty` porque curses "abre `/dev/tty`" | **Rechazado** | Falso y dañino. (1) `initscr()` llama `newterm(name, stdout, stdin)`; el único `fopen("/dev/tty")` de `ncurses/base/lib_initscr.c` está dentro de `#ifdef __CYGWIN__`. (2) La TUI hereda los fd 0/1/2 que systemd abrió (`StandardInput=tty`); los ioctl termios sobre un fd ya abierto no dependen del modo del nodo de dispositivo. (3) Si algo abriera `/dev/tty`, ese nodo es `0666 root:tty` (udev `50-udev-default.rules`: `KERNEL=="tty", GROUP="tty", MODE="0666"`) y basta tener terminal de control. (4) `tty[0-9]*` y los pts son `0620 root:tty`: pertenecer a `tty` daría escritura sobre todas las terminales ajenas (inyección de texto en sesiones). Se documenta la exclusión explícita. | §2.2 tabla, §2.5 tabla de artefactos, §3 nueva fila |
| R1-5 | spec | Important | Vacío de datos / cabecera roja durante la ventana de boot | **Aceptado, modificado** | Válido: el getty arranca antes de que Flask haga su primer `pvesh`. Se define el estado `STARTING` (`warn`, no `crit`) decidido sin privilegios ni subprocesos: `/proc/uptime` < 120 s sin archivo, o snapshot con `resources.status="starting"` fresco; el publicador escribe ese snapshot inicial nada más arrancar y publica de inmediato al ver el lease. Se rechaza consultar `systemctl show -p ActiveState`: la TUI no ejecuta comandos, y `ActiveState=active` no distingue "arrancado pero sin primer `pvesh`" de "con datos", cosa que el estado `starting` del propio snapshot sí hace. Umbral 120 s en lugar de los 30/60 s sugeridos: cubre un Monitor que tarda en importar (gevent, Flask, hilos) en nodos lentos sin esconder una caída real más de dos minutos. | §2.2 tabla (fila "Arranque del nodo"), §2.3 (snapshot inicial, publicación inmediata, estado `starting`), §2.4 cabecera, §2.8 smoke y aceptación 1 |
| R1-6 | spec | Important | Sin procesos de host, un nodo saturado por `vzdump`/scrub deja las listas TOP "tranquilas" | **Aceptado, modificado** | Se añade el indicador barato `host other` (CPU: `cpu_total×threads − Σ cpu×maxcpu`; memoria: `used − ARC − Σ mem`) en el título de cada lista TOP, sólo con snapshot fresco. La lista de procesos del host queda fuera de alcance: el Owner pidió VM/CT, leer `/proc/*/stat` de ~1000 PIDs/s es la parte cara de bashtop y expondría nombres de procesos root antes del login; tecla `p` reservada para un ADR futuro. | §2.4 (indicador), §2.9 P1-T1/P1-T4, grill §6 |
| R1-7 | spec | Minor | TOP MEMORY excluye guests `paused`/en migración | **Aceptado, modificado** | En `/cluster/resources` una VM pausada sigue con `status=running` (el `qmpstatus` no viaja en ese endpoint), así que el filtro exacto propuesto no capta el caso; la regla robusta es incluir todo guest con `mem > 0` sea cual sea su `status` (y excluir plantillas), mostrando el estado en la fila. TOP CPU usa `cpu×maxcpu > 0`. | §2.4 semántica TOP, §2.8 render |
| R1-8 | riesgo | Minor | Sin repintado periódico, los `printk` en tty1 quedan en pantalla hasta pulsar `r` | **Aceptado, modificado** | Repintado completo (`clearok(True)` + `refresh`) cada **30 s** (el Auditor propuso 60 s): un frame completo cuesta milisegundos y 30 s acota la basura visible por debajo del tiempo de reacción de un operador que se acerca a la pantalla. No se toca `loglevel` ni la consola del kernel. | §2.4 "Repintado completo periódico", §2.9 P1-T5, aceptación 2c |
| R1-9 | riesgo | Minor | `remove_console_tui` duplicada o cargada "a decisión del implementador" entre instaladores y menú | **Aceptado** | Biblioteca única `scripts/console/console_lifecycle.sh`: los instaladores la cargan desde el clon (`$TEMP_DIR/scripts/console/…`) y `config_menu.sh` desde `$LOCAL_SCRIPTS/console/…` (llega con el `cp -r ./scripts/*`, `install_proxmenux.sh:877`; el menú ya carga `global/pmx_journal.sh` así, `config_menu.sh:64-65`), **antes** del `rm -rf "$BASE_DIR"` (`:821`). Si falta en `$LOCAL_SCRIPTS`, ninguna versión con consola instaló nada: se omite con aviso. | §2.5 tabla de artefactos y "Desinstalar", §2.9 P2-T2/P2-T3/P2-T5 |
| R1-Q9 | — | — | Dividir en dos PRs upstream | **Aceptado (decisión del Coordinador)** | PR1 = TUI + comando manual root (riesgo cero para el arranque; validable por SSH); PR2 = tty1 (usuario, wrapper, drop-in, instaladores, menú). Se define cómo corre PR1 como root (sin `setpriv`; `console.json` `0600`/`0700`; lógica de grupo latente) y cómo PR2 lo cambia. Q1 se mantiene (pregunta al instalar, default Sí) en PR2: el Owner quiere autoarranque; no se adopta el `AUTOSTART=no` por defecto sugerido para la beta. | Cabecera, §2.1, §2.2, §2.3, §2.5, §2.9 |

Respuestas del Auditor a las preguntas 1–17 que no generaron hallazgo (5 lease/TOCTOU, 6 `setpriv` en util-linux 2.38/2.41, 7 orden de instalación, 10 concurrencia, 11 seguridad, 12 matriz de fallos, 13 coste, 14 rollback, 15 pantalla, 16 testabilidad) se dan por confirmadas; de la 5 se incorpora `lstat`+`S_ISREG` (§2.3) y de la 16 el test de `kill -9` como paso de aceptación 2b (§2.8). La ronda 2 debe revisar de forma acotada: §2.5 paso 9, §2.7 (`sys.path` y caché compartida), §2.2/§2.3 (`starting`), §2.4 (TOP, `host other`, repintado) y §2.9.

---

## (B) Subárbol añadido a AppImage/messages/en/common.json (runtime.consoleTui, legible)

El diff real de ese archivo es una sola línea minificada; esto es lo que se añadió dentro de `runtime`:

```json
{
  "title": "PROXMENUX MONITOR",
  "tabs": {
    "node": "Node",
    "cluster": "Cluster"
  },
  "panels": {
    "cpu": "CPU",
    "memory": "MEMORY",
    "network": "NET",
    "disk": "DISK",
    "cluster": "CLUSTER",
    "topCpu": "TOP CPU (host cores)",
    "topMemory": "TOP MEMORY",
    "nodes": "NODES",
    "guests": "GUESTS"
  },
  "labels": {
    "load": "load",
    "threads": "thr",
    "temp": "temp",
    "arc": "ARC",
    "available": "avail",
    "zram": "ZRAM",
    "swap": "swap",
    "rx": "RX",
    "tx": "TX",
    "read": "R",
    "write": "W",
    "used": "used",
    "uptime": "up",
    "hostCores": "host cores",
    "quorumOk": "QUORUM OK",
    "noQuorum": "NO QUORUM",
    "standalone": "STANDALONE",
    "coverage": "coverage",
    "online": "online",
    "offline": "offline",
    "unknown": "unknown",
    "running": "running",
    "stopped": "stopped",
    "sort": "sort",
    "moreNodes": "+{count} nodes",
    "moreGuests": "+{count} guests",
    "hostOther": "host other"
  },
  "status": {
    "data": "data {age}",
    "noData": "no data",
    "stale": "{age} old",
    "starting": "Monitor starting...",
    "waitingForData": "waiting for data",
    "monitorUnavailable": "MONITOR UNAVAILABLE",
    "terminalTooSmall": "Terminal too small (80x24)"
  },
  "columns": {
    "type": "TYPE",
    "id": "ID",
    "name": "NAME",
    "node": "NODE",
    "status": "STATUS",
    "cpu": "CPU",
    "mem": "MEM",
    "root": "ROOT",
    "guests": "GUESTS",
    "up": "UP",
    "cores": "CORES",
    "uptime": "UPTIME",
    "ha": "HA",
    "tags": "TAGS"
  },
  "footer": {
    "quit": "q login",
    "switch": "Tab cluster",
    "refresh": "r refresh",
    "glyphs": "a glyphs",
    "otherTty": "Ctrl+Alt+F2",
    "sort": "c/m sort"
  }
}
```

---

## (C) Diff completo del PR1 (origin/develop...HEAD), salvo la línea minificada de en/common.json

```diff
diff --git a/.github/workflows/test-console-tui.yml b/.github/workflows/test-console-tui.yml
new file mode 100644
index 00000000..60b26351
--- /dev/null
+++ b/.github/workflows/test-console-tui.yml
@@ -0,0 +1,69 @@
+name: Console TUI tests
+
+# Not in CI: AppImage/scripts/tests/test_console_tui_smoke.py drives the TUI
+# through a pty with real curses and runs on a QA host instead.
+
+on:
+  pull_request:
+    paths:
+      - '.github/workflows/test-console-tui.yml'
+      - 'AppImage/scripts/console_*.py'
+      - 'AppImage/scripts/flask_server.py'
+      - 'AppImage/scripts/tests/test_console_metrics.py'
+      - 'AppImage/scripts/tests/test_console_snapshot.py'
+      - 'AppImage/scripts/tests/test_console_render.py'
+      - 'AppImage/scripts/tests/test_cluster_resources_cache.py'
+      - 'AppImage/scripts/tests/test_cluster_guest_storage_ownership.py'
+      - 'AppImage/scripts/tests/test_lxc_vm_ip_inventory.py'
+      - 'AppImage/scripts/tests/fixtures/console/**'
+      - 'tests/test_console_installer.py'
+      - 'scripts/console/**'
+  push:
+    branches: [main, develop]
+    paths:
+      - '.github/workflows/test-console-tui.yml'
+      - 'AppImage/scripts/console_*.py'
+      - 'AppImage/scripts/flask_server.py'
+      - 'AppImage/scripts/tests/test_console_metrics.py'
+      - 'AppImage/scripts/tests/test_console_snapshot.py'
+      - 'AppImage/scripts/tests/test_console_render.py'
+      - 'AppImage/scripts/tests/test_cluster_resources_cache.py'
+      - 'AppImage/scripts/tests/test_cluster_guest_storage_ownership.py'
+      - 'AppImage/scripts/tests/test_lxc_vm_ip_inventory.py'
+      - 'AppImage/scripts/tests/fixtures/console/**'
+      - 'tests/test_console_installer.py'
+      - 'scripts/console/**'
+  workflow_dispatch:
+
+permissions:
+  contents: read
+
+jobs:
+  console-tui:
+    runs-on: ubuntu-latest
+    timeout-minutes: 10
+    strategy:
+      fail-fast: false
+      matrix:
+        python-version: ['3.11', '3.13']
+    steps:
+      - uses: actions/checkout@v4
+        with:
+          persist-credentials: false
+      - uses: actions/setup-python@v5
+        with:
+          python-version: ${{ matrix.python-version }}
+      - name: Test console metrics sampler
+        run: python -I -B -m unittest discover -s AppImage/scripts/tests -p test_console_metrics.py -v
+      - name: Test console snapshot publisher
+        run: python -I -B -m unittest discover -s AppImage/scripts/tests -p test_console_snapshot.py -v
+      - name: Test console renderer
+        run: python -I -B -m unittest discover -s AppImage/scripts/tests -p test_console_render.py -v
+      - name: Test cluster resources cache
+        run: python -I -B -m unittest discover -s AppImage/scripts/tests -p test_cluster_resources_cache.py -v
+      - name: Test console installer
+        run: python -I -B -m unittest discover -s tests -p test_console_installer.py -v
+      - name: Test cluster guest storage ownership guard
+        run: python -I -B -m unittest discover -s AppImage/scripts/tests -p test_cluster_guest_storage_ownership.py -v
+      - name: Test LXC/VM IP inventory guard
+        run: python -I -B -m unittest discover -s AppImage/scripts/tests -p test_lxc_vm_ip_inventory.py -v
diff --git a/AppImage/README.md b/AppImage/README.md
index 8f51ab23..6c87e5e1 100644
--- a/AppImage/README.md
+++ b/AppImage/README.md
@@ -10,6 +10,7 @@ A modern, responsive dashboard for monitoring Proxmox VE systems built with Next
 - [Features](#features)
 - [Technology Stack](#technology-stack)
 - [Installation](#installation)
+- [Console Monitor (TUI)](#console-monitor-tui)
 - [Authentication & Security](#authentication--security)
   - [Setup Authentication](#setup-authentication)
   - [Two-Factor Authentication (2FA)](#two-factor-authentication-2fa)
@@ -173,6 +174,28 @@ ProxMenux Monitor includes built-in support for reverse proxy configurations. If
 - Adjust API endpoints to work correctly through the proxy
 - Maintain full functionality for all features including authentication and API access
 
+## Console Monitor (TUI)
+
+A read-only text dashboard for the node's physical console or any root shell. It shows the node and cluster state: CPU, memory, network, disks, quorum and coverage, the top guests by CPU and memory, and a guest table. On an installed node, run `proxmenux-console` as root.
+
+```bash
+./ProxMenux-<version>.AppImage --tui                  # interactive
+./ProxMenux-<version>.AppImage --tui --once           # render one frame and exit
+./ProxMenux-<version>.AppImage --tui --glyphs ascii   # acs | unicode | ascii
+```
+
+It reads the snapshot published by the Monitor in `/run/proxmenux-monitor/console.json`. It does not call `pvesh` and does not change any configuration.
+
+| Key | Action |
+|---|---|
+| `q` | Quit |
+| `Tab` | Switch between Node and Cluster views |
+| `r` | Refresh now |
+| `a` | Cycle glyph set (`acs` → `ascii` → `unicode`) |
+| `c` / `m` | Sort the guest table by CPU or memory (Cluster view) |
+
+The text is sanitized before it is drawn, so guest names and tags cannot inject terminal escape sequences. Labels follow the language set in ProxMenux.
+
 
 
 ## Authentication & Security
diff --git a/AppImage/scripts/AppRun b/AppImage/scripts/AppRun
index 74ad4d22..75a0f330 100644
--- a/AppImage/scripts/AppRun
+++ b/AppImage/scripts/AppRun
@@ -9,6 +9,12 @@ APPDIR="$(dirname "$(readlink -f "${0}")")"
 export PATH="${APPDIR}/usr/bin:${PATH}"
 export LD_LIBRARY_PATH="${APPDIR}/usr/lib/x86_64-linux-gnu:${APPDIR}/usr/lib:${APPDIR}/lib/x86_64-linux-gnu:${APPDIR}/lib:${LD_LIBRARY_PATH}"
 
+# Console monitor (TUI). Handled before the Python ABI selection below, whose
+# warnings would dirty the console; `-I` ignores PYTHONPATH anyway.
+if [[ "$1" == "--tui" ]]; then
+    exec python3 -I "${APPDIR}/usr/bin/console_tui.py" "${@:2}"
+fi
+
 # Select the native dependency tree matching the host interpreter.  PVE 8
 # uses CPython 3.11 and PVE 9 uses CPython 3.13; both runtimes are bundled in
 # the same AppImage so its gevent HTTPS/WSS server is independent of the
diff --git a/AppImage/scripts/build_appimage.sh b/AppImage/scripts/build_appimage.sh
index dcdef597..3be0e53e 100755
--- a/AppImage/scripts/build_appimage.sh
+++ b/AppImage/scripts/build_appimage.sh
@@ -127,6 +127,10 @@ cd "$SCRIPT_DIR"
 echo "📋 Copying Flask server..."
 cp "$SCRIPT_DIR/flask_server.py" "$APP_DIR/usr/bin/"
 cp "$SCRIPT_DIR/zram_metrics.py" "$APP_DIR/usr/bin/"
+cp "$SCRIPT_DIR/console_tui.py" "$APP_DIR/usr/bin/"
+cp "$SCRIPT_DIR/console_render.py" "$APP_DIR/usr/bin/"
+cp "$SCRIPT_DIR/console_metrics.py" "$APP_DIR/usr/bin/"
+cp "$SCRIPT_DIR/console_snapshot.py" "$APP_DIR/usr/bin/"
 cp "$SCRIPT_DIR/flask_auth_routes.py" "$APP_DIR/usr/bin/" 2>/dev/null || echo "⚠️  flask_auth_routes.py not found"
 cp "$SCRIPT_DIR/auth_manager.py" "$APP_DIR/usr/bin/" 2>/dev/null || echo "⚠️  auth_manager.py not found"
 cp "$SCRIPT_DIR/jwt_middleware.py" "$APP_DIR/usr/bin/" 2>/dev/null || echo "⚠️  jwt_middleware.py not found"
diff --git a/AppImage/scripts/console_metrics.py b/AppImage/scripts/console_metrics.py
new file mode 100644
index 00000000..060acdc7
--- /dev/null
+++ b/AppImage/scripts/console_metrics.py
@@ -0,0 +1,287 @@
+"""Read local console metrics from procfs and sysfs."""
+
+import os
+from collections import deque
+from pathlib import Path
+
+from zram_metrics import read_zram_metrics
+
+
+def _read_text(path):
+    try:
+        return path.read_text(encoding="utf-8")
+    except (OSError, UnicodeError):
+        return ""
+
+
+def _children(path):
+    try:
+        return sorted(path.iterdir())
+    except OSError:
+        return []
+
+
+def _cpu_counters(root):
+    counters = {}
+    for line in _read_text(root / "proc/stat").splitlines():
+        fields = line.split()
+        if not fields or not fields[0].startswith("cpu"):
+            continue
+        try:
+            values = [int(value) for value in fields[1:9]]
+        except ValueError:
+            continue
+        if len(values) < 4:
+            continue
+        # Guest time is already included in user/nice; do not add it again.
+        counters[fields[0]] = (sum(values), values[3] + (values[4] if len(values) > 4 else 0))
+    return counters
+
+
+def _cpu_fraction(previous, current):
+    if previous is None or current is None:
+        return None
+    total = current[0] - previous[0]
+    idle = current[1] - previous[1]
+    if total <= 0 or idle < 0 or idle > total:
+        return None
+    return max(0.0, min(1.0, (total - idle) / total))
+
+
+def _memory(root):
+    values = {}
+    for line in _read_text(root / "proc/meminfo").splitlines():
+        name, separator, remainder = line.partition(":")
+        if not separator:
+            continue
+        try:
+            values[name] = int(remainder.split()[0]) * 1024
+        except (IndexError, ValueError):
+            continue
+
+    total = values.get("MemTotal", 0)
+    available = values.get("MemAvailable", values.get("MemFree", 0))
+    arc = 0
+    for line in _read_text(root / "proc/spl/kstat/zfs/arcstats").splitlines():
+        fields = line.split()
+        if len(fields) >= 3 and fields[0] == "size":
+            try:
+                arc = max(0, int(fields[2]))
+            except ValueError:
+                pass
+            break
+    # MemAvailable already accounts for reclaimable memory. ARC remains
+    # inside used so the renderer can show it separately and host_other can
+    # subtract it exactly once.
+    return {
+        "total": total,
+        "used": max(0, total - available),
+        "available": available,
+        "cached": values.get("Cached", 0) + values.get("SReclaimable", 0),
+        "arc": arc,
+        "swap_used": max(0, values.get("SwapTotal", 0) - values.get("SwapFree", 0)),
+        "swap_total": values.get("SwapTotal", 0),
+    }
+
+
+def _temperature(root):
+    sensors = []
+    for hwmon in _children(root / "sys/class/hwmon"):
+        name = _read_text(hwmon / "name").strip()
+        priority = {"k10temp": 0, "coretemp": 1}.get(name, 2)
+        for input_path in hwmon.glob("temp*_input"):
+            try:
+                temp = int(_read_text(input_path).strip()) / 1000
+            except ValueError:
+                continue
+            if not -40 <= temp <= 150:
+                continue
+            label = _read_text(input_path.with_name(input_path.name.replace("_input", "_label"))).strip()
+            label_priority = 0 if label.lower() in ("tdie", "package id 0") else 1
+            sensors.append((priority, label_priority, hwmon.name, input_path.name, temp))
+    if sensors:
+        return min(sensors)[-1]
+    for zone in _children(root / "sys/class/thermal"):
+        if not zone.name.startswith("thermal_zone"):
+            continue
+        try:
+            temp = int(_read_text(zone / "temp").strip()) / 1000
+        except ValueError:
+            continue
+        if -40 <= temp <= 150:
+            return temp
+    return None
+
+
+def _network(root):
+    counters = {}
+    ignored = ("vmbr", "fwbr", "fwpr", "fwln", "tap", "veth", "bond")
+    for line in _read_text(root / "proc/net/dev").splitlines()[2:]:
+        name, separator, data = line.partition(":")
+        if not separator:
+            continue
+        name = name.strip()
+        if name == "lo" or name.startswith(ignored):
+            continue
+        device = root / "sys/class/net" / name
+        if not (device / "device").exists():
+            continue
+        try:
+            fields = [int(field) for field in data.split()]
+        except ValueError:
+            continue
+        if len(fields) >= 9:
+            counters[name] = (fields[0], fields[8])
+    return counters
+
+
+def _disk(root):
+    counters = {}
+    for line in _read_text(root / "proc/diskstats").splitlines():
+        fields = line.split()
+        if len(fields) < 10:
+            continue
+        name = fields[2]
+        # /sys/block contains whole devices; a device/ link excludes virtual
+        # block devices (loop, dm, md, zram) that can double count IO.
+        if not (root / "sys/block" / name / "device").exists():
+            continue
+        try:
+            counters[name] = (int(fields[5]) * 512, int(fields[9]) * 512)
+        except ValueError:
+            continue
+    return counters
+
+
+def _rates(previous, current, dt):
+    if previous is None or dt <= 0:
+        return 0, 0
+    first = second = 0
+    for name, values in current.items():
+        old = previous.get(name)
+        if old is None:
+            continue
+        if values[0] >= old[0]:
+            first += values[0] - old[0]
+        if values[1] >= old[1]:
+            second += values[1] - old[1]
+    return first / dt, second / dt
+
+
+def _speed(root, nic):
+    if nic is None:
+        return None
+    try:
+        value = int(_read_text(root / "sys/class/net" / nic / "speed").strip())
+    except ValueError:
+        return None
+    return value if value > 0 else None
+
+
+def _zram(root):
+    metrics = read_zram_metrics(root / "sys/block", root / "proc/swaps")
+    if not metrics["available"]:
+        return None
+    devices = metrics["devices"]
+    original = sum(device["original_bytes"] for device in devices)
+    compressed = sum(device["compressed_bytes"] for device in devices)
+    return {
+        "devices": devices,
+        "capacity_bytes": sum(device["capacity_bytes"] for device in devices),
+        "swap_used_bytes": sum(device["swap_used_bytes"] for device in devices),
+        "compression_ratio": original / compressed if compressed else None,
+    }
+
+
+class LocalSampler:
+    """Keep counter baselines while sampling an injectable proc/sys root."""
+
+    def __init__(self, proc_root="/", storage_paths=None, statvfs=None):
+        self.root = Path(proc_root)
+        self.storage_paths = [("root", "/")] if storage_paths is None else storage_paths
+        self.statvfs = os.statvfs if statvfs is None else statvfs
+        self.previous_time = None
+        self.previous_cpu = None
+        self.previous_net = None
+        self.previous_disk = None
+        self.cpu_history = deque(maxlen=60)
+
+    def _storages(self):
+        storages = []
+        for name, path in self.storage_paths:
+            rooted = self.root / str(path).lstrip("/")
+            try:
+                status = self.statvfs(str(rooted))
+            except OSError:
+                continue
+            total = status.f_blocks * status.f_frsize
+            storages.append({
+                "storage": name,
+                "total": total,
+                "used": max(0, total - status.f_bfree * status.f_frsize),
+                "available": status.f_bavail * status.f_frsize,
+            })
+        return storages
+
+    def sample(self, now):
+        cpu = _cpu_counters(self.root)
+        net = _network(self.root)
+        disk = _disk(self.root)
+        dt = now - self.previous_time if self.previous_time is not None else 0
+        cpu_total = _cpu_fraction(self.previous_cpu.get("cpu") if self.previous_cpu else None,
+                                  cpu.get("cpu")) if dt > 0 else None
+        per_core = [
+            (_cpu_fraction(self.previous_cpu.get(name) if self.previous_cpu else None, cpu[name])
+             or 0) if dt > 0 else 0
+            for name in sorted((name for name in cpu if name != "cpu"),
+                               key=lambda name: int(name[3:]) if name[3:].isdigit() else name)
+        ]
+        if cpu_total is not None:
+            self.cpu_history.append(cpu_total)
+        rx_bps, tx_bps = _rates(self.previous_net, net, dt)
+        read_bps, write_bps = _rates(self.previous_disk, disk, dt)
+        nic = next((name for name in sorted(net)
+                    if _read_text(self.root / "sys/class/net" / name / "operstate").strip() == "up"),
+                   next(iter(sorted(net)), None))
+        loads = _read_text(self.root / "proc/loadavg").split()
+        try:
+            load1, load5, load15 = map(float, loads[:3])
+        except ValueError:
+            load1 = load5 = load15 = 0.0
+        if len(loads) < 3:
+            load1 = load5 = load15 = 0.0
+        result = {
+            "cpu_total": cpu_total,
+            "cpu_history": list(self.cpu_history),
+            "per_core": per_core,
+            "load1": load1, "load5": load5, "load15": load15,
+            "threads": len(per_core),
+            "temp_c": _temperature(self.root),
+            "mem": _memory(self.root),
+            "zram": _zram(self.root),
+            "net": {"rx_bps": rx_bps, "tx_bps": tx_bps, "nic": nic,
+                    "speed_mbps": _speed(self.root, nic)},
+            "disk": {"read_bps": read_bps, "write_bps": write_bps},
+            "storages_local": self._storages(),
+        }
+        if self.previous_time is None or dt > 0:
+            self.previous_time = now
+            self.previous_cpu = cpu
+            self.previous_net = net
+            self.previous_disk = disk
+        return result
+
+
+def host_other(local, guests, node, snapshot_fresh):
+    """Return residual local CPU cores and bytes outside guests."""
+    if not snapshot_fresh:
+        return {"cpu": None, "mem": None}
+    local_guests = [guest for guest in guests if guest.get("node") == node]
+    running = [guest for guest in local_guests if guest.get("status") == "running"]
+    cpu = local.get("cpu_total")
+    cpu_other = None if cpu is None else max(0.0, cpu * local.get("threads", 0) - sum(
+        (guest.get("cpu") or 0) * (guest.get("maxcpu") or 0) for guest in running))
+    memory = local.get("mem", {})
+    mem_other = max(0, memory.get("used", 0) - memory.get("arc", 0) - sum(
+        guest.get("mem") or 0 for guest in local_guests))
+    return {"cpu": cpu_other, "mem": mem_other}
diff --git a/AppImage/scripts/console_render.py b/AppImage/scripts/console_render.py
new file mode 100644
index 00000000..9aa7fd38
--- /dev/null
+++ b/AppImage/scripts/console_render.py
@@ -0,0 +1,630 @@
+"""Pure terminal grid renderer for the ProxMenux console.
+
+Each cell is ``(char, color_role, attrs)``. ``attrs`` is a tuple containing
+``dim``, ``reverse`` or ``bold`` when needed. In ACS mode ``char`` can be one
+of the logical tokens below. The curses caller must map ``ACS_TOKENS[token]``
+to the corresponding ``curses.ACS_*`` value before calling ``addch``; all
+other cells contain one printable character. This module never imports curses.
+"""
+
+import datetime
+import math
+import re
+import unicodedata
+
+
+ACS_TOKENS = {
+    "BOX_H": "ACS_HLINE", "BOX_V": "ACS_VLINE",
+    "BOX_UL": "ACS_ULCORNER", "BOX_UR": "ACS_URCORNER",
+    "BOX_LL": "ACS_LLCORNER", "BOX_LR": "ACS_LRCORNER",
+    "BOX_LT": "ACS_LTEE", "BOX_RT": "ACS_RTEE",
+    "BOX_TT": "ACS_TTEE", "BOX_BT": "ACS_BTEE",
+    "BOX_X": "ACS_PLUS", "BAR_FULL": "ACS_BLOCK",
+    "BAR_EMPTY": "ACS_CKBOARD", "BULLET": "ACS_BULLET",
+}
+
+_UNICODE = dict(zip(ACS_TOKENS, "─│┌┐└┘├┤┬┴┼█░·"))
+_ASCII = {key: ("-" if key == "BOX_H" else "|" if key == "BOX_V"
+                else "#" if key == "BAR_FULL" else "." if key in ("BAR_EMPTY", "BULLET")
+                else "+") for key in ACS_TOKENS}
+_ROLES = frozenset(("title", "text", "dim", "ok", "warn", "crit", "selected"))
+_CONTROL = re.compile(r" +")
+_ANSI = re.compile(r"\x1b\[[0-9;]*[A-Za-z]")
+
+
+def sanitize(text, max_cells, glyphs="acs"):
+    """Transliterate untrusted text into bounded, printable terminal cells."""
+    if text is None or max_cells <= 0:
+        return ""
+    normalized = unicodedata.normalize("NFKD", _ANSI.sub("", str(text)))
+    printable = []
+    for char in normalized:
+        if unicodedata.combining(char):
+            continue
+        if ord(char) < 32 or ord(char) == 127:
+            continue
+        printable.append(char if " " <= char <= "~" else "?")
+    cleaned = _CONTROL.sub(" ", "".join(printable)).strip()
+    if len(cleaned) > max_cells:
+        marker = "…" if glyphs == "unicode" else "~"
+        return cleaned[:max_cells - 1] + marker
+    return cleaned
+
+
+def _number(value, default=0.0):
+    try:
+        result = float(value)
+    except (TypeError, ValueError):
+        return default
+    return result if math.isfinite(result) else default
+
+
+def _pct(value):
+    return max(0, min(100, round(_number(value) * 100)))
+
+
+def _percent(used, total):
+    total = _number(total)
+    return 0 if total <= 0 else max(0, min(100, round(_number(used) * 100 / total)))
+
+
+def _gib(value):
+    return _number(value) / 1073741824
+
+
+def _size(value):
+    value = max(0, _number(value))
+    if value < 1073741824:
+        return f"{value / 1048576:.0f} MiB"
+    return f"{_gib(value):.1f} GiB"
+
+
+def _uptime(seconds):
+    seconds = max(0, int(_number(seconds)))
+    days, hours = divmod(seconds // 3600, 24)
+    return f"{days}d {hours:02d}h" if days else f"{hours}h"
+
+
+def _age(seconds):
+    seconds = max(0, int(_number(seconds)))
+    if seconds < 60:
+        return f"{seconds}s"
+    if seconds < 3600:
+        return f"{seconds // 60}m"
+    return f"{seconds // 3600}h"
+
+
+def _rate(value, bits=False):
+    value = max(0, _number(value)) * (8 if bits else 1)
+    units = ("b/s", "Kb/s", "Mb/s", "Gb/s") if bits else ("B/s", "KiB/s", "MiB/s", "GiB/s")
+    for unit in units:
+        base = 1000 if bits else 1024
+        if value < base or unit == units[-1]:
+            return f"{value:.1f} {unit}"
+        value /= base
+
+
+def _paired_rate(first, second):
+    values = [max(0, _number(first)), max(0, _number(second))]
+    unit = "B/s"
+    for next_unit in ("KiB/s", "MiB/s", "GiB/s"):
+        if max(values) < 1024:
+            break
+        values = [value / 1024 for value in values]
+        unit = next_unit
+    return f"{values[0]:.0f}/{values[1]:.0f} {unit}"
+
+
+class Grid(list):
+    """List of rows of logical ``(char, color_role, attrs)`` cells."""
+
+    def __init__(self, width, height, glyphs):
+        super().__init__([[(' ', 'text', ()) for _ in range(width)] for _ in range(height)])
+        self.width = width
+        self.height = height
+        self.glyphs = glyphs
+
+    def glyph(self, token):
+        if self.glyphs == "acs":
+            return token
+        return (_UNICODE if self.glyphs == "unicode" else _ASCII)[token]
+
+    def put(self, x, y, value, role="text", attrs=(), limit=None):
+        if not 0 <= y < self.height or x >= self.width:
+            return
+        assert role in _ROLES
+        attrs = tuple(attrs)
+        max_x = self.width if limit is None else min(self.width, limit)
+        for index, char in enumerate(str(value)):
+            at = x + index
+            if 0 <= at < max_x:
+                self[y][at] = (char, role, attrs)
+
+    def token(self, x, y, token, role="text", attrs=()):
+        if 0 <= x < self.width and 0 <= y < self.height:
+            self[y][x] = (self.glyph(token), role, tuple(attrs))
+
+    def box(self, x, y, width, height, title):
+        if width < 2 or height < 2:
+            return
+        right, bottom = x + width - 1, y + height - 1
+        for column in range(x + 1, right):
+            self.token(column, y, "BOX_H")
+            self.token(column, bottom, "BOX_H")
+        for row in range(y + 1, bottom):
+            self.token(x, row, "BOX_V")
+            self.token(right, row, "BOX_V")
+        for column, row, token in ((x, y, "BOX_UL"), (right, y, "BOX_UR"),
+                                   (x, bottom, "BOX_LL"), (right, bottom, "BOX_LR")):
+            self.token(column, row, token)
+        self.put(x + 2, y, str(title)[:width - 4], "title", limit=right)
+
+    def bar(self, x, y, width, fraction):
+        if width <= 0:
+            return
+        fraction = max(0, min(1, _number(fraction)))
+        filled = round(width * fraction)
+        role = "crit" if fraction >= .9 else "warn" if fraction >= .7 else "ok"
+        for column in range(width):
+            self.token(x + column, y, "BAR_FULL" if column < filled else "BAR_EMPTY", role)
+
+
+def _label(vm, key, glyphs):
+    result = vm["tr"]("runtime.consoleTui." + key)
+    return str(result) if glyphs == "unicode" else sanitize(result, 200, glyphs)
+
+
+def _label_with(vm, key, glyphs, **values):
+    result = _label(vm, key, glyphs)
+    for name, value in values.items():
+        result = result.replace("{" + name + "}", str(value))
+    return result
+
+
+def _text(grid, x, y, text, width, role="text", untrusted=False):
+    if width <= 0:
+        return
+    if untrusted:
+        text = sanitize(text, width, grid.glyphs)
+    else:
+        text = str(text)[:width]
+    grid.put(x, y, text, role, limit=x + width)
+
+
+def _separator(grid):
+    return "·" if grid.glyphs == "unicode" else "."
+
+
+def _remote_state(vm):
+    snapshot = vm.get("snapshot") or {}
+    resources = snapshot.get("resources") or {}
+    age = _number(snapshot.get("age_s"))
+    status = resources.get("status")
+    if status == "starting" and age <= 15:
+        return "starting"
+    if not snapshot:
+        return "starting" if _number((vm.get("local") or {}).get("uptime"), 999999) < 120 else "unavailable"
+    if age > 60 or status in ("unavailable", "error"):
+        return "unavailable"
+    if status == "stale" or age > 10:
+        return "stale"
+    return "ok" if status == "ok" else "unavailable"
+
+
+def _clock(vm):
+    now = vm.get("now")
+    try:
+        return datetime.datetime.fromtimestamp(_number(now)).strftime("%H:%M:%S")
+    except (OverflowError, OSError, ValueError):
+        return "--:--:--"
+
+
+def _header(grid, vm, state, cluster_view):
+    label = lambda key: _label(vm, key, grid.glyphs)
+    snapshot = vm.get("snapshot") or {}
+    cluster = snapshot.get("cluster") or {}
+    node = snapshot.get("node") or {}
+    grid.put(1, 0, label("title"), "title")
+    tabs_x = 22
+    for view, key in (("node", "tabs.node"), ("cluster", "tabs.cluster")):
+        name = label(key)
+        selected = cluster_view == (view == "cluster")
+        shown = f"[{name}]" if selected else name
+        grid.put(tabs_x, 0, shown, "selected" if selected else "text",
+                 ("reverse",) if selected else ())
+        tabs_x += len(shown) + 1
+    clock = _clock(vm)
+    grid.put(grid.width - 1 - len(clock), 0, clock)
+    remaining = max(0, grid.width - tabs_x - len(clock) - 3)
+    if state == "starting":
+        info, role = label("status.starting"), "warn"
+    elif state == "unavailable":
+        info = label("status.monitorUnavailable") + (f" ({_age(snapshot.get('age_s'))})" if snapshot else "")
+        role = "crit"
+    elif cluster_view:
+        if not cluster.get("is_cluster", True):
+            info, role = label("labels.standalone"), "text"
+        else:
+            up = int(_number(cluster.get("nodes_online")))
+            total = int(_number(cluster.get("nodes_total")))
+            quorum = bool(cluster.get("quorate"))
+            title = label("labels.quorumOk" if quorum else "labels.noQuorum")
+            info = f"{sanitize(cluster.get('name') or '', 24, grid.glyphs)} {title} {up}/{total}"
+            role = "ok" if quorum else "crit"
+            coverage = _coverage(snapshot)
+            if state == "ok" and quorum and coverage == total and total > 0:
+                nodes = _online_nodes(snapshot)
+                mem = sum(_number(n.get("maxmem")) for n in nodes)
+                capacity = sum(_number(n.get("maxcpu")) for n in nodes)
+                cores = sum(_number(n.get("cpu")) * _number(n.get("maxcpu")) for n in nodes)
+                info += f" {mem / 1073741824:.0f} GiB {label('columns.cpu')} {round(cores * 100 / capacity) if capacity else 0}%"
+            else:
+                info += f" {label('labels.coverage')} {coverage}/{total}"
+    else:
+        version = sanitize(node.get("pve_version") or "n/d", 40, grid.glyphs)
+        match = re.search(r"\d+\.\d+", version)
+        version = match.group(1) if match else version
+        name = sanitize(node.get("name") or "", 20, grid.glyphs)
+        info = (f"{name} {_separator(grid)} PVE {version} "
+                f"{_separator(grid)} {label('columns.up').lower()} {_uptime((vm.get('local') or {}).get('uptime'))}")
+        role = "title"
+        if state == "ok" and not cluster.get("is_cluster", True):
+            info = f"{name} {_separator(grid)} {label('labels.standalone')}"
+        elif state == "ok" and cluster.get("quorate") is False:
+            up = int(_number(cluster.get("nodes_online")))
+            total = int(_number(cluster.get("nodes_total")))
+            info = f"{name} {_separator(grid)} {label('labels.noQuorum')} {up}/{total}"
+            role = "crit"
+    _text(grid, tabs_x + 1, 0, info, remaining, role)
+
+
+def _online_nodes(snapshot):
+    return [n for n in (snapshot.get("resources") or {}).get("nodes", [])
+            if n.get("status") == "online"]
+
+
+def _coverage(snapshot):
+    cluster = snapshot.get("cluster") or {}
+    members = {m.get("name") for m in cluster.get("members", []) if m.get("online")}
+    nodes = {n.get("name") for n in _online_nodes(snapshot)}
+    return len(members & nodes)
+
+
+def _footer(grid, vm, state, cluster_view):
+    label = lambda key: _label(vm, key, grid.glyphs)
+    parts = [label("footer.quit"), label("footer.switch")]
+    if cluster_view:
+        parts.append(label("footer.sort"))
+    parts.extend((label("footer.refresh"), label("footer.glyphs")))
+    if not cluster_view:
+        parts.append(label("footer.otherTty"))
+    separator = f" {_separator(grid)} "
+    status = (_label_with(vm, "status.data", grid.glyphs,
+                          age=_age(vm["snapshot"].get("age_s")))
+              if state != "starting" and vm.get("snapshot") else label("status.noData"))
+    help_text = separator.join(parts)
+    _text(grid, 1, grid.height - 1, help_text, max(0, grid.width - len(status) - 4), "dim")
+    _text(grid, grid.width - len(status) - 1, grid.height - 1, status, len(status), "dim")
+
+
+def _cpu_memory(grid, vm, rows):
+    label = lambda key: _label(vm, key, grid.glyphs)
+    local = vm.get("local") or {}
+    mem = local.get("mem") or {}
+    cpu_fraction = _number(local.get("cpu_total"))
+    memory_fraction = _number(mem.get("used")) / max(1, _number(mem.get("total")))
+    left = 43 if grid.width == 80 else (grid.width + 1) // 2
+    right = grid.width - left
+    grid.box(0, 1, left, rows, f"{label('panels.cpu')}  {_pct(cpu_fraction)}%")
+    grid.box(left, 1, right, rows,
+             f"{label('panels.memory')}  {_gib(mem.get('used')):.1f}/{_gib(mem.get('total')):.0f} GiB  {_pct(memory_fraction)}%")
+    grid.bar(2, 2, left - 4, cpu_fraction)
+    grid.bar(left + 2, 2, right - 4, memory_fraction)
+    load = " ".join(f"{_number(local.get('load' + str(i))):.2f}" for i in (1, 5, 15))
+    degree = "°" if grid.glyphs == "unicode" else ""
+    temp = f" {_separator(grid)} {round(_number(local.get('temp_c')))}{degree} C" if local.get("temp_c") is not None else ""
+    _text(grid, 2, 3, f"{label('labels.load')} {load} {_separator(grid)} {int(_number(local.get('threads')))} {label('labels.threads')}{temp}", left - 4)
+    _text(grid, left + 2, 3,
+          f"{label('labels.arc')} {_size(mem.get('arc'))} {_separator(grid)} {label('labels.available')} {_size(mem.get('available'))}", right - 4)
+    zram = local.get("zram") or {}
+    zram_used = zram.get("swap_used_bytes", zram.get("used", 0))
+    zram_total = zram.get("capacity_bytes", zram.get("total", 0))
+    ratio = _number(zram.get("compression_ratio"))
+    _text(grid, left + 2, 4,
+          f"{label('labels.zram')} {_size(zram_used)}/{_size(zram_total)} {ratio:.1f}x {_separator(grid)} {label('labels.swap')} {_size(mem.get('swap_used'))}/{_size(mem.get('swap_total'))}", right - 4)
+    history = local.get("cpu_history") or []
+    history_start = 4
+    history_rows = 1 if rows == 5 else 4
+    per_core = local.get("per_core") or []
+    history_width = (left - 4) // 2 if rows > 5 and grid.width >= 120 and per_core else left - 4
+    for row in range(history_rows):
+        y = history_start + row
+        for index, fraction in enumerate(history[-history_width:]):
+            height = round(max(0, min(1, _number(fraction))) * history_rows)
+            token = "BAR_FULL" if height > history_rows - row - 1 else "BAR_EMPTY"
+            grid.token(2 + index, y, token, "ok" if token == "BAR_FULL" else "dim")
+    if rows > 5 and grid.width >= 120 and per_core:
+        grouped = len(per_core) > 64
+        values = [max(per_core[i:i + 2]) for i in range(0, len(per_core), 2)] if grouped else per_core
+        shown = min(len(values), max(1, (left - history_width - 4) // 15))
+        for index, value in enumerate(values[:shown]):
+            x = history_width + 3 + index * 15
+            _text(grid, x, 5, f"CPU{index}{' x2' if grouped else ''} {_pct(value)}%", 14)
+            grid.bar(x, 6, 5, value)
+
+
+def _strip(grid, vm, y, rows, state):
+    label = lambda key: _label(vm, key, grid.glyphs)
+    local = vm.get("local") or {}
+    snapshot = vm.get("snapshot") or {}
+    cluster = snapshot.get("cluster") or {}
+    standalone = not cluster.get("is_cluster", True) and state == "ok"
+    net_width = 23 if grid.width == 80 else round(grid.width * .28)
+    disk_width = (grid.width - net_width if standalone else 28) if grid.width == 80 else round(grid.width * (.72 if standalone else .34))
+    cluster_x = net_width + disk_width
+    net = local.get("net") or {}
+    disk = local.get("disk") or {}
+    speed = _number(net.get("speed_mbps"))
+    speed_text = f" {speed / 1000:g}G" if speed >= 1000 else f" {speed:g}M" if speed else ""
+    grid.box(0, y, net_width, rows, f"{label('panels.network')} {sanitize(net.get('nic'), 12, grid.glyphs)}{speed_text}")
+    grid.box(net_width, y, disk_width, rows,
+             f"{label('panels.disk')}{' ' + label('labels.standalone') if standalone else ''}")
+    _text(grid, 2, y + 1, f"{label('labels.rx')} {_rate(net.get('rx_bps'), True)}", net_width - 4)
+    _text(grid, 2, y + 2, f"{label('labels.tx')} {_rate(net.get('tx_bps'), True)}", net_width - 4)
+    _text(grid, net_width + 2, y + 1,
+          f"{label('labels.read')} {_rate(disk.get('read_bps'))}  {label('labels.write')} {_rate(disk.get('write_bps'))}", disk_width - 4)
+    storages = local.get("storages_local") or []
+    if storages:
+        storage = storages[0]
+        used = storage.get("used", storage.get("disk"))
+        total = storage.get("total", storage.get("maxdisk"))
+        _text(grid, net_width + 2, y + 2,
+              f"{sanitize(storage.get('storage'), 14, grid.glyphs)} {_percent(used, total)}%", disk_width - 4)
+    if rows >= 5 and len(storages) > 1:
+        storage = storages[1]
+        _text(grid, net_width + 2, y + 3, sanitize(storage.get("storage"), disk_width - 4, grid.glyphs), disk_width - 4)
+    if standalone:
+        return
+    cluster_width = grid.width - cluster_x
+    up = int(_number(cluster.get("nodes_online")))
+    total = int(_number(cluster.get("nodes_total")))
+    title = f"{label('panels.cluster')} {sanitize(cluster.get('name'), 15, grid.glyphs)}"
+    title += f" {_separator(grid)} {label('labels.quorumOk') if cluster.get('quorate') else label('labels.noQuorum')} {up}/{total}"
+    grid.box(cluster_x, y, cluster_width, rows, title)
+    if state == "starting":
+        _text(grid, cluster_x + 2, y + 1, label("status.waitingForData"), cluster_width - 4, "dim")
+        return
+    members = cluster.get("members") or []
+    visible = max(0, rows - 2)
+    if len(members) > visible:
+        visible -= 1
+    nodes = {node.get("name"): node for node in (snapshot.get("resources") or {}).get("nodes", [])}
+    for index, member in enumerate(members[:visible]):
+        node = nodes.get(member.get("name"), {})
+        status = label("labels.online" if member.get("online") else "labels.offline")
+        name = sanitize(member.get("name"), 12, grid.glyphs) + ("*" if member.get("local") else "")
+        value = f"{name} {_pct(node.get('cpu'))}% {_percent(node.get('mem'), node.get('maxmem'))}% {status}"
+        _text(grid, cluster_x + 2, y + 1 + index, value, cluster_width - 4,
+              "crit" if not member.get("online") else "dim" if state != "ok" else "text")
+    if len(members) > visible:
+        _text(grid, cluster_x + 2, y + 1 + visible,
+              _label_with(vm, "labels.moreNodes", grid.glyphs,
+                          count=len(members) - visible), cluster_width - 4, "dim")
+
+
+def _guests(snapshot):
+    return [guest for guest in (snapshot.get("resources") or {}).get("guests", [])
+            if not guest.get("template")]
+
+
+def _cpu_key(guest):
+    return _number(guest.get("cpu")) * _number(guest.get("maxcpu"))
+
+
+def _top_row(grid, guest, x, y, width, mode, standalone):
+    kind = "VM" if guest.get("type") == "qemu" else "CT"
+    identity = f"{kind} {guest.get('vmid', '')} "
+    name_width = max(8, min(24 if grid.width >= 160 else 13, width - (39 if not standalone else 30)))
+    node = "" if standalone else sanitize(guest.get("node"), 10, grid.glyphs)
+    value = (f"{_cpu_key(guest):.1f}c  {_pct(guest.get('cpu'))}%" if mode == "cpu"
+             else _size(guest.get("mem")))
+    right = x + width - 2
+    status = (sanitize(guest.get("status"), 7, grid.glyphs)
+              if guest.get("status") != "running" and mode == "mem" else "")
+    detail = " ".join(part for part in (node, status) if part)
+    available_name = max(8, right - len(value) - len(detail) - len(identity) - x - 6)
+    name = sanitize(guest.get("name"), min(name_width, available_name), grid.glyphs)
+    _text(grid, x + 2, y, identity + name, max(0, width - 4))
+    if detail:
+        _text(grid, right - len(value) - len(detail) - 1, y, detail, len(detail),
+              "warn" if status else "dim")
+    _text(grid, right - len(value), y, value, len(value),
+          "dim" if guest.get("status") != "running" else "text")
+    if grid.width >= 160:
+        extra = f" UP {_uptime(guest.get('uptime'))}"
+        if guest.get("net_rx_bps") is not None:
+            extra += f" NET {_paired_rate(guest.get('net_rx_bps'), guest.get('net_tx_bps'))}"
+        if guest.get("disk_read_bps") is not None:
+            extra += f" DISK {_paired_rate(guest.get('disk_read_bps'), guest.get('disk_write_bps'))}"
+        extra_x = x + 2 + len(identity) + len(name) + 1
+        _text(grid, extra_x, y, extra,
+              max(0, right - len(value) - len(detail) - 2 - extra_x), "dim")
+
+
+def _top(grid, vm, y, rows, state):
+    label = lambda key: _label(vm, key, grid.glyphs)
+    snapshot = vm.get("snapshot") or {}
+    cluster = snapshot.get("cluster") or {}
+    standalone = not cluster.get("is_cluster", True)
+    guests = _guests(snapshot)
+    midpoint = grid.width // 2
+    cpu_title = label("panels.topCpu")
+    memory_title = label("panels.topMemory")
+    if state == "ok":
+        host_other = vm.get("host_other") or {}
+        if host_other.get("cpu") is not None:
+            cpu_title += f" {_separator(grid)} {label('labels.hostOther')} {_number(host_other['cpu']):.1f}c"
+        if host_other.get("mem") is not None:
+            memory_title += f" {_separator(grid)} {label('labels.hostOther')} {_size(host_other['mem'])}"
+    elif snapshot and state in ("stale", "unavailable"):
+        stale = _label_with(vm, "status.stale", grid.glyphs,
+                            age=_age(snapshot.get("age_s")))
+        cpu_title += f" ({stale})"
+        memory_title += f" ({stale})"
+    grid.box(0, y, midpoint, rows, cpu_title)
+    grid.box(midpoint, y, grid.width - midpoint, rows, memory_title)
+    if state == "starting":
+        for x, width in ((0, midpoint), (midpoint, grid.width - midpoint)):
+            _text(grid, x + 2, y + 1, label("status.waitingForData"), width - 4, "dim")
+        return
+    if not snapshot or not guests:
+        return
+    cpu_guests = sorted((g for g in guests if _cpu_key(g) > 0),
+                        key=lambda g: (-_cpu_key(g), int(_number(g.get("vmid")))))
+    mem_guests = sorted((g for g in guests if _number(g.get("mem")) > 0),
+                        key=lambda g: (-_number(g.get("mem")), int(_number(g.get("vmid")))))
+    for x, width, items, mode in ((0, midpoint, cpu_guests, "cpu"),
+                                   (midpoint, grid.width - midpoint, mem_guests, "mem")):
+        visible = max(0, rows - 2)
+        if len(items) > visible:
+            visible -= 1
+        for index, guest in enumerate(items[:visible]):
+            _top_row(grid, guest, x, y + index + 1, width, mode, standalone)
+            if state != "ok":
+                for column in range(x + 1, x + width - 1):
+                    char, _, attrs = grid[y + index + 1][column]
+                    grid[y + index + 1][column] = (char, "dim", attrs)
+        if len(items) > visible:
+            _text(grid, x + 2, y + 1 + visible,
+                  _label_with(vm, "labels.moreGuests", grid.glyphs,
+                              count=len(items) - visible), width - 4, "dim")
+
+
+def _node_view(grid, vm, state):
+    cpu_rows = 8 if grid.height >= 40 else 5
+    strip_rows = 5 if grid.height >= 40 else 4
+    _cpu_memory(grid, vm, cpu_rows)
+    _strip(grid, vm, 1 + cpu_rows, strip_rows, state)
+    top_y = 1 + cpu_rows + strip_rows
+    _top(grid, vm, top_y, grid.height - top_y - 1, state)
+
+
+def _cluster_view(grid, vm, state):
+    label = lambda key: _label(vm, key, grid.glyphs)
+    snapshot = vm.get("snapshot") or {}
+    cluster = snapshot.get("cluster") or {}
+    resources = snapshot.get("resources") or {}
+    members = sorted(cluster.get("members") or [],
+                     key=lambda item: (bool(item.get("online")), str(item.get("name") or "")))
+    nodes = {n.get("name"): n for n in resources.get("nodes") or []}
+    if not cluster.get("is_cluster", True) and not members:
+        local_name = (snapshot.get("node") or {}).get("name")
+        members = [{"name": name, "online": node.get("status") == "online", "local": True}
+                   for name, node in nodes.items() if name == local_name]
+    node_limit = max(1, grid.height // 3 - 2)
+    visible = min(len(members), node_limit)
+    if len(members) > visible:
+        visible = max(0, visible - 1)
+    for x, key in ((1, "columns.node"), (11, "columns.status"),
+                   (20, "columns.cpu"), (34, "columns.mem"),
+                   (63, "columns.root"), (70, "columns.guests")):
+        _text(grid, x, 1, label(key), max(0, grid.width - x - 1), "title")
+    if grid.width >= 90:
+        _text(grid, 81, 1, label("columns.up"), grid.width - 82, "title")
+    guests = _guests(snapshot)
+    for index, member in enumerate(members[:visible]):
+        node = nodes.get(member.get("name"), {}) if member.get("online") else {}
+        node_guests = [g for g in guests if g.get("node") == member.get("name")]
+        running = sum(g.get("status") == "running" for g in node_guests)
+        name = sanitize(member.get("name"), 8, grid.glyphs) + ("*" if member.get("local") else "")
+        status = label("labels.online" if member.get("online") else "labels.offline")
+        cpu = f"{_pct(node.get('cpu'))}%" if node else "--"
+        mem = f"{_size(node.get('mem'))}/{_gib(node.get('maxmem')):.0f} GiB" if node else "--"
+        root = f"{_percent(node.get('disk'), node.get('maxdisk'))}%" if node else "--"
+        row = index + 2
+        role = "crit" if not member.get("online") else "dim" if state != "ok" else "text"
+        _text(grid, 1, row, name, 9, role)
+        _text(grid, 11, row, status, 8, role)
+        _text(grid, 20, row, cpu, 4, role)
+        if node:
+            grid.bar(25, row, 8, node.get("cpu"))
+        _text(grid, 34, row, mem, 17, role)
+        if node:
+            grid.bar(54, row, 8, _number(node.get("mem")) / max(1, _number(node.get("maxmem"))))
+        _text(grid, 63, row, root, 6, role)
+        _text(grid, 70, row, f"{running}/{len(node_guests)}", 7, role)
+        if grid.width >= 90:
+            _text(grid, 81, row, _uptime(node.get("uptime")) if node else "--", 10, role)
+    if len(members) > visible:
+        _text(grid, 2, 2 + visible,
+              _label_with(vm, "labels.moreNodes", grid.glyphs,
+                          count=len(members) - visible), grid.width - 4, "dim")
+    separator_y = 2 + visible + (1 if len(members) > visible else 0)
+    for x in range(grid.width):
+        grid.token(x, separator_y, "BOX_H")
+    running = sum(g.get("status") == "running" for g in guests)
+    title = (f" {label('panels.guests')}  {label('labels.running')} {running} "
+             f"{_separator(grid)} {label('labels.stopped')} {len(guests) - running} "
+             f"{_separator(grid)} {label('labels.sort')}: {vm.get('sort', 'cpu')} (c/m) ")
+    _text(grid, 2, separator_y, title, grid.width - 4, "title")
+    guest_header_y = separator_y + 1
+    name_column = 25 if grid.width >= 160 else 19
+    guest_header = (f"{label('columns.type'):<6}{label('columns.id'):<5}"
+                    f"{label('columns.name'):<{name_column}}{label('columns.node'):<9}"
+                    f"{label('columns.status'):<10}{label('columns.cpu'):<13}{label('columns.mem'):<12}")
+    if grid.width >= 160:
+        guest_header += (f"{label('columns.cores')}  {label('columns.uptime')}  "
+                         f"{label('columns.ha')}  {label('columns.tags')}")
+    _text(grid, 1, guest_header_y, guest_header, grid.width - 2, "title")
+    sort_mem = vm.get("sort") == "mem"
+    ordered = sorted(guests, key=lambda g: (g.get("status") != "running",
+                                           -(_number(g.get("mem")) if sort_mem else _cpu_key(g)),
+                                           int(_number(g.get("vmid")))))
+    start = guest_header_y + 1
+    available = max(0, grid.height - start - 1)
+    if len(ordered) > available:
+        available = max(0, available - 1)
+    for index, guest in enumerate(ordered[:available]):
+        kind = "VM" if guest.get("type") == "qemu" else "CT"
+        name_width = 24 if grid.width >= 160 else 18
+        name = sanitize(guest.get("name"), name_width, grid.glyphs)
+        node = sanitize(guest.get("node"), 8, grid.glyphs)
+        status = sanitize(guest.get("status"), 9, grid.glyphs)
+        base = (f"{kind:<6}{str(guest.get('vmid', '')):<5}{name:<{name_width + 1}}"
+                f"{node:<9}{status:<10}{_cpu_key(guest):.1f}c {_pct(guest.get('cpu')):>3}%")
+        base += f"   {_size(guest.get('mem'))}"
+        if grid.width >= 160:
+            base += (f"  {int(_number(guest.get('maxcpu')))}"
+                     f"  {_uptime(guest.get('uptime'))}"
+                     f"  {sanitize(guest.get('hastate'), 10, grid.glyphs) or '-'}"
+                     f"  {','.join(sanitize(t, 16, grid.glyphs) for t in (guest.get('tags') or [])[:3])}")
+        _text(grid, 1, start + index, base, grid.width - 2,
+              "dim" if guest.get("status") != "running" or state != "ok" else "text")
+    if len(ordered) > available:
+        _text(grid, 2, start + available,
+              _label_with(vm, "labels.moreGuests", grid.glyphs,
+                          count=len(ordered) - available), grid.width - 4, "dim")
+    if state == "starting" and not guests:
+        _text(grid, 2, start, label("status.waitingForData"), grid.width - 4, "dim")
+
+
+def render(view_model, width, height):
+    """Render a ViewModel to a deterministic ``height × width`` Grid."""
+    glyphs = view_model.get("glyphs", "acs")
+    if glyphs not in ("acs", "ascii", "unicode"):
+        glyphs = "acs"
+    grid = Grid(width, height, glyphs)
+    if width < 80 or height < 24:
+        message = _label(view_model, "status.terminalTooSmall", glyphs)
+        _text(grid, max(0, (width - len(message)) // 2), max(0, height // 2),
+              message, width, "warn")
+        return grid
+    state = _remote_state(view_model)
+    cluster_view = view_model.get("selected_view") == "cluster"
+    _header(grid, view_model, state, cluster_view)
+    if cluster_view:
+        _cluster_view(grid, view_model, state)
+    else:
+        _node_view(grid, view_model, state)
+    _footer(grid, view_model, state, cluster_view)
+    return grid
diff --git a/AppImage/scripts/console_snapshot.py b/AppImage/scripts/console_snapshot.py
new file mode 100644
index 00000000..b4c49b42
--- /dev/null
+++ b/AppImage/scripts/console_snapshot.py
@@ -0,0 +1,469 @@
+"""Lease-gated Proxmox snapshot shared by the Monitor and console TUI."""
+
+import grp
+import json
+import math
+import os
+from pathlib import Path
+import re
+import stat
+import tempfile
+import threading
+import time
+
+
+DEFAULT_PATH = Path("/run/proxmenux-monitor/console.json")
+DEFAULT_STATE_DIR = Path("/run/proxmenux-console")
+_STATUS = {"starting", "ok", "stale", "unavailable", "error"}
+_ANSI = re.compile(r"\x1b(?:\[[0-?]*[ -/]*[@-~]|.)")
+_MAX_BYTES = 1024 * 1024
+
+
+def _fields(raw, names):
+    return {name: raw.get(name) for name in names}
+
+
+def normalize_resources(raw):
+    """Map the all-types /cluster/resources response to schema 1 data."""
+    if not isinstance(raw, list):
+        raise ValueError("resources response is not a list")
+    nodes, guests, storages = [], [], []
+    for item in raw:
+        if not isinstance(item, dict):
+            continue
+        kind = item.get("type")
+        if kind == "node":
+            node = _fields(item, ("status", "cpu", "maxcpu", "mem", "maxmem",
+                                  "disk", "maxdisk", "uptime", "level"))
+            node["name"] = item.get("node")
+            node["cgroup_mode"] = item.get("cgroup-mode", item.get("cgroup_mode"))
+            nodes.append(node)
+        elif kind in ("qemu", "lxc"):
+            guest = _fields(item, ("vmid", "name", "node", "status", "cpu", "maxcpu",
+                                   "mem", "maxmem", "disk", "maxdisk", "uptime",
+                                   "netin", "netout", "diskread", "diskwrite",
+                                   "hastate", "template", "lock"))
+            guest["type"] = kind
+            tags = item.get("tags")
+            guest["tags"] = tags.split(";") if isinstance(tags, str) else []
+            if guest["template"] is not None:
+                guest["template"] = bool(guest["template"])
+            guests.append(guest)
+        elif kind == "storage":
+            storage = _fields(item, ("id", "storage", "node", "plugintype", "shared",
+                                     "status", "disk", "maxdisk"))
+            if storage["shared"] is not None:
+                storage["shared"] = bool(storage["shared"])
+            content = item.get("content")
+            storage["content"] = content.split(",") if isinstance(content, str) else (
+                content if isinstance(content, list) else [])
+            storages.append(storage)
+    guests.sort(key=lambda row: (
+        -(row["cpu"] or 0) * (row["maxcpu"] or 0), -(row["mem"] or 0)))
+    truncated = len(guests) > 2000
+    return {"nodes": nodes, "guests": guests[:2000], "storages": storages,
+            "truncated": truncated}
+
+
+def normalize_cluster_status(raw):
+    """Map /cluster/status, including standalone and non-quorate states."""
+    if not isinstance(raw, list):
+        raise ValueError("cluster status response is not a list")
+    cluster = next((item for item in raw if isinstance(item, dict) and
+                    item.get("type") == "cluster"), None)
+    members = []
+    for item in raw:
+        if not isinstance(item, dict) or item.get("type") != "node":
+            continue
+        member = _fields(item, ("name", "nodeid", "ip", "level"))
+        member["online"] = bool(item["online"]) if "online" in item else None
+        member["local"] = bool(item["local"]) if "local" in item else None
+        members.append(member)
+    return {
+        "is_cluster": cluster is not None,
+        "name": cluster.get("name") if cluster else None,
+        "quorate": bool(cluster["quorate"]) if cluster and "quorate" in cluster else None,
+        "nodes_total": cluster.get("nodes") if cluster else len(members),
+        "nodes_online": sum(member["online"] is True for member in members),
+        "config_version": cluster.get("version") if cluster else None,
+        "members": members,
+    }
+
+
+def _empty_cluster():
+    return {"status": "starting", "observed_at": None, "is_cluster": None,
+            "name": None, "quorate": None, "nodes_total": None,
+            "nodes_online": None, "config_version": None, "members": [], "error": None}
+
+
+def _empty_resources():
+    return {"status": "starting", "observed_at": None, "truncated": False,
+            "nodes": [], "guests": [], "storages": [], "error": None}
+
+
+def write_snapshot(data, path=DEFAULT_PATH):
+    """Replace a snapshot atomically, rechecking the optional console group."""
+    path = Path(path)
+    try:
+        gid = grp.getgrnam("proxmenux-console").gr_gid
+    except KeyError:
+        gid = None
+    directory = path.parent
+    directory.mkdir(parents=True, exist_ok=True)
+    os.chmod(directory, 0o750 if gid is not None else 0o700)
+    if gid is not None:
+        os.chown(directory, -1, gid)
+    temporary = None
+    try:
+        with tempfile.NamedTemporaryFile(mode="w", encoding="utf-8", dir=directory,
+                                         delete=False) as stream:
+            temporary = stream.name
+            json.dump(data, stream, ensure_ascii=False, separators=(",", ":"))
+            os.fchmod(stream.fileno(), 0o640 if gid is not None else 0o600)
+            if gid is not None:
+                os.fchown(stream.fileno(), -1, gid)
+        os.replace(temporary, path)
+    finally:
+        if temporary is not None and os.path.exists(temporary):
+            os.unlink(temporary)
+
+
+def _fit_document(data):
+    """Keep large guest inventories under the snapshot size contract."""
+    def size():
+        return len(json.dumps(data, ensure_ascii=False,
+                              separators=(",", ":")).encode("utf-8"))
+
+    if size() <= _MAX_BYTES:
+        return data
+    data["resources"] = dict(data["resources"])
+    data["resources"]["truncated"] = True
+    for section, key in (("resources", "guests"), ("resources", "storages"),
+                         ("resources", "nodes"), ("cluster", "members")):
+        if section == "cluster":
+            data["cluster"] = dict(data["cluster"])
+        rows = data[section][key]
+        low, high = 0, len(rows)
+        while low < high:
+            middle = (low + high + 1) // 2
+            data[section][key] = rows[:middle]
+            if size() <= _MAX_BYTES:
+                low = middle
+            else:
+                high = middle - 1
+        data[section][key] = rows[:low]
+        if size() <= _MAX_BYTES:
+            return data
+    raise ValueError("snapshot exceeds 1 MiB")
+
+
+def touch_lease(state_dir=DEFAULT_STATE_DIR, now=None):
+    """Create or refresh the TUI's client lease."""
+    lease = Path(state_dir) / "client" / "lease"
+    lease.parent.mkdir(parents=True, exist_ok=True)
+    flags = os.O_CREAT | os.O_WRONLY | getattr(os, "O_NOFOLLOW", 0)
+    descriptor = os.open(lease, flags, 0o600)
+    try:
+        if not stat.S_ISREG(os.fstat(descriptor).st_mode):
+            raise ValueError("lease is not a regular file")
+    finally:
+        os.close(descriptor)
+    stamp = time.time() if now is None else now
+    os.utime(lease, (stamp, stamp), follow_symlinks=False)
+    return lease
+
+
+class _Source:
+    def __init__(self, empty):
+        self.data = empty()
+        self.failures = 0
+
+    def succeed(self, data, now):
+        self.data.update(data)
+        self.data.update(status="ok", observed_at=now, error=None)
+        self.failures = 0
+
+    def fail(self, exc):
+        self.failures += 1
+        self.data["error"] = str(exc)[:200]
+        if self.failures >= 3:
+            self.data["status"] = "error"
+        elif self.data["observed_at"] is None:
+            self.data["status"] = "unavailable"
+        else:
+            self.data["status"] = "stale"
+
+
+class Publisher:
+    """One publisher instance; publish_once supports deterministic callers."""
+
+    def __init__(self, get_node_name, get_resources, run_pvesh, read_language,
+                 path=DEFAULT_PATH, state_dir=DEFAULT_STATE_DIR, clock=time.time,
+                 get_pve_version=None, version=None):
+        self.get_node_name = get_node_name
+        self.get_resources = get_resources
+        self.run_pvesh = run_pvesh
+        self.read_language = read_language
+        self.get_pve_version = get_pve_version
+        self.version = version
+        self.path = Path(path)
+        self.lease = Path(state_dir) / "client" / "lease"
+        self.clock = clock
+        self.cluster = _Source(_empty_cluster)
+        self.resources = _Source(_empty_resources)
+        self.last_publish = None
+        self.last_cluster_call = None
+        self.lease_active = False
+        self._pve_version = None
+        self._pve_version_at = None
+
+    def _node(self, now):
+        try:
+            name = self.get_node_name()
+        except Exception as exc:
+            return {"status": "error", "observed_at": None, "name": None,
+                    "pve_version": None, "error": str(exc)[:200]}
+        if (self.get_pve_version is not None and
+                (self._pve_version_at is None or now - self._pve_version_at >= 300)):
+            try:
+                self._pve_version = self.get_pve_version()
+            except Exception:
+                self._pve_version = None
+            self._pve_version_at = now
+        return {"status": "ok", "observed_at": now, "name": name,
+                "pve_version": self._pve_version, "error": None}
+
+    def _document(self, now):
+        try:
+            language = self.read_language() or "en"
+        except Exception:
+            language = "en"
+        return {"schema": 1,
+                "generator": {"name": "proxmenux-monitor", "version": self.version,
+                              "pid": os.getpid()},
+                "observed_at": now, "publish_interval_s": 5,
+                "language": language, "node": self._node(now),
+                "cluster": self.cluster.data, "resources": self.resources.data}
+
+    def write_initial(self, now=None):
+        now = self.clock() if now is None else now
+        write_snapshot(_fit_document(self._document(now)), self.path)
+
+    def _fresh_lease(self, now):
+        try:
+            info = os.lstat(self.lease)
+        except OSError:
+            return False
+        return stat.S_ISREG(info.st_mode) and 0 <= now - info.st_mtime < 15
+
+    def publish_once(self, now=None):
+        now = self.clock() if now is None else now
+        active = self._fresh_lease(now)
+        if not active:
+            self.lease_active = False
+            return False
+        due = (not self.lease_active or self.last_publish is None or
+               now - self.last_publish >= 5)
+        self.lease_active = True
+        if not due:
+            return False
+        try:
+            self.resources.succeed(normalize_resources(self.get_resources()), now)
+        except Exception as exc:
+            self.resources.fail(exc)
+        if self.last_cluster_call is None or now - self.last_cluster_call >= 10:
+            self.last_cluster_call = now
+            try:
+                self.cluster.succeed(normalize_cluster_status(self.run_pvesh()), now)
+            except Exception as exc:
+                self.cluster.fail(exc)
+        write_snapshot(_fit_document(self._document(now)), self.path)
+        self.last_publish = now
+        return True
+
+    def run(self):
+        while True:
+            try:
+                self.publish_once()
+            except Exception:
+                pass
+            time.sleep(2 if not self.lease_active else 1)
+
+
+_publisher = None
+_publisher_lock = threading.Lock()
+
+
+def ensure_publisher(get_node_name, get_resources, run_pvesh, read_language,
+                     path=DEFAULT_PATH, state_dir=DEFAULT_STATE_DIR,
+                     clock=time.time, get_pve_version=None, version=None):
+    """Write STARTING immediately and start exactly one daemon publisher."""
+    global _publisher
+    with _publisher_lock:
+        if _publisher is not None:
+            return _publisher
+        publisher = Publisher(get_node_name, get_resources, run_pvesh,
+                              read_language, path, state_dir, clock,
+                              get_pve_version, version)
+        try:
+            publisher.write_initial()
+        except Exception:
+            pass
+        thread = threading.Thread(target=publisher.run, daemon=True)
+        thread.start()
+        _publisher = publisher
+        return publisher
+
+
+def _text(value, limit):
+    if not isinstance(value, str):
+        return None
+    value = _ANSI.sub("", value)
+    return "".join(char for char in value
+                   if ord(char) >= 32 and not 127 <= ord(char) <= 159)[:limit]
+
+
+def _number(value):
+    if isinstance(value, bool) or not isinstance(value, (int, float)):
+        return None
+    if not math.isfinite(value) or value < 0:
+        return None
+    return value
+
+
+def _boolean(value):
+    return value if isinstance(value, bool) else None
+
+
+def _clean_row(row, fields):
+    if not isinstance(row, dict):
+        return None
+    result = {}
+    for name, kind in fields.items():
+        value = row.get(name)
+        if kind == "name":
+            result[name] = _text(value, 64)
+        elif kind == "text":
+            result[name] = _text(value, 200)
+        elif kind == "number":
+            result[name] = _number(value)
+        elif kind == "bool":
+            result[name] = _boolean(value)
+    return result
+
+
+_NODE_FIELDS = {"name": "name", "pve_version": "text", "error": "text"}
+_MEMBER_FIELDS = {"name": "name", "nodeid": "number", "online": "bool",
+                  "local": "bool", "ip": "text", "level": "text"}
+_RESOURCE_NODE_FIELDS = {"name": "name", "status": "text", "cpu": "number",
+                         "maxcpu": "number", "mem": "number", "maxmem": "number",
+                         "disk": "number", "maxdisk": "number", "uptime": "number",
+                         "level": "text", "cgroup_mode": "number"}
+_GUEST_FIELDS = {"vmid": "number", "type": "text", "name": "name",
+                 "node": "name", "status": "text", "cpu": "number",
+                 "maxcpu": "number", "mem": "number", "maxmem": "number",
+                 "disk": "number", "maxdisk": "number", "uptime": "number",
+                 "netin": "number", "netout": "number", "diskread": "number",
+                 "diskwrite": "number", "hastate": "text", "template": "bool",
+                 "lock": "text"}
+_STORAGE_FIELDS = {"id": "text", "storage": "name", "node": "name",
+                   "plugintype": "text", "shared": "bool", "status": "text",
+                   "disk": "number", "maxdisk": "number"}
+
+
+class Snapshot:
+    """Validated data with attribute access for top-level sections."""
+
+    def __init__(self, data, age_s, reason=None):
+        self.data = data
+        self.age_s = age_s
+        self.reason = reason
+
+    def __getitem__(self, key):
+        return self.data[key]
+
+    def __getattr__(self, key):
+        try:
+            return self.data[key]
+        except KeyError as exc:
+            raise AttributeError(key) from exc
+
+    @classmethod
+    def unavailable(cls, reason):
+        return cls({"schema": 1, "generator": {"name": None, "version": None,
+                                               "pid": None},
+                    "observed_at": None, "publish_interval_s": 5,
+                    "language": "en",
+                    "node": {"status": "error", "observed_at": None,
+                             "name": None, "pve_version": None, "error": reason},
+                    "cluster": {**_empty_cluster(), "status": "unavailable"},
+                    "resources": {**_empty_resources(), "status": "unavailable"}},
+                   None, reason)
+
+
+def load_snapshot(path=DEFAULT_PATH, now=None):
+    """Read untrusted snapshot data without leaking errors to curses."""
+    now = time.time() if now is None else now
+    try:
+        with open(path, encoding="utf-8") as stream:
+            data = json.load(stream)
+        if not isinstance(data, dict) or type(data.get("schema")) is not int or data["schema"] != 1:
+            raise ValueError("unsupported schema")
+        if not all(isinstance(data.get(key), dict) for key in
+                   ("generator", "node", "cluster", "resources")):
+            raise ValueError("invalid sections")
+        if not all(isinstance(data["resources"].get(key), list) for key in
+                   ("nodes", "guests", "storages")):
+            raise ValueError("invalid resource lists")
+        if not isinstance(data["cluster"].get("members"), list):
+            raise ValueError("invalid members")
+        for source in ("cluster", "resources"):
+            if data[source].get("status") not in _STATUS:
+                raise ValueError("invalid source status")
+        if data["node"].get("status") not in ("ok", "error"):
+            raise ValueError("invalid node status")
+        cleaned = {"schema": 1,
+                   "generator": _clean_row(data["generator"],
+                                           {"name": "text", "version": "text", "pid": "number"}),
+                   "observed_at": _number(data.get("observed_at")),
+                   "publish_interval_s": _number(data.get("publish_interval_s")),
+                   "language": _text(data.get("language"), 16) or "en"}
+        cleaned["node"] = _clean_row(data["node"], _NODE_FIELDS)
+        cleaned["node"].update(status=data["node"]["status"],
+                               observed_at=_number(data["node"].get("observed_at")))
+        cluster = data["cluster"]
+        cleaned["cluster"] = _clean_row(cluster,
+            {"is_cluster": "bool", "name": "name", "quorate": "bool",
+             "nodes_total": "number", "nodes_online": "number",
+             "config_version": "number", "error": "text"})
+        cleaned["cluster"].update(status=cluster["status"],
+            observed_at=_number(cluster.get("observed_at")),
+            members=[row for item in cluster["members"][:2000]
+                     if (row := _clean_row(item, _MEMBER_FIELDS)) is not None])
+        resources = data["resources"]
+        cleaned["resources"] = {"status": resources["status"],
+                                "observed_at": _number(resources.get("observed_at")),
+                                "truncated": _boolean(resources.get("truncated")) or False,
+                                "error": _text(resources.get("error"), 200)}
+        for key, fields in (("nodes", _RESOURCE_NODE_FIELDS),
+                            ("guests", _GUEST_FIELDS), ("storages", _STORAGE_FIELDS)):
+            limit = 2000 if key == "guests" else 10000
+            cleaned_rows = []
+            for item in resources[key][:limit]:
+                row = _clean_row(item, fields)
+                if row is None:
+                    continue
+                if key == "guests":
+                    tags = item.get("tags")
+                    row["tags"] = ([_text(tag, 32) for tag in tags[:16]
+                                    if isinstance(tag, str)] if isinstance(tags, list) else [])
+                elif key == "storages":
+                    content = item.get("content")
+                    row["content"] = ([_text(part, 64) for part in content[:32]
+                                       if isinstance(part, str)] if isinstance(content, list) else [])
+                cleaned_rows.append(row)
+            cleaned["resources"][key] = cleaned_rows
+        observed = cleaned["observed_at"]
+        return Snapshot(cleaned, max(0, now - observed) if observed is not None else None)
+    except (OSError, UnicodeError, json.JSONDecodeError, ValueError, TypeError) as exc:
+        return Snapshot.unavailable(str(exc)[:200])
diff --git a/AppImage/scripts/console_tui.py b/AppImage/scripts/console_tui.py
new file mode 100644
index 00000000..3d94f282
--- /dev/null
+++ b/AppImage/scripts/console_tui.py
@@ -0,0 +1,331 @@
+"""Curses entry point for the read-only ProxMenux console."""
+
+import os
+import sys
+_HERE = os.path.dirname(os.path.realpath(__file__))
+if _HERE not in sys.path:
+    sys.path.insert(0, _HERE)
+import console_metrics  # noqa: E402
+import console_render  # noqa: E402
+import console_snapshot  # noqa: E402
+
+import argparse
+import curses
+import json
+import locale
+from pathlib import Path
+import re
+import signal
+import time
+
+
+_ROLES = {"title": curses.COLOR_CYAN, "text": None, "dim": None,
+          "ok": curses.COLOR_GREEN, "warn": curses.COLOR_YELLOW,
+          "crit": curses.COLOR_RED, "selected": None}
+_ASCII_TOKENS = {key: ("-" if key == "BOX_H" else "|" if key == "BOX_V"
+                       else "#" if key == "BAR_FULL" else "."
+                       if key in ("BAR_EMPTY", "BULLET") else "+")
+                 for key in console_render.ACS_TOKENS}
+_COUNTERS = {"netin": "net_rx_bps", "netout": "net_tx_bps",
+             "diskread": "disk_read_bps", "diskwrite": "disk_write_bps"}
+_LANGUAGE = re.compile(r"[A-Za-z]{2,3}(?:[-_][A-Za-z0-9]+)?\Z")
+_stop_requested = False
+
+
+def _catalog_value(catalog, key):
+    value = catalog
+    for part in key.split("."):
+        if not isinstance(value, dict):
+            return None
+        value = value.get(part)
+    return value if isinstance(value, str) else None
+
+
+class Translator:
+    """Resolve keys from the selected catalog, English, then the key."""
+
+    def __init__(self, module_file=__file__):
+        module = Path(module_file).resolve()
+        appdir = Path(os.environ.get("APPDIR") or module.parents[2])
+        self.roots = [appdir / "usr/share/proxmenux/messages",
+                      module.parents[1] / "messages"]
+        self.language = None
+        self.catalogs = {}
+        self.set_language("en")
+
+    def _load(self, language):
+        if language not in self.catalogs:
+            self.catalogs[language] = {}
+            for root in self.roots:
+                try:
+                    with (root / language / "common.json").open(encoding="utf-8") as stream:
+                        value = json.load(stream)
+                except (OSError, UnicodeError, ValueError):
+                    continue
+                if isinstance(value, dict):
+                    self.catalogs[language] = value
+                    break
+        return self.catalogs[language]
+
+    def set_language(self, language):
+        language = (language if isinstance(language, str) and
+                    _LANGUAGE.fullmatch(language) else "en")
+        if language != self.language:
+            self.language = language
+            self.catalogs = {}
+            self._load(language)
+            self._load("en")
+
+    def __call__(self, key):
+        return (_catalog_value(self._load(self.language), key) or
+                _catalog_value(self._load("en"), key) or key)
+
+
+def _configuration_language():
+    try:
+        with open("/usr/local/share/proxmenux/config.json", encoding="utf-8") as stream:
+            language = json.load(stream).get("language")
+        return language if isinstance(language, str) and _LANGUAGE.fullmatch(language) else "en"
+    except (OSError, UnicodeError, ValueError, AttributeError):
+        return "en"
+
+
+def _uptime(proc_root):
+    try:
+        with (Path(proc_root) / "proc/uptime").open(encoding="ascii") as stream:
+            return max(0.0, float(stream.read().split()[0]))
+    except (OSError, UnicodeError, ValueError, IndexError):
+        return 0.0
+
+
+class GuestRates:
+    """Derive guest byte rates only across consecutive valid observations."""
+
+    def __init__(self):
+        self.previous = {}
+        self.observed_at = None
+
+    def update(self, guests, observed_at):
+        dt = (observed_at - self.observed_at
+              if isinstance(observed_at, (int, float)) and
+              isinstance(self.observed_at, (int, float)) else 0)
+        current = {}
+        output = []
+        for item in guests:
+            guest = dict(item)
+            key = (guest.get("type"), guest.get("vmid"))
+            old = self.previous.get(key)
+            stable = (dt > 0 and old is not None and
+                      old.get("node") == guest.get("node") and
+                      isinstance(guest.get("uptime"), (int, float)) and
+                      isinstance(old.get("uptime"), (int, float)) and
+                      guest["uptime"] >= old["uptime"])
+            if stable:
+                stable = all(not (isinstance(old.get(source), (int, float)) and
+                                  isinstance(guest.get(source), (int, float)) and
+                                  guest[source] < old[source]) for source in _COUNTERS)
+            if stable:
+                for source, target in _COUNTERS.items():
+                    first, second = old.get(source), guest.get(source)
+                    if (isinstance(first, (int, float)) and
+                            isinstance(second, (int, float)) and second >= first):
+                        guest[target] = (second - first) / dt
+            current[key] = item
+            output.append(guest)
+        self.previous = current
+        self.observed_at = observed_at
+        return output
+
+
+def _color_pairs():
+    pairs = {role: 0 for role in _ROLES}
+    if not curses.has_colors():
+        return pairs
+    curses.start_color()
+    try:
+        curses.use_default_colors()
+        background = -1
+    except curses.error:
+        background = curses.COLOR_BLACK
+    for number, (role, foreground) in enumerate(_ROLES.items(), 1):
+        try:
+            color = (background if background == -1 else curses.COLOR_WHITE)
+            curses.init_pair(number, color if foreground is None else foreground,
+                             background)
+            pairs[role] = curses.color_pair(number)
+        except curses.error:
+            pass
+    return pairs
+
+
+def _paint(stdscr, grid, pairs):
+    height, width = stdscr.getmaxyx()
+    stdscr.erase()
+    for y, row in enumerate(grid):
+        if y >= height:
+            break
+        for x, (char, role, attrs) in enumerate(row):
+            if x >= width or (y == height - 1 and x == width - 1):
+                continue
+            style = pairs.get(role, 0)
+            for attr in attrs:
+                style |= {"dim": curses.A_DIM, "reverse": curses.A_REVERSE,
+                          "bold": curses.A_BOLD}.get(attr, 0)
+            if role == "dim":
+                style |= curses.A_DIM
+            if role == "selected":
+                style |= curses.A_REVERSE
+            token = console_render.ACS_TOKENS.get(char)
+            if token is not None:
+                glyph = getattr(curses, token, None)
+                if glyph is None:
+                    char = _ASCII_TOKENS[char]
+                else:
+                    char = glyph
+            try:
+                if isinstance(char, int):
+                    stdscr.addch(y, x, char, style)
+                else:
+                    stdscr.addstr(y, x, char, style)
+            except curses.error:
+                # A resize may land between getmaxyx and a cell write.
+                continue
+    stdscr.refresh()
+
+
+def _signal_stop(_signum, _frame):
+    global _stop_requested
+    _stop_requested = True
+
+
+def _parse_args(argv):
+    parser = argparse.ArgumentParser(description="ProxMenux console TUI")
+    parser.add_argument("--once", action="store_true")
+    parser.add_argument("--snapshot", default=console_snapshot.DEFAULT_PATH)
+    parser.add_argument("--proc-root", default="/")
+    parser.add_argument("--state-dir", default=os.environ.get(
+        "PMX_CONSOLE_STATE_DIR", console_snapshot.DEFAULT_STATE_DIR))
+    parser.add_argument("--glyphs", choices=("acs", "unicode", "ascii", "auto"),
+                        default="acs")
+    parser.add_argument("--autostart", action="store_true")
+    return parser.parse_args(argv)
+
+
+def _run(stdscr, args):
+    try:
+        curses.curs_set(0)
+    except curses.error:
+        pass
+    stdscr.timeout(1000)
+    stdscr.keypad(True)
+    pairs = _color_pairs()
+    sampler = console_metrics.LocalSampler(proc_root=args.proc_root)
+    translator = Translator()
+    rates = GuestRates()
+    glyphs = "acs" if args.glyphs == "auto" else args.glyphs
+    selected_view = "node"
+    sort = "cpu"
+    snapshot = None
+    guests = []
+    previous_mtime = object()
+    last_clear = time.monotonic()
+    reload_now = True
+    repaint = False
+    while not _stop_requested:
+        now = time.time()
+        local = sampler.sample(time.monotonic())
+        local["uptime"] = _uptime(args.proc_root)
+        try:
+            console_snapshot.touch_lease(args.state_dir, now)
+        except (OSError, ValueError):
+            pass
+        try:
+            info = os.stat(args.snapshot)
+            mtime = (info.st_dev, info.st_ino, info.st_mtime_ns, info.st_size)
+        except OSError:
+            mtime = None
+        if reload_now or mtime != previous_mtime:
+            snapshot = console_snapshot.load_snapshot(args.snapshot, now)
+            if snapshot.reason is None:
+                resources = snapshot.data.get("resources") or {}
+                guests = rates.update(resources.get("guests") or [],
+                                      resources.get("observed_at"))
+            else:
+                rates = GuestRates()
+                guests = []
+            previous_mtime = mtime
+            reload_now = False
+        if snapshot is not None and snapshot.reason is None:
+            data = dict(snapshot.data)
+            observed = data.get("observed_at")
+            data["age_s"] = max(0, now - observed) if observed is not None else None
+            data["resources"] = dict(data.get("resources") or {}, guests=guests)
+            language = data.get("language")
+        else:
+            data = None
+            language = _configuration_language()
+        translator.set_language(language)
+        resources = (data or {}).get("resources") or {}
+        node = (data or {}).get("node") or {}
+        fresh = (resources.get("status") == "ok" and data["age_s"] is not None
+                 and data["age_s"] <= 10)
+        other = console_metrics.host_other(local, guests, node.get("name"), fresh)
+        view_model = {"local": local, "snapshot": data, "host_other": other,
+                      "glyphs": glyphs, "lang": translator.language, "tr": translator,
+                      "now": now, "selected_view": selected_view, "sort": sort}
+        height, width = stdscr.getmaxyx()
+        if repaint or time.monotonic() - last_clear >= 30:
+            stdscr.clearok(True)
+            last_clear = time.monotonic()
+            repaint = False
+        _paint(stdscr, console_render.render(view_model, width, height), pairs)
+        if args.once or _stop_requested:
+            break
+        key = stdscr.getch()
+        if key in (ord("q"), ord("Q")):
+            break
+        if key in (9, curses.KEY_LEFT, curses.KEY_RIGHT):
+            selected_view = "cluster" if selected_view == "node" else "node"
+        elif key == ord("1"):
+            selected_view = "node"
+        elif key == ord("2"):
+            selected_view = "cluster"
+        elif key == ord("r"):
+            reload_now = True
+            repaint = True
+            try:
+                console_snapshot.touch_lease(args.state_dir, time.time())
+            except (OSError, ValueError):
+                pass
+        elif key == ord("a"):
+            glyphs = {"acs": "ascii", "ascii": "unicode", "unicode": "acs"}[glyphs]
+        elif key == curses.KEY_RESIZE:
+            repaint = True
+        elif selected_view == "cluster" and key in (ord("c"), ord("m")):
+            sort = "cpu" if key == ord("c") else "mem"
+
+
+def main(argv=None):
+    global _stop_requested
+    try:
+        locale.setlocale(locale.LC_ALL, "")
+    except locale.Error:
+        pass
+    args = _parse_args(argv)
+    _stop_requested = False
+    previous = {}
+    for signum in (signal.SIGINT, signal.SIGTERM, signal.SIGHUP):
+        previous[signum] = signal.signal(signum, _signal_stop)
+    try:
+        curses.wrapper(_run, args)
+    except Exception as exc:
+        print("console_tui: " + str(exc).replace("\n", " "), file=sys.stderr)
+        return 1
+    finally:
+        for signum, handler in previous.items():
+            signal.signal(signum, handler)
+    return 0
+
+
+if __name__ == "__main__":
+    sys.exit(main())
diff --git a/AppImage/scripts/flask_server.py b/AppImage/scripts/flask_server.py
index d8be04d6..2820e12a 100644
--- a/AppImage/scripts/flask_server.py
+++ b/AppImage/scripts/flask_server.py
@@ -1756,7 +1756,8 @@ _AVAILABLE_UPDATES_MIN_INTERVAL = 30  # seconds between apt recounts when apt st
 
 # Cache for pvesh cluster resources (reduces repeated API calls)
 _pvesh_cache = {
-    'cluster_resources_vm': None,
+    'cluster_resources': None,
+    # Existing callers reset this timestamp to invalidate the resources cache.
     'cluster_resources_vm_time': 0,
     'cluster_resources_storage': None,
     'cluster_resources_storage_time': 0,
@@ -2756,8 +2757,8 @@ def _action_cancel(unit_name):
     return True
 
 
-def get_cached_pvesh_cluster_resources_vm():
-    """Get cluster VM resources with a per-process cache.
+def get_cached_pvesh_cluster_resources():
+    """Get all cluster resources with a per-process cache.
 
     `_PVESH_CACHE_TTL` controls cache freshness; `_pvesh_cluster_resources_lock`
     serialises in-flight calls so a thundering herd of dashboard fetches
@@ -2768,9 +2769,9 @@ def get_cached_pvesh_cluster_resources_vm():
     global _pvesh_cache
     now = time.time()
 
-    if _pvesh_cache['cluster_resources_vm'] is not None and \
+    if _pvesh_cache['cluster_resources'] is not None and \
        now - _pvesh_cache['cluster_resources_vm_time'] < _PVESH_CACHE_TTL:
-        return _pvesh_cache['cluster_resources_vm']
+        return _pvesh_cache['cluster_resources']
 
     with _pvesh_cluster_resources_lock:
         # Re-check inside the lock — another thread may have refreshed
@@ -2778,23 +2779,29 @@ def get_cached_pvesh_cluster_resources_vm():
         # pattern: only the first thread runs `pvesh`, everyone else
         # benefits from the same result without spawning duplicates.
         now = time.time()
-        if _pvesh_cache['cluster_resources_vm'] is not None and \
+        if _pvesh_cache['cluster_resources'] is not None and \
            now - _pvesh_cache['cluster_resources_vm_time'] < _PVESH_CACHE_TTL:
-            return _pvesh_cache['cluster_resources_vm']
+            return _pvesh_cache['cluster_resources']
 
         try:
             result = subprocess.run(
-                ['pvesh', 'get', '/cluster/resources', '--type', 'vm', '--output-format', 'json'],
+                ['pvesh', 'get', '/cluster/resources', '--output-format', 'json'],
                 capture_output=True, text=True, timeout=10
             )
             if result.returncode == 0:
                 data = json.loads(result.stdout)
-                _pvesh_cache['cluster_resources_vm'] = data
+                _pvesh_cache['cluster_resources'] = data
                 _pvesh_cache['cluster_resources_vm_time'] = now
                 return data
         except Exception:
             pass
-        return _pvesh_cache['cluster_resources_vm'] or []
+        return _pvesh_cache['cluster_resources'] or []
+
+
+def get_cached_pvesh_cluster_resources_vm():
+    """Get VM and container resources from the shared cluster cache."""
+    return [r for r in get_cached_pvesh_cluster_resources()
+            if r.get('type') in ('qemu', 'lxc')]
 
 
 # Sprint 14 perf pass: three backup endpoints (`/api/backups`,
@@ -5916,6 +5923,34 @@ try:
 except Exception:
     pass
 
+# Console TUI snapshot publisher: one daemon thread that writes the
+# cluster snapshot while a console client holds its lease.
+# Optional by design — the Monitor must start even if the module fails.
+def _console_run_pvesh():
+    result = subprocess.run(['pvesh', 'get', '/cluster/status', '--output-format', 'json'],
+                            capture_output=True, text=True, timeout=10)
+    if result.returncode != 0:
+        raise OSError(f"pvesh exited with {result.returncode}")
+    return json.loads(result.stdout)
+
+
+def _console_read_language():
+    try:
+        with open('/usr/local/share/proxmenux/config.json', encoding='utf-8') as stream:
+            return json.load(stream).get('language')
+    except Exception:
+        return None
+
+
+try:
+    import console_snapshot
+    console_snapshot.ensure_publisher(
+        get_proxmox_node_name, get_cached_pvesh_cluster_resources,
+        _console_run_pvesh, _console_read_language,
+        get_pve_version=get_proxmox_version, version=None)
+except Exception:
+    pass
+
 
 # ─── Per-NIC live rate (moving window) ──────────────────────────
 # Keep the last ~30s of byte counters and derive the rate from the
diff --git a/AppImage/scripts/tests/fixtures/console/proc_disk_1 b/AppImage/scripts/tests/fixtures/console/proc_disk_1
new file mode 100644
index 00000000..91905f61
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/proc_disk_1
@@ -0,0 +1,4 @@
+   8       0 sda 0 0 100 0 0 0 200 0 0 0 0 0 0 0 0 0 0
+   8       1 sda1 0 0 50 0 0 0 100 0 0 0 0 0 0 0 0 0 0
+ 259       0 nvme0n1 0 0 300 0 0 0 400 0 0 0 0 0 0 0 0 0 0
+ 259       1 nvme0n1p1 0 0 150 0 0 0 200 0 0 0 0 0 0 0 0 0
diff --git a/AppImage/scripts/tests/fixtures/console/proc_disk_2 b/AppImage/scripts/tests/fixtures/console/proc_disk_2
new file mode 100644
index 00000000..72af122e
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/proc_disk_2
@@ -0,0 +1,4 @@
+   8       0 sda 0 0 102 0 0 0 204 0 0 0 0 0 0 0 0 0 0
+   8       1 sda1 0 0 500 0 0 0 1000 0 0 0 0 0 0 0 0 0 0
+ 259       0 nvme0n1 0 0 306 0 0 0 408 0 0 0 0 0 0 0 0 0 0
+ 259       1 nvme0n1p1 0 0 1500 0 0 0 2000 0 0 0 0 0 0 0 0 0 0
diff --git a/AppImage/scripts/tests/fixtures/console/proc_net_1 b/AppImage/scripts/tests/fixtures/console/proc_net_1
new file mode 100644
index 00000000..9d16984f
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/proc_net_1
@@ -0,0 +1,7 @@
+Inter-|   Receive                                                |  Transmit
+ face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
+  eno1: 1000 0 0 0 0 0 0 0 2000 0 0 0 0 0 0 0
+  eno2: 3000 0 0 0 0 0 0 0 4000 0 0 0 0 0 0 0
+    lo: 100 0 0 0 0 0 0 0 100 0 0 0 0 0 0 0
+  vmbr0: 100 0 0 0 0 0 0 0 100 0 0 0 0 0 0 0
+  bond0: 100 0 0 0 0 0 0 0 100 0 0 0 0 0 0 0
diff --git a/AppImage/scripts/tests/fixtures/console/proc_net_2 b/AppImage/scripts/tests/fixtures/console/proc_net_2
new file mode 100644
index 00000000..7ad24809
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/proc_net_2
@@ -0,0 +1,7 @@
+Inter-|   Receive                                                |  Transmit
+ face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
+  eno1: 1300 0 0 0 0 0 0 0 2500 0 0 0 0 0 0 0
+  eno2: 3700 0 0 0 0 0 0 0 4500 0 0 0 0 0 0 0
+    lo: 9999 0 0 0 0 0 0 0 9999 0 0 0 0 0 0 0
+  vmbr0: 9999 0 0 0 0 0 0 0 9999 0 0 0 0 0 0 0
+  bond0: 9999 0 0 0 0 0 0 0 9999 0 0 0 0 0 0 0
diff --git a/AppImage/scripts/tests/fixtures/console/proc_stat_1 b/AppImage/scripts/tests/fixtures/console/proc_stat_1
new file mode 100644
index 00000000..ac4938be
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/proc_stat_1
@@ -0,0 +1,3 @@
+cpu  100 0 50 850 0 0 0 0 0 0
+cpu0 50 0 25 425 0 0 0 0 0 0
+cpu1 50 0 25 425 0 0 0 0 0 0
diff --git a/AppImage/scripts/tests/fixtures/console/proc_stat_2 b/AppImage/scripts/tests/fixtures/console/proc_stat_2
new file mode 100644
index 00000000..b207ee2e
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/proc_stat_2
@@ -0,0 +1,3 @@
+cpu  120 0 60 920 0 0 0 0 0 0
+cpu0 60 0 30 460 0 0 0 0 0 0
+cpu1 60 0 30 460 0 0 0 0 0 0
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_cluster_resources.json b/AppImage/scripts/tests/fixtures/console/pvesh_cluster_resources.json
new file mode 100644
index 00000000..e9ff04a9
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_cluster_resources.json
@@ -0,0 +1,8 @@
+[
+  {"type":"node","node":"p3-01","status":"online","cpu":0.07,"maxcpu":16,"mem":1717986918,"maxmem":34359738368,"disk":21474836480,"maxdisk":53687091200,"uptime":612000,"level":"","cgroup-mode":2},
+  {"type":"node","node":"p3-02","status":"offline"},
+  {"type":"qemu","vmid":101,"name":"plex","node":"p3-01","status":"running","cpu":0.6,"maxcpu":2,"mem":4294967296,"maxmem":8589934592,"disk":0,"maxdisk":34359738368,"uptime":3600,"netin":123456789,"netout":98765432,"diskwrite":0,"tags":"media;prod","template":0},
+  {"type":"lxc","vmid":102,"name":"nginx","node":"p3-02","status":"unknown","tags":"edge;\u001b[31mred\nline","template":0},
+  {"type":"storage","id":"storage/p3-01/local-zfs","storage":"local-zfs","node":"p3-01","plugintype":"zfspool","shared":0,"status":"available","disk":21474836480,"maxdisk":53687091200,"content":"images,rootdir"},
+  {"type":"pool","pool":"unused"}
+]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_cluster_status.json b/AppImage/scripts/tests/fixtures/console/pvesh_cluster_status.json
new file mode 100644
index 00000000..3b11f0fd
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_cluster_status.json
@@ -0,0 +1,5 @@
+[
+  {"type":"cluster","name":"p3","quorate":1,"nodes":2,"version":7},
+  {"type":"node","name":"p3-01","nodeid":1,"online":1,"local":1,"ip":"node-one.invalid","level":""},
+  {"type":"node","name":"p3-02","nodeid":2,"online":0,"local":0,"ip":"node-two.invalid","level":""}
+]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_no_quorum_status.json b/AppImage/scripts/tests/fixtures/console/pvesh_no_quorum_status.json
new file mode 100644
index 00000000..67cbaf74
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_no_quorum_status.json
@@ -0,0 +1,5 @@
+[
+  {"type":"cluster","name":"p3","quorate":0,"nodes":2,"version":8},
+  {"type":"node","name":"p3-01","nodeid":1,"online":1,"local":1,"ip":"node-one.invalid","level":""},
+  {"type":"node","name":"p3-02","nodeid":2,"online":0,"local":0,"ip":"node-two.invalid","level":""}
+]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_resources_full.json b/AppImage/scripts/tests/fixtures/console/pvesh_resources_full.json
new file mode 100644
index 00000000..c354ac8d
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_resources_full.json
@@ -0,0 +1,10 @@
+[
+  {"id": "node/p3-01", "type": "node", "node": "p3-01", "status": "online", "cpu": 0.07, "maxcpu": 16},
+  {"id": "qemu/101", "type": "qemu", "vmid": 101, "name": "plex", "node": "p3-01", "status": "running", "cpu": 0.6, "maxcpu": 2, "mem": 4294967296, "maxmem": 8589934592, "tags": "media;prod"},
+  {"id": "storage/p3-01/local-zfs", "type": "storage", "storage": "local-zfs", "node": "p3-01", "status": "available"},
+  {"id": "lxc/205", "type": "lxc", "vmid": 205, "name": "nginx", "node": "p3-02", "status": "running", "cpu": 0.3, "maxcpu": 1, "mem": 858993459, "maxmem": 2147483648},
+  {"id": "node/p3-02", "type": "node", "node": "p3-02", "status": "online", "cpu": 0.09, "maxcpu": 16},
+  {"id": "qemu/103", "type": "qemu", "vmid": 103, "name": "db", "node": "p3-02", "status": "stopped", "cpu": 0, "maxcpu": 4, "mem": 0, "maxmem": 8589934592},
+  {"id": "pool/media", "type": "pool", "pool": "media"},
+  {"id": "sdn/vnet/media", "type": "sdn", "sdn": "media"}
+]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_resources_vm.json b/AppImage/scripts/tests/fixtures/console/pvesh_resources_vm.json
new file mode 100644
index 00000000..b8dbcd29
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_resources_vm.json
@@ -0,0 +1,5 @@
+[
+  {"id": "qemu/101", "type": "qemu", "vmid": 101, "name": "plex", "node": "p3-01", "status": "running", "cpu": 0.6, "maxcpu": 2, "mem": 4294967296, "maxmem": 8589934592, "tags": "media;prod"},
+  {"id": "lxc/205", "type": "lxc", "vmid": 205, "name": "nginx", "node": "p3-02", "status": "running", "cpu": 0.3, "maxcpu": 1, "mem": 858993459, "maxmem": 2147483648},
+  {"id": "qemu/103", "type": "qemu", "vmid": 103, "name": "db", "node": "p3-02", "status": "stopped", "cpu": 0, "maxcpu": 4, "mem": 0, "maxmem": 8589934592}
+]
diff --git a/AppImage/scripts/tests/fixtures/console/pvesh_standalone_status.json b/AppImage/scripts/tests/fixtures/console/pvesh_standalone_status.json
new file mode 100644
index 00000000..249e6347
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/pvesh_standalone_status.json
@@ -0,0 +1,3 @@
+[
+  {"type":"node","name":"p3-01","nodeid":1,"online":1,"local":1,"ip":"node-one.invalid","level":""}
+]
diff --git a/AppImage/scripts/tests/fixtures/console/viewmodel_cluster.json b/AppImage/scripts/tests/fixtures/console/viewmodel_cluster.json
new file mode 100644
index 00000000..104337b2
--- /dev/null
+++ b/AppImage/scripts/tests/fixtures/console/viewmodel_cluster.json
@@ -0,0 +1,43 @@
+{
+  "glyphs": "ascii",
+  "lang": "en",
+  "now": 1760704496,
+  "selected_view": "node",
+  "sort": "cpu",
+  "host_other": {"cpu": 1.6, "mem": 8589934592},
+  "local": {
+    "cpu_total": 0.7,
+    "cpu_history": [0.1, 0.4, 0.8, 0.2, 0.6],
+    "per_core": [0.1, 0.8, 0.2, 0.3],
+    "load1": 0.42,
+    "load5": 0.51,
+    "load15": 0.48,
+    "threads": 4,
+    "temp_c": 48,
+    "uptime": 619200,
+    "mem": {"total": 34359738368, "used": 17179869184, "available": 17179869184, "cached": 1073741824, "arc": 2147483648, "swap_used": 0, "swap_total": 8589934592},
+    "zram": {"capacity_bytes": 3435973836, "swap_used_bytes": 150323855, "compression_ratio": 1.8},
+    "net": {"rx_bps": 400000, "tx_bps": 100000, "nic": "eno1", "speed_mbps": 2500},
+    "disk": {"read_bps": 44040192, "write_bps": 8388608},
+    "storages_local": [{"storage": "local-zfs", "used": 21474836480, "total": 53687091200}]
+  },
+  "snapshot": {
+    "schema": 1,
+    "age_s": 2,
+    "language": "en",
+    "node": {"name": "p3-01", "pve_version": "pve-manager/9.0.3/abc"},
+    "cluster": {"status": "ok", "is_cluster": true, "name": "p3", "quorate": true, "nodes_total": 3, "nodes_online": 2, "members": [{"name": "p3-01", "online": true, "local": true}, {"name": "p3-02", "online": true, "local": false}, {"name": "p3-03", "online": false, "local": false}]},
+    "resources": {
+      "status": "ok",
+      "nodes": [{"name": "p3-01", "status": "online", "cpu": 0.07, "maxcpu": 4, "mem": 17179869184, "maxmem": 34359738368, "disk": 21474836480, "maxdisk": 53687091200, "uptime": 619200}, {"name": "p3-02", "status": "online", "cpu": 0.09, "maxcpu": 8, "mem": 19327352832, "maxmem": 34359738368, "disk": 21474836480, "maxdisk": 53687091200, "uptime": 619200}],
+      "guests": [
+        {"vmid": 101, "type": "qemu", "name": "plex", "node": "p3-01", "status": "running", "cpu": 0.6, "maxcpu": 2, "mem": 4294967296, "maxmem": 8589934592, "uptime": 3600, "template": false, "tags": ["media"]},
+        {"vmid": 103, "type": "qemu", "name": "db", "node": "p3-02", "status": "running", "cpu": 0.05, "maxcpu": 2, "mem": 6442450944, "maxmem": 8589934592, "uptime": 7200, "template": false},
+        {"vmid": 205, "type": "lxc", "name": "nginx", "node": "p3-02", "status": "running", "cpu": 0.3, "maxcpu": 1, "mem": 858993459, "maxmem": 2147483648, "uptime": 1800, "template": false},
+        {"vmid": 110, "type": "qemu", "name": "win", "node": "p3-01", "status": "unknown", "cpu": 0, "maxcpu": 4, "mem": 2147483648, "maxmem": 4294967296, "uptime": 0, "template": false},
+        {"vmid": 999, "type": "lxc", "name": "template", "node": "p3-01", "status": "stopped", "cpu": 0.9, "maxcpu": 1, "mem": 4294967296, "template": true}
+      ],
+      "storages": [{"storage": "local-zfs", "node": "p3-01", "disk": 21474836480, "maxdisk": 53687091200, "status": "available"}]
+    }
+  }
+}
diff --git a/AppImage/scripts/tests/test_cluster_resources_cache.py b/AppImage/scripts/tests/test_cluster_resources_cache.py
new file mode 100644
index 00000000..a02b1e05
--- /dev/null
+++ b/AppImage/scripts/tests/test_cluster_resources_cache.py
@@ -0,0 +1,145 @@
+"""Characterize the shared pvesh cluster resources cache without Flask imports."""
+
+import ast
+import json
+import subprocess
+import threading
+import unittest
+from pathlib import Path
+from types import SimpleNamespace
+from unittest.mock import Mock
+
+
+SERVER_SOURCE = Path(__file__).resolve().parents[1] / "flask_server.py"
+FIXTURES = Path(__file__).resolve().parent / "fixtures" / "console"
+
+
+def load_server_functions(*names):
+    """Compile selected production functions without importing host-bound Flask state."""
+    tree = ast.parse(SERVER_SOURCE.read_text(encoding="utf-8"))
+    functions = {
+        node.name: node
+        for node in tree.body
+        if isinstance(node, ast.FunctionDef) and node.name in names
+    }
+    missing = set(names) - functions.keys()
+    if missing:
+        raise AssertionError(f"Missing production functions: {sorted(missing)}")
+    module = ast.Module(body=[functions[name] for name in names], type_ignores=[])
+    ast.fix_missing_locations(module)
+    namespace = {}
+    exec(compile(module, str(SERVER_SOURCE), "exec"), namespace)
+    return namespace
+
+
+class ClusterResourcesCacheTests(unittest.TestCase):
+    def setUp(self):
+        self.full = json.loads((FIXTURES / "pvesh_resources_full.json").read_text(encoding="utf-8"))
+        self.vm = json.loads((FIXTURES / "pvesh_resources_vm.json").read_text(encoding="utf-8"))
+        self.clock = Mock(return_value=100.0)
+        self.run = Mock(return_value=SimpleNamespace(returncode=0, stdout=json.dumps(self.full)))
+        self.server = load_server_functions(
+            "get_cached_pvesh_cluster_resources", "get_cached_pvesh_cluster_resources_vm"
+        )
+        self.server.update({
+            "_pvesh_cache": {
+                "cluster_resources": None,
+                "cluster_resources_vm_time": 0,
+            },
+            "_pvesh_cluster_resources_lock": threading.Lock(),
+            "_PVESH_CACHE_TTL": 5,
+            "time": SimpleNamespace(time=self.clock),
+            "subprocess": SimpleNamespace(run=self.run),
+            "json": json,
+        })
+
+    def full_resources(self):
+        return self.server["get_cached_pvesh_cluster_resources"]()
+
+    def vm_resources(self):
+        return self.server["get_cached_pvesh_cluster_resources_vm"]()
+
+    def test_one_pvesh_call_serves_full_and_vm_within_ttl(self):
+        self.assertEqual(self.full_resources(), self.full)
+        self.assertEqual(self.vm_resources(), self.vm)
+        self.assertEqual(self.full_resources(), self.full)
+        self.assertEqual(self.run.call_count, 1)
+        self.run.assert_called_once_with(
+            ["pvesh", "get", "/cluster/resources", "--output-format", "json"],
+            capture_output=True, text=True, timeout=10,
+        )
+
+    def test_vm_filter_matches_type_vm_fixture_exactly_in_original_order(self):
+        self.assertEqual(self.vm_resources(), self.vm)
+        self.assertEqual([row["id"] for row in self.vm_resources()],
+                         ["qemu/101", "lxc/205", "qemu/103"])
+        self.assertEqual(self.run.call_count, 1)
+
+    def test_failure_and_timeout_return_last_cache_or_empty_list(self):
+        self.run.side_effect = [
+            SimpleNamespace(returncode=1, stdout=""),
+            subprocess.TimeoutExpired("pvesh", 10),
+            SimpleNamespace(returncode=0, stdout=json.dumps(self.full)),
+            subprocess.TimeoutExpired("pvesh", 10),
+        ]
+        self.assertEqual(self.full_resources(), [])
+        self.assertEqual(self.vm_resources(), [])
+        self.assertEqual(self.full_resources(), self.full)
+        self.clock.return_value = 105.0
+        self.assertEqual(self.vm_resources(), self.vm)
+        self.assertEqual(self.run.call_count, 4)
+
+    def test_concurrent_full_and_vm_reads_share_one_in_flight_call(self):
+        entered = threading.Event()
+        release = threading.Event()
+        results = {}
+
+        def slow_run(*_args, **_kwargs):
+            entered.set()
+            if not release.wait(2):
+                raise AssertionError("pvesh fixture was not released")
+            return SimpleNamespace(returncode=0, stdout=json.dumps(self.full))
+
+        self.run.side_effect = slow_run
+        full_thread = threading.Thread(target=lambda: results.update(full=self.full_resources()))
+        vm_thread = threading.Thread(target=lambda: results.update(vm=self.vm_resources()))
+        full_thread.start()
+        try:
+            self.assertTrue(entered.wait(2))
+            vm_thread.start()
+        finally:
+            release.set()
+            full_thread.join(2)
+            if vm_thread.ident is not None:
+                vm_thread.join(2)
+
+        self.assertFalse(full_thread.is_alive())
+        self.assertFalse(vm_thread.is_alive())
+        self.assertEqual(results, {"full": self.full, "vm": self.vm})
+        self.assertEqual(self.run.call_count, 1)
+
+    def test_ttl_expiry_triggers_exactly_one_refresh(self):
+        self.assertEqual(self.vm_resources(), self.vm)
+        self.clock.return_value = 104.999
+        self.assertEqual(self.full_resources(), self.full)
+        self.assertEqual(self.run.call_count, 1)
+        self.clock.return_value = 105.0
+        self.assertEqual(self.full_resources(), self.full)
+        self.assertEqual(self.vm_resources(), self.vm)
+        self.assertEqual(self.run.call_count, 2)
+
+    def test_legacy_invalidation_timestamp_refreshes_shared_cache(self):
+        self.assertEqual(self.full_resources(), self.full)
+        self.server["_pvesh_cache"]["cluster_resources_vm_time"] = 0
+        self.assertEqual(self.vm_resources(), self.vm)
+        self.assertEqual(self.run.call_count, 2)
+
+    def test_successful_empty_response_is_cached_for_ttl(self):
+        self.run.return_value = SimpleNamespace(returncode=0, stdout="[]")
+        self.assertEqual(self.full_resources(), [])
+        self.assertEqual(self.vm_resources(), [])
+        self.assertEqual(self.run.call_count, 1)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/AppImage/scripts/tests/test_console_metrics.py b/AppImage/scripts/tests/test_console_metrics.py
new file mode 100644
index 00000000..7cd09dde
--- /dev/null
+++ b/AppImage/scripts/tests/test_console_metrics.py
@@ -0,0 +1,217 @@
+"""Local console metrics from an injected proc/sys tree."""
+
+import sys
+import tempfile
+import unittest
+from pathlib import Path
+from types import SimpleNamespace
+
+
+SCRIPTS_DIR = Path(__file__).resolve().parents[1]
+if str(SCRIPTS_DIR) not in sys.path:
+    sys.path.insert(0, str(SCRIPTS_DIR))
+
+from console_metrics import LocalSampler, host_other
+
+
+FIXTURES = Path(__file__).parent / "fixtures" / "console"
+
+
+class LocalSamplerTests(unittest.TestCase):
+    def setUp(self):
+        temporary = tempfile.TemporaryDirectory()
+        self.addCleanup(temporary.cleanup)
+        self.root = Path(temporary.name)
+        self.write("proc/loadavg", "0.25 0.50 0.75 1/100 1\n")
+        self.write("proc/meminfo", (
+            "MemTotal: 10000 kB\nMemFree: 1000 kB\nMemAvailable: 4000 kB\n"
+            "Cached: 2000 kB\nSReclaimable: 500 kB\n"
+            "SwapTotal: 3000 kB\nSwapFree: 2500 kB\n"
+        ))
+        self.write("proc/swaps", "Filename Type Size Used Priority\n")
+        self.write("proc/stat", (FIXTURES / "proc_stat_1").read_text())
+        self.write("proc/net/dev", (FIXTURES / "proc_net_1").read_text())
+        self.write("proc/diskstats", (FIXTURES / "proc_disk_1").read_text())
+        for nic in ("eno1", "eno2", "lo", "vmbr0", "bond0"):
+            self.write(f"sys/class/net/{nic}/speed", "2500\n")
+            self.write(f"sys/class/net/{nic}/operstate", "up\n")
+            (self.root / "sys/class/net" / nic / "device").mkdir()
+        for disk in ("sda", "nvme0n1"):
+            (self.root / "sys/block" / disk / "device").mkdir(parents=True)
+
+    def write(self, path, content):
+        target = self.root / path
+        target.parent.mkdir(parents=True, exist_ok=True)
+        target.write_text(content, encoding="utf-8")
+
+    def sampler(self, **kwargs):
+        return LocalSampler(proc_root=self.root, storage_paths=[], **kwargs)
+
+    def second_counters(self):
+        for target, fixture in (("proc/stat", "proc_stat_2"),
+                                ("proc/net/dev", "proc_net_2"),
+                                ("proc/diskstats", "proc_disk_2")):
+            self.write(target, (FIXTURES / fixture).read_text())
+
+    def test_two_samples_give_exact_cpu_network_and_whole_disk_rates(self):
+        sampler = self.sampler()
+        first = sampler.sample(10.0)
+        self.assertIsNone(first["cpu_total"])
+        self.assertEqual(first["per_core"], [0, 0])
+        self.assertEqual(first["cpu_history"], [])
+        self.assertEqual(first["net"]["rx_bps"], 0)
+        self.assertEqual(first["disk"]["read_bps"], 0)
+
+        self.second_counters()
+        second = sampler.sample(11.0)
+        self.assertEqual(second["cpu_total"], 0.3)
+        self.assertEqual(second["per_core"], [0.3, 0.3])
+        self.assertEqual(second["cpu_history"], [0.3])
+        self.assertEqual(second["threads"], 2)
+        self.assertEqual((second["load1"], second["load5"], second["load15"]),
+                         (0.25, 0.5, 0.75))
+        self.assertEqual(second["net"], {
+            "rx_bps": 1000, "tx_bps": 1000, "nic": "eno1", "speed_mbps": 2500,
+        })
+        self.assertEqual(second["disk"], {"read_bps": 4096, "write_bps": 6144})
+
+    def test_only_physical_nics_with_device_are_counted(self):
+        for nic in ("fwbr1", "fwpr1", "fwln1", "tap1", "veth1", "wlan0"):
+            self.write(f"sys/class/net/{nic}/speed", "1000\n")
+            (self.root / "sys/class/net" / nic / "device").mkdir()
+        (self.root / "sys/class/net/eno2/device").rmdir()
+        self.write("sys/class/net/eno1/operstate", "down\n")
+        self.write("sys/class/net/wlan0/operstate", "up\n")
+        self.write("proc/net/dev", (FIXTURES / "proc_net_1").read_text() +
+                   "  fwbr1: 1 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0\n"
+                   "  fwpr1: 1 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0\n"
+                   "  fwln1: 1 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0\n"
+                   "   tap1: 1 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0\n"
+                   "  veth1: 1 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0\n"
+                   "  wlan0: 10 0 0 0 0 0 0 0 20 0 0 0 0 0 0 0\n")
+        sampler = self.sampler()
+        sampler.sample(1.0)
+        second = (self.root / "proc/net/dev").read_text()
+        second = second.replace("wlan0: 10", "wlan0: 110").replace(
+            "0 0 0 0 0 0 0 20", "0 0 0 0 0 0 0 220")
+        second = second.replace("eno2: 3000", "eno2: 10000")
+        for nic in ("fwbr1", "fwpr1", "fwln1", "tap1", "veth1"):
+            second = second.replace(f"{nic}: 1", f"{nic}: 1001")
+        self.write("proc/net/dev", second)
+        result = sampler.sample(2.0)["net"]
+        self.assertEqual(result["rx_bps"], 100)
+        self.assertEqual(result["tx_bps"], 200)
+        self.assertEqual(result["nic"], "wlan0")
+
+    def test_mem_used_includes_arc_once_and_cached_is_separate(self):
+        self.write("proc/spl/kstat/zfs/arcstats", "13 1 0x01 1 1 1\nname type data\nsize 4 1048576\n")
+        result = self.sampler().sample(1.0)["mem"]
+        self.assertEqual(result, {
+            "total": 10240000, "used": 6144000, "available": 4096000,
+            "cached": 2560000, "arc": 1048576,
+            "swap_used": 512000, "swap_total": 3072000,
+        })
+        (self.root / "proc/spl/kstat/zfs/arcstats").unlink()
+        without_arc = self.sampler().sample(1.0)["mem"]
+        self.assertEqual(without_arc["arc"], 0)
+        self.assertEqual(without_arc["used"], result["used"])
+
+    def test_hwmon_priority_and_thermal_fallback(self):
+        self.write("sys/class/hwmon/hwmon0/name", "other\n")
+        self.write("sys/class/hwmon/hwmon0/temp1_input", "42000\n")
+        self.write("sys/class/hwmon/hwmon1/name", "coretemp\n")
+        self.write("sys/class/hwmon/hwmon1/temp1_input", "51000\n")
+        self.write("sys/class/hwmon/hwmon2/name", "k10temp\n")
+        self.write("sys/class/hwmon/hwmon2/temp1_input", "48000\n")
+        self.write("sys/class/hwmon/hwmon2/temp1_label", "Tctl\n")
+        self.write("sys/class/hwmon/hwmon2/temp2_input", "46000\n")
+        self.write("sys/class/hwmon/hwmon2/temp2_label", "Tdie\n")
+        self.assertEqual(self.sampler().sample(1.0)["temp_c"], 46.0)
+        (self.root / "sys/class/hwmon/hwmon2/temp2_input").unlink()
+        self.assertEqual(self.sampler().sample(1.0)["temp_c"], 48.0)
+        (self.root / "sys/class/hwmon/hwmon2/temp1_input").unlink()
+        self.assertEqual(self.sampler().sample(1.0)["temp_c"], 51.0)
+        (self.root / "sys/class/hwmon/hwmon1/temp1_input").unlink()
+        self.assertEqual(self.sampler().sample(1.0)["temp_c"], 42.0)
+        (self.root / "sys/class/hwmon/hwmon0/temp1_input").unlink()
+        self.write("sys/class/thermal/thermal_zone0/temp", "39000\n")
+        self.assertEqual(self.sampler().sample(1.0)["temp_c"], 39.0)
+        (self.root / "sys/class/thermal/thermal_zone0/temp").unlink()
+        self.assertIsNone(self.sampler().sample(1.0)["temp_c"])
+
+    def test_zram_uses_injected_paths(self):
+        self.write("sys/block/zram0/disksize", "1073741824\n")
+        self.write("sys/block/zram0/mm_stat", "2097152 1048576 1179648\n")
+        self.write("sys/block/zram0/comp_algorithm", "lzo [zstd] lz4\n")
+        self.write("proc/swaps", "Filename Type Size Used Priority\n/dev/zram0 partition 1048576 1024 100\n")
+        zram = self.sampler().sample(1.0)["zram"]
+        self.assertEqual(zram["devices"][0]["algorithm"], "zstd")
+        self.assertEqual(zram["capacity_bytes"], 1073741824)
+        self.assertEqual(zram["swap_used_bytes"], 1048576)
+        self.assertEqual(zram["compression_ratio"], 2.0)
+        (self.root / "sys/block/zram0/mm_stat").unlink()
+        self.assertIsNone(self.sampler().sample(1.0)["zram"])
+
+    def test_counter_decrease_resets_without_negative_rates(self):
+        sampler = self.sampler()
+        sampler.sample(1.0)
+        self.write("proc/stat", "cpu 1 0 1 8 0 0 0 0 0 0\ncpu0 1 0 0 4 0 0 0 0 0 0\n")
+        self.write("proc/net/dev", (FIXTURES / "proc_net_1").read_text().replace("eno1: 1000", "eno1: 1"))
+        self.write("proc/diskstats", (FIXTURES / "proc_disk_1").read_text().replace("sda 0 0 100", "sda 0 0 1"))
+        reset = sampler.sample(2.0)
+        self.assertIsNone(reset["cpu_total"])
+        self.assertEqual(reset["net"]["rx_bps"], 0)
+        self.assertEqual(reset["disk"]["read_bps"], 0)
+        self.second_counters()
+        later = sampler.sample(3.0)
+        self.assertIsNotNone(later["cpu_total"])
+        self.assertGreaterEqual(later["net"]["rx_bps"], 0)
+        self.assertGreaterEqual(later["disk"]["read_bps"], 0)
+
+    def test_history_is_capped_at_sixty(self):
+        sampler = self.sampler()
+        sampler.sample(0.0)
+        for index in range(1, 66):
+            self.write("proc/stat", f"cpu {100 + index} 0 50 850 0 0 0 0 0 0\n")
+            sample = sampler.sample(float(index))
+        self.assertEqual(len(sample["cpu_history"]), 60)
+        self.assertEqual(sample["cpu_history"][-1], 1.0)
+
+    def test_statvfs_is_injectable(self):
+        seen = []
+
+        def fake_statvfs(path):
+            seen.append(path)
+            return SimpleNamespace(f_frsize=4096, f_blocks=100, f_bfree=40, f_bavail=30)
+
+        sampler = LocalSampler(proc_root=self.root, storage_paths=[("local", "/")],
+                               statvfs=fake_statvfs)
+        self.assertEqual(sampler.sample(1.0)["storages_local"], [{
+            "storage": "local", "total": 409600,
+            "used": 245760, "available": 122880,
+        }])
+        self.assertEqual(seen, [str(self.root)])
+
+
+class HostOtherTests(unittest.TestCase):
+    def test_arithmetic_filters_local_running_guests(self):
+        local = {"cpu_total": 0.5, "threads": 8, "mem": {"used": 10_000, "arc": 1000}}
+        guests = [
+            {"node": "p3-01", "status": "running", "cpu": 0.5, "maxcpu": 2, "mem": 3000},
+            {"node": "p3-01", "status": "stopped", "cpu": 0.5, "maxcpu": 2, "mem": 2000},
+            {"node": "p3-02", "status": "running", "cpu": 0.5, "maxcpu": 2, "mem": 4000},
+        ]
+        self.assertEqual(host_other(local, guests, "p3-01", snapshot_fresh=True),
+                         {"cpu": 3.0, "mem": 4000})
+
+    def test_clamps_and_omits_stale_snapshot(self):
+        local = {"cpu_total": 0.1, "threads": 2, "mem": {"used": 1000, "arc": 500}}
+        guests = [{"node": "p3-01", "status": "running", "cpu": 1, "maxcpu": 2, "mem": 2000}]
+        self.assertEqual(host_other(local, guests, "p3-01", snapshot_fresh=True),
+                         {"cpu": 0.0, "mem": 0})
+        self.assertEqual(host_other(local, guests, "p3-01", snapshot_fresh=False),
+                         {"cpu": None, "mem": None})
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/AppImage/scripts/tests/test_console_render.py b/AppImage/scripts/tests/test_console_render.py
new file mode 100644
index 00000000..723c9248
--- /dev/null
+++ b/AppImage/scripts/tests/test_console_render.py
@@ -0,0 +1,283 @@
+"""Contract tests for the terminal-independent console renderer."""
+
+import copy
+import json
+import sys
+import unittest
+from pathlib import Path
+
+
+SCRIPTS_DIR = Path(__file__).resolve().parents[1]
+if str(SCRIPTS_DIR) not in sys.path:
+    sys.path.insert(0, str(SCRIPTS_DIR))
+
+from console_render import ACS_TOKENS, render, sanitize
+
+
+CATALOG = json.loads((SCRIPTS_DIR.parent / "messages" / "en" / "common.json")
+                     .read_text(encoding="utf-8"))["runtime"]["consoleTui"]
+
+
+def tr(key):
+    value = CATALOG
+    try:
+        for part in key.removeprefix("runtime.consoleTui.").split("."):
+            value = value[part]
+    except (KeyError, TypeError):
+        return key
+    return value
+
+
+def lines(grid):
+    return ["".join(cell[0] for cell in row) for row in grid]
+
+
+class ConsoleRenderTests(unittest.TestCase):
+    def setUp(self):
+        fixture = Path(__file__).parent / "fixtures" / "console" / "viewmodel_cluster.json"
+        self.vm = json.loads(fixture.read_text(encoding="utf-8"))
+        self.vm["tr"] = tr
+
+    def test_exact_grid_dimensions_for_both_views(self):
+        for view in ("node", "cluster"):
+            for width, height in ((80, 24), (128, 48), (240, 67)):
+                with self.subTest(view=view, size=(width, height)):
+                    self.vm["selected_view"] = view
+                    grid = render(self.vm, width, height)
+                    self.assertEqual(len(grid), height)
+                    self.assertTrue(all(len(row) == width for row in grid))
+                    self.assertTrue(all(len(cell) == 3 for row in grid for cell in row))
+
+    def test_node_80x24_panel_positions_and_collapse(self):
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("[Node] Cluster", rows[0])
+        self.assertIn("CPU", rows[1])
+        self.assertIn("MEMORY", rows[1])
+        self.assertEqual(rows[1][42:44], "++")
+        self.assertIn("NET", rows[6])
+        self.assertEqual(rows[6][22:24], "++")
+        self.assertEqual(rows[6][50:52], "++")
+        self.assertIn("TOP CPU", rows[10])
+        self.assertIn("TOP MEMORY", rows[10])
+        self.assertEqual(rows[5][0], "+")
+        self.assertNotIn("CPU0", "\n".join(rows))
+        self.assertIn("q login", rows[23])
+        self.assertIn("data 2s", rows[23])
+        self.assertIn("+2 nodes", "\n".join(rows[6:10]))
+        self.assertIn("TOP CPU (host cores)", rows[10])
+        self.assertEqual(rows[10].count("(host cores)"), 1)
+
+    def test_selected_tab_uses_selected_role(self):
+        grid = render(self.vm, 80, 24)
+        self.assertEqual(grid[0][22], ("[", "selected", ("reverse",)))
+        self.vm["selected_view"] = "cluster"
+        grid = render(self.vm, 80, 24)
+        self.assertEqual(grid[0][22][1], "text")
+        self.assertEqual(grid[0][27][1], "selected")
+
+    def test_top_lists_use_host_cores_and_memory_not_status(self):
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("plex", rows[11][:40])
+        self.assertIn("nginx", rows[12][:40])
+        self.assertIn("db", rows[11][40:])
+        self.assertIn("plex", rows[12][40:])
+        self.assertIn("win", "\n".join(row[40:] for row in rows[11:23]))
+        self.assertNotIn("template", "\n".join(rows))
+
+    def test_host_other_titles_only_with_fresh_snapshot(self):
+        rows = lines(render(self.vm, 128, 48))
+        self.assertIn("host other 1.6c", rows[14])
+        self.assertIn("host other 8.0 GiB", rows[14])
+        self.vm["snapshot"]["resources"]["status"] = "stale"
+        rows = lines(render(self.vm, 128, 48))
+        self.assertNotIn("host other", rows[14])
+        self.assertIn("old", rows[14])
+
+    def test_host_other_none_is_hidden_independently(self):
+        self.vm["host_other"]["cpu"] = None
+        rows = lines(render(self.vm, 128, 48))
+        self.assertNotIn("host other", rows[14][:64])
+        self.assertIn("host other 8.0 GiB", rows[14][64:])
+
+    def test_stale_and_unavailable_age_labels(self):
+        self.vm["snapshot"]["age_s"] = 120
+        rows = lines(render(self.vm, 128, 48))
+        self.assertIn("(2m old)", rows[14])
+        self.vm["snapshot"]["age_s"] = 180
+        rows = lines(render(self.vm, 128, 48))
+        self.assertIn("MONITOR UNAVAILABLE (3m)", rows[0])
+        self.vm["snapshot"]["age_s"] = 3600
+        self.assertIn("(1h old)", lines(render(self.vm, 128, 48))[14])
+
+    def test_pve_version_accepts_plain_and_manager_forms(self):
+        for version in ("9.0.6", "pve-manager/9.0.6/abc"):
+            with self.subTest(version=version):
+                self.vm["snapshot"]["node"]["pve_version"] = version
+                self.assertIn("PVE 9.0", lines(render(self.vm, 80, 24))[0])
+
+    def test_translation_with_stray_brace_keeps_rendering(self):
+        self.vm["tr"] = lambda key: "data {age} {" if key.endswith("status.data") else tr(key)
+        self.assertIn("data 2s {", lines(render(self.vm, 80, 24))[-1])
+
+    def test_cluster_offline_first_and_coverage_suppresses_totals(self):
+        self.vm["selected_view"] = "cluster"
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("p3-03", rows[2])
+        self.assertIn("offline", rows[2])
+        self.assertIn("coverage 2/3", rows[0])
+        self.assertNotIn("64 GiB", rows[0])
+
+    def test_cluster_totals_require_quorum_and_full_coverage(self):
+        self.vm["selected_view"] = "cluster"
+        cluster = self.vm["snapshot"]["cluster"]
+        cluster["nodes_total"] = 2
+        cluster["members"] = cluster["members"][:2]
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("64 GiB", rows[0])
+        cluster["quorate"] = False
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("NO QUORUM", rows[0])
+        self.assertNotIn("64 GiB", rows[0])
+
+    def test_cluster_node_bars_and_overflow_markers(self):
+        self.vm["selected_view"] = "cluster"
+        grid = render(self.vm, 80, 24)
+        self.assertIn("#", lines(grid)[3])
+        self.assertIn(".", lines(grid)[3])
+        cluster = self.vm["snapshot"]["cluster"]
+        for index in range(5):
+            cluster["members"].append({"name": f"p3-{index + 10}", "online": False})
+        cluster["nodes_total"] = len(cluster["members"])
+        guests = self.vm["snapshot"]["resources"]["guests"]
+        for index in range(13):
+            extra = copy.deepcopy(guests[0])
+            extra["vmid"] = 300 + index
+            extra["name"] = f"guest-{index}"
+            guests.append(extra)
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("+3 nodes", "\n".join(rows))
+        self.assertIn("+5 guests", "\n".join(rows))
+
+    def test_cluster_guest_sort_and_status(self):
+        self.vm["selected_view"] = "cluster"
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("plex", rows[7])
+        self.assertIn("nginx", rows[8])
+        self.vm["sort"] = "mem"
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("db", rows[7])
+        self.assertIn("plex", rows[8])
+        self.assertIn("unknown", "\n".join(rows))
+
+    def test_host_other_uses_viewmodel_values(self):
+        self.vm["host_other"] = {"cpu": 0.0, "mem": 0}
+        rows = lines(render(self.vm, 128, 48))
+        self.assertIn("host other 0.0c", rows[14])
+        self.assertIn("host other 0 MiB", rows[14])
+
+    def test_node_growth_and_standalone(self):
+        rows = lines(render(self.vm, 128, 48))
+        self.assertIn("CPU0", "\n".join(rows[:9]))
+        self.assertIn("TOP CPU", rows[14])
+        guest = self.vm["snapshot"]["resources"]["guests"][0]
+        guest["net_rx_bps"] = 1024
+        guest["net_tx_bps"] = 2048
+        guest["disk_read_bps"] = 4096
+        guest["disk_write_bps"] = 8192
+        rows = lines(render(self.vm, 240, 67))
+        self.assertEqual(rows[65][0], "+")
+        self.assertIn("q login", rows[66])
+        self.assertIn("UP", rows[14 + 1])
+        self.assertIn("NET", rows[14 + 1])
+        self.assertIn("DISK", rows[14 + 1])
+        self.vm["selected_view"] = "cluster"
+        rows = lines(render(self.vm, 240, 67))
+        self.assertIn("CORES", "\n".join(rows[:8]))
+        self.assertIn("UPTIME", "\n".join(rows[:8]))
+        self.assertIn("HA", "\n".join(rows[:8]))
+        self.assertIn("TAGS", "\n".join(rows[:8]))
+        self.vm["selected_view"] = "node"
+        self.vm["snapshot"]["cluster"]["is_cluster"] = False
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("STANDALONE", rows[0])
+        self.assertIn("STANDALONE", rows[6])
+        self.assertNotIn("CLUSTER", rows[6])
+        self.vm["selected_view"] = "cluster"
+        self.vm["snapshot"]["cluster"]["members"] = []
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("p3-01", rows[2])
+
+    def test_no_quorum_and_stale_roles(self):
+        self.vm["selected_view"] = "cluster"
+        self.vm["snapshot"]["cluster"]["quorate"] = False
+        grid = render(self.vm, 80, 24)
+        header = lines(grid)[0]
+        self.assertEqual(grid[0][header.index("NO QUORUM")][1], "crit")
+        self.vm["selected_view"] = "node"
+        grid = render(self.vm, 80, 24)
+        self.assertIn("NO QUORUM", lines(grid)[0])
+        self.vm["snapshot"]["resources"]["status"] = "stale"
+        grid = render(self.vm, 80, 24)
+        self.assertEqual(grid[11][3][1], "dim")
+
+    def test_status_starting_warn_unavailable_crit(self):
+        self.vm["snapshot"]["resources"]["status"] = "starting"
+        grid = render(self.vm, 80, 24)
+        self.assertIn("Monitor starting", lines(grid)[0])
+        self.assertEqual(grid[0][lines(grid)[0].index("Monitor starting")][1], "warn")
+        self.assertIn("waiting for data", "\n".join(lines(grid)))
+        self.vm["snapshot"]["resources"]["status"] = "unavailable"
+        grid = render(self.vm, 80, 24)
+        self.assertIn("MONITOR UNAVAILABLE", lines(grid)[0])
+        self.assertEqual(grid[0][lines(grid)[0].index("MONITOR UNAVAILABLE")][1], "crit")
+
+    def test_absent_snapshot_uses_local_uptime_for_starting(self):
+        self.vm["snapshot"] = None
+        self.vm["local"]["uptime"] = 80
+        grid = render(self.vm, 80, 24)
+        self.assertIn("Monitor starting", lines(grid)[0])
+        self.assertEqual(grid[0][lines(grid)[0].index("Monitor starting")][1], "warn")
+        self.vm["local"]["uptime"] = 180
+        grid = render(self.vm, 80, 24)
+        self.assertIn("MONITOR UNAVAILABLE", lines(grid)[0])
+        self.assertEqual(grid[0][lines(grid)[0].index("MONITOR UNAVAILABLE")][1], "crit")
+
+    def test_small_terminal_message(self):
+        for width, height in ((79, 24), (80, 23)):
+            rows = lines(render(self.vm, width, height))
+            self.assertIn("Terminal too small (80x24)", "\n".join(rows))
+            self.assertNotIn("TOP CPU", "\n".join(rows))
+
+    def test_ascii_mode_and_sanitize(self):
+        self.assertEqual(sanitize("\x1b[31mhi\nč", 20, "ascii"), "hic")
+        self.assertEqual(sanitize("abcdefghijkl", 6, "ascii"), "abcde~")
+        self.assertEqual(sanitize("abcdefghijkl", 6, "unicode"), "abcde…")
+        for view in ("node", "cluster"):
+            self.vm["selected_view"] = view
+            self.assertTrue(all(ord(cell[0]) <= 0x7e for row in render(self.vm, 80, 24) for cell in row))
+
+    def test_acs_glyphs_are_logical_tokens_for_curses_layer(self):
+        self.vm["glyphs"] = "acs"
+        grid = render(self.vm, 80, 24)
+        self.assertEqual(grid[1][0][0], "BOX_UL")
+        self.assertEqual(ACS_TOKENS[grid[1][0][0]], "ACS_ULCORNER")
+        self.assertEqual(grid[2][0][0], "BOX_V")
+
+    def test_unicode_glyph_mode_uses_box_and_bar_characters(self):
+        self.vm["glyphs"] = "unicode"
+        grid = render(self.vm, 80, 24)
+        self.assertEqual(grid[1][0][0], "┌")
+        self.assertEqual(grid[2][0][0], "│")
+        self.assertIn(grid[2][2][0], ("█", "░"))
+
+    def test_snapshot_and_nic_names_are_sanitized(self):
+        self.vm["snapshot"]["node"]["name"] = "p3\x1b[31m-01"
+        self.vm["local"]["net"]["nic"] = "eno\n1"
+        rows = lines(render(self.vm, 80, 24))
+        self.assertIn("p3-01", rows[0])
+        self.assertIn("eno1", rows[6])
+        self.assertNotIn("[31m", "\n".join(rows))
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/AppImage/scripts/tests/test_console_snapshot.py b/AppImage/scripts/tests/test_console_snapshot.py
new file mode 100644
index 00000000..0397784e
--- /dev/null
+++ b/AppImage/scripts/tests/test_console_snapshot.py
@@ -0,0 +1,289 @@
+"""Tests for the console snapshot contract and lease-gated publisher."""
+
+import json
+import os
+from pathlib import Path
+import sys
+import tempfile
+import unittest
+from unittest import mock
+
+
+SCRIPTS_DIR = Path(__file__).resolve().parents[1]
+if str(SCRIPTS_DIR) not in sys.path:
+    sys.path.insert(0, str(SCRIPTS_DIR))
+
+import console_snapshot
+
+
+FIXTURES = Path(__file__).parent / "fixtures" / "console"
+
+
+def fixture(name):
+    return json.loads((FIXTURES / name).read_text(encoding="utf-8"))
+
+
+class ConsoleSnapshotTests(unittest.TestCase):
+    def setUp(self):
+        self.temporary = tempfile.TemporaryDirectory()
+        self.addCleanup(self.temporary.cleanup)
+        self.root = Path(self.temporary.name)
+        self.path = self.root / "monitor" / "console.json"
+        self.state_dir = self.root / "console"
+        self.resources = fixture("pvesh_cluster_resources.json")
+        self.status = fixture("pvesh_cluster_status.json")
+        console_snapshot._publisher = None
+        self.addCleanup(setattr, console_snapshot, "_publisher", None)
+
+    def publisher(self, resources=None, status=None, version=None):
+        return console_snapshot.Publisher(
+            get_node_name=lambda: "p3-01",
+            get_resources=lambda: self.resources if resources is None else resources(),
+            run_pvesh=lambda: self.status if status is None else status(),
+            read_language=lambda: "es",
+            path=self.path,
+            state_dir=self.state_dir,
+            clock=lambda: 100.0,
+            version=version,
+        )
+
+    def lease(self, now=100):
+        lease = self.state_dir / "client" / "lease"
+        lease.parent.mkdir(parents=True, exist_ok=True)
+        lease.write_text("", encoding="utf-8")
+        os.utime(lease, (now, now))
+        return lease
+
+    def test_normalizes_complete_resources_and_missing_fields(self):
+        result = console_snapshot.normalize_resources(self.resources)
+        self.assertEqual(set(result), {"nodes", "guests", "storages", "truncated"})
+        self.assertEqual(result["nodes"][0]["name"], "p3-01")
+        self.assertEqual(result["nodes"][0]["cgroup_mode"], 2)
+        self.assertIsNone(result["nodes"][1]["cpu"])
+        self.assertEqual(result["guests"][0]["tags"], ["media", "prod"])
+        self.assertIsNone(result["guests"][0]["diskread"])
+        self.assertEqual(result["guests"][1]["tags"], ["edge", "\x1b[31mred\nline"])
+        self.assertEqual(result["storages"][0]["content"], ["images", "rootdir"])
+
+    def test_cluster_quorum_offline_and_standalone(self):
+        cluster = console_snapshot.normalize_cluster_status(self.status)
+        self.assertEqual((cluster["is_cluster"], cluster["quorate"]), (True, True))
+        self.assertEqual((cluster["nodes_total"], cluster["nodes_online"]), (2, 1))
+        self.assertFalse(cluster["members"][1]["online"])
+        no_quorum = console_snapshot.normalize_cluster_status(
+            fixture("pvesh_no_quorum_status.json"))
+        self.assertEqual((no_quorum["quorate"], no_quorum["nodes_online"]),
+                         (False, 1))
+        standalone = console_snapshot.normalize_cluster_status(fixture("pvesh_standalone_status.json"))
+        self.assertEqual((standalone["is_cluster"], standalone["name"]), (False, None))
+
+    def test_truncates_2500_guests_by_cpu_cores_then_memory(self):
+        guests = [{"type": "qemu", "vmid": i, "cpu": i / 2500, "maxcpu": 2,
+                   "mem": 2500 - i} for i in range(2500)]
+        result = console_snapshot.normalize_resources(guests)
+        self.assertTrue(result["truncated"])
+        self.assertEqual(len(result["guests"]), 2000)
+        self.assertEqual((result["guests"][0]["vmid"], result["guests"][-1]["vmid"]),
+                         (2499, 500))
+
+    def test_published_file_stays_within_one_megabyte(self):
+        large = [{"type": "qemu", "vmid": i, "name": "g" * 1000,
+                  "cpu": i / 2500, "maxcpu": 2} for i in range(2500)]
+        publisher = self.publisher(resources=lambda: large)
+        self.lease()
+        publisher.publish_once(100.0)
+        data = json.loads(self.path.read_text(encoding="utf-8"))
+        self.assertLessEqual(self.path.stat().st_size, 1024 * 1024)
+        self.assertTrue(data["resources"]["truncated"])
+        self.assertEqual(data["resources"]["guests"][0]["vmid"], 2499)
+
+    def test_initial_snapshot_and_idempotent_ensure(self):
+        with mock.patch.object(console_snapshot.threading, "Thread") as thread:
+            first = console_snapshot.ensure_publisher(
+                lambda: "p3-01", lambda: self.resources, lambda: self.status,
+                lambda: "es", path=self.path, state_dir=self.state_dir,
+                clock=lambda: 100.0, version="9.9.9",
+            )
+            second = console_snapshot.ensure_publisher(
+                lambda: "other", lambda: [], lambda: [], lambda: "en",
+                path=self.path, state_dir=self.state_dir,
+            )
+        self.assertIs(first, second)
+        self.assertEqual(thread.call_count, 1)
+        data = json.loads(self.path.read_text(encoding="utf-8"))
+        self.assertEqual((data["cluster"]["status"], data["resources"]["status"]),
+                         ("starting", "starting"))
+        self.assertEqual(data["node"]["name"], "p3-01")
+        self.assertEqual(data["generator"]["version"], "9.9.9")
+
+    def test_unwritable_snapshot_path_does_not_break_ensure_publisher(self):
+        blocked = self.root / "blocked"
+        blocked.write_text("", encoding="utf-8")
+        with mock.patch.object(console_snapshot.threading, "Thread") as thread:
+            publisher = console_snapshot.ensure_publisher(
+                lambda: "p3-01", lambda: self.resources, lambda: self.status,
+                lambda: "es", path=blocked / "console.json", state_dir=self.state_dir,
+                clock=lambda: 100.0,
+            )
+        self.assertIsInstance(publisher, console_snapshot.Publisher)
+        self.assertEqual(thread.call_count, 1)
+        thread.return_value.start.assert_called_once_with()
+
+    def test_generator_version_comes_from_caller(self):
+        self.lease()
+        self.publisher(version="9.9.9").publish_once(100.0)
+        data = json.loads(self.path.read_text(encoding="utf-8"))
+        self.assertEqual(data["generator"]["version"], "9.9.9")
+        self.publisher().publish_once(100.0)
+        data = json.loads(self.path.read_text(encoding="utf-8"))
+        self.assertIsNone(data["generator"]["version"])
+
+    def test_lease_gates_calls_and_publishes_immediately(self):
+        calls = {"resources": 0, "cluster": 0}
+        def resources():
+            calls["resources"] += 1
+            return self.resources
+        def status():
+            calls["cluster"] += 1
+            return self.status
+        publisher = self.publisher(resources, status)
+        self.assertFalse(publisher.publish_once(100.0))
+        self.assertEqual(calls, {"resources": 0, "cluster": 0})
+        self.lease()
+        self.assertTrue(publisher.publish_once(100.0))
+        self.assertEqual(calls, {"resources": 1, "cluster": 1})
+        self.assertFalse(publisher.publish_once(104.0))
+        self.assertTrue(publisher.publish_once(105.0))
+        self.assertEqual(calls, {"resources": 2, "cluster": 1})
+        self.assertTrue(publisher.publish_once(110.0))
+        self.assertEqual(calls, {"resources": 3, "cluster": 2})
+        self.assertFalse(publisher.publish_once(115.0))
+        self.assertEqual(calls, {"resources": 3, "cluster": 2})
+
+    def test_symlink_and_directory_lease_are_ignored(self):
+        publisher = self.publisher()
+        lease = self.state_dir / "client" / "lease"
+        lease.parent.mkdir(parents=True)
+        target = self.root / "target"
+        target.write_text("", encoding="utf-8")
+        lease.symlink_to(target)
+        self.assertFalse(publisher.publish_once(100.0))
+        lease.unlink()
+        lease.mkdir()
+        self.assertFalse(publisher.publish_once(102.0))
+
+    def test_failure_states_preserve_last_success_and_trim_error(self):
+        fail = False
+        def resources():
+            if fail:
+                raise RuntimeError("x" * 300)
+            return self.resources
+        publisher = self.publisher(resources)
+        self.lease(now=100)
+        publisher.publish_once(100.0)
+        fail = True
+        publisher.publish_once(105.0)
+        data = json.loads(self.path.read_text(encoding="utf-8"))
+        self.assertEqual(data["resources"]["status"], "stale")
+        self.assertEqual(data["resources"]["observed_at"], 100.0)
+        self.assertEqual(len(data["resources"]["error"]), 200)
+        publisher.publish_once(110.0)
+        self.lease(now=115)
+        publisher.publish_once(115.0)
+        data = json.loads(self.path.read_text(encoding="utf-8"))
+        self.assertEqual(data["resources"]["status"], "error")
+        self.assertEqual(len(data["resources"]["guests"]), 2)
+
+    def test_unavailable_before_first_success(self):
+        def fail():
+            raise RuntimeError("offline")
+        publisher = self.publisher(fail, fail)
+        self.lease()
+        publisher.publish_once(100.0)
+        data = json.loads(self.path.read_text(encoding="utf-8"))
+        self.assertEqual(data["resources"]["status"], "unavailable")
+        self.assertEqual(data["cluster"]["status"], "unavailable")
+
+    def test_atomic_write_and_permissions_with_and_without_group(self):
+        data = {"schema": 1}
+        with mock.patch.object(console_snapshot.grp, "getgrnam", side_effect=KeyError):
+            console_snapshot.write_snapshot(data, self.path)
+            self.assertEqual(self.path.stat().st_mode & 0o777, 0o600)
+            self.assertEqual(self.path.parent.stat().st_mode & 0o777, 0o700)
+        group = mock.Mock(gr_gid=os.getgid())
+        with mock.patch.object(console_snapshot.grp, "getgrnam", return_value=group), \
+             mock.patch.object(console_snapshot.os, "replace", wraps=os.replace) as replace:
+            console_snapshot.write_snapshot(data, self.path)
+            self.assertEqual(replace.call_count, 1)
+            self.assertEqual(Path(replace.call_args.args[0]).parent, self.path.parent)
+        self.assertEqual(self.path.stat().st_mode & 0o777, 0o640)
+        self.assertEqual(self.path.parent.stat().st_mode & 0o777, 0o750)
+
+    def test_reader_rejects_bad_schema_corruption_and_clock_skew(self):
+        self.assertEqual(console_snapshot.load_snapshot(self.path, 100).resources["status"],
+                         "unavailable")
+        self.path.parent.mkdir()
+        self.path.write_text('{', encoding="utf-8")
+        self.assertEqual(console_snapshot.load_snapshot(self.path, 100).resources["status"],
+                         "unavailable")
+        self.path.write_text('{"schema":2}', encoding="utf-8")
+        self.assertEqual(console_snapshot.load_snapshot(self.path, 100).resources["status"],
+                         "unavailable")
+        publisher = self.publisher()
+        self.lease()
+        publisher.publish_once(100.0)
+        self.assertEqual(console_snapshot.load_snapshot(self.path, 99).age_s, 0)
+
+    def test_reader_sanitizes_names_tags_and_negative_metrics(self):
+        publisher = self.publisher()
+        self.lease()
+        publisher.publish_once(100.0)
+        data = json.loads(self.path.read_text(encoding="utf-8"))
+        guest = data["resources"]["guests"][0]
+        guest["name"] = "\x1b[31m" + "a" * 80
+        guest["cpu"] = -1
+        guest["tags"] = ["\x1b[31m" + "t" * 40] * 20
+        self.path.write_text(json.dumps(data), encoding="utf-8")
+        read = console_snapshot.load_snapshot(self.path, 101)
+        guest = read.resources["guests"][0]
+        self.assertEqual(guest["name"], "a" * 64)
+        self.assertIsNone(guest["cpu"])
+        self.assertEqual(guest["tags"], ["t" * 32] * 16)
+
+    def test_reader_strips_c1_control_characters(self):
+        publisher = self.publisher()
+        self.lease()
+        publisher.publish_once(100.0)
+        data = json.loads(self.path.read_text(encoding="utf-8"))
+        data["resources"]["guests"][0]["name"] = "\u009b31mok\u0080\u009f"
+        self.path.write_text(json.dumps(data), encoding="utf-8")
+        guest = console_snapshot.load_snapshot(self.path, 101).resources["guests"][0]
+        self.assertNotIn("\u009b", guest["name"])
+        self.assertEqual(guest["name"], "31mok")
+
+    def test_touch_lease_creates_regular_file_and_refreshes_timestamp(self):
+        lease = console_snapshot.touch_lease(self.state_dir, now=100)
+        self.assertTrue(lease.is_file())
+        self.assertEqual(lease.stat().st_mtime, 100)
+        console_snapshot.touch_lease(self.state_dir, now=105)
+        self.assertEqual(lease.stat().st_mtime, 105)
+
+    def test_pve_version_callback_is_cached_for_300_seconds(self):
+        calls = []
+        def version():
+            calls.append(1)
+            return "pve-manager/9.0.3"
+        publisher = console_snapshot.Publisher(
+            lambda: "p3-01", lambda: self.resources, lambda: self.status,
+            lambda: "en", path=self.path, state_dir=self.state_dir,
+            get_pve_version=version,
+        )
+        self.assertEqual(publisher._node(100)["pve_version"], "pve-manager/9.0.3")
+        self.assertEqual(publisher._node(399)["pve_version"], "pve-manager/9.0.3")
+        self.assertEqual(len(calls), 1)
+        publisher._node(400)
+        self.assertEqual(len(calls), 2)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/AppImage/scripts/tests/test_console_tui_smoke.py b/AppImage/scripts/tests/test_console_tui_smoke.py
new file mode 100644
index 00000000..f8ad44eb
--- /dev/null
+++ b/AppImage/scripts/tests/test_console_tui_smoke.py
@@ -0,0 +1,232 @@
+"""Terminal smoke tests and pure controller tests for the console TUI."""
+
+import fcntl
+import json
+import os
+from pathlib import Path
+import pty
+import select
+import signal
+import struct
+import sys
+import tempfile
+import termios
+import time
+import unittest
+from unittest import mock
+
+
+SCRIPTS = Path(__file__).resolve().parents[1]
+FIXTURES = Path(__file__).parent / "fixtures" / "console"
+sys.path.insert(0, str(SCRIPTS))
+
+import console_tui  # noqa: E402
+
+
+def guest(vmid=101, node="p3-01", uptime=30, **counters):
+    return {"type": "qemu", "vmid": vmid, "node": node, "uptime": uptime,
+            "netin": counters.get("netin", 100),
+            "netout": counters.get("netout", 200),
+            "diskread": counters.get("diskread", 300),
+            "diskwrite": counters.get("diskwrite", 400)}
+
+
+class GuestRateTests(unittest.TestCase):
+    def test_rates_require_advanced_observation_and_stable_guest(self):
+        rates = console_tui.GuestRates()
+        first = rates.update([guest()], 10)
+        self.assertNotIn("net_rx_bps", first[0])
+        second = rates.update([guest(uptime=35, netin=150, netout=220,
+                                     diskread=340, diskwrite=440)], 15)
+        self.assertEqual({key: second[0][key] for key in
+                          ("net_rx_bps", "net_tx_bps", "disk_read_bps", "disk_write_bps")},
+                         {"net_rx_bps": 10, "net_tx_bps": 4,
+                          "disk_read_bps": 8, "disk_write_bps": 8})
+        self.assertNotIn("net_rx_bps", rates.update([guest(netin=999)], 15)[0])
+        self.assertNotIn("net_rx_bps", rates.update([guest(node="p3-02")], 20)[0])
+        self.assertNotIn("net_rx_bps", rates.update([guest(node="p3-02", uptime=1)], 25)[0])
+        self.assertNotIn("net_rx_bps", rates.update([guest(node="p3-02", uptime=2,
+                                                            netin=1)], 30)[0])
+
+    def test_missing_counter_resets_its_rate(self):
+        rates = console_tui.GuestRates()
+        rates.update([guest(diskread=None)], 10)
+        result = rates.update([guest(uptime=35, netin=120, diskread=400)], 12)[0]
+        self.assertEqual(result["net_rx_bps"], 10)
+        self.assertNotIn("disk_read_bps", result)
+
+    def test_decreased_counter_resets_all_rates(self):
+        rates = console_tui.GuestRates()
+        rates.update([guest()], 10)
+        result = rates.update([guest(uptime=35, netin=120, netout=100)], 12)[0]
+        self.assertNotIn("net_rx_bps", result)
+        next_result = rates.update([guest(uptime=36, netin=140,
+                                          netout=120)], 14)[0]
+        self.assertEqual(next_result["net_rx_bps"], 10)
+
+
+class TranslationTests(unittest.TestCase):
+    def test_language_key_english_and_raw_key_fallback(self):
+        with tempfile.TemporaryDirectory() as temp:
+            appdir = Path(temp)
+            folder = appdir / "usr/share/proxmenux/messages"
+            for language, values in (("en", {"runtime": {"consoleTui": {
+                    "title": "English title", "onlyEnglish": "English fallback"}}}),
+                                     ("es", {"runtime": {"consoleTui": {
+                    "title": "Título"}}})):
+                target = folder / language / "common.json"
+                target.parent.mkdir(parents=True)
+                target.write_text(json.dumps(values), encoding="utf-8")
+            with mock.patch.dict(os.environ, {"APPDIR": str(appdir)}):
+                catalog = console_tui.Translator()
+                catalog.set_language("es")
+                self.assertEqual(catalog("runtime.consoleTui.title"), "Título")
+                self.assertEqual(catalog("runtime.consoleTui.onlyEnglish"), "English fallback")
+                self.assertEqual(catalog("runtime.consoleTui.missing"),
+                                 "runtime.consoleTui.missing")
+                catalog.set_language("fr")
+                self.assertEqual(catalog("runtime.consoleTui.title"), "English title")
+
+    def test_packaged_layout_resolves_catalog_from_appdir(self):
+        with tempfile.TemporaryDirectory() as temp:
+            appdir = Path(temp) / "AppDir"
+            module = appdir / "usr/bin/console_tui.py"
+            module.parent.mkdir(parents=True)
+            module.write_text("", encoding="utf-8")
+            target = appdir / "usr/share/proxmenux/messages/en/common.json"
+            target.parent.mkdir(parents=True)
+            target.write_text(json.dumps({"runtime": {"consoleTui": {"title": "PACKAGED"}}}),
+                              encoding="utf-8")
+            with mock.patch.dict(os.environ):
+                os.environ.pop("APPDIR", None)
+                catalog = console_tui.Translator(module_file=str(module))
+                self.assertEqual(catalog("runtime.consoleTui.title"), "PACKAGED")
+
+
+class TerminalSmokeTests(unittest.TestCase):
+    def setUp(self):
+        self.temp = tempfile.TemporaryDirectory()
+        self.addCleanup(self.temp.cleanup)
+        self.root = Path(self.temp.name)
+        self.proc_root = self.root / "machine"
+        for target, fixture in (("proc/stat", "proc_stat_1"),
+                                ("proc/net/dev", "proc_net_1"),
+                                ("proc/diskstats", "proc_disk_1")):
+            path = self.proc_root / target
+            path.parent.mkdir(parents=True, exist_ok=True)
+            path.write_bytes((FIXTURES / fixture).read_bytes())
+        for target, content in (("proc/meminfo", "MemTotal: 1000000 kB\n"
+                                 "MemAvailable: 500000 kB\n"),
+                                ("proc/loadavg", "0.25 0.50 0.75 1/10 1\n"),
+                                ("proc/uptime", "30.0 10.0\n")):
+            path = self.proc_root / target
+            path.write_text(content, encoding="utf-8")
+        self.snapshot = self.root / "snapshot.json"
+        self.state_dir = self.root / "state"
+        self.other_cwd = self.root / "elsewhere"
+        self.other_cwd.mkdir()
+
+    def write_snapshot(self):
+        document = {"schema": 1, "generator": {"name": "monitor", "version": "1"},
+                    "observed_at": time.time(), "publish_interval_s": 5,
+                    "language": "en", "node": {"status": "ok", "name": "p3-01"},
+                    "cluster": {"status": "ok", "members": [], "is_cluster": False},
+                    "resources": {"status": "ok", "observed_at": time.time(),
+                                  "nodes": [], "guests": [], "storages": []}}
+        self.snapshot.write_text(json.dumps(document), encoding="utf-8")
+
+    def spawn(self, once=False):
+        child, master = pty.fork()
+        if child == 0:
+            fcntl.ioctl(1, termios.TIOCSWINSZ, struct.pack("HHHH", 24, 80, 0, 0))
+            os.chdir(self.other_cwd)
+            os.environ["TERM"] = "linux"
+            os.environ.pop("APPDIR", None)
+            args = [sys.executable, "-I", str(SCRIPTS / "console_tui.py")]
+            if once:
+                args.append("--once")
+            args.extend(("--snapshot", str(self.snapshot), "--proc-root",
+                         str(self.proc_root), "--state-dir", str(self.state_dir)))
+            os.execv(sys.executable, args)
+        self.addCleanup(self._stop, child, master)
+        return child, master
+
+    @staticmethod
+    def _stop(child, master):
+        try:
+            os.kill(child, signal.SIGKILL)
+        except ProcessLookupError:
+            pass
+        try:
+            os.waitpid(child, os.WNOHANG)
+        except ChildProcessError:
+            pass
+        os.close(master)
+
+    def collect(self, child, master, timeout=2, wanted=None):
+        output = bytearray()
+        deadline = time.monotonic() + timeout
+        while time.monotonic() < deadline:
+            readable, _, _ = select.select([master], [], [], 0.05)
+            if readable:
+                try:
+                    output.extend(os.read(master, 65536))
+                except OSError:
+                    pass
+            if wanted is not None and wanted in output:
+                return bytes(output), None
+            ended, status = os.waitpid(child, os.WNOHANG)
+            if ended:
+                while select.select([master], [], [], 0)[0]:
+                    try:
+                        chunk = os.read(master, 65536)
+                    except OSError:
+                        break
+                    if not chunk:
+                        break
+                    output.extend(chunk)
+                return bytes(output), os.waitstatus_to_exitcode(status)
+        self.fail("TUI did not respond before deadline")
+
+    def test_once_with_snapshot_from_different_cwd_under_isolated_python(self):
+        self.write_snapshot()
+        child, master = self.spawn(once=True)
+        output, code = self.collect(child, master)
+        self.assertEqual(code, 0, output.decode("ascii", "replace"))
+        self.assertIn(b"PROXMENUX MONITOR", output)
+
+    def test_q_exits_interactive_loop_within_two_seconds(self):
+        self.write_snapshot()
+        child, master = self.spawn()
+        output, _ = self.collect(child, master, wanted=b"PROXMENUX MONITOR")
+        self.assertIn(b"PROXMENUX MONITOR", output)
+        os.write(master, b"q")
+        _, code = self.collect(child, master)
+        self.assertEqual(code, 0)
+
+    def test_once_without_snapshot_shows_local_panels_and_starting(self):
+        child, master = self.spawn(once=True)
+        output, code = self.collect(child, master)
+        self.assertEqual(code, 0, output.decode("ascii", "replace"))
+        self.assertIn(b"CPU", output)
+        self.assertIn(b"MEMORY", output)
+        self.assertIn(b"Monitor starting", output)
+
+    def test_once_without_snapshot_after_boot_shows_unavailable(self):
+        (self.proc_root / "proc/uptime").write_text("180.0 10.0\n", encoding="utf-8")
+        child, master = self.spawn(once=True)
+        output, code = self.collect(child, master)
+        self.assertEqual(code, 0, output.decode("ascii", "replace"))
+        self.assertIn(b"MONITOR UNAVAILABLE", output)
+
+    def test_sigterm_restores_terminal_and_exits_zero(self):
+        child, master = self.spawn()
+        self.collect(child, master, wanted=b"PROXMENUX MONITOR")
+        os.kill(child, signal.SIGTERM)
+        output, code = self.collect(child, master)
+        self.assertEqual(code, 0)
+        self.assertTrue(b"\x1b[?25h" in output or b"\x0f" in output)
+
+
+if __name__ == "__main__":
+    unittest.main()
diff --git a/install_proxmenux.sh b/install_proxmenux.sh
index e2aee65a..5d6f655a 100755
--- a/install_proxmenux.sh
+++ b/install_proxmenux.sh
@@ -573,6 +573,10 @@ install_proxmenux_monitor() {
         return 1
     fi
 
+    if [ -f "$TEMP_DIR/scripts/console/proxmenux-console" ]; then
+        install -m 0755 "$TEMP_DIR/scripts/console/proxmenux-console" /usr/local/bin/proxmenux-console
+    fi
+
     msg_ok "ProxMenux Monitor v$appimage_version installed."
 
     if [ "$service_exists" = false ]; then
diff --git a/install_proxmenux_beta.sh b/install_proxmenux_beta.sh
index 90022cd2..1eefefaf 100644
--- a/install_proxmenux_beta.sh
+++ b/install_proxmenux_beta.sh
@@ -458,6 +458,10 @@ install_proxmenux_monitor() {
         return 1
     fi
 
+    if [ -f "$TEMP_DIR/scripts/console/proxmenux-console" ]; then
+        install -m 0755 "$TEMP_DIR/scripts/console/proxmenux-console" /usr/local/bin/proxmenux-console
+    fi
+
     # Copy shutdown-notify.sh script for systemd ExecStop
     local shutdown_script_src="$TEMP_DIR/scripts/shutdown-notify.sh"
     local shutdown_script_dst="$MONITOR_INSTALL_DIR/scripts/shutdown-notify.sh"
diff --git a/scripts/console/proxmenux-console b/scripts/console/proxmenux-console
new file mode 100755
index 00000000..a639ce26
--- /dev/null
+++ b/scripts/console/proxmenux-console
@@ -0,0 +1,18 @@
+#!/bin/bash
+# Launch the ProxMenux Monitor console TUI in the current terminal.
+# Root only: the TUI is read-only and runs as the root that invoked it.
+# PMX_CONSOLE_LAUNCHER overrides the AppRun path (used by tests).
+
+LAUNCHER="${PMX_CONSOLE_LAUNCHER:-/usr/local/share/proxmenux/monitor-app/AppRun}"
+
+if [ "$(id -u)" -ne 0 ]; then
+    echo "proxmenux-console must be run as root." >&2
+    exit 1
+fi
+
+if [ ! -x "$LAUNCHER" ]; then
+    echo "ProxMenux Monitor is not installed: $LAUNCHER is missing or not executable." >&2
+    exit 1
+fi
+
+exec "$LAUNCHER" --tui "$@"
diff --git a/scripts/menus/config_menu.sh b/scripts/menus/config_menu.sh
index 547b2244..87e98cbb 100644
--- a/scripts/menus/config_menu.sh
+++ b/scripts/menus/config_menu.sh
@@ -128,6 +128,9 @@ uninstall_proxmenux_monitor() {
     echo " - Config dir ${MONITOR_CONFIG_DIR} does not exist (ok)"
     fi
 
+    # 4b. Remove the manual console launcher installed with the Monitor
+    pmx_remove_file "/usr/local/bin/proxmenux-console"
+
     # 5. Reload systemd
     echo " - Recargando systemd..."
     systemctl daemon-reload > /dev/null 2>&1
@@ -817,6 +820,7 @@ uninstall_proxmenu() {
 
         echo "50" ; echo "Removing ProxMenu files..."
         pmx_remove_file "$INSTALL_DIR/$MENU_SCRIPT"
+        pmx_remove_file "/usr/local/bin/proxmenux-console"
         pmx_record_execution "Remove ProxMenux application directory" "rm -rf $BASE_DIR"
         rm -rf "$BASE_DIR"
         remove_registry_logins
diff --git a/tests/test_console_installer.py b/tests/test_console_installer.py
new file mode 100644
index 00000000..5435abd4
--- /dev/null
+++ b/tests/test_console_installer.py
@@ -0,0 +1,101 @@
+"""Manual console launcher and its install/uninstall wiring.
+
+Run: python3 -m unittest discover -s tests -p test_console_installer.py -v
+Requires Bash. The launcher runs with a restricted PATH that holds only a fake
+`id`; AppRun is a fake that records its argv (NUL-separated, exact bytes).
+"""
+from pathlib import Path
+import re
+import subprocess
+import tempfile
+import unittest
+
+ROOT = Path(__file__).resolve().parents[1]
+LAUNCHER = ROOT / "scripts/console/proxmenux-console"
+INSTALLERS = ("install_proxmenux.sh", "install_proxmenux_beta.sh")
+CONFIG_MENU = "scripts/menus/config_menu.sh"
+INSTALL_LINE = re.compile(
+    r'^\s*install -m 0755 "\$TEMP_DIR/scripts/console/proxmenux-console" /usr/local/bin/proxmenux-console$',
+    re.M)
+REMOVE_LINE = re.compile(r'^\s*pmx_remove_file "/usr/local/bin/proxmenux-console"$', re.M)
+
+
+def function(path, name):
+    text = (ROOT / path).read_text()
+    match = re.search(r"^" + re.escape(name) + r"\(\) \{\n.*?^\}", text, re.M | re.S)
+    assert match, (path, name)
+    return match[0]
+
+
+class Launcher(unittest.TestCase):
+    def run_launcher(self, uid, args=(), apprun_exists=True, apprun_executable=True):
+        with tempfile.TemporaryDirectory(prefix="pmx-console-") as tmp:
+            tmp = Path(tmp)
+            bin_dir = tmp / "bin"
+            bin_dir.mkdir()
+            fake_id = bin_dir / "id"
+            fake_id.write_text(f"#!/bin/sh\necho {uid}\n")
+            fake_id.chmod(0o755)
+            apprun = tmp / "AppRun"
+            if apprun_exists:
+                # ${0%/*} instead of dirname: PATH holds only the fake id.
+                apprun.write_text(
+                    r"""#!/bin/sh
+printf '%s\0' "$@" > "${0%/*}/argv"
+""")
+                apprun.chmod(0o755 if apprun_executable else 0o644)
+            env = {"PATH": str(bin_dir), "PMX_CONSOLE_LAUNCHER": str(apprun)}
+            result = subprocess.run(["/bin/bash", "--noprofile", "--norc", str(LAUNCHER), *args],
+                                    env=env, text=True, capture_output=True, timeout=10)
+            argv_file = tmp / "argv"
+            argv = argv_file.read_bytes().split(b"\0")[:-1] if argv_file.exists() else None
+            return result, argv
+
+    def test_non_root_is_refused_before_apprun(self):
+        result, argv = self.run_launcher(uid=1000, args=("x",))
+        self.assertEqual(result.returncode, 1)
+        self.assertIn("must be run as root", result.stderr)
+        self.assertIsNone(argv)
+
+    def test_root_execs_apprun_with_tui_then_exact_args(self):
+        args = ("--foo", "two words", "")
+        result, argv = self.run_launcher(uid=0, args=args)
+        self.assertEqual(result.returncode, 0, result.stderr)
+        self.assertEqual(argv, [b"--tui", b"--foo", b"two words", b""])
+
+    def test_missing_apprun_reports_monitor_not_installed(self):
+        result, argv = self.run_launcher(uid=0, apprun_exists=False)
+        self.assertEqual(result.returncode, 1)
+        self.assertIn("ProxMenux Monitor is not installed", result.stderr)
+        self.assertIsNone(argv)
+
+    def test_non_executable_apprun_is_refused(self):
+        result, argv = self.run_launcher(uid=0, apprun_executable=False)
+        self.assertEqual(result.returncode, 1)
+        self.assertIn("ProxMenux Monitor is not installed", result.stderr)
+        self.assertIsNone(argv)
+
+
+class InstallerWiring(unittest.TestCase):
+    def test_installers_copy_launcher_in_monitor_install(self):
+        for name in INSTALLERS:
+            with self.subTest(installer=name):
+                body = function(name, "install_proxmenux_monitor")
+                matches = list(INSTALL_LINE.finditer(body))
+                self.assertEqual(len(matches), 1)
+                line = matches[0].start()
+                extract = body.index('extract_appimage_to_runtime_dir "$target_path"')
+                service = body.index('if [ "$service_exists" = false ]')
+                self.assertLess(extract, line)
+                self.assertLess(line, service)
+
+    def test_uninstall_paths_remove_launcher(self):
+        monitor = function(CONFIG_MENU, "uninstall_proxmenux_monitor")
+        self.assertRegex(monitor, REMOVE_LINE)
+        full = function(CONFIG_MENU, "uninstall_proxmenu")
+        self.assertRegex(full, REMOVE_LINE)
+        self.assertLess(REMOVE_LINE.search(full).start(), full.index('rm -rf "$BASE_DIR"'))
+
+
+if __name__ == "__main__":
+    unittest.main()
```
