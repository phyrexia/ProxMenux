# Grill — Consola TUI de ProxMenux Monitor en la salida de video del nodo

- Fecha: 2026-10-17
- Revisión: 2 (2026-10-17, tras auditoría ronda 1: §5 y §6 ajustados; entrega en dos PRs secuenciales, ver ADR §2.9 y §8)
- Rol: Designer (Fable). Documento de concepción; no es implementación.
- Rama: `feature/console-tui` sobre `origin/develop` 227780ab.
- ADR asociado: `.notes/docs/adrs/001-console-tui.md`.

## 1. Problema en una línea

Un nodo Proxmox conectado a un monitor por HDMI/VGA muestra hoy un `login:` inerte; el Owner quiere que esa pantalla muestre, sin iniciar sesión, el uso y rendimiento del nodo y —si forma parte de un clúster— del clúster, al estilo bashtop, y que al salir devuelva el login normal.

## 2. Quién la usa y con qué frecuencia

- **Operador del homelab o de la sala de servidores** que pasa frente al rack o tiene el nodo conectado a una pantalla permanente: mira la consola varias veces al día, sin teclado la mayor parte del tiempo. Es el caso del Owner (dos nodos P3 Lite con HDMI).
- **Administrador que necesita entrar por la consola física** (red caída, SSH indisponible): necesita salir de la TUI con una tecla y encontrarse con `login:` en segundos. Caso raro pero crítico: la TUI jamás puede impedir el login.
- **Usuarios upstream de ProxMenux** (`MacRimi/ProxMenux`, PRs a `develop`): heterogéneos; muchos sin monitor conectado. Para ellos la función no debe costar CPU ni cambiar el comportamiento del sistema cuando nadie mira.

## 3. Línea base hoy

- `getty@tty1.service` de Debian (systemd 252 en PVE 8 / Debian 12; 257.13 en PVE 9 / Debian 13) lanza `agetty` en tty1 con `Restart=always`, `RestartSec=0`, `Type=idle`, `TTYVTDisallocate=yes` y `UnsetEnvironment=LANG LANGUAGE LC_*`. Nada de ProxMenux toca las TTY.
- ProxMenux Monitor es un servicio Flask (`proxmenux-monitor.service`, `User=root`, `Restart=on-failure`) que corre `AppRun` extraído en `/usr/local/share/proxmenux/monitor-app/` (`install_proxmenux.sh:528-692`). Expone el dashboard web en el puerto 8008; sus endpoints de métricas requieren autenticación JWT.
- El backend ya cachea `pvesh get /cluster/resources --type vm` con TTL de 5 s y single-flight (`AppImage/scripts/flask_server.py:1766`, `:2759-2800`), resuelve el nombre del nodo local leyendo el symlink `/etc/pve/local` (`:122-154`) y tiene lectores de zram inyectables por ruta (`AppImage/scripts/zram_metrics.py`).
- `AppRun` (`AppImage/scripts/AppRun`) usa el **python3 del host** (`cp311`/`cp313`) y sólo añade al `PYTHONPATH` los `dist-packages` empaquetados; no empaqueta intérprete propio. El flag existente es `--translate`; cualquier otra invocación arranca Flask.
- Pruebas: `unittest` puro, sin pytest; lanes en `.github/workflows/test-*.yml` con `python -I -B -m unittest discover -s <dir> -p <file>`; fixtures en `AppImage/scripts/tests/` para módulos del backend y `tests/` para scripts bash (patrón de bash con PATH restringido en `tests/test_tui_count_messages.py:51-54`).
- i18n: los scripts bash llaman `translate "..."` y un bot genera `lang/<lang>.json` (`scripts/utils.sh:244-277`, `.github/scripts/build_translation_cache.py`). El backend Python lee `AppImage/messages/<locale>/common.json`, empaquetado en `usr/share/proxmenux/messages/` (`AppImage/scripts/build_appimage.sh:178-181`, `AppImage/scripts/notification_templates.py:25-73`), y el bot `build-i18n-messages.yml` rellena las traducciones faltantes a partir del `en`.

## 4. Supuestos desafiados

