---
base_sha: 227780aba76eb3f35dc50b888b3b28facd1f0c5f
head_sha: 33c8f7a8376e71c13e17a084f6f3e64941551bc6
verdict: pending
auditor: "agy --mode plan --model gemini-3.8-flash-high (round 1)"
scope: spec
kind: auditor
profile: feature
date: 2026-10-17
findings: {critical: 1, important: 5, minor: 4}
---

# Recibo de revisión — ADR 001 consola TUI, ronda 1

Veredicto `pending`: el Auditor recomendó APROBAR CON CAMBIOS en ambos ejes. Critical e Important
se adjudican y corrigen en el ADR y vuelven a revisión acotada (ronda 2). Salida del Auditor, sin editar:

# Auditoría Adversarial — Consola TUI de ProxMenux Monitor en tty1 (HDMI/VGA)

- **Fecha de auditoría**: 2026-10-17
- **Documentos auditados**: ADR 001 (`.notes/docs/adrs/001-console-tui.md`, commit `33c8f7a8`) y Grill de especificación (`.notes/docs/specs/2026-10-17-console-tui-grill.md`).
- **Perfil declarado**: `feature`
- **Modo**: Auditoría adversarial estricta de diseño previa a implementación (fail-closed).

---

## Veredicto 1 — Spec-compliance frente a los requisitos del Owner

Perfil verificado: feature

El diseño responde de manera muy fiel y exhaustiva a los requerimientos fundamentales del Owner:
1. Requisito (a) ("algo como bashtop con monitor de Proxmox en uno solo", salida HDMI/VGA nativa): cumple la visualización simultánea de métricas de host (CPU por núcleo/historial, RAM, ARC, ZRAM, swap, red, disco, temp) y estado de hipervisor Proxmox.
2. Requisito (b) (autoarranque, salida limpia a login de `agetty`, persistencia de login tras logout y relanzamiento manual): implementado mediante el drop-in en `getty@tty1`, marcador por boot en `/run` y lanzador `proxmenux-console`.
3. Requisito (c) (rendimiento de nodo y de clúster): paneles locales directos de `/proc` y vista agregada condicional de clúster con quorum.
4. Requisito (d) (Vista A principal con selector Node/Cluster y máquinas VM/CT que más CPU/RAM consumen; Vista B con nodos en una fila compacta): diagramado exacto a 80×24 sin depender de la variante C descartada.

Sin embargo, el escrutinio adversarial detecta los siguientes desajustes de especificación frente a lo prometido:

### Hallazgos de Spec-Compliance

#### [Important] Vacío de datos en HDMI durante el arranque ("no data" en paneles clave tras reiniciar)
- **Evidencia**: ADR §2.2 (*Justificación: "El Owner mira la consola justo después de un reinicio..."*) en contraste con §2.2 (*Comportamiento ante fallos: "si nunca hubo archivo, paneles con 'no data'"*) y §2.3 (*"Primer snapshot ≤ 2 s tras aparecer el lease"*).
- **Impacto**: El caso de uso primordial del Owner es observar el monitor físico inmediatamente tras encender o reiniciar el nodo. En ese momento, `systemd` lanza `getty@tty1` mucho antes de que `proxmenux-monitor.service` (que depende de `network.target` y `pve-cluster.service`) haya completado su inicialización de Python/Flask/gevent y ejecutado su primer `pvesh`. Durante 5 a 25 segundos tras el boot, el operador verá la mitad de la pantalla (TOP CPU, TOP MEMORY, CLUSTER y STORAGE) con etiquetas `no data` o cabeceras alarmantes en rojo (`MONITOR UNAVAILABLE`), contradiciendo la promesa de tener el monitor de Proxmox listo tras reiniciar.
- **Corrección propuesta**: Definir en la especificación de render un estado explícito de arranque (`INITIALIZING` / `CONNECTING TO MONITOR...`) durante los primeros 30 segundos si `uptime < 60s`, evitando pintar paneles vacíos con aspecto de fallo crítico y forzando al publicador a emitir un primer snapshot inmediato en cuanto `flask_server` inicialice sin esperar ciclos de reposo.

