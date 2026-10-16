---
base_sha: 227780aba76eb3f35dc50b888b3b28facd1f0c5f
head_sha: 4a06659a2aa8b18d1f19bd7c58c966d9490a6909
verdict: approved
auditor: "agy --mode plan --model gemini-3.8-flash-high (round 2, scoped delta review)"
scope: spec
kind: auditor
profile: feature
date: 2026-10-17
findings: {critical: 0, important: 0, minor: 0}
---

# Recibo de revisión — ADR 001 consola TUI (fase 3, spec)

Ronda 1 (`2026-10-17-console-tui-redteam-verdict-r1.md`, sobre 33c8f7a8): APROBAR CON CAMBIOS,
1 Critical / 5 Important / 4 Minor. El Designer adjudicó todo en la revisión 2 (4a06659a, ADR §8).
Ronda 2 (revisión acotada al delta, brief `2026-10-17-console-tui-redteam-brief-r2.md`): APROBAR en
ambos ejes, 0/0/0, los 10 hallazgos de la ronda 1 cerrados, rechazo de R1-4 (grupo `tty`) validado.

Un recibo `scope: spec` no exige vigencia frente a commits posteriores: su `head_sha` es el del ADR.

Salida del Auditor, ronda 2, sin editar:

# Auditoría adversarial — ronda 2 (revisión acotada) — ADR 001 consola TUI de ProxMenux Monitor

## Veredicto 1 — Spec-compliance

Perfil verificado: feature

No se detectan incumplimientos ni regresiones de especificación en el delta de la revisión 2. La ventana de arranque queda resuelta formalmente mediante el estado semántico `STARTING` (sin recurrir a subprocesos ni comandos externos), el cálculo de `host other` cubre el ángulo muerto de procesos del host (como `vzdump` o scrub de ZFS) sin incurrir en el coste ni la exposición de listar PIDs root, y la inclusión de cualquier guest con `mem > 0` en TOP MEMORY garantiza la visibilidad de guests pausados o en migración.

Resumen: 0 Critical, 0 Important, 0 Minor  
Recomendación: **APROBAR**

---

## Veredicto 2 — Riesgo / Calidad de diseño

El delta de la revisión 2 resuelve de manera impecable los riesgos técnicos señalados en la ronda 1. La división en dos PRs desacopla por completo la entrega del núcleo de la TUI de los cambios en el ciclo de vida de `getty@tty1`. La generalización de la caché de recursos en `flask_server.py` está respaldada por la arquitectura interna de PVE (`API2Tools::extract_vm_stats`), y el uso de `stty sane` + `ESC c` repara el emulador del VT sin alterar fuentes ni mapas de teclado.

Resumen: 0 Critical, 0 Important, 0 Minor  
Recomendación: **APROBAR**

---

## Análisis detallado del delta y preguntas de control

### 1. Evaluación del hallazgo rechazado en la ronda 1 (R1-4: grupo `tty`)

El rechazo por parte del Designer es **técnicamente correcto e irrebatible**, y el hallazgo de la ronda 1 queda formalmente **retirado**:

1. **Implementación de ncurses**: En `ncurses/base/lib_initscr.c`, `initscr()` delega en `newterm(name, stdout, stdin)`. La apertura explícita de `/dev/tty` mediante `fopen` existe exclusivamente dentro de bloques condicionales `#ifdef __CYGWIN__`. En Linux, curses opera sobre los descriptores estándar ya abiertos.
2. **Herencia de descriptores**: Bajo systemd con `StandardInput=tty`, el proceso hereda los descriptores de archivo 0, 1 y 2 ya abiertos por PID 1 antes del cambio de privilegios. Las llamadas `ioctl` de terminal (`TCGETS`, `TCSETS*`, etc.) operan directamente sobre el descriptor abierto y no realizan revalidación de permisos sobre el nodo del sistema de archivos.
3. **Semántica de `/dev/tty`**: En cualquier sistema Linux estándar (regla udev `50-udev-default.rules`), el nodo `/dev/tty` (terminal de control del proceso) tiene permisos `0666 root:tty`. Si un subcomponente lo abriera, cualquier proceso con terminal de control asignada tiene acceso de lectura y escritura sin pertenecer al grupo `tty`.
4. **Superficie de ataque y riesgo real**: Los nodos de terminales físicas y pseudoterminales (`/dev/tty1`–`63`, `/dev/pts/*`) tienen permisos `0620 root:tty`. Conceder el grupo `tty` a `proxmenux-console` le otorgaría permisos de escritura sobre las terminales de otros usuarios, permitiendo inyectar secuencias de escape o caracteres arbitrarios en sesiones de administradores en consola o pts. Por tanto, excluirlo del grupo preserva el principio de mínimo privilegio.

