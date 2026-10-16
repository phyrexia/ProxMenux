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

- [ ] P1-T1 `console_metrics.py` + `host other` (principal, Codex gpt-6-sol) — despachada
- [ ] P1-T2 `console_snapshot.py` (principal, Codex gpt-6-sol) — despachada
- [ ] P1-T3 caché compartida `pvesh` en `flask_server.py` (principal, Codex gpt-6-sol) — despachada
- [ ] P1-T4 `console_render.py` (principal, Codex gpt-6-sol) — despachada
- [ ] P1-T6 i18n `runtime.consoleTui` en `en/common.json` (light, Haiku) — despachada
- [ ] P1-T5 `console_tui.py` (principal) — tras T1, T2, T4
- [ ] P1-T7 hook del publicador en `flask_server.py` (light) — tras T2, T3
- [ ] P1-T8 `AppRun --tui` + `build_appimage.sh` + `AppImage/README.md` (light) — tras T5
- [ ] P1-T9 comando `proxmenux-console` root-only + instaladores + `config_menu.sh` (light) — tras T8
- [ ] P1-T10 CI `test-console-tui.yml` (light)
- [ ] P1-T11 docs web + CHANGELOG + README (light)

## DAG PR2 — `feature/console-tui-autostart` (tras mergear PR1)

- [ ] P2-T1 wrapper getty · P2-T2 `console_lifecycle.sh` · P2-T3 instaladores · P2-T4 lanzador con `setpriv` · P2-T5 menú · P2-T6 CI · P2-T7 docs

## Bloqueos / decisiones del Owner

- [ ] QA remota: `qf-test` rechaza por nombre `web/.npmrc` (versionado upstream). El clasificador de permisos bloqueó sacarlo del índice durante el snapshot. Opciones: (a) el Owner autoriza esa maniobra para esta serie, (b) pedido al dueño de qf-test de una exclusión por ruta (ver AgentBus #14807, falsos positivos similares).
- [ ] ¿Empujar `notes/console-tui` al fork público o dejarla local?
- [ ] Preguntas abiertas del grill (Q1–Q5): se avanza con los defaults recomendados salvo indicación.