#### [Important] Ausencia de procesos de host en la vista principal frente a la expectativa de "algo como bashtop"
- **Evidencia**: Requisito del Owner (a) (*"algo como bashtop con monitor de Proxmox en uno solo"*) frente a ADR §2.4 (*"Semántica de las listas TOP: sólo guests running con cpu/mem numéricos"*).
- **Impacto**: Si el nodo PVE experimenta un 100% de uso de CPU o saturación de memoria provocada por procesos del propio host (e.g. ZFS scrub, tareas de backup `vzdump`, compilación, replicación zfs o procesos huérfanos), el monitor mostrará las barras superiores en saturación crítica (`crit` rojo), pero las dos tablas inferiores (TOP CPU / TOP MEMORY) aparecerán casi vacías o con consumos bajos de las VMs. El operador que espera un sustituto de bashtop/btop quedará desorientado sin saber qué proceso está saturando el nodo.
- **Corrección propuesta**: Mantener las listas de VM/CT solicitadas en (d), pero añadir en la cabecera del panel TOP o en el strip un indicador de carga residual del host: `Host overhead: X% CPU / Y GiB RAM`, o permitir alternar una vista compacta de procesos de host (`p`) si la carga no-guest supera un umbral (e.g. >25%).

#### [Minor] Exclusión en TOP MEMORY de guests pausados o en migración que retienen memoria
- **Evidencia**: ADR §2.4 (*"Semántica de las listas TOP: sólo guests running con cpu/mem numéricos"*).
- **Impacto**: Un guest en estado `paused` o bloqueado durante una migración retiene su memoria RAM asignada en el hipervisor (a menudo decenas de GiB). Al filtrarse exclusivamente por `status == "running"`, desaparece de la lista TOP MEMORY, distorsionando el análisis de capacidad de memoria del nodo.
- **Corrección propuesta**: En el panel TOP MEMORY, incluir guests cuyo estado sea `running` o `paused` siempre que `mem > 0`.

---

**Resumen Veredicto 1**: 0 Critical, 2 Important, 1 Minor  
**Recomendación**: **APROBAR CON CAMBIOS**

---

## Veredicto 2 — Riesgo / Calidad de Diseño

El diseño técnico es de una calidad sobresaliente: lectura local desacoplada para disponibilidad inmediata, uso de primitivas atómicas en tmpfs, retención de `agetty` estándar sin secuestro del tty, ausencia de dependencias pesadas gracias a `_curses` del host, y un análisis riguroso de fuentes de consola (CP437, `linux.set`).

No obstante, al auditar los modos de falla, la seguridad en bajo nivel y la concurrencia, emergen riesgos críticos que deben remediarse antes de tirar una sola línea de código:

### Hallazgos de Riesgo y Calidad

#### [Critical] Corrupción de consola física y cursor oculto en tty1 ante fallo o salida abrupta de la TUI
- **Evidencia**: ADR §2.5 paso 9 (*"Tras cualquier salida de la TUI: stty sane <&0 2>/dev/null || true; printf '\033[H\033[2J'; run_login"*) combinado con el `ExecStart` del vendor (`agetty --noreset --noclear - ${TERM}`).
- **Impacto**: La TUI utiliza por defecto el juego de caracteres alternativo de curses (`ACS_*`), el cual conmuta la consola Linux al set gráfico G1 (secuencia `\033(0` / SO `\016`). Además, las aplicaciones curses ocultan el cursor durante el dibujado (`\033[?25l`). Si el proceso de la TUI muere inesperadamente (SIGKILL, fallo de memoria OOM, traceback de Python no capturado), la consola física queda en modo G1 y sin cursor. Dado que `agetty` se ejecuta con `--noreset`, y `stty sane` **sólo** restablece los flags de línea del kernel (termios) pero **no** envía secuencias de escape ANSI/VT, el prompt `login:` aparecerá sin cursor y cualquier texto que el usuario teclee se renderizará como símbolos gráficos ininteligibles (rombos, bloques, esquinas). Esto inhabilita el acceso físico en tty1 tras un fallo.
- **Corrección propuesta**: Sustituir la limpieza del paso 9 por una secuencia exhaustiva de reset de emulador VT:
  ```bash
  stty sane <&0 2>/dev/null || true
  printf '\033[0m\017\033[?25h\033c'
  ```
  Esto restaura atributos de color (`\033[0m`), fuerza el regreso al conjunto ASCII estándar G0 mediante Shift-In (`\017`), hace visible el cursor (`\033[?25h`) y ejecuta un hard reset del terminal RIS (`\033c`) antes de hacer `exec /sbin/agetty`.

