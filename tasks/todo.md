# Console TUI (tty1 HDMI/VGA) — ProxMenux Monitor

Perfil: feature · ADR: `docs/adrs/001-console-tui.md` · Grill: `docs/specs/2026-10-17-console-tui-grill.md`
Code branch: `feature/console-tui` (from `origin/develop` 227780ab) · Notes branch: `notes/console-tui` (this one, local only until the Owner says otherwise — the fork is public)

## Estado al 2026-10-17

- El despacho del 2026-10-16 a Orca en qf-infra nunca arrancó (Codex se detuvo a las 18:05 leyendo `orca --help`); sin worktree, rama ni PR. Se retoma aquí.

## Fases

- [x] 1. Concepción — grill escrito con requisitos del Owner (hilo 2026-10-14/16)
- [x] 2. Diseño — ADR 001 `proposed` (Fable)
- [ ] 3. Auditoría adversarial del ADR (Gemini 3.8 Flash vía `agy`) → recibo `docs/analysis/2026-10-17-console-tui-redteam-verdict.md`; Critical/Important corregidos; ADR `accepted`
- [ ] 4. Implementación (DAG abajo), TDD, un commit por tarea
- [ ] 5. QA remota (qf-test) + auditoría del diff + PR a `MacRimi/ProxMenux` `develop` desde el fork, sin atribución IA

## DAG de implementación (ADR §2.9)

- [ ] T1 `console_metrics.py` — lectores `/proc`/`/sys` + tasas (principal)
- [ ] T2 `console_snapshot.py` — contrato, normalizador, estados, escritura atómica, lease, lector (principal)
- [ ] T4 `console_render.py` — render puro a `Grid`, vistas A/B, glifos, colapso, `sanitize` (principal)
- [ ] T6 catálogo i18n `runtime.consoleTui.*` en `AppImage/messages/en/common.json` (light)
- [ ] T8 wrapper getty + lanzador manual en `scripts/console/` (principal)
- [ ] T3 publicador en `flask_server.py` (light) — tras T2
- [ ] T5 `console_tui.py` — bucle curses, teclas, i18n, señales (principal) — tras T1, T2, T4
- [ ] T9 instaladores `install_console_tui` / `remove_console_tui` (principal) — tras T8
- [ ] T7 empaquetado `AppRun --tui`, `build_appimage.sh` (light) — tras T5
- [ ] T10 menú `config_menu.sh` + desinstalación (light) — tras T9
- [ ] T11 CI `test-console-tui.yml` (light)
- [ ] T12 docs web + CHANGELOG + README (light)

## Bloqueos / decisiones del Owner

- [ ] QA remota: `qf-test` rechaza por nombre `web/.npmrc` (versionado upstream). El clasificador de permisos bloqueó sacarlo del índice durante el snapshot. Opciones: (a) el Owner autoriza esa maniobra para esta serie, (b) pedido al dueño de qf-test de una exclusión por ruta (ver AgentBus #14807, falsos positivos similares).
- [ ] ¿Empujar `notes/console-tui` al fork público o dejarla local?
- [ ] Preguntas abiertas del grill (Q1–Q5): se avanza con los defaults recomendados salvo indicación.