### 2. Evaluación de riesgos y regresiones en el delta

- **(a) Secuencia `stty sane` + `ESC c` (RIS) ante `agetty --noreset --noclear`**:
  - *Fuentes primarias*: En el subsistema VT del kernel Linux (`drivers/tty/vt/vt.c:reset_terminal()`), `ESC c` (RIS) reinicia variables internas del emulador de terminal (`vc_charset`, `default_attr()`, `vc_deccm`, tabuladores) y limpia la pantalla (`csi_J`).
  - *Fuentes de consola y teclado*: `reset_terminal()` **no** invoca `con_font_set` ni `set_palette`; los glifos de fuente cargados por `console-setup` en la memoria del controlador de vídeo se conservan íntegros. Del mismo modo, `vt_reset_keyboard()` en `keyboard.c` restablece el modo de teclado a `VC_XLATE`/`VC_UNICODE` y limpia estados transitorios de LEDs, pero no recarga ni borra la tabla de keymaps (`KDSKBENT`).
  - *Modo UTF-8*: `vc_utf` se restablece a `default_utf8`. En Debian y Proxmox VE el kernel se compila con `CONFIG_VT_HW_CONSOLE_BINDING=y` y `default_utf8=1`, por lo que el VT permanece en modo UTF-8. No hay pérdida de configuración operativa.
- **(b) Generalización de la caché `pvesh` a todos los tipos**:
  - En `PVE/API2/Cluster.pm`, el método `resources` extrae los datos de guests invocando `PVE::API2Tools::extract_vm_stats($vmid, $data, $rrd)` de manera idéntica tanto si se pasa `--type vm` como si no se pasa ningún tipo (en cuyo caso añade además bloques para `node`, `storage`, `pool` y `sdn`).
  - Filtrar la lista completa resultante con `[r for r in resources if r.get('type') in ('qemu', 'lxc')]` produce exactamente los mismos diccionarios y claves que la consulta específica de VMs. Los consumidores actuales del dashboard web no experimentan alteración de esquema ni de contenido.
- **(c) Ejecución de la TUI como root en PR1**:
  - En PR1, `proxmenux-console` es un comando manual invocado interactivamente por un operador que ya ha autenticado una sesión como `root` (vía SSH o shell local).
  - La TUI es estrictamente de solo lectura (lee `/proc`, `/sys` y `console.json`, sin invocar subprocesos, ni modificar `/etc`, ni abrir sockets de red) y sanea todas las cadenas mostradas en pantalla contra secuencias de escape ANSI. No introduce vectores de escalada ni mutación de estado. Es una solución segura y pragmática que permite validar la TUI antes de introducir el usuario de sistema en PR2.
- **(d) Estado `STARTING` decidido por `/proc/uptime < 120 s`**:
  - *Falsos positivos (Monitor caído durante el arranque)*: Durante los primeros 120 s tras el encendido, si el servicio Monitor fallara de inmediato, la TUI mostrará `Monitor starting…` en lugar de `MONITOR UNAVAILABLE` durante como máximo 2 minutos. Esto es aceptable y deseable durante la ventana de arranque mientras convergen los servicios de systemd.
  - *Falsos negativos (Reinicio del Monitor con uptime largo)*: Si el host lleva días encendido y el Monitor se reinicia, el publicador escribe un snapshot inicial inmediato con `resources.status = "starting"`. La condición (ii) detecta este estado mientras la edad del archivo sea ≤ 15 s, mostrando `Monitor starting…` independientemente del uptime.