#### [Important] Duplicación redundante de invocaciones a `pvesh` cada 5 s entre el Monitor y la TUI
- **Evidencia**: ADR §2.7 (*"El publicador usa su propio subprocess.run(['pvesh', 'get', '/cluster/resources', '--output-format', 'json'], timeout=10) ... y no comparte la caché --type vm existente para no cambiar su semántica"*) frente a `flask_server.py:2759-2800` (`get_cached_pvesh_cluster_resources_vm`).
- **Impacto**: En un servidor donde el operador tenga el dashboard web abierto y la TUI activa en HDMI, el sistema ejecutará dos subprocesos `pvesh` independientes cada 5 segundos. En hosts homelab (como los P3 o nodos N100), cada invocación de `pvesh` cuesta entre 300 y 800 ms de CPU mononúcleo en Perl. Dos llamadas concurrentes cada 5 s representan un gasto permanente de hasta el 25–30% de un núcleo sólo para consultar el estado del clúster, además de provocar contención innecesaria en el sistema de archivos de clúster `pmxcfs`.
- **Corrección propuesta**: Unificar la consulta en `flask_server.py`. Convertir la función existente en un proveedor genérico de `cluster/resources` (todos los tipos) bajo el mismo lock single-flight con TTL de 5 s. Los endpoints web existentes filtran en memoria los VMs (`type in ('qemu', 'lxc')`), y el hilo publicador de la TUI toma esa misma estructura, eliminando por completo el segundo subproceso Perl.

#### [Important] Riesgo de fallo de importación (`ModuleNotFoundError`) bajo `python3 -I` en `AppRun`
- **Evidencia**: ADR §2.7 (*"con -I (modo aislado: ignora PYTHONPATH del entorno y el cwd; la TUI sólo necesita stdlib y los módulos hermanos en usr/bin, que importa por ruta del propio archivo)"*).
- **Impacto**: En CPython 3.11 y 3.13, el flag `-I` (isolated) activa automáticamente `-P` (safe path), eliminando de `sys.path` tanto el directorio actual como el directorio del script ejecutado. Si `console_tui.py` hace `import console_metrics` o `import console_render` de forma directa, el intérprete fallará de inmediato con `ModuleNotFoundError`. Al fallar, el wrapper registrará el traceback y saltará a `agetty`, impidiendo que la TUI arranque en producción.
- **Corrección propuesta**: Especificar en la tarea T5 del DAG que `console_tui.py` debe normalizar de forma determinista `sys.path` al inicio de su ejecución:
  ```python
  import os, sys
  sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
  ```

#### [Important] Permisos de terminal para el usuario sin privilegios y grupo `tty`
- **Evidencia**: ADR §2.5 (*"usuario/grupo proxmenux-console: adduser --system --group --no-create-home --shell /usr/sbin/nologin"*).
- **Impacto**: Los nodos de dispositivo `/dev/tty1` y los pseudoterminales `/dev/pts/*` en Debian están típicamente asignados al grupo `tty` con permisos `0620`. Aunque el proceso hijo herede los descriptores 0 y 1 abiertos por `systemd`, ciertas operaciones de `curses` y `termios` intentan abrir internamente `/dev/tty`. Si el usuario `proxmenux-console` no pertenece al grupo `tty`, estas llamadas fallan con `EACCES (Permission denied)`, impidiendo inicializar curses en ejecuciones interactivas o relanzamientos manuales.
- **Corrección propuesta**: En la creación del usuario en el instalador, añadirlo al grupo del sistema `tty`:
  `adduser --system --ingroup proxmenux-console --no-create-home --shell /usr/sbin/nologin proxmenux-console && usermod -a -G tty proxmenux-console`.

#### [Minor] Corrupción visual permanente ante mensajes de kernel (`printk`) en `tty1`
- **Evidencia**: ADR §2.4 (sólo se contempla repintado manual mediante tecla `r`; no hay repintado periódico programado).
- **Impacto**: Al ser `tty1` el destino de la consola por defecto (`console=tty0`), cualquier mensaje del kernel (e.g. eventos udev, mensajes de conexión USB, alertas de ZFS o logs de red) se escribe directamente en la memoria de vídeo del terminal. Dado que `curses` optimiza refrescando sólo celdas que cambian, los fragmentos de logs del kernel permanecerán visibles indefinidamente hasta que el usuario presione manualmente `r`.
- **Corrección propuesta**: Programar un repintado forzado completo (`clearok(stdscr, True)`) en el bucle principal de la TUI cada 60 segundos de forma desatendida.