| Supuesto | Desafío | Respuesta (hilo del Owner o evidencia) |
|---|---|---|
| "Basta con lanzar btop/bashtop en tty1" | btop no sabe de Proxmox (guests, clúster, quorum) y su modo TTY usa 16 colores y símbolos propios; no hay forma de integrar la vista de clúster. | Owner 2026-10-14 11:24: "algo como bashtop **con monitor de proxmox** en uno solo". Se construye una TUI propia que toma de bashtop la densidad y el refresco, no el binario. |
| "La TUI puede sustituir al getty mientras corre" | `Restart=always` + `RestartSec=0` reinician el getty al salir: si la TUI es el `ExecStart`, al pulsar `q` volvería a aparecer y nadie podría loguearse. | Owner 2026-10-14 11:25: salir al login y poder relanzarla. Solución: TUI **una vez por arranque** (marcador en `/run`) y después `agetty`; relanzamiento manual con un comando. |
| "La TUI debe correr como root para leer todo" | Antes del login cualquiera con acceso físico ve la pantalla; un proceso root con teclado es superficie de ataque (p. ej. `curses` + nombres de guests maliciosos). | Separar: el servicio Monitor (ya root) publica un snapshot; la TUI corre como usuario de sistema sin shell y lee `/proc`, `/sys` y el snapshot. Confirmado como requisito de diseño, no preguntado al Owner. |
| "Toda la información viene del snapshot del Monitor" | Si el Monitor está parado, actualizándose o en periodo de gracia de arranque (`startup_grace.py`: 3–5 min), la consola quedaría vacía justo cuando más se mira (tras un reinicio). | CPU/RAM/carga/red/disco/temperatura/zram locales se leen **directo de `/proc` y `/sys`** (todo legible sin privilegios); del snapshot sólo lo que exige `pvesh` (clúster, guests, almacenamiento PVE). |
| "Los gráficos braille de btop sirven en la consola" | La consola Linux renderiza con fuentes de 256/512 glifos: el font VGA del kernel es CP437 (sin braille ni bloques parciales ▁▂▃); los fontsets de console-setup (`Lat15.256`, `Uni2.512`…) tampoco los incluyen. | Juego de glifos reducido y verificado: líneas de caja vía `ACS_*` de curses (mapeo garantizado en terminfo `linux`), `█ ░ ▒`, y modo ASCII. Ver ADR §Glifos. |
| "Hay 16 colores" | En modo texto VGA una fuente de 512 glifos reduce el primer plano a 8 colores (`console-setup(5)`); con framebuffer (el caso HDMI moderno) no. | Diseñar para **8 colores + atributos**; el brillo (`A_BOLD`) es decorativo, nunca portador de significado. |
| "La vista principal debe ser la densa tipo bashtop (C)" | El Owner validó maquetas. | Owner 2026-10-16 17:45: **A** es la principal (sirve standalone y clúster) y debe incluir las máquinas que más consumen CPU y memoria; **B** es la vista de clúster con nodos resumidos; **C se descarta**. |
| "`netin/netout` de los guests son tráfico del host" | Son bytes acumulados por guest desde su arranque (`/cluster/resources`, esquema oficial). | La red del host sale de `/proc/net/dev` (NIC físicas); las tasas por guest se derivan entre snapshots y sólo se muestran en pantallas anchas. |
| "Instalar activa la TUI de inmediato" | Si el instalador reinicia `getty@tty1` con una sesión abierta en tty1, mata al usuario que está instalando desde la consola. | Activación inmediata **sólo si tty1 tiene un `agetty` ocioso sin sesión**; si no, en el siguiente arranque. Nunca `chvt`. |
| "Desinstalar ProxMenux basta con `rm -rf $BASE_DIR`" | `config_menu.sh:821` borra `$BASE_DIR`; si el drop-in de getty apuntara ahí, tty1 entraría en bucle de reinicio y **se perdería el login**. | El wrapper vive fuera de `$BASE_DIR` (`/usr/local/sbin`) y, sin la TUI, ejecuta `agetty` igual; la desinstalación retira el drop-in y recarga systemd **antes** de borrar lo demás. |

## 5. Restricciones duras (con números)

### Refresco
- Métricas locales (CPU, RAM, carga, red, disco, temperatura, zram): muestreo y repintado **cada 1 s** (btop usa 2000 ms por defecto; 1 s es el tick de `/proc/stat` y mantiene la sensación "en vivo" sin coste apreciable).
- Snapshot del Monitor (`console.json`): publicación **cada 5 s** mientras haya un consumidor (alineado con `_PVESH_CACHE_TTL = 5`), con `/cluster/status` cada 10 s. Sin consumidor: **0 llamadas a `pvesh`**, sólo un `stat()` cada 2 s.
- Historial: ventana de **60 muestras** de CPU total (60 s) en memoria de la TUI; nada persiste en disco.
- Repintado completo forzado **cada 30 s** (además de `r` y de los cambios de tamaño): `tty1` recibe los `printk` del kernel y curses sólo repinta diferencias.

