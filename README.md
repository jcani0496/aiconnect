# aiconnect

Remote-control layer for Claude Code and Codex CLI sessions running on your Macs, accessible as a PWA from your phone over Tailscale.

**Status:** Fase 1 · planificado y listo para ejecutar.

## Qué resuelve

Three pain points:

1. **Unificación** — un único inbox para conversaciones Claude + Codex en tus Macs.
2. **Organización por proyecto** — agrupación lógica con tags y filtros que cruzan máquinas y CLIs.
3. **Acceso al código local desde el móvil** — los prompts desde el teléfono ejecutan en la Mac apropiada con acceso nativo al repo.

Auth 100% vía suscripción (Claude Max · ChatGPT Pro). Sin API tokens.

## Arquitectura

```
[PWA móvil/desktop]  ←→  [Tailscale]  ←→  [Agente en cada Mac]
                                                │
                                         - Envuelve claude / codex CLI
                                         - WebSocket bindeado al tailnet
                                         - SQLite local (FTS5)
                                         - LaunchAgent auto-start
```

Datos 100% en tus Macs. Sin backend central, sin cuentas, sin residuo en servicios externos.

## Documentación

- **Spec de diseño** — [`docs/superpowers/specs/2026-04-18-aiconnect-design.md`](docs/superpowers/specs/2026-04-18-aiconnect-design.md)
- **Plan 1 · Agente macOS + pareo** — [`docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan1-agente.md`](docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan1-agente.md)
- **Plan 2 · PWA core + E2E** — [`docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan2-pwa.md`](docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan2-pwa.md)
- **Plan 3 · Extras mobile-first** — [`docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan3-extras.md`](docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan3-extras.md)
- **Guía de ejecución en Mac** — [`docs/superpowers/EXECUTION.md`](docs/superpowers/EXECUTION.md)

## Estado

- [x] Brainstorming + spec cerrado
- [x] Mockups completos (móvil 26 pantallas + web 6 pantallas)
- [x] Plan 1 (Agente) — 29 tareas
- [x] Plan 2 (PWA) — 22 tareas
- [x] Plan 3 (Extras) — 11 tareas
- [ ] Plan 1 ejecutado
- [ ] Plan 2 ejecutado
- [ ] Plan 3 ejecutado
- [ ] Dogfood 1 semana

## Próximo paso

Clona este repo en tu Mac y sigue [`docs/superpowers/EXECUTION.md`](docs/superpowers/EXECUTION.md).
