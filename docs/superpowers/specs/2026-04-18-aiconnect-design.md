# aiconnect — Design Spec

**Date:** 2026-04-18
**Status:** Approved for implementation planning (Fase 1 / MVP)
**Owner:** jcani
**Revision:** 2 — añadido model switching, voice input, image attachment, quick prompts (Fase 1); plan mode, @mentions, aiconnect memory, cross-CLI handoff (Fase 2); Skills Manager (Fase 3); sección de Design System.

## 1. Propósito

aiconnect es una capa de control remoto para sesiones de **Claude Code CLI** y **Codex CLI** que corren en Macs del usuario, accesible desde un **PWA móvil**. Permite continuar conversaciones de coding, revisar output y aprobar acciones sin estar físicamente frente a la Mac.

**El dolor que resuelve (los tres puntos confirmados por el usuario):**

1. **Unificación** — un único inbox para conversaciones Claude + Codex.
2. **Organización por proyecto** — agrupación lógica con tags y filtros que cruzan máquinas y CLIs.
3. **Acceso al código local desde el móvil** — los prompts enviados desde el teléfono ejecutan en la Mac apropiada, con acceso nativo al repo.

**Restricción clave:** autenticación exclusivamente vía suscripciones Claude Max y ChatGPT Pro (sin API tokens). Los CLIs oficiales ya autentican así; aiconnect los envuelve sin tocar credenciales.

**Identidad de producto:** herramienta para developers que valoran el control de su infra, la privacidad y una experiencia técnica cuidada. No busca ser consumer-friendly en el sentido masivo; busca ser la herramienta que un developer escoge deliberadamente.

## 2. No-goals

- **No espeja ni importa** conversaciones preexistentes de Claude.ai Desktop o ChatGPT/Codex Desktop — esas APIs internas no son accesibles sin violar TOS. aiconnect es para conversaciones **nuevas** que ocurren a través de él.
- **No resuelve migración de contexto entre Macs.** Un proyecto vive en una máquina; si el mismo repo existe en dos Macs son dos proyectos distintos.
- **No controla apps GUI** (Claude.ai Desktop, ChatGPT Desktop). Solo envuelve los CLIs oficiales.
- **No revende tokens de IA.** El usuario trae su suscripción.
- **No tiene backend central** en Fase 1 (quizás sí en Nivel 2/3 de distribución futura).

## 3. Arquitectura a alto nivel

Tres componentes:

### 3.1. Agente (daemon en cada Mac)

- Proceso background lanzado al login vía LaunchAgent
- Envuelve `claude` y `codex` como subprocesos vivos (uno por conversación activa)
- Expone API HTTP + WebSocket, bindeada exclusivamente a la IP del tailnet (`100.x.x.x`)
- Persiste conversaciones, mensajes, eventos en SQLite local con FTS5 built-in
- Distribuido como `brew tap jcani/aiconnect` para Fase 1

### 3.2. PWA (frontend móvil)

- Instalable en iOS/Android vía "Add to Home Screen"
- Se conecta directo a cada agente vía Tailscale MagicDNS (`mac-a.tailnet.ts.net`)
- Agrega en cliente el historial de múltiples agentes para dar inbox unificado
- Hospedada como sitio estático en un dominio público (ej: `aiconnect.jcani.dev`)

### 3.3. Tailscale (infraestructura externa)

- Resuelve NAT, identidad entre dispositivos, y cifrado E2E
- El usuario instala cliente Tailscale en cada Mac y en el móvil
- aiconnect nunca toca credenciales de red ni opera como servicio de red

**Consecuencias explícitas de este diseño:**

- Datos 100% en las Macs del usuario. Sin VPS, sin nube, sin terceros almacenando conversaciones.
- Mac apagada → sus conversaciones no visibles hasta que regrese. Trade-off aceptado.
- Sin costo mensual (Tailscale free tier basta para uso personal).
- Añadir una Mac nueva no requiere cambios en aiconnect — solo instalar el agente.

## 4. Modelo de datos

Cuatro entidades principales + modelos como concepto transversal.

### 4.1. Machine

Representa una Mac con agente instalado.

- `id` (UUID)
- `hostname` (Tailscale MagicDNS)
- `display_name` (user-defined, ej: "mac-trabajo")
- `last_seen`
- `fingerprint` (verificación de identidad del agente)

### 4.2. Project

Unidad de agrupación. Un proyecto vive en **una sola máquina**. Si el mismo repo está clonado en dos Macs, son dos proyectos independientes.