### Presupuesto de recursos en el nodo
- TUI: **≤ 2 % de un núcleo** de promedio a 240×67 y **≤ 40 MiB RSS**; primer frame en **≤ 1,5 s** desde el exec.
- Publicador en el Monitor: **≤ 1 `pvesh get /cluster/resources` por 5 s + 1 `/cluster/status` por 10 s** mientras la TUI esté viva; JSON **≤ 1 MiB** (tope de 2000 guests, marcado `truncated`).
- Ninguna llamada a `smartctl`, `ipmitool`, `sensors`, `rrddata` ni a los endpoints `/nodes/{node}/status` remotos por la TUI: un dato que no esté en `/proc`, `/sys` o en `/cluster/resources` + `/cluster/status` se muestra como `n/d`.

### Pantalla
- Tamaño mínimo **80×24**; por debajo se muestra sólo "Terminal too small (80×24)" y las teclas siguen funcionando.
- Tamaños de diseño: 80×24/25 (VGA texto), 128×48 (1024×768, fuente 8×16), 240×67 (1920×1080, fuente 8×16). El layout se recalcula en cada `KEY_RESIZE`.
- 8 colores + `A_REVERSE`/`A_BOLD`/`A_DIM`; sin truecolor, sin 256 colores, sin ratón.
- Glifos: por defecto `ACS_*` de curses y `█ ░ ▒` (todos presentes en CP437 y en el conjunto `linux.set` de console-setup); opción `unicode` añade `▄ ▀`; `ascii` usa `# . - | +`. Sin braille ni bloques parciales U+2581–2587.

### Clúster y datos
- Diseñado para **≤ 16 nodos y ≤ 1000 guests** (los nodos que no quepan se resumen como "+N nodes"); el snapshot admite hasta 2000 guests.
- Umbrales de antigüedad: muestra local > 3 s → `STALE`; snapshot ≤ 15 s fresco; 15–60 s `stale` (ámbar, con edad); > 60 s o ausente → `Monitor unavailable` (rojo) manteniendo los últimos valores atenuados.
- Ventana de arranque: sin snapshot y `/proc/uptime` < **120 s**, o snapshot con estado `starting`, se muestra `Monitor starting…` en ámbar, nunca el rojo de caída; el publicador escribe un snapshot `starting` nada más arrancar y el primero con datos en cuanto ve el lease.
- Consumo del host fuera de los guests: indicador `host other` (CPU en núcleos, memoria en GiB) en los títulos de las listas TOP, calculado como total local menos suma de guests (y menos ARC en memoria).
- Agregados de clúster sólo con quorum y con datos frescos de todos los nodos online; de lo contrario se muestra cobertura (`2/3 nodes`), nunca un total aparentemente completo.
- Sin quorum: panel de clúster en rojo `NO QUORUM`, nodos offline marcados; la TUI nunca escribe en `/etc/pve` ni intenta reparar nada.

### Ciclo de vida
- Autoarranque **una vez por boot**, sólo en `tty1`, sólo si `/dev/tty0` existe (condición propia de `getty@`). Salida con `q` o `Ctrl+C` → login en **< 2 s**. Un fallo de la TUI (traceback, falta de Python, snapshot corrupto) → login igual.
- Un reinicio del getty tras logout **no** vuelve a mostrar la TUI; se relanza con `proxmenux-console` desde cualquier shell.
- `tty2–tty6`, `serial-getty@*`, SSH, modo rescate y `autovt@` no se tocan.

### Seguridad
- La TUI corre como usuario de sistema dedicado (`proxmenux-console`, sin shell, sin home, sin sudo, sin pertenencia a `www-data`), con `setpriv --no-new-privs --reset-env`.
- Sólo lectura: ningún comando de administración, ningún shell, ninguna escritura fuera de `/run/proxmenux-console/client/`.
- Nombres y tags de guests son **no confiables**: se filtran a imprimibles ASCII y se truncan por celdas antes de llegar a curses.
- Artefactos de arranque (drop-in, wrapper, marcador, configuración) son root-owned; el usuario de la TUI no puede alterar la política de arranque.

## 6. Fuera de alcance (explícito)

