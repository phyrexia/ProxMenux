# Console TUI (tty1 HDMI/VGA) — ProxMenux Monitor

Perfil: feature · ADR: `docs/adrs/001-console-tui.md` · Grill: `docs/specs/2026-10-17-console-tui-grill.md`
Code branch: `feature/console-tui` (from `origin/develop` 227780ab) · Notes branch: `notes/console-tui` (this one, local only until the Owner says otherwise — the fork is public)

## Estado al 2026-10-17

- El despacho del 2026-10-16 a Orca en qf-infra nunca arrancó (Codex se detuvo a las 18:05 leyendo `orca --help`); sin worktree, rama ni PR. Se retoma aquí.

## Fases

- [x] 1. Concepción — grill escrito con requisitos del Owner (hilo 2026-10-14/16)
- [x] 2. Diseño — ADR 001 `proposed` (Fable)
- [x] 3. Auditoría adversarial del ADR — ronda 1 APROBAR CON CAMBIOS (1C/5I/4m), revisión 2 adjudicada, ronda 2 APROBAR 0/0/0 → recibo `docs/analysis/2026-10-17-console-tui-redteam-verdict.md`; ADR `accepted` (617441ec)
- [ ] 4. Implementación (DAG abajo), TDD, un commit por tarea
- [ ] 5. QA remota (qf-test) + auditoría del diff + PR a `MacRimi/ProxMenux` `develop` desde el fork, sin atribución IA

Entrega en dos PR (ADR rev. 2 §2.9): PR1 TUI manual por root; PR2 autoarranque en tty1 (rama `feature/console-tui-autostart` desde `develop` tras mergear PR1).

## DAG PR1 — `feature/console-tui`

- [x] P1-T6 i18n `runtime.consoleTui` — 757ce219 (Haiku)
- [x] P1-T3 caché compartida `pvesh` — 149eb2e6 (Codex gpt-6-sol)
- [x] P1-T1 `console_metrics.py` + `host other` — f5120d36 (Codex gpt-6-sol)
- [x] P1-T2 `console_snapshot.py` — 8b5277f6 (Codex gpt-6-sol + ronda de corrección Haiku 5.5: publicador nunca tumba el Monitor, versión desde el caller, C1)
- [x] P1-T7 hook del publicador — 4cd35bcf (Haiku 5.5; integración: reusa `get_proxmox_version`)
- [x] P1-T4 `console_render.py` — 04b26b43 (Codex gpt-6-sol + ronda de corrección: placeholders, `_age`, `host_other` del ViewModel, `PVE x.y`, claves de columnas, tests con catálogo real)
- [x] P1-T5 `console_tui.py` — 55eec7ff (Codex gpt-6-sol + ronda Haiku 5.5: ruta de catálogos empaquetados `parents[2]`, `setlocale`, muestreo monotónico)
- [x] P1-T8 `AppRun --tui` + build + `AppImage/README.md` — 1b819d67 (Haiku 5.5)
- [x] P1-T9 comando `proxmenux-console` + instaladores + menú — 31e2e6cf (Haiku 5.5)
- [x] P1-T10 CI `test-console-tui.yml` — 65127237 (Haiku 5.5)
- [x] P1-T11 docs — absorbida: el upstream no recibe CHANGELOG/README/docs web de contribuidores (sólo MacRimi los edita); la documentación va en `AppImage/README.md` (T8) y en el cuerpo del PR, ofreciendo la página de Settings
- [x] Fase 5 PR1 — auditoría del diff: ronda 1 sobre 65127237 APROBAR CON CAMBIOS (0C/4I/7m); fix 8ae1a74a + 5c7a8ab0; ronda 2 APROBAR 0/0/0 → recibo `docs/analysis/2026-10-17-console-tui-pr1-diff-verdict.md` (vigente sobre 5c7a8ab0)
- [x] Escaneo de credenciales del diff (TruffleHog estático): 0 candidatos; pedido AgentBus #17938 resuelto (el corte del autoreview es el defecto de base de #16699)
- [x] Rama `feature/console-tui` empujada al fork `phyrexia/ProxMenux` (sin PR)
- [ ] **QA remota PR1** (bloqueada, decisión del Owner por `web/.npmrc`): `unittest` de los 8 archivos de prueba en 3.11/3.13, smoke de pty, `bash -n` de scripts, build del AppImage y `AppRun --tui --once`
- [ ] Aceptación en hardware (Owner, paso 0 de ADR §2.8): `proxmenux-console` por SSH en un P3 con el AppImage de la rama
- [ ] Abrir PR1 a `MacRimi/ProxMenux` `develop` (sin atribución IA; cuerpo con evidencia, rollback y oferta de la página de docs) — tras QA verde
- [ ] PR2 (autoarranque tty1) — tras mergear PR1

## Revisión (2026-10-17)

- Lo que funcionó: diseño de Fable con fuentes primarias (vt.c, ncurses, Cluster.pm) que resistió dos rondas de auditoría; partir el DAG en archivos sin solape permitió 5 implementadores en paralelo sin conflictos.
- Lo que la revisión del coordinador atrapó y la auditoría no: placeholders del catálogo duplicados por el render, ruta de catálogos empaquetados (`parents[1]` vs `[2]`), falta de `setlocale`, publicador que podía tumbar el Monitor, orden de commits (T9 antes que T8 habría dejado `AppRun --tui` arrancando Flask).
- Ruido de herramientas: la inicialización de `.ai-memory` de cada sesión Codex duplica un bloque en `.gitignore` del worktree (revertido dos veces); `agy` headless deja la salida vacía si el modelo intenta escribir un archivo (se resolvió pidiéndole responder sólo en texto).

## DAG PR2 — `feature/console-tui-autostart` (tras mergear PR1)

- [ ] P2-T1 wrapper getty · P2-T2 `console_lifecycle.sh` · P2-T3 instaladores · P2-T4 lanzador con `setpriv` · P2-T5 menú · P2-T6 CI · P2-T7 docs

## Bloqueos / decisiones del Owner

- [ ] QA remota: `qf-test` rechaza por nombre `web/.npmrc` (versionado upstream). El clasificador de permisos bloqueó sacarlo del índice durante el snapshot. Opciones: (a) el Owner autoriza esa maniobra para esta serie, (b) pedido al dueño de qf-test de una exclusión por ruta (ver AgentBus #14807, falsos positivos similares).
- [ ] ¿Empujar `notes/console-tui` al fork público o dejarla local?
- [ ] Preguntas abiertas del grill (Q1–Q5): se avanza con los defaults recomendados salvo indicación.