- `id` (UUID)
- `machine_id` (FK)
- `name`
- `working_dir` (path absoluto en esa Mac)
- `tags` (array, cruza máquinas lógicamente)
- `color` (opcional, UI)
- `permission_preset` (default: "ask-for-side-effects" | "auto-accept-edits" | "full-auto")
- `default_model_claude_code` (modelo preferido al crear conversaciones Claude — opcional)
- `default_model_codex` (modelo preferido al crear conversaciones Codex — opcional)
- `memory` (Fase 2: contexto persistente inyectado como system prompt prefix)

### 4.3. Conversation

- `id` (UUID)
- `project_id` (FK)
- `cli` (`claude-code` | `codex`) — **inmutable**, fijado al crear
- `current_model` — el modelo activo (puede cambiar mid-conversation)
- `title` (auto-generado o editable)
- `status` (`idle` | `running` | `awaiting_approval` | `error` | `interrupted`)
- `mode` (Fase 2: `active` | `plan-only` — si `plan-only`, el CLI corre sin permisos de escritura)
- `created_at`, `last_activity_at`

### 4.4. Message / Event

Cada turn se compone de eventos estructurados persistidos incrementalmente:

- `id` (ULID para orden temporal)
- `conversation_id` (FK)
- `type` (`user_prompt` | `assistant_text_delta` | `tool_use` | `tool_result` | `permission_request` | `permission_response` | `model_change` | `mode_change` | `error` | `turn_complete`)
- `payload` (JSON con contenido tipado según `type`)
- `attachments` (imágenes, audio — Fase 1 admite imágenes en `user_prompt`)
- `timestamp`
- Metadata indexada: archivos tocados, comandos ejecutados, approvals, modelo activo en el momento del evento

**FTS5:** índice full-text sobre contenido textual de mensajes. Habilitado desde día 1 aunque el UI de búsqueda llegue en Fase 2.

### 4.5. Model (concepto transversal)

Los modelos disponibles por CLI se descubren en runtime desde el agente (ejecutando `claude --list-models` o equivalente). El agente cachea la lista. En la PWA:

- **Al crear proyecto:** opcional fijar default por CLI
- **Al crear conversación:** override del default si aplica
- **Durante conversación:** tap en el badge de modelo del header → sheet con lista → selección → evento `model_change` registrado y `/model` enviado al CLI

**Razonamiento:** cambiar modelo mid-conversation es un caso de uso real (Sonnet para exploración rápida, Opus para razonamiento pesado). Debe ser fricción baja.

## 5. Flujo de datos

### 5.1. Envío de un prompt

```
1. Usuario escribe en PWA (o dicta por voz, o adjunta imagen), toca enviar
2. PWA → WebSocket → Agente de la Mac del proyecto
3. Agente localiza proceso CLI de la conversación (o lo arranca por primera vez)
   - cwd = working_dir del proyecto
   - CLI en modo stream-json (stdin/stdout estructurado)
   - CLI arrancado con `--model <current_model>` si aplica
4. Agente escribe prompt (+ attachments si hay) en stdin
5. CLI emite eventos JSON por stdout en vivo:
   - text_delta, tool_use, permission_request, tool_result, turn_complete
6. Agente persiste cada evento en SQLite + lo reenvía por WebSocket a PWA
7. PWA renderiza en tiempo real (markdown, diffs, tool indicators)
```

### 5.2. Approvals

- Modo default: **"ask-for-side-effects"** — pregunta antes de editar archivos, bash, network
- Cuando llega `permission_request`:
  - CLI queda bloqueado esperando respuesta
  - PWA muestra sheet modal + Push notification (Fase 2: con botones quick-approve/deny en la propia notificación)
  - Usuario aprueba/deniega (opcionalmente con nota)
  - Respuesta viaja de vuelta hasta stdin → CLI continúa
- Sin timeout por default — el pending queda indefinidamente hasta que usuario responda

### 5.3. Cambio de modelo mid-conversation

1. Usuario tap en badge de modelo en header
2. Sheet con modelos disponibles para el CLI activo
3. Selección → agente envía `/model <nombre>` al stdin del CLI
4. CLI confirma cambio; agente emite evento `model_change`
5. PWA refleja nuevo modelo en el header

### 5.4. Persistencia y reconexión

- **Proceso CLI vivo entre prompts** — no spawn por turn. Preserva contexto real del CLI sin depender de `--resume`.
- Eventos se escriben a SQLite **a medida que llegan**, no al final del turn.
- Si PWA pierde conexión, al reconectar:
  - Re-engancha WebSocket del agente
  - Solicita eventos `> last_received_id` desde SQLite
  - Retoma streaming en vivo

