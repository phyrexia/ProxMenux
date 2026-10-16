# ADR 001 — Consola TUI de ProxMenux Monitor en tty1 (HDMI/VGA)

- Estado: **proposed**
- Fecha: 2026-10-17
- Autor del diseño: Designer (Fable). Implementación autorizada por el Owner el 2026-10-16 17:48.
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

Una TUI propia en Python `curses` (stdlib del host, sin dependencias nuevas) empaquetada en el AppImage del Monitor y lanzada con `AppRun --tui`. Lee **métricas locales directamente de `/proc` y `/sys`** cada 1 s, y lee de un **snapshot JSON** (`/run/proxmenux-monitor/console.json`) publicado por el servicio Monitor lo que exige `pvesh` (clúster, guests, almacenamiento PVE, versión). En `tty1` la lanza **una vez por arranque** un wrapper root del `getty@tty1` (drop-in de dos líneas), como **usuario de sistema sin privilegios**; al salir, el wrapper ejecuta el `agetty` del fabricante. Relanzamiento manual con `proxmenux-console`. Todo es sólo lectura y toda la configuración de arranque es root-owned.

### 2.2 Modelo de procesos y privilegios

| Pieza | Corre como | Qué hace | Qué no hace |
|---|---|---|---|
| **Publicador** (`console_snapshot.py`, hilo daemon dentro de `flask_server.py`) | root (proceso del Monitor) | Cada 5 s, sólo si existe un *lease* reciente, llama `pvesh get /cluster/resources --output-format json` (todos los tipos) y cada 10 s `pvesh get /cluster/status --output-format json`; normaliza y escribe `console.json` de forma atómica. | No expone endpoints HTTP, no escribe credenciales, no toca `/etc/pve`. |
| **TUI** (`console_tui.py` + `console_metrics.py` + `console_render.py`) | `proxmenux-console` (system user, `nologin`, sin home, sin sudo, no en `www-data`) | Lee `/proc`, `/sys` y `console.json`; renueva el lease; pinta con curses; responde al teclado. | No ejecuta comandos externos (ni `pvesh`, ni `sensors`), no escribe fuera de `/run/proxmenux-console/client/`, no importa `flask_server.py`. |
| **Wrapper getty** (`/usr/local/sbin/proxmenux-console-getty`) | root (contexto de `getty@tty1`) | Decide por boot si lanza la TUI, la lanza con `setpriv` y después hace `exec` de `agetty`. | No hace `chvt`, no cambia permisos del tty, no reinicia nada. |
| **Lanzador manual** (`/usr/local/bin/proxmenux-console`) | quien lo invoque | Si es root, baja a `proxmenux-console` con `setpriv`; si es miembro del grupo, ejecuta directo; si no, explica. Vuelve al shell al salir. | No toca el marcador por boot. |

**Local desde `/proc` y no desde el snapshot — justificación.** (a) El Owner mira la consola justo después de un reinicio, cuando el Monitor arranca, se actualiza o está en gracia (`startup_grace.py`: 180–300 s); la consola debe mostrar el nodo aunque el Monitor esté parado o desactivado desde `config_menu.sh:403-440`. (b) Cadencia de 1 s sin IPC ni carga extra en el proceso gevent del Monitor. (c) Todo lo necesario es legible sin privilegios: `/proc/stat`, `/proc/meminfo`, `/proc/loadavg`, `/proc/uptime`, `/proc/net/dev`, `/proc/diskstats`, `/sys/class/net/*/{operstate,speed,device}`, `/sys/class/hwmon/*/{name,temp*_input,temp*_label}`, `/sys/class/thermal/thermal_zone*/temp`, `/proc/spl/kstat/zfs/arcstats`, `/sys/block/zram*/{disksize,mm_stat,comp_algorithm}` y `/proc/swaps` (se reutiliza `zram_metrics.read_zram_metrics`). (d) Es testeable con fixtures por inyección de rutas, como ya hace `test_zram_metrics.py`.

**Comportamiento ante fallos.**