- **(e) Carga de `console_lifecycle.sh` desde `$LOCAL_SCRIPTS` antes de `rm -rf "$BASE_DIR"`**:
  - En `config_menu.sh` (`uninstall_proxmenu`), el archivo `console_lifecycle.sh` se importa en el proceso Bash activo antes de ejecutar la eliminación recursiva de `$BASE_DIR`. Las funciones (`remove_console_tui`, etc.) residen en la memoria del intérprete y proceden a desregistrar el drop-in, ejecutar `systemctl daemon-reload` y limpiar los binarios y directorios del sistema de forma segura.

### 3. Coherencia del DAG en PR1 y PR2 (§2.9)

- **Aislamiento funcional**: Cada tarea representa un cambio cohesivo y comprobable por su propio conjunto de pruebas automatizadas.
- **Propiedad de archivos sin solape**:
  - En PR1, los módulos (`console_metrics.py`, `console_snapshot.py`, `console_render.py`, `console_tui.py`) tienen propietarios únicos en sus tareas iniciales.
  - La única concurrencia sobre `flask_server.py` está serializada y delimitada a regiones físicas disjuntas: P1-T3 posee exclusivamente la región de la caché de recursos (`:1758-1800` y `:2759-2797`), mientras que P1-T7 añade el hilo publicador en la inicialización (`:5900`), dependiendo explícitamente de P1-T3.
- **Independencia de PR1**: PR1 empaqueta el AppImage con la opción `--tui` y el comando `proxmenux-console` para root. No requiere el drop-in de systemd, ni el wrapper de getty, ni el usuario `proxmenux-console`. Puede desplegarse, probarse por SSH y validarse en producción sin alterar la secuencia de arranque de `tty1`.

---

## Tabla de cierre de hallazgos — Auditoría ronda 1

| # | Eje | Sev. R1 | Estado R2 | Razón de cierre / evaluación |
|---|---|---|---|---|
| **R1-1** | riesgo | Critical | **Cerrado** | Se incorpora `stty sane` + `ESC c` (RIS) tras cualquier salida de la TUI, restaurando disciplina de línea, cursor y mapas de caracteres sin degradar fuentes ni keymaps de la consola. |
| **R1-2** | riesgo | Important | **Cerrado** | Se unifica la caché single-flight de `/cluster/resources` a todos los tipos con TTL de 5 s; los consumidores de guests filtran en memoria sin generar llamadas duplicadas a `pvesh`. |
| **R1-3** | riesgo | Important | **Cerrado** | `console_tui.py` resuelve su ruta canónica vía `os.path.realpath(__file__)` y la inyecta al inicio de `sys.path`, neutralizando el comportamiento de `-P` bajo `python3 -I`. |
| **R1-4** | riesgo | Important | **Cerrado** *(Rechazo validado)* | Evidencia concluyente: ncurses usa los fd 0/1/2 heredados de systemd, `/dev/tty` es `0666` y pertenecer al grupo `tty` otorgaría capacidad innecesaria de inyección sobre otras terminales. |
| **R1-5** | spec | Important | **Cerrado** | Se define el estado `STARTING` (advertencia ámbar) evaluando `/proc/uptime < 120 s` y el snapshot inicial inmediato con `status="starting"`, sin ejecutar subprocesos en la TUI. |
| **R1-6** | spec | Important | **Cerrado** | Se añade el indicador aritmético `host other` en los títulos de TOP CPU y TOP MEMORY a partir del snapshot fresco, señalando carga del host sin el coste de muestrear PIDs. |
| **R1-7** | spec | Minor | **Cerrado** | TOP MEMORY evalúa la condición `mem > 0` con independencia del estado del guest (excluyendo plantillas), capturando correctamente instancias en pausa o migración. |
| **R1-8** | riesgo | Minor | **Cerrado** | Se añade repintado forzado completo (`clearok(True)` + `refresh()`) cada 30 s en el bucle principal para purgar mensajes esporádicos de `printk` en `tty1`. |
| **R1-9** | riesgo | Minor | **Cerrado** | Se centraliza el ciclo de vida en `scripts/console/console_lifecycle.sh`, cargándose desde `$LOCAL_SCRIPTS` antes de borrar el árbol de directorios en la desinstalación. |
| **R1-Q9** | proceso | — | **Cerrado** | El DAG divide la entrega en PR1 (núcleo TUI manual root) y PR2 (integración en `tty1`), desacoplando el riesgo sobre el arranque del sistema. |