### 5.5. Sleep prevention

- Agente activa `caffeinate -i` mientras exista al menos una conversación activa
- Pantalla puede dormir; sistema no se suspende
- Sin conversaciones activas, comportamiento normal de sleep

## 6. Onboarding y seguridad

### 6.1. Instalación del agente

```bash
brew tap jcani/aiconnect
brew install aiconnect-agent
aiconnect init   # pide permisos, crea LaunchAgent, imprime QR de pareo
```

### 6.2. Pareo con PWA

1. Usuario abre PWA (primera vez)
2. Escanea QR del terminal (o tipea código de 6 dígitos)
3. PWA recibe: hostname Tailscale + token largo + fingerprint del agente
4. Datos guardados en storage del PWA; futuras sesiones no piden pareo

### 6.3. Capas de seguridad

- **Capa 1 — Tailscale:** agente bindeado solo a IP tailnet. Inaccesible desde fuera del tailnet del usuario.
- **Capa 2 — Token por dispositivo:** cada PWA paireada tiene su propio token. Revocable con `aiconnect agent revoke <token-id>`.
- **Capa 3 — Fingerprint:** PWA recuerda fingerprint del agente. Si cambia, pide reconfirmar.

### 6.4. Decisiones explícitas sobre lo que NO hay

- Sin OAuth / sin login de usuario en PWA. Dispositivo paireado + token + tailnet prueba identidad.
- Sin almacenamiento de credenciales Claude/Codex. El agente invoca los CLIs que ya autentican localmente.
- Sin cuentas, sin backend nuestro. `brew uninstall` + borrar PWA = residuo cero.

## 7. Manejo de errores

Principio rector: **ningún error pierde conversación.** SQLite se escribe incrementalmente.

| Falla | Comportamiento |
|---|---|
| Mac apagada o suspendida | PWA marca agente offline. Prompts tipeados se encolan si usuario insiste. |
| WiFi móvil cae mid-streaming | Reconexión automática, replay de eventos perdidos desde SQLite. |
| CLI crashea mid-turn | Turn marcado `error` con stderr capturado. Botón "reintentar desde último prompt". |
| Agente crashea | LaunchAgent reinicia. Conversaciones en curso marcadas `interrupted`. |
| Token revocado / fingerprint cambiado | PWA fuerza re-pareo. |
| Tailscale caído | PWA muestra "sin red aiconnect". Cache local visible read-only. |
| Approval pendiente sin respuesta | CLI queda bloqueado indefinidamente. Push recordatoria cada X horas. |
| Disco lleno | Agente emite evento error. Conversaciones read-only hasta liberar espacio. |
| Modelo no disponible | Agente informa, sugiere modelos válidos. Conversación continúa con el modelo previo. |

**Fuera de alcance:**

- Conflictos de escritura entre conversaciones paralelas sobre mismos archivos (problema del usuario, igual que con dos Cursors abiertos).
- Recuperación de sesiones CLI corruptas internamente (ofrecemos "nuevo turn desde cero con contexto reinyectado").

## 8. Scope del MVP (Fase 1)

**Objetivo:** software que el usuario usa todos los días en una semana de trabajo real. Si no logra eso, el MVP no está terminado.

### Incluido en Fase 1

**Core de control remoto:**
- Agente para **una Mac**
- **Claude Code CLI solamente**
- Proyectos: crear, listar, pinneados a working dir
- Conversaciones con streaming y approvals
- Pareo vía QR + nombrado de máquina por usuario
- SQLite con esquema completo (FTS5 habilitado aunque UI venga después)
- Reconexión automática tras network drop
- PWA instalable con UI funcional (sin pulido visual avanzado más allá del design system)

**Selección y cambio de modelo:**
- Model selector al crear proyecto (default por CLI)
- Model override al crear conversación
- Cambio de modelo mid-conversation vía tap en badge del header
- Registro del cambio como evento en el historial

**Features mobile-first:**
- **Voice input** — tap-hold del botón mic para dictar prompts usando Web Speech API. Transcripción en vivo editable antes de enviar
- **Image attachment** — botón para adjuntar desde cámara o galería. Se envía como multimodal al CLI (ambos Claude Code y Codex lo soportan en versiones actuales)
- **Quick prompts** — biblioteca local de prompts guardados (ej: "code review", "escribe tests", "explica"). Acceso rápido desde el input. Editables/eliminables.

### Explícitamente excluido de Fase 1