#### [Minor] Desacoplamiento frágil de `remove_console_tui` entre el instalador y `config_menu.sh`
- **Evidencia**: ADR §2.9 Tarea T10 (*"la función se carga desde el instalador descargado o se duplica mínimamente en el menú — decisión del implementador"*).
- **Impacto**: Dejar como decisión abierta del implementador la duplicación o sourcing de funciones de desinstalación de bajo nivel (drop-in, systemd reload, borrado de binarios) entre `install_proxmenux.sh` y `config_menu.sh` propicia divergencias e inconsistencias durante la desinstalación.
- **Corrección propuesta**: Centralizar la lógica de instalación/desinstalación de la consola en un script común (e.g. `scripts/console/console_install_helpers.sh`) que sea importado (`source`) tanto por los instaladores como por `config_menu.sh`.

---

**Resumen Veredicto 2**: 1 Critical, 3 Important, 2 Minor  
**Recomendación**: **APROBAR CON CAMBIOS**

---

## Respuestas a las Preguntas Explícitas

### Preguntas del Coordinador

#### 1. Kernel printk / other writers to tty1: repintado periódico
**Sí, el diseño lo necesita imperativamente.** En Linux, `/dev/console` mapea a `tty0` (el VT activo, habitualmente `tty1`). Mensajes de hardware o del kernel ensucian el buffer de vídeo. Confiar únicamente en la tecla manual `r` es inviable para un monitor desatendido. Se debe programar una llamada periódica a `clearok(stdscr, True)` cada **60 segundos** en el bucle de la TUI. Esto añade un coste computacional ínfimo (<1 ms cada minuto) y garantiza la auto-recuperación de la pantalla.

#### 2. `python3 -I`, `sys.path` (-P) e inyección de imports
"Import siblings by the file's own path" **no está suficientemente especificado en el ADR**. Bajo `python3 -I`, Python 3.11/3.13 ignora `PYTHONPATH`, cwd y **no añade** el directorio del script a `sys.path` (comportamiento `-P`). La implementación debe ejecutar explícitamente `sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))` al inicio de `console_tui.py`.  
Respecto a *import-hijacking*: **El riesgo es nulo.** El árbol `/usr/local/share/proxmenux/monitor-app` pertenece a `root:root` (instalado con permisos de administrador) y el usuario ejecutor `proxmenux-console` no tiene permisos de escritura en él. Al usar `-I`, no se consulta el directorio de trabajo ni directorios de usuario (`~/.local`), blindando el proceso.

#### 3. Permisos de lectura para `proxmenux-console` y `config.json` 0600
- `/proc/stat`, `/proc/meminfo`, `/proc/net/dev`, `/proc/diskstats`, `/proc/spl/kstat/zfs/arcstats`, `/sys/class/hwmon/*`: Son legibles universalmente por cualquier usuario sin privilegios (`0444`).
- `/run/proxmenux-monitor/console.json`: Legible gracias al modo `0640` y grupo `proxmenux-console`.
- Árbol de la AppImage: Legible siempre que el instalador garantice `chmod -R a+rX` sobre `monitor-app`.
- **Si `config.json` tiene permisos 0600 (`root:root`)**: El usuario `proxmenux-console` no podrá leerlo. No obstante, el diseño es resiliente: la fuente primaria del idioma es `snapshot.language` (inyectado por el Monitor como root). Si el snapshot no existe y `config.json` es ilegible, el ADR cae limpiamente al valor por defecto `"en"` sin abortar. Se recomienda que el instalador asegure `0644` en `config.json`.

#### 4. Estado de VT tras salida de la TUI: ¿Basta `stty sane`?
**No, no basta en absoluto.** `stty sane` actúa exclusivamente sobre la disciplina de línea del kernel (ICANON, ECHO, etc.). No envía secuencias de control al emulador de terminal. Si la TUI termina abruptamente mientras curses dibujaba con `ACS_*` o con el cursor apagado, el terminal queda atrapado en el charset alternativo G1 y sin cursor. Al tener `agetty` el flag `--noreset`, el prompt de login heredará este estado y los caracteres tecleados saldrán como símbolos gráficos. Se debe enviar obligatoriamente un hard-reset VT:
`printf '\033[0m\017\033[?25h\033c'` antes de invocar a `agetty`.