- Acciones administrativas (start/stop/migrar guests, apagar nodo, shell embebida).
- Lista de procesos del host al estilo bashtop (tecla `p`): el Owner pidió VM/CT; leer `/proc/*/stat` de ~1000 PIDs cada segundo es la parte cara de bashtop y mostraría nombres de procesos root antes del login. La necesidad real (saber si el nodo está saturado por algo que no es un guest) se cubre con el indicador `host other`.
- Ejecutar comandos desde la TUI (`systemctl`, `pvesh`, `sensors`): todo lo que necesita se lee de `/proc`, `/sys` y el snapshot.
- Ratón, truecolor, 256 colores, fuentes personalizadas, cambio de VT (`chvt`), autologin.
- Históricos más allá de 60 s en memoria; nada de RRD ni persistencia.
- Métricas remotas por nodo que exijan llamadas adicionales (`rrddata`, `/nodes/{node}/status`, red/zram/temperatura de nodos remotos): la fila por nodo de la vista B usa sólo `/cluster/resources` + `/cluster/status`.
- Temperaturas vía `ipmitool`/`sensors`; sólo `hwmon`/`thermal` de `/sys`.
- SMART, uso interno de discos de VM sin guest agent, almacenamiento compartido deduplicado por capacidad (se marca `shared`, no se agrega).
- Soporte de consola serial o `TERM` distinto de `linux` como objetivo de diseño (funciona si curses tiene terminfo, pero no se valida).
- Idiomas con alfabetos no latinos en la consola (la fuente no los tiene): se translitera a ASCII cuando faltan glifos.
- Cambiar el comportamiento del dashboard web o sus endpoints.

## 7. Criterio de no construir

No se construye (o se detiene) si en la validación en hardware del Owner ocurre cualquiera de estas cosas y no tiene arreglo acotado:

1. Tras `q`, el `login:` tarda > 5 s o reaparece la TUI (bucle de getty).
2. Con el servicio Monitor parado, la consola no muestra CPU/RAM locales en vivo.
3. El publicador del snapshot añade > 1 llamada `pvesh` cada 5 s mientras la TUI corre, o sigue llamando a `pvesh` cuando no hay TUI.
4. Al desinstalar ProxMenux, tty1 queda sin login tras reiniciar.
5. Los glifos por defecto se ven como cuadros de sustitución en la consola de los P3 y el modo ASCII no lo corrige.

## 8. Preguntas abiertas para el Owner (cada una con default para no bloquear)

| # | Pregunta | Default recomendado | Por qué |
|---|---|---|---|
| Q1 | ¿El autoarranque en tty1 queda **activado por defecto** para todos los usuarios upstream que instalan Monitor, o se pregunta en el instalador? | **Preguntar en el instalador con un `--yesno` cuyo valor por defecto es Sí**; instalaciones no interactivas → Sí. La respuesta se persiste en `/etc/proxmenux/console.conf` y las actualizaciones la respetan. | Para el Owner equivale a "activado"; para upstream, cambiar la TTY de todo el mundo sin preguntar reduce la probabilidad de aceptación del PR. |
| Q2 | ¿Mostrar **nombres de guests y tags** antes del login (los ve cualquiera con acceso físico)? | **Sí**, con opción `PRIVACY=ids` en `console.conf` para mostrar sólo VMID y tipo. | El acceso físico ya expone la consola de arranque y el teclado; el Owner pidió "las máquinas que más consumen", que sin nombre pierde utilidad. |
| Q3 | Orden de "top CPU": ¿por **núcleos de host consumidos** (`cpu × maxcpu`) o por % de lo asignado? | **Núcleos de host**, mostrando ambos (`1.2c 60%`). | Es lo que afecta al nodo; un CT con 1 vCPU al 100 % pesa menos que una VM de 8 al 40 %. |
| Q4 | ¿Idioma de la consola = idioma configurado en ProxMenux (`config.json .language`), con transliteración ASCII para glifos que la fuente no tiene? | **Sí.** | Coherente con menús y Monitor; el `getty` elimina `LANG`, así que no hay otra fuente razonable. |
| Q5 | ¿El relanzamiento manual exige root (`setpriv` al usuario dedicado) o se permite a cualquier usuario del grupo `proxmenux-console`? | **Root o miembro del grupo**; el lanzador decide. | En PVE casi todo se opera como root; abrir el grupo cuesta nada y no amplía privilegios. |

Ninguna de estas preguntas bloquea la implementación con los defaults indicados.

## 9. Fuentes consultadas y qué aportaron

