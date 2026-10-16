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