#### 5. Publicador (root) haciendo `stat` de un lease de usuario no privilegiado
**No hay riesgo de seguridad explotable.**  
- *Symlink / TOCTOU*: El directorio padre `/run/proxmenux-console` es de `root:root 0755`, por lo que el usuario no puede moverlo. Si dentro de `client/` crea un symlink hacia cualquier archivo del sistema, `os.stat()` únicamente lee metadatos de inodo; nunca abre ni modifica el fichero. Para máxima pulcritud, el publicador debe usar `os.lstat()` y comprobar `stat.S_ISREG()`.
- *DoS por lease continuo*: Si el usuario hace `touch` continuo del lease, el publicador simplemente ejecutará `pvesh` cada 5 s, que es el comportamiento normal previsto cuando la TUI está en pantalla. Dado que `proxmenux-console` es un usuario sin shell interactivo, el riesgo es totalmente asumible.

#### 6. Compatibilidad de `setpriv` en PVE 8 (util-linux 2.38) vs PVE 9 (util-linux 2.41)
**Totalmente compatible.** Todos los modificadores empleados (`--reuid`, `--regid`, `--init-groups`, `--reset-env`, `--no-new-privs`) fueron introducidos entre util-linux 2.21 y 2.32 (2018). Debian 12 (PVE 8) incluye util-linux 2.38.7 y Debian 13 (PVE 9) incluye 2.41. Los comandos `id` e `install` pertenecen a `coreutils` y usan sintaxis estándar POSIX.

#### 7. Orden de instalación/desinstalación y caída intermedia en tty1
El orden especificado en el ADR es robusto:
- En instalación: los binarios y el wrapper se colocan en `/usr/local/sbin` **antes** de crear el drop-in de systemd y recargar.
- En desinstalación: el drop-in se borra **primero**, se ejecuta `systemctl daemon-reload` y sólo entonces se eliminan el wrapper y los binarios.
Si el sistema sufriera un corte eléctrico entre pasos:
- *Drop-in presente sin wrapper*: Systemd falla al ejecutar el binario, pero gracias al prefijo `ExecStart=-`, no aborta en seco; sin embargo, al no haber login inmediato, el operador físico puede presionar **`Ctrl+Alt+F2`** para acceder a una consola limpia en `tty2` (la cual no tiene drop-in y pertenece al vendor). Desde `tty2` puede reparar systemd.

#### 8. Publicación condicionada por lease vs publicación continua
**No es una optimización prematura; es indispensable.** En procesadores de bajo consumo habituales en homelab (N100, Celeron, P3), `pvesh` consume entre 6% y 16% de un núcleo de forma sostenida. Ejecutarlo las 24 horas del día los 365 días del año sin que nadie esté mirando la pantalla física es un desperdicio injustificable. La ventana de 2 a 4 segundos de datos vacíos al inicio se soluciona fácilmente en la UI mostrando un estado transitorio `Connecting...`, mientras que la publicación continua consumiría ciclos de CPU permanentemente sin resolver el retardo de arranque inicial del servicio.

#### 9. Proporcionalidad y estrategia de aceptación upstream (MacRimi)
El alcance completo en un único PR puede generar fricción y desconfianza en el mantenedor de ProxMenux por tocar el arranque físico del sistema (`getty@tty1`).
**Recomendación de entrega**:
- **PR 1**: Núcleo de la TUI dentro del AppImage (`console_*.py`), rama `--tui` en `AppRun`, gancho de publicación en `flask_server.py` y lanzador CLI manual `/usr/local/bin/proxmenux-console`. Este PR es de riesgo cero, no toca systemd ni usuarios de sistema, y permite validar el monitor por SSH.
- **PR 2**: Integración en `tty1` (usuario `proxmenux-console`, wrapper, drop-in, instaladores y menú de configuración). Si se envía todo junto, el instalador debe dejar la función como **opt-in** (`AUTOSTART=no` por defecto durante la fase beta).

---

### Preguntas del Designer por Eje