- **Memoria del proyecto** (`.ai-memory` vectorial, consulta "monitor console tui"): devolvió commits de i18n de la "TUI" bash (`whiptail/dialog`) y el `tests/test_tui_count_messages.py`; útil para confirmar que en este repo "TUI" ya designa los menús bash —de ahí el nombre **console** para esta función— y para adoptar el patrón de pruebas bash con PATH restringido. Nada sobre getty/curses.
- **Second Brain** (`sb query "consola tui getty curses" --research biblioteca`): nada útil (resultados sobre HUD de Claude Code, un libro de fundraising y notas de red). No se usó como base.
- **Hilo del Owner** (`owner-thread.md`): requisitos confirmados (§4), elección de A como vista principal con top guests y B como vista de clúster, descarte de C, autorización de ejecución.
- **Diseño previo (Astra, 2026-10-14)**: base de la separación recolector/renderizador, del marcador por boot y de la sanitización. Se conserva y se corrige en el ADR (§Qué se rechazó).
- **systemd `units/getty@.service.in` v252 y v257** y **Debian 257.13-1~deb13u1** (sources.debian.org): `ExecStart` exacto en cada versión (`-o '-p -- \\u' --noclear - $TERM` en 252; `-o '-- \\u' --noreset --noclear - ${TERM}` en 257), `Restart=always`, `RestartSec=0`, `Type=idle`, `TTYVTDisallocate=yes`, `UnsetEnvironment=LANG…`. Determina el diseño del wrapper y del marcador.
- **systemd.service(5)** (man7): la cadena vacía `ExecStart=` reinicia la lista (patrón del drop-in), `Type=idle` retrasa ≤ 5 s, prefijo `-` convierte fallos en éxito, un sólo `ExecStart` salvo `oneshot`. Justifica el drop-in de dos líneas y descarta `ExecStartPre`.
- **agetty(8)**: semántica de `-o … \u` con `--` (aviso de seguridad), `--noclear`, `--noreset`, `-` como puerto ya conectado a stdin, `TERM` por defecto `linux` en VT.
- **Proxmox API (`apidoc.js` oficial, descargado y parseado)**: campos y descripciones literales de `/cluster/resources` (`cpu` "CPU utilization", `maxcpu`, `mem`/`maxmem` en bytes, `netin/netout`/`diskread/diskwrite` acumulados desde el arranque del guest, `uptime`, `status`, `tags`, `hastate`, `lock`, `shared`, `level`), de `/cluster/status` (`quorate`, `nodes`, `online`, `local`, `nodeid`, `ip`, `version`; permiso `Sys.Audit`) y de `/nodes/{node}/status` (`loadavg`, `memory`, `rootfs`, `pveversion`). Define el contrato del snapshot.
- **pvesh(1)**: "Only root is allowed to do that" → el recolector debe ser root (el Monitor ya lo es) y la TUI no.
- **packages.debian.org**: `libpython3.13-stdlib` (trixie) y `libpython3.11-stdlib` (bookworm) incluyen `lib-dynload/_curses*.so` y `curses/`; `ncurses-base` incluye `/usr/share/terminfo/l/linux`. Conclusión: no hay que empaquetar nada para curses.
- **console-setup 1.242~deb13u1** (`Fonts/fontsets/` → `Lat15.256`, `Uni2.512`…; `Fonts/linux.set`; `Fonts/useful.set`) y **console-setup(5)**: el conjunto ACS de la consola mapea 13 puntos de caja + `2588/2591/2592`; `useful.set` añade `2580/2584/258C/2590/2593` y dobles sólo "si quedan posiciones libres"; sin braille ni 2581–2587; las fuentes de 512 glifos reducen a 8 colores en modo texto. Define el juego de glifos y la paleta.
- **Wikipedia, Code page 437**: confirma caja simple/doble y `▀ ▄ █ ▌ ▐ ░ ▒ ▓` en el font VGA del kernel; sin bloques parciales ni braille.
- **btop README**: `update_ms = 2000` por defecto; modo TTY de 16 colores con símbolos `tty`; `graph_symbol` braille/block/tty. Referencia de convenciones de refresco y de degradación en TTY.
- **kernel fbcon**: fuente por defecto 8×16 (`VGA8x16`); no fija columnas × filas, se derivan de resolución/fuente (1920/8 × 1080/16 = 240×67; 1024/8 × 768/16 = 128×48).
- **Repositorio (worktree)**: `AppRun`, `build_appimage.sh:60-200`, `flask_server.py` (`:122-154`, `:1758-1800`, `:2495-2510`, `:2759-2800`, `:5870-5900`), `zram_metrics.py`, `install_proxmenux.sh:43-64,310-330,528-692,813`, `install_proxmenux_beta.sh:415,524`, `scripts/menus/config_menu.sh:55-140,403-440,578-612,814-821`, `scripts/utils.sh:228-277`, `notification_templates.py:25-73`, `.github/workflows/test-lvm-disk-policy.yml`, `test-offline-qualified.yml`, `.github/scripts/run_offline_qualified.py`, `build_translation_cache.py:285-345`, `tests/test_tui_count_messages.py`, `AppImage/scripts/tests/test_zram_metrics.py`, `web/app/[locale]/docs/settings/proxmenux-monitor/page.tsx`.