- Segunda Mac (trivial añadir en Fase 2)
- Codex CLI
- UI de búsqueda (el índice existe, la UI llega en Fase 2)
- Tags en proyectos (el campo existe, la UI completa llega en Fase 2)
- Push notifications (polling basta para empezar)
- Dark mode toggle (dark es el default y único por ahora)
- Plan mode, @mentions, aiconnect memory (todo Fase 2)
- Skills Manager (Fase 3)
- Distribución a terceros (signed installer, notarización)

### Criterio de "Fase 1 terminada"

Usuario usa aiconnect para trabajo real durante **una semana completa** sin pelearse con él. Si hay fricción, se arregla antes de empezar Fase 2.

## 9. Roadmap Fase 2 y Fase 3

### Fase 2 — Expansión

- **Codex CLI integrado** con su propio model picker
- **Multi-máquina:** inbox unificado agregando de varios agentes en cliente
- **UI de búsqueda full-text** (FTS5) con filtros por proyecto/CLI/máquina/tag
- **Tags completos** con UI de gestión
- **Plan mode toggle** — conversaciones read-only explícitas para brainstorming sin riesgo de acciones
- **@file / @project mentions** — autocomplete rápido al tipear `@` para referenciar archivos del repo o proyectos
- **aiconnect memory por proyecto** — contexto persistente inyectado como system prompt prefix (ej: "este proyecto usa Drizzle, prefiere named exports, evita try/catch excesivo")
- **Cross-CLI context handoff** — botón en conversación Claude para "abrir esta conversación en Codex" (exporta resumen + contexto técnico, nuevo start en el otro CLI)
- **Search dentro de conversación** — Cmd+F estilo jump entre matches, archivos, commits mencionados
- **Web Push notifications** — approval pending, turn completion, errores
- **Push actionable** — botones quick-approve/deny directamente en la notificación (iOS 16.4+, Android)
- **Onboarding improvements** — segunda máquina fluida, re-pareo guiado

### Fase 3 — Pulido / potencial apertura

- **Skills Manager** — catálogo, instalación, habilitación por proyecto, revisión antes de instalar. **Nota:** merece su propio ciclo de brainstorming antes de implementarse, por complejidad de seguridad y asimetría entre CLIs
- **Búsqueda semántica** (sqlite-vec + embeddings locales) para "encuentra la conversación donde trabajé en auth middleware"
- **Checkpoints / time-travel** — snapshot de estado de conversación en cualquier punto, rollback si la ejecución se va al carajo
- **Collaboration** — compartir snapshot read-only de una conversación con un teammate vía link temporal
- **Cost/usage tracking** — porcentaje consumido de Max plan / ChatGPT Pro, agregado por proyecto y por día
- **Voice output / read-aloud** — TTS del turn del assistant mientras manejas/caminas
- **Widgets y Apple Watch** — summary de conversaciones activas, approvals pendientes, quick-approve desde el watch
- **iOS Shortcuts** — atajos para prompts predefinidos
- **Presets de permisos por proyecto** — perfiles guardados ("sandbox total", "auto edits", "full auto con check-in")
- **Onboarding para usuarios no-técnicos** — signed .pkg, alternativa no-Tailscale guiada

## 10. Stack técnico

### Agente

- **Runtime:** Bun
- **Lenguaje:** TypeScript
- **Build:** `bun build --compile` → binario único para brew
- **Stdio bridge:** APIs nativas de Bun para subprocesos
- **WebSocket:** WebSocket server nativo de Bun
- **Storage:** SQLite con Drizzle ORM
  - FTS5 built-in (Fase 1)
  - sqlite-vec (Fase 3)

### PWA

- **Framework:** React + Vite
- **Lenguaje:** TypeScript
- **Styling:** Tailwind CSS + tokens del design system
- **Service Worker:** installability + cache offline básico + Web Push (Fase 2)
- **Voice input:** Web Speech API (gratis, cliente)
- **Image attachment:** `<input type="file" accept="image/*" capture="environment">` + Canvas API para resize pre-upload
- **Estado:** a decidir en plan (Zustand o React Query + estado local)

### Protocolo PWA ↔ Agente

- WebSocket con mensajes JSON tipados
- Versionado desde día 1 (`protocol_version` en handshake) para permitir evolución sin romper

## 11. Estrategia de testing

### Agente

- **Unit tests:** wrapper de CLI con stdio mockeado
- **Integration tests:** stub-CLI binario que emite eventos stream-json canned
- **E2E:** manual con `claude` real instalado

### PWA

- **Component tests:** Vitest
- **E2E crítico:** Playwright contra agente fake (enviar prompt → ver streaming → aprobar → ver resultado → cambiar modelo → voice input → image attach)

### Dogfood