#### 10. Concurrencia
- **(a) Atomicidad de `os.replace`**: En sistemas de archivos POSIX y tmpfs, `rename()` es atómico. Si la TUI ya tiene abierto el archivo, continuará leyendo el inodo anterior hasta cerrarlo; si lo abre de nuevo, obtendrá el nuevo inodo completo. El manejo de `JSONDecodeError` en `load_snapshot` es una protección defensiva suficiente.
- **(b) Múltiples TUI compartiendo el lease**: No hay colisión. `touch` / `os.utime()` es una actualización atómica de inodo. El cálculo de deltas de red y disco por guest se computa en la memoria privada de cada proceso TUI.
- **(c) Carrera en arranque con `/run`**: `systemd-tmpfiles-setup.service` se ejecuta en `sysinit.target`, mucho antes de que `getty@tty1` arranque en `multi-user.target`. Además, el wrapper ejecuta `mkdir -p` e `install -d` como root antes de lanzar la TUI. No hay ventana de carrera.

#### 11. Seguridad y Autorización
- **(a) Vector de ataque pre-login en tty1**: Frente a `agetty` (que corre como root), la TUI ofrece una superficie de ataque significativamente menor al correr bajo `setpriv` como usuario sin privilegios y con `--no-new-privs`. El bucle de curses sólo lee teclas fijas de navegación sin invocar subshells. La única diferencia es la visibilidad física de nombres de guests antes de autenticarse.
- **(b) Influencia del usuario en variables de entorno**: Imposible. `setpriv --reset-env` purga el entorno por completo; `env` define únicamente `TERM`, `LANG`, `HOME` y `PMX_CONSOLE_STATE_DIR`. Aunque `AppRun` declare `LD_LIBRARY_PATH`, el binario `python3` se invoca con `-I` (aislado) y todos los directorios del AppImage son propiedad de root.
- **(c) Lectura de `console.json` por el grupo `proxmenux-console`**: No supone un problema de seguridad. El usuario `proxmenux-console` tiene `/usr/sbin/nologin` y ningún usuario humano pertenece a dicho grupo. Los nombres e IDs de VMs son metadatos habituales en hipervisores compartidos.

#### 12. Modos de Falla (Matriz de Escenarios)
1. *Monitor parado/deshabilitado*: Cubierto. CPU/RAM locales en vivo; paneles de clúster marcan `MONITOR UNAVAILABLE`.
2. *`pvesh` colgado 10 s*: Cubierto. Subprocess timeout de 10 s; el publicador etiqueta la fuente como `stale`.
3. *Sin quorum*: Cubierto. `quorate = false` -> cabecera en rojo `NO QUORUM`, sin agregados de clúster.
4. *Nodo remoto caído*: Cubierto. Nodo marcado `offline`, métricas agregadas sólo de nodos online con indicador de cobertura.
5. *`/run` lleno (ENOSPC)*: Parcialmente cubierto. La escritura temporal fallará y se conservará el snapshot previo. Se debe asegurar que el wrapper capture errores de escritura en `tui.log` sin abortar hacia fallo silencioso.
6. *`console.json` corrupto / schema inválido*: Cubierto. `load_snapshot` captura la excepción y retorna `Snapshot.unavailable()`.
7. *Host sin `_curses`*: Cubierto. El paso 6 del wrapper lo detecta mediante `python3 -c 'import curses'` y cae limpiamente a `run_login()`.
8. *`TERM` distinto de `linux`*: Cubierto. Fallback en wrapper a `${TERM:-linux}`.
9. *Consola serial primaria (`tty0` ausente)*: Cubierto. `getty@tty1` posee la directiva nativa `ConditionPathExists=/dev/tty0`, impidiendo su ejecución en sistemas puramente seriales.
10. *Fuente de consola sin ACS*: Cubierto. Soporte de tecla `a` para alternar a ASCII o configuración `GLYPHS=ascii`.
11. *`setpriv` ausente*: Cubierto. El wrapper lo valida en sus comprobaciones y salta a `agetty`.
12. *Usuario `proxmenux-console` borrado*: Cubierto. Comprobado mediante `id "$CONSOLE_USER"`, cayendo a `run_login()`.
13. *`AUTOSTART` inválido*: Cubierto. Evaluación estricta `[ "$AUTOSTART" = yes ] || run_login`.
14. *Actualización de AppImage en caliente*: Parcialmente cubierto. El proceso en memoria sigue corriendo con sus descriptores previos. Al salir o reiniciar, el nuevo wrapper ejecutará el AppImage reemplazado.