| Situación | Local (A, columna izquierda y strip) | Clúster / guests / storage PVE |
|---|---|---|
| Monitor parado, deshabilitado o arrancando | En vivo | `console.json` ausente o con `observed_at` > 60 s → cabecera `MONITOR UNAVAILABLE` en rojo; los últimos valores quedan atenuados con su edad (`data 3m old`); si nunca hubo archivo, paneles con `no data` |
| `pvesh` falla o tarda > 10 s | En vivo | `resources.status = "stale"` con `error`; tras 3 fallos seguidos `"error"`; valores previos atenuados con edad |
| Sin quorum | En vivo | `cluster.quorate = false` → `NO QUORUM` rojo; nodos `online=false` en rojo; guests de nodos offline aparecen con `status` tal cual lo da PVE (normalmente `unknown`) y sin métricas; **no se calcula total de clúster**, se muestra `coverage 1/2 nodes` |
| Nodo remoto inalcanzable con quorum | En vivo | Fila del nodo `offline`; agregados sólo de los online y marcados con cobertura |
| Standalone (sin `corosync.conf`) | En vivo | `cluster.is_cluster = false` → cabecera `STANDALONE`; la pestaña **Cluster** muestra la vista B con un solo nodo y todos los guests (sigue siendo útil como tabla) |
| Terminal < 80×24 | — | Sólo el mensaje `Terminal too small (80x24)`; `q` sigue funcionando |

### 2.3 Contrato de `console.json`

- Ruta: `/run/proxmenux-monitor/console.json`. Directorio `/run/proxmenux-monitor` creado por el publicador: `root:proxmenux-console 0750` (si el grupo no existe, `0700`). Archivo `0640 root:proxmenux-console`.
- Escritura atómica: `tempfile.NamedTemporaryFile(dir=same_dir, delete=False)` → `json.dump` → `os.fchmod(0o640)` → `os.fchown` al grupo → `os.replace`. `/run` es tmpfs: sin `fsync`.
- Intervalo: 5 s mientras `/run/proxmenux-console/client/lease` tenga `mtime` < 15 s; en reposo, un `os.stat()` cada 2 s y nada más. Primer snapshot ≤ 2 s tras aparecer el lease.
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

Semántica de `status` por fuente: `ok` = última llamada exitosa dentro de 2× su intervalo; `stale` = hubo éxito antes pero la última falló (se conservan datos); `unavailable` = nunca hubo éxito desde que arrancó el publicador (listas vacías); `error` = 3 fallos consecutivos (listas conservadas, `error` con el mensaje recortado a 200 caracteres). `observed_at` del documento es el instante de escritura; los `observed_at` por fuente son los de su último éxito, de modo que la TUI muestre la edad real del dato remoto y no la del archivo.

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

Semántica de las listas TOP: sólo guests `running` con `cpu`/`mem` numéricos; **TOP CPU** ordena por `cpu × maxcpu` (núcleos de host consumidos) y muestra `1.2c` y `60 %` (de lo asignado); **TOP MEMORY** ordena por `mem` y muestra GiB con una decimal (`MiB` si < 1 GiB). En clúster se listan guests de todos los nodos con la columna `node`; en standalone la columna `node` se omite y el espacio va al nombre. Si el snapshot está stale, las listas se pintan en `dim` y el título lleva `(2m old)`.

Cabecera: título, pestañas (`[Node]` en `selected`), nombre del nodo, `PVE x.y` (del snapshot, `n/d` si no hay), uptime local, reloj local a la derecha. Si el snapshot está en estado crítico, la mitad derecha se sustituye por `MONITOR UNAVAILABLE (3m)` en `crit`.

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
| `q`, `Q`, `Ctrl+C` (SIGINT), SIGTERM, SIGHUP | salir limpiamente (`curses.endwin`), código 0 |
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

| Ruta instalada | Origen en el repo | Rol |
|---|---|---|
| `/etc/systemd/system/getty@tty1.service.d/proxmenux-console.conf` | generado por el instalador | drop-in |
| `/usr/local/sbin/proxmenux-console-getty` | `scripts/console/proxmenux-console-getty` | wrapper del getty (fuera de `$BASE_DIR` a propósito) |
| `/usr/local/bin/proxmenux-console` | `scripts/console/proxmenux-console` | lanzador manual |
| `/etc/proxmenux/console.conf` | generado si no existe | `AUTOSTART=yes|no`, `GLYPHS=auto|unicode|ascii`, `PRIVACY=names|ids`, `CONSOLE_USER=proxmenux-console` |
| `/etc/tmpfiles.d/proxmenux-console.conf` | generado | `d /run/proxmenux-console 0755 root root -` y `d /run/proxmenux-console/client 0755 proxmenux-console proxmenux-console -` |
| usuario/grupo `proxmenux-console` | `adduser --system --group --no-create-home --shell /usr/sbin/nologin` | identidad de la TUI |

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
9. Tras cualquier salida de la TUI: `stty sane <&0 2>/dev/null || true; printf '\033[H\033[2J'`; `run_login`.