- Cada fase se considera "terminada" solo después de **una semana completa de uso real** sin regresiones que impidan trabajar.

## 12. Decisiones abiertas (a resolver en plan)

- Formato exacto de los mensajes del protocolo WebSocket (schema detallado)
- Manejo exacto del flag `--output-format stream-json` de Claude Code y equivalencias en Codex (a verificar versiones actuales)
- Mecanismo específico para descubrir modelos disponibles por CLI (CLI command vs. hardcoded list con update)
- Estructura específica de migrations SQLite (Drizzle migrate vs. manual)
- Detalles del LaunchAgent plist (intervalos de reinicio, logs, etc.)
- Elección concreta de librería de state en PWA (Zustand vs. React Query + contexts vs. otro)
- Política de retención: ¿conversaciones viejas se archivan automáticamente? ¿O viven para siempre?
- Comportamiento exacto del voice input: ¿commitear al soltar mic o requerir tap de confirm? Preferencia por confirm para evitar envíos accidentales.
- Formato de persistencia de imágenes: ¿en SQLite como BLOB o en filesystem con referencias? Preferencia por filesystem con hash-addressed para dedup.

Estos puntos se resuelven al escribir el plan de implementación (`writing-plans`), no bloquean el diseño.

## 13. Design System

La identidad visual es parte del diseño, no cosmética aplicada después. Documentada acá para guiar implementación.

### 13.1. Principios

- **Dark editorial + engineer DNA.** Serif editorial en títulos + monospace técnico en metadata. No sans-only genérico.
- **Estructura, no decoración.** Hairline borders (`1px solid rgba(255,255,255,0.06)`), sin gradientes, sin shadows, sin glass-morphism.
- **Esquinas rectas** en contenedores y cards. Radius sutil (≤ 4px) solo en elementos táctiles (botones) donde aporta a la usabilidad.
- **Labels en mono small-caps** como tratamiento dashboard técnico (`STEP · 01/02`, `TURN · 03`, `CONV · 2F9A1C`).
- **Un solo acento fuerte** (lima `#d2ff3b`) + un acento de alerta (coral `#ff5c3a`). Nada más. Evita paleta "startup violeta-indigo".

### 13.2. Tokens

**Color:**
- `--bg: #0a0e14` — azul-negro profundo base
- `--surface-1: #10151d` — superficie elevada nivel 1
- `--surface-2: #171d27` — superficie elevada nivel 2
- `--surface-3: #1f2733` — superficie elevada nivel 3
- `--border: rgba(255,255,255,0.06)` — hairline estándar
- `--border-strong: rgba(255,255,255,0.12)` — hairline enfatizado
- `--text: #e8e4d9` — texto primario (off-white cálido, no blanco puro)
- `--text-dim: #8a8f99` — secundario
- `--text-faint: #4a4e58` — terciario
- `--accent: #d2ff3b` — lima ácido (live, focus, primary actions)
- `--accent-dim: rgba(210,255,59,0.15)` — lima con alpha para fondos
- `--coral: #ff5c3a` — alertas, approvals
- `--coral-dim: rgba(255,92,58,0.12)` — coral con alpha para fondos

**Tipografía:**
- **Display:** `"Fraunces", Georgia, serif` — weights 400/600/700. Para títulos grandes, nombres de proyecto/conversación, preguntas de approval.
- **Sans UI:** `"Inter", -apple-system, sans-serif` — weights 400/500/600. Para cuerpo de mensajes, texto de UI general.
- **Mono:** `"JetBrains Mono", "SF Mono", monospace` — weights 400/500/600. Para hostnames, paths, comandos, IDs de conversación, labels técnicos.

**Escala:**
- Display: 28px / 24px / 22px (grandes → settings → inbox header)
- Serif mediano: 18px / 16px / 15px (títulos de sección, nombres de máquinas, titulares de conversación)
- Body: 13px / 12.5px (mensajes)
- Mono metadata: 11px / 10.5px / 10px
- Small-caps labels: 9.5px con letter-spacing 0.15em, uppercase

### 13.3. Patrones

- **User turn** = bloque con barra lateral lima de 2px (no burbuja iMessage)
- **Assistant turn** = texto plano sobre fondo sin contenedor, con label `CLAUDE · HH:MM` arriba
- **Tool use** = línea compacta con prefijo `▸` en lima, fondo `surface-1`, mono
- **Approval** = card con borde coral, pregunta en serif, comando en bloque negro con prompt `$` lima
- **Model badge** = mono con `●` lima si live, aparece en chat header clickeable
- **Status pills** = caracteres geométricos (`●` online, `○` offline, `◆` generic), no emojis