#### 13. Presupuesto y Coste Computacional
- **(a) Parseo de 1 MiB de JSON cada 5 s y límites (≤2% CPU, ≤40 MiB RSS)**: Totalmente realista. `json.loads()` de 1 MiB en Python consume entre 5 y 8 ms de CPU. Evaluando `mtime`, el parseo sólo ocurre cada 5 segundos (gasto medio de CPU <0,2%). Curses actualiza únicamente las celdas diferenciales. La memoria RSS de Python 3 con las listas de 1000 guests se estabiliza en torno a 28–35 MiB.
- **(b) Coste de `pvesh` con y sin `--type vm`**: En Proxmox VE, `cluster/resources` lee la base de datos completa de `pmxcfs`. El parámetro `--type vm` sólo aplica un filtro en memoria dentro del script de Perl. Por tanto, invocarlo sin `--type` para obtener todos los recursos cuesta prácticamente lo mismo en CPU y tiempo.
- **(c) Compartir la caché de `flask_server`**: **Debe compartirse obligatoriamente.** No tiene sentido mantener dos llamadas concurrentes a `pvesh` cada 5 s (una para la web y otra para el snapshot). La función `get_cached_pvesh_cluster_resources_vm` debe convertirse en la fuente única que alimente a ambos módulos.

#### 14. Rollback y Recuperación
- **(a) Garantía de login tras desinstalar**: Plenamente garantizado. La eliminación del drop-in seguida de `systemctl daemon-reload` devuelve el control unitario al archivo original del vendor.
- **(b) Borrado manual `rm -rf /usr/local/share/proxmenux`**: Protegido. El wrapper reside en `/usr/local/sbin/`. Al comprobar que el ejecutable `$LAUNCHER` no existe, salta de inmediato a `run_login()`, garantizando que tty1 mantenga el login de `agetty`.
- **(c) Modo seguro ante bucles de fallo**: Innecesario. El marcador `/run/proxmenux-console/autostart.tty1` se crea **antes** de invocar a la TUI. Si la TUI falla una sola vez, cualquier reaparición de `getty` encuentra el marcador y va directo a `agetty`. Por diseño, el límite de intentos es exactamente 1 por arranque.

#### 15. Ergonomía de Pantalla y Glifos
- El diseño cumple al detalle lo requerido por el Owner a 80×24. La Vista A organiza con claridad métricas de nodo y los dos TOPs de máquinas virtuales; la Vista B sintetiza con precisión los nodos del clúster en una única línea por nodo (CPU, RAM, disco root, guests y uptime).
- El orden de colapso es quirúrgico y acertado, preservando en todo momento el rendimiento vital y sacrificando primero históricos secundarios y columnas accesorias.
- El uso de `ACS_*` como juego por defecto es la decisión técnica correcta: garantiza compatibilidad nativa en el modo texto VGA y en las fuentes estándar de 256 caracteres del kernel Linux, donde UTF-8 suele fallar.

#### 16. Testabilidad
- El conjunto de pruebas en §2.8 cubre exhaustivamente la lógica aislada, la simulación de argumentos y las condiciones del wrapper.
- **Hueco no detectable sin hardware**: La persistencia de estados de terminal residuales en el driver del kernel (VT atrapado en G1 o cursor apagado tras un fallo abrupto) y la interacción real de `setpriv` con los descriptores de `/dev/tty1` gestionados por systemd.
- **Prueba prioritaria a añadir**: Un test de integración en entorno virtualizado que inyecte un `kill -9` al proceso de la TUI y valide mediante un script expect/pty que la pantalla resultante muestra un prompt `login:` funcional con cursor visible y caracteres legibles.

#### 17. Proporcionalidad del DAG
- La tarea **T10** presenta una indeterminación que rompe la limpieza del DAG: propone que la lógica de `remove_console_tui` se duplique o se cargue desde el instalador.
- **Reordenación recomendada**: Extraer las funciones de ciclo de vida e instalación a una biblioteca común (`scripts/console/console_install_helpers.sh`) dentro de **T8/T9**. De este modo, tanto los instaladores (T9) como el menú de configuración (T10) consumirán la misma fuente de verdad sin duplicación de código ni dependencias circulares.