El wrapper no cambia dueño ni permisos del tty: la TUI hereda los fd 0/1/2 ya abiertos por systemd (`StandardInput=tty`), y curses trabaja sobre ellos; no necesita abrir `/dev/tty1`.

#### Qué hace cada salida

| Evento | Resultado |
|---|---|
| `q` / `Ctrl+C` | TUI termina 0 → pantalla limpia → `agetty` → `login:` en < 2 s |
| Traceback de la TUI | stderr en `tui.log`, mismo camino → `login:` |
| Logout de una sesión en tty1 | `Restart=always` relanza el wrapper → marcador presente → `agetty` directo (sin TUI) |
| `systemctl restart getty@tty1` | ídem (marcador persiste hasta el reinicio) |
| Reinicio del nodo | `/run` se vacía → TUI de nuevo una vez |
| `AUTOSTART=no` | `agetty` directo en todos los casos |
| `/dev/tty0` ausente (sin VT) | `getty@tty1` no arranca por su propia condición; nada que hacer |

#### Relanzamiento manual

`proxmenux-console` desde cualquier shell (SSH, otra TTY, la propia tty1 ya logueada): ejecuta `AppRun --tui` como `proxmenux-console` (vía `setpriv` si es root) y al salir devuelve el shell; no toca el marcador. Para volver a verla en tty1 sin reiniciar: `rm /run/proxmenux-console/autostart.tty1 && systemctl restart getty@tty1` (documentado; sólo si nadie está logueado en tty1).

#### Menú de ProxMenux (`scripts/menus/config_menu.sh`)

Junto a las entradas del Monitor (`:586-602`), cuando `console.conf` existe: `Console monitor (HDMI/VGA): launch now`, `Console monitor: enable/disable autostart on tty1` (edita `AUTOSTART` en `console.conf`) y `Console monitor: status` (muestra `AUTOSTART`, si el marcador existe, última línea de `tui.log`). Todo con `translate "..."`.

#### Instalación / actualización / desinstalación

- **Instalar** (`install_proxmenux.sh` y `install_proxmenux_beta.sh`, tras `create_monitor_service` / la actualización): función `install_console_tui`: crear usuario, copiar wrapper y lanzador a `/usr/local/{sbin,bin}`, escribir tmpfiles y `systemd-tmpfiles --create` del archivo, escribir drop-in, `daemon-reload`; escribir `console.conf` **sólo si no existe** (pregunta Q1 del grill con default Sí; `AUTOSTART=no` si el usuario rehúsa). Si `AUTOSTART=yes` y tty1 está ocioso (`loginctl list-sessions --no-legend` sin sesión en `tty1` **y** `ps -o comm= -t tty1` es exactamente `agetty`) → `systemctl restart getty@tty1.service` para mostrarla ya; si no, se mostrará en el próximo arranque y se informa. Nunca `chvt`.
- **Actualizar**: misma función (idempotente): rescribe wrapper, lanzador, tmpfiles y drop-in; **no toca `console.conf`** (respeta un `AUTOSTART=no` explícito); no reinicia getty si ya hay marcador o sesión.
- **Desinstalar Monitor** (`uninstall_proxmenux_monitor`, `config_menu.sh:95-137`) y **desinstalar ProxMenux** (`uninstall_proxmenu`, antes del `rm -rf "$BASE_DIR"` de `:821`): función `remove_console_tui`: en este orden, borrar drop-in → `daemon-reload` → borrar wrapper, lanzador, tmpfiles, `console.conf`, `/run/proxmenux-console` → `deluser --system proxmenux-console` (si existe y no tiene procesos; si los tiene, se deja y se informa). Si tty1 ejecuta en este momento sólo la TUI (sin sesión), `systemctl restart getty@tty1` para devolver el getty del fabricante; si hay sesión, se deja para el próximo respawn. Sólo se borran rutas propias enumeradas; nunca `getty@.service` ni otros drop-ins.
- **Interacción con otras consolas**: `serial-getty@*`, `tty2–6`, `autovt@`, SSH y `rescue.target` no se tocan. El drop-in es por instancia (`getty@tty1.service.d`), así que `getty@tty2` etc. siguen con el fabricante.

