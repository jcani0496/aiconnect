# Guía de ejecución · Fase 1

Esta guía asume que vas a ejecutar los planes en tu Mac porque el agente es macOS-only. La planificación se hizo en Windows; la ejecución corre en Mac.

## Pre-requisitos en tu Mac

Antes de arrancar la ejecución de Plan 1:

### 1. Tailscale

```bash
brew install tailscale
brew services start tailscale   # o sudo tailscaled
sudo tailscale up
tailscale ip -4                 # verifica que obtengas un 100.x.x.x
```

Instala Tailscale también en tu móvil y verifica que ambos dispositivos estén en el mismo tailnet.

### 2. Claude Code CLI

```bash
brew install anthropic/claude/claude   # o el método oficial vigente
claude --version
claude                                  # una sola vez para autenticar con tu Max plan
```

### 3. Bun

```bash
curl -fsSL https://bun.sh/install | bash
# Reinicia el terminal para que el PATH tome efecto
bun --version                           # debe reportar ≥ 1.1
```

### 4. Mover el repo a tu Mac

Clonar o transferir este directorio a tu Mac. Recomendado:

```bash
# En Windows (tu máquina actual), crea un bundle git del repo ya inicializado:
cd D:/PROYECTOS/aiconnect
git bundle create aiconnect.bundle --all

# Cópialo a la Mac (scp, iCloud, USB, lo que prefieras)

# En la Mac, clona desde el bundle:
git clone aiconnect.bundle aiconnect
cd aiconnect
```

Alternativa simple: crea un repo vacío en GitHub, `git remote add origin <url>` y push, luego clone desde la Mac.

## Ejecución de Plan 1 con subagentes

Tienes dos formas:

### Forma 1 · Claude Code en tu Mac (recomendado)

Si tienes `claude` CLI en la Mac (lo necesitas para el runtime del agente de todos modos), abre una sesión de Claude Code en el directorio del repo:

```bash
cd ~/aiconnect
claude
```

Dentro de la sesión, dile:

> "Quiero ejecutar `docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan1-agente.md` usando subagent-driven-development. Empieza con Task 1."

Claude leerá el plan y arrancará el ciclo: dispatcha subagente implementer → spec reviewer → code quality reviewer, tarea por tarea. Revisas al final de cada tarea antes de que avance a la siguiente.

### Forma 2 · Ejecución manual tarea por tarea

Si prefieres control total, ejecuta las tareas tú mismo siguiendo el plan. Cada tarea tiene pasos explícitos con código completo (TDD: test → red → impl → green → commit). Tiempo estimado: ~2 semanas con atención plena.

## Orden de ejecución de los 3 planes

```
Plan 1 (Agente)  ──▶  Plan 2 (PWA)  ──▶  Plan 3 (Extras)
    ~2 sem              ~1.5 sem           ~4 días
```

**Dependencia crítica entre Plan 1 y Plan 2:** el protocolo WebSocket (Zod schemas en `agent/src/protocol/messages.ts`). Una vez congelado, Plan 2 puede arrancar en paralelo.

**Plan 3 depende de Plan 2** (amplía la PWA).

**Protocol v2 de Plan 3** toca ambos repos (schema attachments) — hazlo en un commit coordinado.

## Criterio de "Fase 1 terminada"

Estás usando aiconnect en trabajo real durante una semana sin pelearte con él. Si algún bug te frustra, arréglalo antes de avanzar a Fase 2.

## Si algo falla

- **`tailscale` no detecta IP:** `sudo tailscale status`, verifica que el daemon esté corriendo.
- **`claude` no autentica:** abre una sesión interactiva manual primero (`claude` sin flags), completa el OAuth, luego el agente hereda esa auth.
- **Bun no encuentra módulos:** `bun install` en el directorio `agent/` o `pwa/` respectivamente.
- **LaunchAgent no arranca al login:** revisa `launchctl list | grep aiconnect` y el plist en `~/Library/LaunchAgents/`.

## Después de Fase 1

Fase 2 — Codex CLI, multi-máquina, search UI, tags completos, plan mode, @mentions, aiconnect memory, push actionable. Cada feature merece su propio spec + plan cuando corresponda.

Fase 3 — Skills Manager (requiere su propio brainstorming por complejidad de seguridad), búsqueda semántica, collaboration, cost tracking, Watch/Shortcuts.