### 2.6 i18n

- **TUI (Python)**: claves bajo `runtime.consoleTui.*` en `AppImage/messages/en/common.json`; el bot `build-i18n-messages.yml` rellena `de es fr it pt sk sv`. Los catálogos ya se empaquetan en `usr/share/proxmenux/messages/<locale>/common.json` (`build_appimage.sh:178-181`). La TUI resuelve `APPDIR/usr/share/proxmenux/messages/<lang>/common.json` → `en` → clave, con la misma lógica de `notification_templates._catalog_value` (`:52-73`) reimplementada en 20 líneas dentro de `console_tui.py` (no se importa `notification_templates` para no arrastrar `urllib` ni la caché de notificaciones). Idioma: `snapshot.language` si hay snapshot; si no, `.language` de `/usr/local/share/proxmenux/config.json` (misma fuente que `load_language`, `scripts/utils.sh:228-236`); si no, `en`. Textos pasan por `sanitize` cuando los glifos no son `unicode`.
- Claves iniciales (T6 puede avanzar en paralelo): `title`, `tabs.node`, `tabs.cluster`, `panels.cpu`, `panels.memory`, `panels.network`, `panels.disk`, `panels.cluster`, `panels.topCpu`, `panels.topMemory`, `panels.nodes`, `panels.guests`, `labels.load`, `labels.threads`, `labels.temp`, `labels.arc`, `labels.available`, `labels.zram`, `labels.swap`, `labels.rx`, `labels.tx`, `labels.read`, `labels.write`, `labels.used`, `labels.uptime`, `labels.hostCores`, `labels.quorumOk`, `labels.noQuorum`, `labels.standalone`, `labels.coverage`, `labels.online`, `labels.offline`, `labels.unknown`, `labels.running`, `labels.stopped`, `labels.sort`, `labels.moreNodes`, `labels.moreGuests`, `status.data`, `status.noData`, `status.stale`, `status.monitorUnavailable`, `status.terminalTooSmall`, `columns.type`, `columns.id`, `columns.name`, `columns.node`, `columns.status`, `columns.cpu`, `columns.mem`, `columns.root`, `columns.guests`, `columns.up`, `footer.quit`, `footer.switch`, `footer.refresh`, `footer.glyphs`, `footer.otherTty`, `footer.sort`.
- **Bash** (instaladores, wrapper, menú): `translate "..."` como el resto del repo; el bot de `lang/*.json` las recoge de `scripts/**/*.sh` y de los instaladores. El wrapper imprime lo mínimo (nada en el camino feliz).
- **Docs web**: `web/messages/en/docs/settings/console-monitor.json` + `web/app/[locale]/docs/settings/console-monitor/page.tsx`, siguiendo `docs/settings/proxmenux-monitor`; traducción por `build-web-docs-i18n.yml`.

### 2.7 Empaquetado

- `AppImage/scripts/AppRun`: nueva rama antes del arranque de Flask: `if [[ "$1" == "--tui" ]]; then exec python3 -I "${APPDIR}/usr/bin/console_tui.py" "${@:2}"; fi` — sin los `echo` de banner (ensuciarían la consola) y con `-I` (modo aislado: ignora `PYTHONPATH` del entorno y el cwd; la TUI sólo necesita stdlib y los módulos hermanos en `usr/bin`, que importa por ruta del propio archivo). Las ramas `--translate` y por defecto no cambian.
- `AppImage/scripts/build_appimage.sh:128-148`: añadir `cp` de `console_tui.py`, `console_render.py`, `console_metrics.py`, `console_snapshot.py` a `$APP_DIR/usr/bin/` (sin `2>/dev/null ||` : son obligatorios, como `zram_metrics.py` en `:129`).
- Python/curses: nada que empaquetar; el host aporta `_curses` (verificado en `libpython3.11-stdlib` y `libpython3.13-stdlib`) y `ncurses-base` el terminfo `linux`. El wrapper comprueba `import curses` antes de lanzar (paso 6).
- `flask_server.py`: `import console_snapshot` y `console_snapshot.ensure_publisher(get_node_name=get_proxmox_node_name, run_pvesh=<callable>, read_language=<callable>)` al cargar el módulo, junto a `_ensure_net_rate_sampler()` (`:5900` aprox.). El publicador usa su propio `subprocess.run(['pvesh', 'get', '/cluster/resources', '--output-format', 'json'], timeout=10)` (todos los tipos: nodos, guests y storages en una sola llamada) y no comparte la caché `--type vm` existente para no cambiar su semántica.

### 2.8 Estrategia de pruebas (sólo en workers QA remotos)

Las pruebas nunca se ejecutan en el Mac: corren en los P3 habilitados o en `qf-infra` mediante `qf-test up --generic --auto --allow-qf-infra` → `exec` → `artifacts` → `down`, en un contenedor Debian 13 con `python3`, `bash`, `util-linux`, `jq`. CI upstream: nuevo `.github/workflows/test-console-tui.yml` con el mismo patrón que `test-lvm-disk-policy.yml` (`python -I -B -m unittest discover -s <dir> -p <file> -v`, Python 3.11 y 3.13 en matriz).

| Capa | Archivo de prueba | Qué prueba (fixtures en `AppImage/scripts/tests/fixtures/console/`) |
|---|---|---|
| Métricas locales | `AppImage/scripts/tests/test_console_metrics.py` | Dos muestras de `/proc/stat`, `/proc/net/dev`, `/proc/diskstats` con `dt=1.0` → porcentajes y tasas exactas; filtro de NIC físicas (sólo con `device/`, excluye `lo`, `vmbr*`, `fwbr*`, `fwpr*`, `fwln*`, `tap*`, `veth*`, bonds); `meminfo` con y sin ARC (no doble conteo); `hwmon` `k10temp`/`coretemp`/sin sensor → `None`; zram vía `zram_metrics` con rutas inyectadas; `statvfs` simulado. |
| Snapshot | `AppImage/scripts/tests/test_console_snapshot.py` | Normalización desde JSON de `pvesh` grabados: clúster 2 nodos, standalone, sin quorum, nodo offline, campos ausentes (`diskread` null), `tags` con `;` y caracteres de control, 2500 guests → `truncated`; estados `ok/stale/unavailable/error` tras 0/1/3 fallos; escritura atómica (archivo temporal + `replace`, permisos 0640) en `TMPDIR`; lease: sin lease no se invoca `run_pvesh` (callable contado); lector rechaza `schema != 1` y JSON corrupto sin excepción; `age_s` con reloj atrasado → 0. |
| Render | `AppImage/scripts/tests/test_console_render.py` | `render(view_model, 80, 24, glyphs)` y `(128, 48)`, `(240, 67)`: dimensiones exactas, ninguna fila supera `width`, celdas clave (cabecera, títulos, filas TOP ordenadas por `cpu×maxcpu` y `mem`, `+N nodes`), colapso a 80×24 (historial 1 fila, sin per-core), modo `ascii` sin bytes > 0x7E, `sanitize` elimina `\x1b[31m`, `\n`, `č → c`, trunca a celdas; vista B: offline primero, totales sólo con quorum y cobertura completa; `Terminal too small` a 79×24 y 80×23. |
| Wrapper getty | `tests/test_console_getty_wrapper.py` | Bash con `PATH` restringido (patrón `tests/test_tui_count_messages.py:51-54`), `PMX_CONSOLE_*` apuntando a un fake root; `agetty`, `setpriv`, `python3`, `stty`, `id`, `install` falsos que registran argv: sin marcador → lanza TUI y luego `agetty` con argv exacto `-o '-- \u' --noreset --noclear - linux`; marcador presente → sólo `agetty`; `AUTOSTART=no` → sólo `agetty`; conf escribible por otros → ignorada; lanzador que sale 1 → `agetty` igual; marcador creado antes de invocar el lanzador (orden en el log); `TERM` ausente → `linux`. |
| Instalador | `tests/test_console_installer.py` | Extracción acotada de `install_console_tui`/`remove_console_tui` (helper `function()` de `test_tui_count_messages.py`), `systemctl`, `loginctl`, `ps`, `adduser`, `deluser`, `systemd-tmpfiles` falsos y prefijo de rutas por variable: drop-in con contenido exacto; `console.conf` no se sobrescribe si existe; reinicio de `getty@tty1` sólo con tty1 ocioso; desinstalación borra sólo las rutas propias y en el orden drop-in → reload → resto; idempotencia (dos ejecuciones = mismo árbol). |
| Smoke TUI | `AppImage/scripts/tests/test_console_tui_smoke.py` | `pty.fork()` con `TERM=linux`, `console_tui.py --once --snapshot <fixture> --proc-root <fixture>` pinta un frame y sale 0; `q` enviado por el pty termina el proceso < 2 s; sin snapshot → sale 0 con `no data`. Se ejecuta sólo en el worker QA (necesita pty y curses reales). |

**Aceptación en hardware (Owner, P3 con HDMI)**, tras instalar el AppImage de la rama en un nodo de prueba:

1. Reiniciar: la TUI aparece en el monitor tras el getty; los glifos de caja y barras se ven correctos (si no, `a` y anotar la fuente activa: `cat /etc/default/console-setup`).
2. `q` → `login:` en < 2 s; iniciar sesión, `exit` → vuelve `login:`, no la TUI.
3. `proxmenux-console` desde SSH y desde tty1 logueada → se abre y al salir vuelve el shell.
4. `Ctrl+Alt+F2` → getty normal en tty2; `Ctrl+Alt+F1` → la TUI sigue (o el login si ya se salió).
5. Pestaña Cluster muestra ambos P3 online y los guests; `c`/`m` cambian el orden.
6. `systemctl stop proxmenux-monitor` → en ≤ 60 s la cabecera marca `MONITOR UNAVAILABLE`, CPU/RAM/red locales siguen en vivo; `start` → vuelve a `data Ns`.
7. Desconectar la red del nodo → `NO QUORUM` en rojo con cobertura `1/2`; reconectar → `QUORUM OK 2/2`.
8. Poner `AUTOSTART=no`, reinstalar/actualizar → sigue en `no`; reiniciar → sólo `login:`.
9. Desinstalar Monitor desde el menú → reiniciar → `login:` del fabricante en tty1; `systemctl cat getty@tty1` sin drop-in de ProxMenux.
10. `top` en el nodo durante 5 min con la TUI abierta: `python3` de la TUI ≤ 2 % CPU, ≤ 40 MiB RSS; `journalctl -u proxmenux-monitor` sin errores del publicador.

### 2.9 Descomposición de la implementación (DAG)

Un commit por tarea; propiedad de archivos sin solapes; **nivel**: *principal* (lógica compleja, contratos) o *light*. Ninguna tarea edita archivos de otra; lo que cruce fronteras se resuelve en este ADR, no en el código.

| # | Tarea | Archivos propios | Depende de | Pruebas que la cierran | Nivel |
|---|---|---|---|---|---|
| T1 | Lectores locales `/proc`/`/sys` con rutas inyectables y cálculo de tasas (`LocalSampler.sample(now) -> dict`, historial de 60) | `AppImage/scripts/console_metrics.py`, `AppImage/scripts/tests/test_console_metrics.py`, `AppImage/scripts/tests/fixtures/console/proc_*` | — | `test_console_metrics.py` | principal |
| T2 | Contrato del snapshot: normalizador desde JSON de `pvesh`, estados por fuente, escritura atómica, lease, `ensure_publisher(...)`, `load_snapshot(...)` | `AppImage/scripts/console_snapshot.py`, `AppImage/scripts/tests/test_console_snapshot.py`, `AppImage/scripts/tests/fixtures/console/pvesh_*.json` | — | `test_console_snapshot.py` | principal |
| T3 | Arranque del publicador en el Monitor (import + `ensure_publisher` junto a `_ensure_net_rate_sampler`, callables de nodo/pvesh/idioma, creación de `/run/proxmenux-monitor` con grupo) | `AppImage/scripts/flask_server.py` (sólo el bloque nuevo) | T2 | `test_console_snapshot.py` (ya cubre el módulo); verificación manual del hilo en QA (`journalctl`) | light |
| T4 | Motor de render puro: `Grid`, paneles, glifos, roles de color, vistas A y B, reglas de colapso, `sanitize`, textos vía callable `tr(key)` | `AppImage/scripts/console_render.py`, `AppImage/scripts/tests/test_console_render.py`, `AppImage/scripts/tests/fixtures/console/viewmodel_*.json` | — (el `ViewModel` está fijado en §2.4) | `test_console_render.py` | principal |
| T5 | Entrada `console_tui.py`: argumentos (`--autostart`, `--once`, `--snapshot`, `--proc-root`, `--glyphs`), bucle curses (1 s, `KEY_RESIZE`, teclas), carga de snapshot y edad, tasas por guest entre snapshots, lease, i18n (carga de catálogo + idioma), señales, salida limpia | `AppImage/scripts/console_tui.py`, `AppImage/scripts/tests/test_console_tui_smoke.py` | T1, T2, T4 | `test_console_tui_smoke.py` (pty, QA remoto) | principal |
| T6 | Catálogo i18n `runtime.consoleTui.*` en inglés (lista de §2.6) | `AppImage/messages/en/common.json` | — (claves fijadas aquí) | `python -c 'json.load'` + prueba en T5 de que toda clave usada existe en `en` | light |
| T7 | Empaquetado: rama `--tui` en `AppRun`, `cp` de los cuatro módulos en `build_appimage.sh`, nota en `AppImage/README.md` | `AppImage/scripts/AppRun`, `AppImage/scripts/build_appimage.sh`, `AppImage/README.md` | T5 (nombres de archivo) | `bash -n`; build del AppImage en QA remoto y `AppRun --tui --once` sobre fixture | light |
| T8 | Wrapper del getty y lanzador manual | `scripts/console/proxmenux-console-getty`, `scripts/console/proxmenux-console`, `tests/test_console_getty_wrapper.py` | — | `test_console_getty_wrapper.py` | principal |
| T9 | Funciones de instalación/actualización/desinstalación (`install_console_tui`, `remove_console_tui`, detección de tty1 ocioso, pregunta de autoarranque) en ambos instaladores, llamadas tras la creación/actualización del servicio Monitor | `install_proxmenux.sh`, `install_proxmenux_beta.sh`, `tests/test_console_installer.py` | T8 (rutas) | `test_console_installer.py` | principal |
| T10 | Menú de configuración: entradas de consola (lanzar, autoarranque, estado) y llamada a `remove_console_tui` en `uninstall_proxmenux_monitor` y antes del `rm -rf "$BASE_DIR"` de `uninstall_proxmenu` (la función se carga desde el instalador descargado o se duplica mínimamente en el menú — decisión del implementador, documentada en el commit) | `scripts/menus/config_menu.sh` | T9 (nombres) | extensión de `tests/test_console_installer.py` acotada al menú (fake `dialog`, `systemctl`) | light |
| T11 | CI: `test-console-tui.yml` (matriz 3.11/3.13; lanes de los cinco archivos de prueba excepto el smoke de pty, que queda para QA remoto) | `.github/workflows/test-console-tui.yml` | T1, T2, T4, T8, T9 (nombres de archivo) | el propio workflow en verde | light |
| T12 | Documentación: página web de docs (JSON + `page.tsx`), entrada de `CHANGELOG.md`, sección en `README.md` del repo | `web/messages/en/docs/settings/console-monitor.json`, `web/app/[locale]/docs/settings/console-monitor/page.tsx`, `CHANGELOG.md`, `README.md` | T5, T9, T10 (comportamiento final) | revisión de enlaces y `npm run build` de `web/` en QA remoto | light |

Orden sugerido: T1 ∥ T2 ∥ T4 ∥ T6 ∥ T8 → T3, T5, T9 → T7, T10 → T11 → T12. Integración, build del AppImage, ejecución de las lanes en el worker QA y la auditoría adversarial (Gemini vía `agy`, lectura) las hace el Coordinador; la aceptación en hardware (§2.8) la hace el Owner.

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

## 4. Consecuencias

Positivas:
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
- Nueva superficie de instalación (drop-in, usuario, tmpfiles, dos binarios) que los instaladores deben mantener idempotente en dos archivos casi gemelos (`install_proxmenux.sh` y `_beta.sh`).

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

Ver §9 del grill (`.notes/docs/specs/2026-10-17-console-tui-grill.md`); las cifras del repo citadas aquí (`flask_server.py:1766, 2759-2800, 5891-5897, 2499-2508, 122-154`; `install_proxmenux.sh:528, 608, 646, 625-640`; `install_proxmenux_beta.sh:415, 524`; `config_menu.sh:95-137, 403-440, 578-612, 821`; `utils.sh:228-277`; `build_appimage.sh:128-148, 178-181`; `notification_templates.py:25-73`) se verificaron en el worktree el 2026-10-17.
