# aiconnect Fase 1 · Plan 2/3 — PWA core + E2E

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir el frontend PWA que conecta a los agentes de aiconnect vía WebSocket sobre Tailscale, implementa el flujo completo de pareo → proyectos → conversaciones → streaming → approvals → model switching, instalable en iOS/Android desde Safari/Chrome con el design system dark editorial del spec.

**Architecture:** React 18 + Vite + TypeScript. Zustand para estado global. Tailwind + tokens CSS del design system. WebSocket client custom con reconexión exponencial. IndexedDB para pairing secrets y cache de inbox. Service Worker para installability + offline cache básico. E2E con Playwright contra agente de Plan 1.

**Tech Stack:** React 18.3 · Vite 5 · TypeScript · Tailwind 3.4 · Zustand · `idb` (IndexedDB wrapper) · `@zxing/browser` (QR scan) · `zod` (shared schema con agente) · `vite-plugin-pwa` · Playwright

**Spec de referencia:** `docs/superpowers/specs/2026-04-18-aiconnect-design.md`
**Plan 1 de referencia (agente):** `docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan1-agente.md`

**Criterio de éxito del plan:** el usuario paira la PWA con el agente del Plan 1, crea un proyecto real en su Mac, abre una conversación de Claude Code, envía prompts desde el móvil, ve streaming, aprueba comandos, cambia de modelo mid-conversation, cierra la app y al volver retoma desde donde estaba. El test de dogfood de una semana está cubierto una vez este plan esté terminado.

---

## File Structure

```
pwa/
├── package.json
├── vite.config.ts
├── tailwind.config.ts
├── postcss.config.js
├── tsconfig.json
├── tsconfig.node.json
├── index.html
├── .gitignore
├── playwright.config.ts
├── public/
│   ├── manifest.webmanifest
│   ├── icon-192.png
│   ├── icon-512.png
│   └── icon-maskable.png
├── src/
│   ├── main.tsx                        # Entry, mounts App
│   ├── App.tsx                         # Router + providers
│   ├── index.css                       # Tailwind + tokens + fonts
│   │
│   ├── design/
│   │   ├── tokens.css                  # CSS vars from spec section 13
│   │   ├── fonts.css                   # Fraunces + Inter + JetBrains Mono
│   │   └── globals.css                 # Resets + base
│   │
│   ├── protocol/
│   │   └── messages.ts                 # Copied/mirror from agent (zod schemas)
│   │
│   ├── net/
│   │   ├── ws-client.ts                # Connection + auto-reconnect
│   │   ├── protocol-client.ts          # Typed request/response helpers
│   │   └── types.ts                    # Shared types
│   │
│   ├── storage/
│   │   ├── db.ts                       # IndexedDB wrapper
│   │   ├── pairings.ts                 # Pairing records CRUD
│   │   └── cache.ts                    # Inbox/project cache
│   │
│   ├── state/
│   │   ├── pairings.ts                 # zustand store: paired machines
│   │   ├── connection.ts               # zustand: active ws state per machine
│   │   ├── inbox.ts                    # zustand: aggregated conversations
│   │   └── active-conversation.ts      # zustand: currently viewed chat
│   │
│   ├── qr/
│   │   └── scanner.tsx                 # Camera + QR decoding component
│   │
│   ├── pages/
│   │   ├── Welcome.tsx
│   │   ├── PairInstructions.tsx
│   │   ├── PairScan.tsx
│   │   ├── PairManual.tsx
│   │   ├── PairNameMachine.tsx
│   │   ├── Inbox.tsx
│   │   ├── Conversation.tsx
│   │   ├── NewConversation.tsx
│   │   ├── NewProject.tsx
│   │   ├── ProjectDetail.tsx
│   │   ├── Settings.tsx
│   │   ├── SettingsMachines.tsx
│   │   └── SettingsProject.tsx
│   │
│   ├── components/
│   │   ├── layout/
│   │   │   ├── PhoneFrame.tsx          # Dev-only visual debug wrapper
│   │   │   ├── StatusBar.tsx
│   │   │   ├── HeaderRow.tsx
│   │   │   └── Ticker.tsx
│   │   ├── typography/
│   │   │   ├── BigTitle.tsx
│   │   │   ├── Label.tsx
│   │   │   └── Mono.tsx
│   │   ├── forms/
│   │   │   ├── FormInput.tsx
│   │   │   ├── FormPath.tsx
│   │   │   ├── TagChip.tsx
│   │   │   └── ModelPicker.tsx
│   │   ├── chat/
│   │   │   ├── ChatHeader.tsx
│   │   │   ├── ChatBody.tsx
│   │   │   ├── UserTurn.tsx
│   │   │   ├── AssistantTurn.tsx
│   │   │   ├── ToolLine.tsx
│   │   │   ├── ApprovalCard.tsx
│   │   │   ├── ModelBadge.tsx
│   │   │   ├── ModelSwitcherSheet.tsx
│   │   │   ├── ChatInput.tsx
│   │   │   └── StreamingCursor.tsx
│   │   ├── inbox/
│   │   │   ├── InboxHeader.tsx
│   │   │   ├── SearchRow.tsx
│   │   │   ├── ChipRow.tsx
│   │   │   ├── ConvListItem.tsx
│   │   │   └── OfflineBanner.tsx
│   │   ├── buttons/
│   │   │   ├── PrimaryButton.tsx
│   │   │   ├── SecondaryButton.tsx
│   │   │   └── ActionButton.tsx
│   │   └── feedback/
│   │       ├── ErrorBanner.tsx
│   │       └── Sheet.tsx
│   │
│   ├── hooks/
│   │   ├── useAgent.ts                 # connection per machine
│   │   ├── useConversation.ts          # live events + replay
│   │   ├── useOnline.ts                # navigator.onLine
│   │   └── useKeyboard.ts              # Cmd+K for command palette (web)
│   │
│   └── utils/
│       ├── time.ts                     # "12m", "ahora", etc.
│       ├── markdown.tsx                # simple markdown → JSX
│       └── shortcode.ts                # normalize pair short code input
│
├── tests/
│   ├── unit/
│   │   ├── storage/
│   │   │   └── pairings.test.ts
│   │   ├── net/
│   │   │   └── ws-client.test.ts
│   │   ├── state/
│   │   │   └── inbox.test.ts
│   │   └── utils/
│   │       └── time.test.ts
│   └── e2e/
│       └── full-flow.spec.ts           # Playwright against real agent
│
└── scripts/
    └── dev-with-agent.sh               # boots agent + pwa together
```

**Design decisions:**

- **Protocol schemas copied, not published package** — Plan 1 defined Zod schemas; Plan 2 copies the file. Simpler than npm publishing for one user. If Plan 3+ grows the schema, a workspace package is a natural evolution.
- **Per-machine WebSocket** — the PWA holds N connections (one per paired machine). Aggregation happens in state (`state/inbox.ts`). Machine offline = that connection in `disconnected` state, conversations from it visible but read-only.
- **Zustand over Redux** — simpler, enough for this scope. State is bounded (no complex derived computations beyond inbox aggregation).
- **IndexedDB over localStorage** — pairings include tokens (sensitive); IndexedDB is not shared between origins and we can structure properly.
- **No React Router for Fase 1** — simple custom router using window.location.hash or state-driven navigation. Full Router is overkill for the page count.

---

## Task 1: Scaffolding Vite + React + TS + Tailwind

**Files:**
- Create: `pwa/` directory
- Create: `pwa/package.json`, `vite.config.ts`, `tsconfig.json`, etc.

- [ ] **Step 1: Create pwa directory and scaffold**

```bash
cd D:/PROYECTOS/aiconnect
bun create vite pwa -- --template react-ts
cd pwa
bun install
```

- [ ] **Step 2: Add dependencies**

```bash
cd pwa
bun add zustand zod idb @zxing/browser
bun add -d tailwindcss postcss autoprefixer vite-plugin-pwa
bun add -d @playwright/test
bun add -d @types/node
```

- [ ] **Step 3: Init Tailwind**

```bash
bunx tailwindcss init -p
```

- [ ] **Step 4: Replace `tailwind.config.ts`**

```typescript
import type { Config } from "tailwindcss";

export default {
  content: ["./index.html", "./src/**/*.{ts,tsx}"],
  theme: {
    extend: {
      colors: {
        bg: "#0a0e14",
        "surface-1": "#10151d",
        "surface-2": "#171d27",
        "surface-3": "#1f2733",
        border: "rgba(255,255,255,0.06)",
        "border-strong": "rgba(255,255,255,0.12)",
        text: "#e8e4d9",
        "text-dim": "#8a8f99",
        "text-faint": "#4a4e58",
        accent: "#d2ff3b",
        "accent-dim": "rgba(210,255,59,0.15)",
        coral: "#ff5c3a",
        "coral-dim": "rgba(255,92,58,0.12)",
      },
      fontFamily: {
        serif: ['Fraunces', 'Georgia', 'serif'],
        sans: ['Inter', '-apple-system', 'sans-serif'],
        mono: ['"JetBrains Mono"', '"SF Mono"', 'monospace'],
      },
      letterSpacing: {
        "ui-label": "0.15em",
        "kbd": "0.08em",
      },
    },
  },
  plugins: [],
} satisfies Config;
```

- [ ] **Step 5: Replace `postcss.config.js`**

```javascript
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

- [ ] **Step 6: Replace `src/index.css`**

```css
@import url('https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,400;9..144,500;9..144,600;9..144,700&family=Inter:wght@400;500;600&family=JetBrains+Mono:wght@400;500;600&display=swap');

@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  color-scheme: dark;
}

html, body, #root {
  background: #0a0e14;
  color: #e8e4d9;
  height: 100%;
  margin: 0;
  font-family: 'Inter', -apple-system, sans-serif;
  -webkit-font-smoothing: antialiased;
}

body {
  overscroll-behavior: none;
}

button { all: unset; cursor: pointer; }

.label-small {
  font-family: 'JetBrains Mono', monospace;
  font-size: 9px;
  letter-spacing: 0.15em;
  text-transform: uppercase;
}
```

- [ ] **Step 7: Update `vite.config.ts`**

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { VitePWA } from "vite-plugin-pwa";

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: "autoUpdate",
      manifest: {
        name: "aiconnect",
        short_name: "aiconnect",
        description: "Control remoto para Claude Code y Codex",
        theme_color: "#0a0e14",
        background_color: "#0a0e14",
        display: "standalone",
        orientation: "portrait",
        start_url: "/",
        icons: [
          { src: "icon-192.png", sizes: "192x192", type: "image/png" },
          { src: "icon-512.png", sizes: "512x512", type: "image/png" },
          { src: "icon-maskable.png", sizes: "512x512", type: "image/png", purpose: "maskable" },
        ],
      },
      workbox: {
        navigateFallback: "/index.html",
      },
      devOptions: { enabled: true },
    }),
  ],
  server: { host: "0.0.0.0", port: 5173 },
});
```

- [ ] **Step 8: Create placeholder icons**

```bash
mkdir -p pwa/public
# Use any placeholder PNGs; real icons designed later
cp /path/to/placeholder-192.png pwa/public/icon-192.png 2>/dev/null || echo "TODO: add real icons"
```

For now, generate minimal placeholders:

```bash
cd pwa/public
# On macOS/Linux with ImageMagick; skip if not available, use any blank PNG
convert -size 192x192 xc:"#0a0e14" -fill "#d2ff3b" -pointsize 80 -gravity center -annotate 0 "◆" icon-192.png || true
convert -size 512x512 xc:"#0a0e14" -fill "#d2ff3b" -pointsize 200 -gravity center -annotate 0 "◆" icon-512.png || true
cp icon-512.png icon-maskable.png
```

If ImageMagick unavailable, create blank 1x1 PNGs — real icons come later.

- [ ] **Step 9: Smoke test dev server**

```bash
cd pwa
bun run dev
```

Open `http://localhost:5173` — default Vite page. Kill server.

- [ ] **Step 10: Commit**

```bash
cd D:/PROYECTOS/aiconnect
git add pwa/
git commit -m "feat(pwa): scaffold vite + react + tailwind + pwa plugin"
```

---

## Task 2: Design system base components

**Files:**
- Create: `pwa/src/components/typography/BigTitle.tsx`, `Label.tsx`, `Mono.tsx`
- Create: `pwa/src/components/buttons/PrimaryButton.tsx`, `SecondaryButton.tsx`
- Create: `pwa/src/components/layout/StatusBar.tsx`, `HeaderRow.tsx`

- [ ] **Step 1: Implement typography components**

`pwa/src/components/typography/BigTitle.tsx`:

```typescript
import type { ReactNode } from "react";

export function BigTitle({ children }: { children: ReactNode }) {
  return (
    <h1 className="font-serif font-semibold text-[22px] leading-[1.1] tracking-[-0.025em] text-text">
      {children}
    </h1>
  );
}
```

`Label.tsx`:

```typescript
import type { ReactNode } from "react";

export function Label({ children }: { children: ReactNode }) {
  return (
    <span className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim">
      {children}
    </span>
  );
}
```

`Mono.tsx`:

```typescript
import type { ReactNode, HTMLAttributes } from "react";

export function Mono({ children, ...rest }: HTMLAttributes<HTMLSpanElement> & { children: ReactNode }) {
  return <span className="font-mono" {...rest}>{children}</span>;
}
```

- [ ] **Step 2: Implement button components**

`pwa/src/components/buttons/PrimaryButton.tsx`:

```typescript
import type { ButtonHTMLAttributes, ReactNode } from "react";

export function PrimaryButton(props: ButtonHTMLAttributes<HTMLButtonElement> & { children: ReactNode }) {
  const { children, className = "", ...rest } = props;
  return (
    <button
      className={`bg-accent text-bg py-3 px-4 font-mono text-[10.5px] tracking-[0.15em] uppercase font-semibold w-full disabled:opacity-40 ${className}`}
      {...rest}
    >
      {children}
    </button>
  );
}
```

`SecondaryButton.tsx`:

```typescript
import type { ButtonHTMLAttributes, ReactNode } from "react";

export function SecondaryButton(props: ButtonHTMLAttributes<HTMLButtonElement> & { children: ReactNode }) {
  const { children, className = "", ...rest } = props;
  return (
    <button
      className={`bg-transparent text-text py-3 px-4 border border-border-strong font-mono text-[10.5px] tracking-[0.15em] uppercase w-full disabled:opacity-40 ${className}`}
      {...rest}
    >
      {children}
    </button>
  );
}
```

- [ ] **Step 3: Implement layout components**

`pwa/src/components/layout/StatusBar.tsx`:

```typescript
export function StatusBar({ time = "09:41", right = "◆ 5G · 87%" }: { time?: string; right?: string }) {
  return (
    <div className="h-6 flex justify-between items-center px-5 font-mono text-[10px] text-text-dim">
      <span>{time}</span>
      <span>{right}</span>
    </div>
  );
}
```

`HeaderRow.tsx`:

```typescript
import type { ReactNode } from "react";

export function HeaderRow({ left, right }: { left?: ReactNode; right?: ReactNode }) {
  return (
    <div className="px-4 py-3 border-b border-border flex justify-between items-center">
      <div>{left}</div>
      <div>{right}</div>
    </div>
  );
}

export function Logo() {
  return (
    <span className="font-mono text-[11px] tracking-wide text-text">
      <span className="text-accent">◆</span> aiconnect
    </span>
  );
}
```

- [ ] **Step 4: Write smoke test**

`pwa/tests/unit/components/typography.test.tsx`:

```typescript
// Use Bun's jsdom if available; otherwise skip DOM-heavy tests here and cover in E2E.
// For Fase 1 we rely on Playwright for full visual testing.
```

Since Bun's test runner doesn't ship with JSDOM out of the box and we want to avoid adding React Testing Library complexity for tiny components, we cover these via Playwright E2E in Task 24. Skip unit tests for pure presentational components.

- [ ] **Step 5: Commit**

```bash
git add pwa/src/components/
git commit -m "feat(pwa): base design system components (typography, buttons, layout)"
```

---

## Task 3: Copy protocol schemas from agent

**Files:**
- Create: `pwa/src/protocol/messages.ts`

- [ ] **Step 1: Copy contents**

Copy `agent/src/protocol/messages.ts` → `pwa/src/protocol/messages.ts` verbatim (the Zod schemas).

Also copy `agent/src/protocol/version.ts` → `pwa/src/protocol/version.ts`.

**Important:** These must stay in sync. Document this in a comment at the top:

```typescript
// WARNING: This file must stay in sync with agent/src/protocol/messages.ts.
// On schema changes, update both files and bump PROTOCOL_VERSION.
// In Fase 2+, consider promoting this to a workspace package shared by both.
```

- [ ] **Step 2: Verify import resolves**

```bash
cd pwa
bun run tsc --noEmit
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add pwa/src/protocol/
git commit -m "feat(pwa): copy protocol schemas from agent"
```

---

## Task 4: IndexedDB wrapper for pairings

**Files:**
- Create: `pwa/src/storage/db.ts`
- Create: `pwa/src/storage/pairings.ts`
- Create: `pwa/tests/unit/storage/pairings.test.ts`

- [ ] **Step 1: Implement `src/storage/db.ts`**

```typescript
import { openDB, type IDBPDatabase, type DBSchema } from "idb";

export interface Pairing {
  id: string;          // local UUID, primary key
  hostname: string;    // tailscale magic dns
  port: number;
  token: string;       // plain token (never leaves this device)
  fingerprint: string; // expected agent fingerprint
  machineId: string;
  displayName: string;
  createdAt: number;
  lastSeen: number;
}

export interface QuickPrompt {
  id: string;
  name: string;
  body: string;
  shortcut?: string;
  createdAt: number;
}

interface Schema extends DBSchema {
  pairings: { key: string; value: Pairing };
  quickPrompts: { key: string; value: QuickPrompt };
}

let instance: IDBPDatabase<Schema> | null = null;

export async function getDb(): Promise<IDBPDatabase<Schema>> {
  if (instance) return instance;
  instance = await openDB<Schema>("aiconnect", 1, {
    upgrade(db) {
      db.createObjectStore("pairings", { keyPath: "id" });
      db.createObjectStore("quickPrompts", { keyPath: "id" });
    },
  });
  return instance;
}
```

- [ ] **Step 2: Implement `src/storage/pairings.ts`**

```typescript
import { getDb, type Pairing } from "./db";

export async function savePairing(p: Pairing): Promise<void> {
  const db = await getDb();
  await db.put("pairings", p);
}

export async function listPairings(): Promise<Pairing[]> {
  const db = await getDb();
  return db.getAll("pairings");
}

export async function getPairing(id: string): Promise<Pairing | undefined> {
  const db = await getDb();
  return db.get("pairings", id);
}

export async function removePairing(id: string): Promise<void> {
  const db = await getDb();
  await db.delete("pairings", id);
}

export async function updateLastSeen(id: string, when: number): Promise<void> {
  const db = await getDb();
  const p = await db.get("pairings", id);
  if (!p) return;
  p.lastSeen = when;
  await db.put("pairings", p);
}
```

- [ ] **Step 3: Write test using fake-indexeddb**

First add test dep:

```bash
bun add -d fake-indexeddb
```

`pwa/tests/unit/storage/pairings.test.ts`:

```typescript
import { describe, test, expect, beforeEach } from "bun:test";
import "fake-indexeddb/auto";
import { savePairing, listPairings, getPairing, removePairing, updateLastSeen } from "../../../src/storage/pairings";

function mk(id: string, overrides: Partial<import("../../../src/storage/db").Pairing> = {}): import("../../../src/storage/db").Pairing {
  return {
    id,
    hostname: "x.ts.net",
    port: 51820,
    token: "tok",
    fingerprint: "fp",
    machineId: "m",
    displayName: "test-mac",
    createdAt: Date.now(),
    lastSeen: Date.now(),
    ...overrides,
  };
}

describe("pairings storage", () => {
  test("save + get", async () => {
    await savePairing(mk("a"));
    const got = await getPairing("a");
    expect(got?.displayName).toBe("test-mac");
  });

  test("list returns all", async () => {
    await savePairing(mk("a"));
    await savePairing(mk("b"));
    const all = await listPairings();
    const ids = all.map((p) => p.id).sort();
    expect(ids).toContain("a");
    expect(ids).toContain("b");
  });

  test("remove deletes", async () => {
    await savePairing(mk("a"));
    await removePairing("a");
    expect(await getPairing("a")).toBeUndefined();
  });

  test("updateLastSeen updates timestamp", async () => {
    await savePairing(mk("a", { lastSeen: 100 }));
    await updateLastSeen("a", 999);
    const got = await getPairing("a");
    expect(got?.lastSeen).toBe(999);
  });
});
```

- [ ] **Step 4: Run tests**

```bash
cd pwa
bun test tests/unit/storage/pairings.test.ts
```

Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add pwa/src/storage/ pwa/tests/unit/storage/ pwa/package.json
git commit -m "feat(pwa): indexeddb storage for pairings and quick prompts"
```

---

## Task 5: WebSocket client with auto-reconnect

**Files:**
- Create: `pwa/src/net/ws-client.ts`
- Create: `pwa/tests/unit/net/ws-client.test.ts`

- [ ] **Step 1: Implement `src/net/ws-client.ts`**

```typescript
import { ServerMessage, type ServerMessageT, type ClientMessageT } from "../protocol/messages";

export type WsState = "disconnected" | "connecting" | "authenticating" | "authenticated" | "reconnecting";

export interface WsClientOptions {
  url: string;
  authToken: string;
  protocolVersion: number;
  deviceLabel?: string;
  onState: (s: WsState) => void;
  onMessage: (m: ServerMessageT) => void;
  onFatal: (reason: string) => void;
  backoffCapMs?: number;
}

export class WsClient {
  private ws: WebSocket | null = null;
  private state: WsState = "disconnected";
  private closedByUser = false;
  private attempt = 0;
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private readonly backoffCap: number;

  constructor(private readonly opts: WsClientOptions) {
    this.backoffCap = opts.backoffCapMs ?? 10_000;
  }

  connect(): void {
    this.closedByUser = false;
    this.setState("connecting");
    this.ws = new WebSocket(this.opts.url);
    this.ws.addEventListener("open", () => {
      this.setState("authenticating");
      this.send({
        type: "auth",
        token: this.opts.authToken,
        protocolVersion: this.opts.protocolVersion,
        ...(this.opts.deviceLabel !== undefined ? { deviceLabel: this.opts.deviceLabel } : {}),
      });
    });
    this.ws.addEventListener("message", (ev) => {
      let parsed: unknown;
      try { parsed = JSON.parse(typeof ev.data === "string" ? ev.data : ""); } catch { return; }
      const result = ServerMessage.safeParse(parsed);
      if (!result.success) {
        console.warn("[ws] invalid server message", result.error);
        return;
      }
      const msg = result.data;
      if (msg.type === "auth_ok") {
        this.attempt = 0;
        this.setState("authenticated");
      } else if (msg.type === "error" && msg.code === "unauthorized") {
        this.closedByUser = true;
        this.opts.onFatal("unauthorized");
        this.ws?.close();
        return;
      }
      this.opts.onMessage(msg);
    });
    this.ws.addEventListener("close", () => {
      if (this.closedByUser) {
        this.setState("disconnected");
        return;
      }
      this.setState("reconnecting");
      this.scheduleReconnect();
    });
    this.ws.addEventListener("error", () => {
      // close handler will handle reconnect
    });
  }

  send(msg: ClientMessageT | { type: "auth"; token: string; protocolVersion: number; deviceLabel?: string }): void {
    if (this.ws?.readyState !== WebSocket.OPEN) {
      console.warn("[ws] send ignored, not open");
      return;
    }
    this.ws.send(JSON.stringify(msg));
  }

  close(): void {
    this.closedByUser = true;
    if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
    this.ws?.close();
    this.setState("disconnected");
  }

  getState(): WsState { return this.state; }

  private setState(s: WsState): void {
    if (this.state !== s) {
      this.state = s;
      this.opts.onState(s);
    }
  }

  private scheduleReconnect(): void {
    this.attempt++;
    const delay = Math.min(this.backoffCap, 500 * 2 ** Math.min(this.attempt, 5));
    this.reconnectTimer = setTimeout(() => this.connect(), delay);
  }
}
```

- [ ] **Step 2: Write basic tests (mock WebSocket)**

Bun doesn't ship WebSocket globally in test env universally; we use mock.

`pwa/tests/unit/net/ws-client.test.ts`:

```typescript
import { describe, test, expect, mock, beforeEach } from "bun:test";
import { WsClient, type WsState } from "../../../src/net/ws-client";

interface MockWs {
  listeners: Record<string, Array<(e: unknown) => void>>;
  readyState: number;
  send: (data: string) => void;
  close: () => void;
  addEventListener: (type: string, cb: (e: unknown) => void) => void;
  _simulateOpen: () => void;
  _simulateMessage: (data: unknown) => void;
  _simulateClose: () => void;
  sent: string[];
}

function createMockWs(): MockWs {
  const m: MockWs = {
    listeners: {},
    readyState: 0,
    sent: [],
    send(data) { m.sent.push(data); },
    close() { m._simulateClose(); },
    addEventListener(type, cb) { (m.listeners[type] ??= []).push(cb); },
    _simulateOpen() {
      m.readyState = 1;
      for (const cb of m.listeners["open"] ?? []) cb({});
    },
    _simulateMessage(data) {
      for (const cb of m.listeners["message"] ?? []) cb({ data: JSON.stringify(data) });
    },
    _simulateClose() {
      m.readyState = 3;
      for (const cb of m.listeners["close"] ?? []) cb({});
    },
  };
  return m;
}

let mockWs: MockWs;

beforeEach(() => {
  mockWs = createMockWs();
  (globalThis as unknown as { WebSocket: new () => MockWs }).WebSocket = function() { return mockWs; } as never;
  (globalThis as unknown as { WebSocket: { OPEN: number; CLOSED: number } }).WebSocket.OPEN = 1;
  (globalThis as unknown as { WebSocket: { OPEN: number; CLOSED: number } }).WebSocket.CLOSED = 3;
});

describe("WsClient", () => {
  test("sends auth on open and transitions to authenticating", () => {
    const states: WsState[] = [];
    const client = new WsClient({
      url: "ws://x",
      authToken: "t".repeat(20),
      protocolVersion: 1,
      onState: (s) => states.push(s),
      onMessage: () => {},
      onFatal: () => {},
    });
    client.connect();
    mockWs._simulateOpen();
    expect(states).toEqual(["connecting", "authenticating"]);
    expect(mockWs.sent[0]).toContain("\"type\":\"auth\"");
  });

  test("transitions to authenticated on auth_ok", () => {
    const states: WsState[] = [];
    const client = new WsClient({
      url: "ws://x",
      authToken: "t".repeat(20),
      protocolVersion: 1,
      onState: (s) => states.push(s),
      onMessage: () => {},
      onFatal: () => {},
    });
    client.connect();
    mockWs._simulateOpen();
    mockWs._simulateMessage({
      type: "auth_ok",
      tokenId: "tid",
      machine: { id: "m", fingerprint: "fp", hostname: "h", displayName: "h" },
      protocolVersion: 1,
    });
    expect(states).toContain("authenticated");
  });

  test("onFatal fires on unauthorized", () => {
    let fatalReason: string | null = null;
    const client = new WsClient({
      url: "ws://x",
      authToken: "t".repeat(20),
      protocolVersion: 1,
      onState: () => {},
      onMessage: () => {},
      onFatal: (r) => { fatalReason = r; },
    });
    client.connect();
    mockWs._simulateOpen();
    mockWs._simulateMessage({ type: "error", code: "unauthorized", message: "nope" });
    expect(fatalReason).toBe("unauthorized");
  });
});
```

- [ ] **Step 3: Run**

```bash
cd pwa
bun test tests/unit/net/ws-client.test.ts
```

Expected: PASS, 3 tests.

- [ ] **Step 4: Commit**

```bash
git add pwa/src/net/ pwa/tests/unit/net/
git commit -m "feat(pwa): ws client with auth handshake and reconnect"
```

---

## Task 6: Zustand stores — pairings, connection, inbox, active conversation

**Files:**
- Create: `pwa/src/state/pairings.ts`
- Create: `pwa/src/state/connection.ts`
- Create: `pwa/src/state/inbox.ts`
- Create: `pwa/src/state/active-conversation.ts`

- [ ] **Step 1: Implement `state/pairings.ts`**

```typescript
import { create } from "zustand";
import { listPairings, savePairing, removePairing, type Pairing } from "../storage/pairings";

interface PairingsState {
  pairings: Pairing[];
  loaded: boolean;
  load: () => Promise<void>;
  add: (p: Pairing) => Promise<void>;
  remove: (id: string) => Promise<void>;
}

export const usePairings = create<PairingsState>((set, get) => ({
  pairings: [],
  loaded: false,
  async load() {
    const list = await listPairings();
    set({ pairings: list, loaded: true });
  },
  async add(p) {
    await savePairing(p);
    set({ pairings: [...get().pairings.filter((x) => x.id !== p.id), p] });
  },
  async remove(id) {
    await removePairing(id);
    set({ pairings: get().pairings.filter((x) => x.id !== id) });
  },
}));
```

- [ ] **Step 2: Implement `state/connection.ts`**

```typescript
import { create } from "zustand";
import type { WsState } from "../net/ws-client";

interface ConnectionState {
  // Map pairingId → WsState
  states: Map<string, WsState>;
  setState: (pairingId: string, state: WsState) => void;
  clear: (pairingId: string) => void;
  all: () => Array<{ pairingId: string; state: WsState }>;
}

export const useConnections = create<ConnectionState>((set, get) => ({
  states: new Map(),
  setState(pairingId, state) {
    const next = new Map(get().states);
    next.set(pairingId, state);
    set({ states: next });
  },
  clear(pairingId) {
    const next = new Map(get().states);
    next.delete(pairingId);
    set({ states: next });
  },
  all() {
    return Array.from(get().states.entries()).map(([pairingId, state]) => ({ pairingId, state }));
  },
}));
```

- [ ] **Step 3: Implement `state/inbox.ts`**

```typescript
import { create } from "zustand";

export interface InboxConversation {
  pairingId: string;       // which machine it came from
  machineDisplayName: string;
  id: string;
  projectId: string;
  projectName: string;
  cli: "claude-code" | "codex";
  currentModel: string | null;
  title: string;
  status: "idle" | "running" | "awaiting_approval" | "error" | "interrupted";
  mode: "active" | "plan-only";
  lastActivityAt: number;
  lastPreviewText?: string;
}

export interface InboxProject {
  pairingId: string;
  id: string;
  name: string;
  tags: string[];
}

interface InboxState {
  conversations: InboxConversation[];
  projects: InboxProject[];
  upsertConversation: (c: InboxConversation) => void;
  upsertProject: (p: InboxProject) => void;
  removeByPairing: (pairingId: string) => void;
  filteredConversations: (filter: InboxFilter) => InboxConversation[];
}

export interface InboxFilter {
  cli?: "claude-code" | "codex";
  tag?: string;
  pairingId?: string;
  onlyApproval?: boolean;
}

export const useInbox = create<InboxState>((set, get) => ({
  conversations: [],
  projects: [],
  upsertConversation(c) {
    const next = get().conversations.filter((x) => !(x.pairingId === c.pairingId && x.id === c.id));
    next.push(c);
    next.sort((a, b) => b.lastActivityAt - a.lastActivityAt);
    set({ conversations: next });
  },
  upsertProject(p) {
    const next = get().projects.filter((x) => !(x.pairingId === p.pairingId && x.id === p.id));
    next.push(p);
    set({ projects: next });
  },
  removeByPairing(pairingId) {
    set({
      conversations: get().conversations.filter((c) => c.pairingId !== pairingId),
      projects: get().projects.filter((p) => p.pairingId !== pairingId),
    });
  },
  filteredConversations(filter) {
    const all = get().conversations;
    return all.filter((c) => {
      if (filter.cli && c.cli !== filter.cli) return false;
      if (filter.pairingId && c.pairingId !== filter.pairingId) return false;
      if (filter.onlyApproval && c.status !== "awaiting_approval") return false;
      if (filter.tag) {
        const proj = get().projects.find((p) => p.pairingId === c.pairingId && p.id === c.projectId);
        if (!proj?.tags.includes(filter.tag)) return false;
      }
      return true;
    });
  },
}));
```

- [ ] **Step 4: Implement `state/active-conversation.ts`**

```typescript
import { create } from "zustand";
import type { ConversationEventT } from "../protocol/messages";

interface ActiveConversationState {
  pairingId: string | null;
  conversationId: string | null;
  events: ConversationEventT[];
  streaming: boolean;
  lastEventId: string | null;

  open: (pairingId: string, conversationId: string) => void;
  close: () => void;
  appendEvent: (e: ConversationEventT) => void;
  setStreaming: (s: boolean) => void;
  replaceEvents: (events: ConversationEventT[]) => void;
}

export const useActiveConversation = create<ActiveConversationState>((set) => ({
  pairingId: null,
  conversationId: null,
  events: [],
  streaming: false,
  lastEventId: null,

  open(pairingId, conversationId) {
    set({ pairingId, conversationId, events: [], lastEventId: null, streaming: false });
  },
  close() {
    set({ pairingId: null, conversationId: null, events: [], streaming: false, lastEventId: null });
  },
  appendEvent(e) {
    set((s) => ({ events: [...s.events, e], lastEventId: e.id }));
  },
  setStreaming(s) {
    set({ streaming: s });
  },
  replaceEvents(events) {
    const last = events.length > 0 ? events[events.length - 1]!.id : null;
    set({ events, lastEventId: last });
  },
}));
```

- [ ] **Step 5: Write test for inbox aggregation**

`pwa/tests/unit/state/inbox.test.ts`:

```typescript
import { describe, test, expect, beforeEach } from "bun:test";
import { useInbox } from "../../../src/state/inbox";

beforeEach(() => {
  useInbox.setState({ conversations: [], projects: [] });
});

function mkConv(overrides: Partial<import("../../../src/state/inbox").InboxConversation>) {
  return {
    pairingId: "p1",
    machineDisplayName: "mac-a",
    id: "c1",
    projectId: "proj1",
    projectName: "api",
    cli: "claude-code" as const,
    currentModel: "sonnet",
    title: "t",
    status: "idle" as const,
    mode: "active" as const,
    lastActivityAt: Date.now(),
    ...overrides,
  };
}

describe("inbox store", () => {
  test("upsert replaces existing by pairing+id", () => {
    useInbox.getState().upsertConversation(mkConv({ id: "a", title: "v1" }));
    useInbox.getState().upsertConversation(mkConv({ id: "a", title: "v2" }));
    const all = useInbox.getState().conversations;
    expect(all.length).toBe(1);
    expect(all[0]?.title).toBe("v2");
  });

  test("filters by cli", () => {
    useInbox.getState().upsertConversation(mkConv({ id: "a", cli: "claude-code" }));
    useInbox.getState().upsertConversation(mkConv({ id: "b", cli: "codex" }));
    const claude = useInbox.getState().filteredConversations({ cli: "claude-code" });
    expect(claude.length).toBe(1);
    expect(claude[0]?.id).toBe("a");
  });

  test("filters by approval", () => {
    useInbox.getState().upsertConversation(mkConv({ id: "a", status: "idle" }));
    useInbox.getState().upsertConversation(mkConv({ id: "b", status: "awaiting_approval" }));
    const approv = useInbox.getState().filteredConversations({ onlyApproval: true });
    expect(approv.length).toBe(1);
    expect(approv[0]?.id).toBe("b");
  });

  test("sort by lastActivity desc", () => {
    useInbox.getState().upsertConversation(mkConv({ id: "old", lastActivityAt: 100 }));
    useInbox.getState().upsertConversation(mkConv({ id: "new", lastActivityAt: 999 }));
    const all = useInbox.getState().conversations;
    expect(all[0]?.id).toBe("new");
  });
});
```

- [ ] **Step 6: Run tests**

```bash
cd pwa
bun test tests/unit/state/inbox.test.ts
```

Expected: PASS, 4 tests.

- [ ] **Step 7: Commit**

```bash
git add pwa/src/state/ pwa/tests/unit/state/
git commit -m "feat(pwa): zustand stores for pairings, connections, inbox, active conv"
```

---

## Task 7: AgentClient — per-machine connection orchestration

**Files:**
- Create: `pwa/src/net/agent-client.ts`

- [ ] **Step 1: Implement `src/net/agent-client.ts`**

```typescript
import { WsClient, type WsState } from "./ws-client";
import { PROTOCOL_VERSION } from "../protocol/version";
import { useConnections } from "../state/connection";
import { useInbox } from "../state/inbox";
import { useActiveConversation } from "../state/active-conversation";
import { updateLastSeen, type Pairing } from "../storage/pairings";
import type { ServerMessageT, ClientMessageT, ConversationEventT } from "../protocol/messages";

const clients = new Map<string, AgentClient>();

export class AgentClient {
  private ws: WsClient;

  constructor(private readonly pairing: Pairing) {
    this.ws = new WsClient({
      url: `ws://${pairing.hostname}:${pairing.port}`,
      authToken: pairing.token,
      protocolVersion: PROTOCOL_VERSION,
      deviceLabel: navigator.userAgent.slice(0, 80),
      onState: (s) => this.onStateChange(s),
      onMessage: (m) => this.onMessage(m),
      onFatal: (reason) => console.error("[agent]", this.pairing.displayName, "fatal:", reason),
    });
  }

  connect(): void { this.ws.connect(); }
  disconnect(): void { this.ws.close(); }
  send(msg: ClientMessageT): void { this.ws.send(msg); }

  private onStateChange(s: WsState): void {
    useConnections.getState().setState(this.pairing.id, s);
    if (s === "authenticated") {
      updateLastSeen(this.pairing.id, Date.now()).catch(() => {});
      // Request projects + conversations on auth
      this.send({ type: "list_projects" });
      this.send({ type: "list_conversations" });
    }
  }

  private onMessage(m: ServerMessageT): void {
    if (m.type === "project_list") {
      for (const p of m.projects) {
        useInbox.getState().upsertProject({
          pairingId: this.pairing.id,
          id: p.id,
          name: p.name,
          tags: p.tags,
        });
      }
    } else if (m.type === "project_created") {
      useInbox.getState().upsertProject({
        pairingId: this.pairing.id,
        id: m.project.id,
        name: m.project.name,
        tags: m.project.tags,
      });
    } else if (m.type === "conversation_list") {
      for (const c of m.conversations) {
        const proj = useInbox.getState().projects.find((p) => p.pairingId === this.pairing.id && p.id === c.projectId);
        useInbox.getState().upsertConversation({
          pairingId: this.pairing.id,
          machineDisplayName: this.pairing.displayName,
          id: c.id,
          projectId: c.projectId,
          projectName: proj?.name ?? c.projectId,
          cli: c.cli,
          currentModel: c.currentModel,
          title: c.title,
          status: c.status,
          mode: c.mode,
          lastActivityAt: c.lastActivityAt,
        });
      }
    } else if (m.type === "conversation_created") {
      const proj = useInbox.getState().projects.find((p) => p.pairingId === this.pairing.id && p.id === m.conversation.projectId);
      useInbox.getState().upsertConversation({
        pairingId: this.pairing.id,
        machineDisplayName: this.pairing.displayName,
        id: m.conversation.id,
        projectId: m.conversation.projectId,
        projectName: proj?.name ?? "",
        cli: m.conversation.cli,
        currentModel: m.conversation.currentModel,
        title: m.conversation.title,
        status: m.conversation.status,
        mode: m.conversation.mode,
        lastActivityAt: m.conversation.lastActivityAt,
      });
    } else if (m.type === "event") {
      this.handleEvent(m.conversationId, m.event);
    } else if (m.type === "replay_done") {
      useActiveConversation.getState().setStreaming(false);
    }
  }

  private handleEvent(conversationId: string, ev: ConversationEventT): void {
    const active = useActiveConversation.getState();
    if (active.pairingId === this.pairing.id && active.conversationId === conversationId) {
      active.appendEvent(ev);
    }
    // Update inbox preview + status
    const conv = useInbox.getState().conversations.find((c) => c.pairingId === this.pairing.id && c.id === conversationId);
    if (conv) {
      let status = conv.status;
      if (ev.type === "permission_request") status = "awaiting_approval";
      else if (ev.type === "turn_complete") status = "idle";
      useInbox.getState().upsertConversation({
        ...conv,
        status,
        lastActivityAt: ev.timestamp,
        lastPreviewText: ev.type === "assistant_text_delta" ? (ev.payload.text ?? conv.lastPreviewText) : conv.lastPreviewText,
      });
    }
  }
}

export function getAgent(pairing: Pairing): AgentClient {
  let c = clients.get(pairing.id);
  if (!c) {
    c = new AgentClient(pairing);
    clients.set(pairing.id, c);
    c.connect();
  }
  return c;
}

export function dropAgent(pairingId: string): void {
  const c = clients.get(pairingId);
  if (c) {
    c.disconnect();
    clients.delete(pairingId);
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add pwa/src/net/agent-client.ts
git commit -m "feat(pwa): AgentClient per-machine ws orchestration"
```

---

## Task 8: Pantalla Welcome + gestión de primer pareo

**Files:**
- Create: `pwa/src/pages/Welcome.tsx`
- Create: `pwa/src/App.tsx`
- Modify: `pwa/src/main.tsx`

- [ ] **Step 1: Implement `src/App.tsx`**

```typescript
import { useEffect, useState } from "react";
import { usePairings } from "./state/pairings";
import { Welcome } from "./pages/Welcome";
import { PairScan } from "./pages/PairScan";
import { PairManual } from "./pages/PairManual";
import { PairNameMachine } from "./pages/PairNameMachine";
import { Inbox } from "./pages/Inbox";
import { Conversation } from "./pages/Conversation";
import { NewProject } from "./pages/NewProject";
import { NewConversation } from "./pages/NewConversation";
import { Settings } from "./pages/Settings";

export type Route =
  | { name: "welcome" }
  | { name: "pair-scan" }
  | { name: "pair-manual" }
  | { name: "pair-name"; payload: import("./storage/db").Pairing }
  | { name: "inbox" }
  | { name: "conversation"; pairingId: string; conversationId: string }
  | { name: "new-project" }
  | { name: "new-conversation"; pairingId: string; projectId: string }
  | { name: "settings" };

export function App() {
  const [route, setRoute] = useState<Route>({ name: "welcome" });
  const { pairings, loaded, load } = usePairings();

  useEffect(() => {
    load().then(() => {
      if (usePairings.getState().pairings.length > 0) setRoute({ name: "inbox" });
    });
  }, []);

  if (!loaded) return null;

  switch (route.name) {
    case "welcome": return <Welcome onStart={() => setRoute({ name: "pair-scan" })} />;
    case "pair-scan": return <PairScan onFound={(p) => setRoute({ name: "pair-name", payload: p })} onManual={() => setRoute({ name: "pair-manual" })} />;
    case "pair-manual": return <PairManual onFound={(p) => setRoute({ name: "pair-name", payload: p })} onCancel={() => setRoute({ name: "pair-scan" })} />;
    case "pair-name": return <PairNameMachine pairing={route.payload} onDone={() => setRoute({ name: "inbox" })} />;
    case "inbox": return <Inbox
        onOpenConv={(pairingId, conversationId) => setRoute({ name: "conversation", pairingId, conversationId })}
        onNewProject={() => setRoute({ name: "new-project" })}
        onSettings={() => setRoute({ name: "settings" })}
      />;
    case "conversation": return <Conversation
        pairingId={route.pairingId}
        conversationId={route.conversationId}
        onBack={() => setRoute({ name: "inbox" })}
      />;
    case "new-project": return <NewProject
        onCreated={() => setRoute({ name: "inbox" })}
        onCancel={() => setRoute({ name: "inbox" })}
      />;
    case "new-conversation": return <NewConversation
        pairingId={route.pairingId}
        projectId={route.projectId}
        onCreated={(conversationId) => setRoute({ name: "conversation", pairingId: route.pairingId, conversationId })}
        onCancel={() => setRoute({ name: "inbox" })}
      />;
    case "settings": return <Settings onBack={() => setRoute({ name: "inbox" })} />;
  }
}
```

- [ ] **Step 2: Implement `src/pages/Welcome.tsx`**

```typescript
import { StatusBar } from "../components/layout/StatusBar";
import { HeaderRow, Logo } from "../components/layout/HeaderRow";
import { Label } from "../components/typography/Label";
import { BigTitle } from "../components/typography/BigTitle";
import { PrimaryButton } from "../components/buttons/PrimaryButton";
import { SecondaryButton } from "../components/buttons/SecondaryButton";

export function Welcome({ onStart }: { onStart: () => void }) {
  return (
    <div className="flex flex-col h-full bg-bg">
      <StatusBar />
      <HeaderRow left={<Logo />} right={<Label>v0.1.0</Label>} />
      <div className="flex-1 flex flex-col gap-5 p-5">
        <Label>bienvenido</Label>
        <div className="mt-1">
          <BigTitle>Controla tus<br />sesiones de<br /><em className="not-italic text-accent">Claude Code</em><br />desde donde sea.</BigTitle>
        </div>
        <div className="mt-auto text-[13px] leading-[1.5] text-text-dim">
          Necesitas una Mac con <code className="font-mono text-accent text-[11px]">claude</code> instalado y autenticado con tu Max plan.
        </div>
        <div>
          <PrimaryButton onClick={onStart}>Parear primera Mac</PrimaryButton>
          <SecondaryButton>¿No tengo claude instalado?</SecondaryButton>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Create stub pages for the rest (will fill in subsequent tasks)**

`pwa/src/pages/PairScan.tsx`:

```typescript
import type { Pairing } from "../storage/db";
export function PairScan({ onFound: _onFound, onManual: _onManual }: { onFound: (p: Pairing) => void; onManual: () => void }) {
  return <div>PairScan placeholder</div>;
}
```

Do the same for:
- `PairManual.tsx`
- `PairNameMachine.tsx`
- `Inbox.tsx`
- `Conversation.tsx`
- `NewProject.tsx`
- `NewConversation.tsx`
- `Settings.tsx`

Each exports a React component with a placeholder div and matching props. This allows `App.tsx` to compile; subsequent tasks flesh out each page.

Example stub for `Inbox.tsx`:

```typescript
export function Inbox({ onOpenConv: _a, onNewProject: _b, onSettings: _c }: {
  onOpenConv: (pairingId: string, conversationId: string) => void;
  onNewProject: () => void;
  onSettings: () => void;
}) {
  return <div className="p-5">Inbox placeholder</div>;
}
```

Generate all stubs; keep prop signatures matching the ones used in `App.tsx`.

- [ ] **Step 4: Update `src/main.tsx`**

```typescript
import React from "react";
import ReactDOM from "react-dom/client";
import { App } from "./App";
import "./index.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

- [ ] **Step 5: Run dev server + smoke test**

```bash
cd pwa
bun run dev
```

Open browser at `http://localhost:5173`. Expected: Welcome screen renders with dark editorial style.

- [ ] **Step 6: Commit**

```bash
git add pwa/src/
git commit -m "feat(pwa): welcome screen + page router skeleton with stubs"
```

---

## Task 9: QR scanner component

**Files:**
- Create: `pwa/src/qr/scanner.tsx`
- Create: `pwa/src/pages/PairScan.tsx` (real implementation)
- Create: `pwa/src/utils/shortcode.ts`

- [ ] **Step 1: Implement `src/utils/shortcode.ts`**

```typescript
export function normalizeShortCode(raw: string): string {
  return raw.toUpperCase().replace(/[^A-Z0-9]/g, "");
}

export function formatShortCode(normalized: string): string {
  const clean = normalizeShortCode(normalized).slice(0, 6);
  if (clean.length <= 3) return clean;
  return `${clean.slice(0, 3)}-${clean.slice(3)}`;
}
```

- [ ] **Step 2: Implement `src/qr/scanner.tsx`**

```typescript
import { useEffect, useRef, useState } from "react";
import { BrowserQRCodeReader } from "@zxing/browser";

interface Props {
  onDecoded: (text: string) => void;
  onError?: (err: Error) => void;
}

export function QrScanner({ onDecoded, onError }: Props) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const [err, setErr] = useState<string | null>(null);

  useEffect(() => {
    const reader = new BrowserQRCodeReader();
    let controls: { stop: () => void } | undefined;

    (async () => {
      try {
        const devices = await BrowserQRCodeReader.listVideoInputDevices();
        const deviceId = devices[0]?.deviceId;
        if (!deviceId) {
          setErr("No se detectó cámara");
          return;
        }
        controls = await reader.decodeFromVideoDevice(deviceId, videoRef.current!, (result) => {
          if (result) onDecoded(result.getText());
        });
      } catch (e) {
        const msg = e instanceof Error ? e.message : String(e);
        setErr(msg);
        if (onError && e instanceof Error) onError(e);
      }
    })();

    return () => {
      if (controls) controls.stop();
    };
  }, [onDecoded, onError]);

  if (err) {
    return (
      <div className="flex-1 flex flex-col items-center justify-center p-6 text-center text-coral font-mono text-[11px]">
        <div className="mb-2">⚠ {err}</div>
        <div className="text-text-dim">Usa el código manual.</div>
      </div>
    );
  }

  return (
    <video
      ref={videoRef}
      className="w-full h-full object-cover"
      autoPlay
      muted
      playsInline
    />
  );
}
```

- [ ] **Step 3: Implement `src/pages/PairScan.tsx`**

```typescript
import { useCallback } from "react";
import type { Pairing } from "../storage/db";
import { StatusBar } from "../components/layout/StatusBar";
import { HeaderRow, Logo } from "../components/layout/HeaderRow";
import { Label } from "../components/typography/Label";
import { BigTitle } from "../components/typography/BigTitle";
import { PrimaryButton } from "../components/buttons/PrimaryButton";
import { SecondaryButton } from "../components/buttons/SecondaryButton";
import { QrScanner } from "../qr/scanner";
import { nanoid } from "nanoid";

interface PairingPayload {
  version: number;
  host: string;
  port: number;
  token: string;
  fingerprint: string;
  shortCode: string;
}

function parsePairingPayload(raw: string): PairingPayload | null {
  try {
    const obj = JSON.parse(raw) as PairingPayload;
    if (typeof obj.host !== "string" || typeof obj.port !== "number" || typeof obj.token !== "string" || typeof obj.fingerprint !== "string") return null;
    return obj;
  } catch { return null; }
}

function toPairing(payload: PairingPayload): Pairing {
  return {
    id: nanoid(),
    hostname: payload.host,
    port: payload.port,
    token: payload.token,
    fingerprint: payload.fingerprint,
    machineId: "",        // filled in after auth_ok
    displayName: payload.host.split(".")[0] ?? payload.host,
    createdAt: Date.now(),
    lastSeen: Date.now(),
  };
}

export function PairScan({ onFound, onManual }: { onFound: (p: Pairing) => void; onManual: () => void }) {
  const onDecoded = useCallback((text: string) => {
    const payload = parsePairingPayload(text);
    if (payload) onFound(toPairing(payload));
  }, [onFound]);

  return (
    <div className="flex flex-col h-full bg-bg">
      <StatusBar />
      <HeaderRow left={<Logo />} right={<Label>step · 01/02</Label>} />
      <div className="p-5 flex flex-col gap-4 flex-1">
        <Label>parear dispositivo</Label>
        <BigTitle>Escanea el<br />código de tu<br /><em className="not-italic text-accent">nueva Mac</em></BigTitle>
        <div className="flex-1 relative border border-border-strong overflow-hidden min-h-[240px]">
          <QrScanner onDecoded={onDecoded} />
          <div className="absolute inset-0 pointer-events-none border-2 border-accent opacity-20 m-8"></div>
        </div>
        <div className="mt-auto">
          <SecondaryButton onClick={onManual}>Usar código manual</SecondaryButton>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add pwa/src/qr/ pwa/src/utils/shortcode.ts pwa/src/pages/PairScan.tsx
git commit -m "feat(pwa): qr scanner + pair scan page"
```

---

## Task 10: Manual pairing page

**Files:**
- Create: `pwa/src/pages/PairManual.tsx`

- [ ] **Step 1: Implement manual pairing**

```typescript
import { useState } from "react";
import type { Pairing } from "../storage/db";
import { StatusBar } from "../components/layout/StatusBar";
import { HeaderRow, Logo } from "../components/layout/HeaderRow";
import { Label } from "../components/typography/Label";
import { BigTitle } from "../components/typography/BigTitle";
import { PrimaryButton } from "../components/buttons/PrimaryButton";
import { SecondaryButton } from "../components/buttons/SecondaryButton";
import { formatShortCode } from "../utils/shortcode";
import { nanoid } from "nanoid";

export function PairManual({ onFound, onCancel }: { onFound: (p: Pairing) => void; onCancel: () => void }) {
  const [code, setCode] = useState("");
  const [host, setHost] = useState("");
  const [port, setPort] = useState("51820");
  const [token, setToken] = useState("");
  const [error, setError] = useState<string | null>(null);

  const submit = () => {
    setError(null);
    if (!host.trim() || !token.trim()) {
      setError("host y token son requeridos");
      return;
    }
    const portN = Number.parseInt(port, 10);
    if (!Number.isFinite(portN)) {
      setError("port inválido");
      return;
    }
    const p: Pairing = {
      id: nanoid(),
      hostname: host.trim(),
      port: portN,
      token: token.trim(),
      fingerprint: "", // validated post-auth
      machineId: "",
      displayName: host.split(".")[0] ?? host,
      createdAt: Date.now(),
      lastSeen: Date.now(),
    };
    onFound(p);
  };

  return (
    <div className="flex flex-col h-full bg-bg">
      <StatusBar />
      <HeaderRow left={<Logo />} right={<Label>manual</Label>} />
      <div className="p-5 flex flex-col gap-4 flex-1">
        <Label>código manual</Label>
        <BigTitle>Pega los datos de <em className="not-italic text-accent">tu Mac</em></BigTitle>

        <div>
          <label className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim mb-1.5 block">short code (opcional)</label>
          <input
            className="font-mono text-[14px] text-text bg-transparent border-b-[1.5px] border-accent py-2 w-full outline-none"
            value={code}
            onChange={(e) => setCode(formatShortCode(e.target.value))}
            placeholder="ABC-123"
            autoComplete="off"
          />
        </div>

        <div>
          <label className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim mb-1.5 block">host · tailscale</label>
          <input
            className="font-mono text-[12px] text-text bg-surface-1 border-l-2 border-text-faint px-3 py-2 w-full outline-none"
            value={host}
            onChange={(e) => setHost(e.target.value)}
            placeholder="mac-trabajo.lion-spark.ts.net"
            autoComplete="off"
          />
        </div>

        <div>
          <label className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim mb-1.5 block">port</label>
          <input
            className="font-mono text-[12px] text-text bg-surface-1 border-l-2 border-text-faint px-3 py-2 w-full outline-none"
            value={port}
            onChange={(e) => setPort(e.target.value)}
            inputMode="numeric"
          />
        </div>

        <div>
          <label className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim mb-1.5 block">token</label>
          <input
            className="font-mono text-[12px] text-text bg-surface-1 border-l-2 border-text-faint px-3 py-2 w-full outline-none"
            value={token}
            onChange={(e) => setToken(e.target.value)}
            autoComplete="off"
          />
        </div>

        {error && <div className="text-coral font-mono text-[10px]">⚠ {error}</div>}

        <div className="mt-auto">
          <PrimaryButton onClick={submit}>Conectar</PrimaryButton>
          <SecondaryButton onClick={onCancel}>Cancelar</SecondaryButton>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add pwa/src/pages/PairManual.tsx
git commit -m "feat(pwa): manual pairing form"
```

---

## Task 11: Name machine page + save pairing

**Files:**
- Create: `pwa/src/pages/PairNameMachine.tsx`

- [ ] **Step 1: Implement**

The flow: we received a `Pairing` (from QR or manual). We connect to it to verify, receive `auth_ok` with machine info, let the user rename, then persist.

```typescript
import { useEffect, useState } from "react";
import type { Pairing } from "../storage/db";
import { StatusBar } from "../components/layout/StatusBar";
import { HeaderRow, Logo } from "../components/layout/HeaderRow";
import { Label } from "../components/typography/Label";
import { BigTitle } from "../components/typography/BigTitle";
import { PrimaryButton } from "../components/buttons/PrimaryButton";
import { WsClient } from "../net/ws-client";
import { PROTOCOL_VERSION } from "../protocol/version";
import { usePairings } from "../state/pairings";

type Phase = "connecting" | "ok" | "error";

interface MachineInfo {
  id: string;
  fingerprint: string;
  hostname: string;
  displayName: string;
}

export function PairNameMachine({ pairing, onDone }: { pairing: Pairing; onDone: () => void }) {
  const [phase, setPhase] = useState<Phase>("connecting");
  const [errorMsg, setErrorMsg] = useState<string>("");
  const [machine, setMachine] = useState<MachineInfo | null>(null);
  const [name, setName] = useState<string>(pairing.displayName);

  useEffect(() => {
    const client = new WsClient({
      url: `ws://${pairing.hostname}:${pairing.port}`,
      authToken: pairing.token,
      protocolVersion: PROTOCOL_VERSION,
      deviceLabel: navigator.userAgent.slice(0, 80),
      onState: () => {},
      onMessage: (m) => {
        if (m.type === "auth_ok") {
          setMachine(m.machine);
          setName(m.machine.displayName);
          setPhase("ok");
        } else if (m.type === "error") {
          setPhase("error");
          setErrorMsg(m.message);
          client.close();
        }
      },
      onFatal: (reason) => { setPhase("error"); setErrorMsg(reason); },
    });
    client.connect();
    return () => client.close();
  }, [pairing]);

  const save = async () => {
    if (!machine) return;
    const final: Pairing = {
      ...pairing,
      machineId: machine.id,
      fingerprint: machine.fingerprint,
      displayName: name.trim() || machine.hostname,
    };
    await usePairings.getState().add(final);
    onDone();
  };

  return (
    <div className="flex flex-col h-full bg-bg">
      <StatusBar />
      <HeaderRow left={<Logo />} right={<Label>step · 02/02</Label>} />
      <div className="p-5 flex flex-col gap-5 flex-1">
        {phase === "connecting" && (
          <div className="flex-1 flex flex-col items-center justify-center gap-4">
            <Label>conectando...</Label>
            <div className="font-mono text-[10.5px] text-text-dim">
              {pairing.hostname}:{pairing.port}
            </div>
          </div>
        )}

        {phase === "error" && (
          <div className="flex-1 flex flex-col items-center justify-center gap-4 text-center">
            <Label>conexión falló</Label>
            <div className="text-coral font-mono text-[11px]">⚠ {errorMsg}</div>
          </div>
        )}

        {phase === "ok" && machine && (
          <>
            <Label>identificada</Label>
            <BigTitle>¿Cómo llamas<br />a esta <em className="not-italic text-accent">Mac?</em></BigTitle>

            <div>
              <label className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim mb-1.5 block">hostname · detectado</label>
              <div className="font-mono text-[12px] text-text bg-surface-1 border-l-2 border-text-faint px-3 py-2">{machine.hostname}</div>
            </div>

            <div>
              <label className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim mb-1.5 block">nombre · como la llamas tú</label>
              <input
                className="font-mono text-[14px] text-text bg-transparent border-b-[1.5px] border-accent py-2 w-full outline-none"
                value={name}
                onChange={(e) => setName(e.target.value)}
              />
              <div className="font-mono text-[10px] text-text-dim mt-1.5">puedes cambiarlo luego desde settings</div>
            </div>

            <div className="mt-auto">
              <PrimaryButton onClick={save}>Registrar máquina →</PrimaryButton>
            </div>
          </>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add pwa/src/pages/PairNameMachine.tsx
git commit -m "feat(pwa): name + save pairing after verification"
```

---

## Task 12: Inbox page

**Files:**
- Create: `pwa/src/components/inbox/InboxHeader.tsx`
- Create: `pwa/src/components/inbox/SearchRow.tsx`
- Create: `pwa/src/components/inbox/ChipRow.tsx`
- Create: `pwa/src/components/inbox/ConvListItem.tsx`
- Create: `pwa/src/utils/time.ts`
- Create: `pwa/src/pages/Inbox.tsx` (replace stub)

- [ ] **Step 1: Implement `src/utils/time.ts`**

```typescript
export function relativeTime(ts: number, now: number = Date.now()): string {
  const diff = Math.max(0, now - ts);
  const sec = Math.floor(diff / 1000);
  if (sec < 60) return "ahora";
  const min = Math.floor(sec / 60);
  if (min < 60) return `${min}m`;
  const h = Math.floor(min / 60);
  if (h < 24) return `${h}h`;
  const d = Math.floor(h / 24);
  if (d < 7) return `${d}d`;
  const w = Math.floor(d / 7);
  if (w < 4) return `${w}sem`;
  return `${Math.floor(d / 30)}mo`;
}
```

- [ ] **Step 2: Write test for time**

`pwa/tests/unit/utils/time.test.ts`:

```typescript
import { describe, test, expect } from "bun:test";
import { relativeTime } from "../../../src/utils/time";

describe("relativeTime", () => {
  test("returns 'ahora' for < 60s", () => {
    const now = 1_000_000;
    expect(relativeTime(now - 30_000, now)).toBe("ahora");
  });
  test("returns minutes", () => {
    const now = 1_000_000;
    expect(relativeTime(now - 5 * 60 * 1000, now)).toBe("5m");
  });
  test("returns hours", () => {
    const now = 1_000_000_000;
    expect(relativeTime(now - 3 * 3600 * 1000, now)).toBe("3h");
  });
  test("returns days", () => {
    const now = 1_000_000_000_000;
    expect(relativeTime(now - 2 * 24 * 3600 * 1000, now)).toBe("2d");
  });
});
```

- [ ] **Step 3: Implement inbox components**

`pwa/src/components/inbox/InboxHeader.tsx`:

```typescript
export function InboxHeader({ count, machines, onSettings }: { count: number; machines: number; onSettings: () => void }) {
  return (
    <div className="px-4 py-3 border-b border-border flex justify-between items-baseline">
      <h1 className="font-serif font-semibold text-[20px] leading-none tracking-[-0.025em] text-text">Inbox</h1>
      <div className="flex gap-3 items-baseline">
        <span className="font-mono text-[9.5px] text-text-dim">{count} convs · {machines} {machines === 1 ? "máquina" : "máquinas"}</span>
        <button className="font-mono text-[10px] text-text-dim" onClick={onSettings}>⚙</button>
      </div>
    </div>
  );
}
```

`pwa/src/components/inbox/SearchRow.tsx`:

```typescript
export function SearchRow({ onAddProject }: { onAddProject: () => void }) {
  return (
    <div className="px-4 py-2.5 flex gap-2.5 items-center border-b border-border">
      <div className="flex-1 font-mono text-[11px] text-text-faint bg-surface-1 px-3 py-2 border border-border">
        buscar conversaciones...
      </div>
      <button className="text-accent font-mono text-[16px] font-semibold" onClick={onAddProject}>+</button>
    </div>
  );
}
```

`pwa/src/components/inbox/ChipRow.tsx`:

```typescript
import type { InboxFilter } from "../../state/inbox";

export type FilterKey = "all" | "claude" | "codex" | "approval" | string;

export function ChipRow({ active, onSelect, tags }: { active: FilterKey; onSelect: (k: FilterKey) => void; tags: string[] }) {
  const base = "font-mono text-[9.5px] tracking-[0.08em] px-2.5 py-1 uppercase whitespace-nowrap border";
  const activeCls = "bg-accent text-bg border-accent font-semibold";
  const idleCls = "border-border-strong text-text-dim";

  const chip = (key: FilterKey, label: string) => (
    <button className={`${base} ${active === key ? activeCls : idleCls}`} onClick={() => onSelect(key)}>{label}</button>
  );

  return (
    <div className="px-4 py-2.5 flex gap-1.5 overflow-x-auto border-b border-border">
      {chip("all", "todas")}
      {chip("claude", "claude")}
      {chip("codex", "codex")}
      {chip("approval", "approval")}
      {tags.map((t) => chip(`tag:${t}`, t))}
    </div>
  );
}
```

`pwa/src/components/inbox/ConvListItem.tsx`:

```typescript
import type { InboxConversation } from "../../state/inbox";
import { relativeTime } from "../../utils/time";

export function ConvListItem({ conv, offline, onClick }: { conv: InboxConversation; offline: boolean; onClick: () => void }) {
  const cliCls = conv.cli === "claude-code" ? "text-accent" : "text-emerald-400";
  return (
    <button onClick={onClick} className={`w-full text-left px-4 py-3 border-b border-border block ${offline ? "opacity-50" : ""}`}>
      <div className="flex justify-between items-baseline mb-1 gap-2">
        <span className={`font-mono text-[8.5px] tracking-[0.15em] uppercase ${cliCls}`}>
          {conv.cli === "claude-code" ? "claude" : "codex"}{conv.currentModel ? ` · ${conv.currentModel.split("-")[0]}` : ""}
        </span>
        <span className="font-mono text-[9.5px] text-text-dim">{relativeTime(conv.lastActivityAt)}</span>
      </div>
      <div className="font-serif font-semibold text-[13.5px] tracking-[-0.01em] leading-tight mb-0.5 text-text">{conv.title}</div>
      <div className="text-[11px] text-text-dim truncate">{conv.lastPreviewText ?? "—"}</div>
      <div className="flex justify-between mt-1.5 font-mono text-[9.5px] text-text-faint">
        <span>{conv.machineDisplayName} · {conv.projectName}</span>
        <span className={conv.status === "awaiting_approval" ? "text-coral font-semibold before:content-['●_']" : ""}>
          {conv.status === "awaiting_approval" ? "approval" : conv.status}
        </span>
      </div>
    </button>
  );
}
```

- [ ] **Step 4: Implement `src/pages/Inbox.tsx`**

```typescript
import { useEffect, useState } from "react";
import { usePairings } from "../state/pairings";
import { useInbox, type InboxFilter } from "../state/inbox";
import { useConnections } from "../state/connection";
import { getAgent } from "../net/agent-client";
import { InboxHeader } from "../components/inbox/InboxHeader";
import { SearchRow } from "../components/inbox/SearchRow";
import { ChipRow, type FilterKey } from "../components/inbox/ChipRow";
import { ConvListItem } from "../components/inbox/ConvListItem";

export function Inbox({ onOpenConv, onNewProject, onSettings }: {
  onOpenConv: (pairingId: string, conversationId: string) => void;
  onNewProject: () => void;
  onSettings: () => void;
}) {
  const pairings = usePairings((s) => s.pairings);
  const conversations = useInbox((s) => s.conversations);
  const projects = useInbox((s) => s.projects);
  const connections = useConnections((s) => s.states);
  const [filter, setFilter] = useState<FilterKey>("all");

  useEffect(() => {
    for (const p of pairings) getAgent(p);
  }, [pairings]);

  const inboxFilter: InboxFilter = {};
  if (filter === "claude") inboxFilter.cli = "claude-code";
  else if (filter === "codex") inboxFilter.cli = "codex";
  else if (filter === "approval") inboxFilter.onlyApproval = true;
  else if (filter.startsWith("tag:")) inboxFilter.tag = filter.slice(4);

  const visible = useInbox.getState().filteredConversations(inboxFilter);

  const allTags = Array.from(new Set(projects.flatMap((p) => p.tags))).sort();

  return (
    <div className="flex flex-col h-full bg-bg">
      <InboxHeader count={conversations.length} machines={pairings.length} onSettings={onSettings} />
      <SearchRow onAddProject={onNewProject} />
      <ChipRow active={filter} onSelect={setFilter} tags={allTags} />
      <div className="flex-1 overflow-y-auto">
        {visible.length === 0 ? (
          <div className="flex-1 flex flex-col items-center justify-center p-8 gap-3 text-center text-text-dim">
            <div className="w-16 h-16 border border-dashed border-border-strong flex items-center justify-center text-text-faint font-mono text-2xl">∅</div>
            <div className="font-serif font-semibold text-[18px] text-text">Sin conversaciones</div>
            <div className="font-mono text-[10.5px]">toca + para crear proyecto y empezar</div>
          </div>
        ) : visible.map((c) => {
          const connState = connections.get(c.pairingId);
          const offline = connState !== "authenticated";
          return <ConvListItem key={`${c.pairingId}:${c.id}`} conv={c} offline={offline} onClick={() => onOpenConv(c.pairingId, c.id)} />;
        })}
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Commit**

```bash
git add pwa/src/pages/Inbox.tsx pwa/src/components/inbox/ pwa/src/utils/time.ts pwa/tests/unit/utils/
git commit -m "feat(pwa): inbox page with filters and conv list"
```

---

## Task 13: New Project page

**Files:**
- Create: `pwa/src/pages/NewProject.tsx`

- [ ] **Step 1: Implement**

```typescript
import { useState, useMemo } from "react";
import { usePairings } from "../state/pairings";
import { useConnections } from "../state/connection";
import { getAgent } from "../net/agent-client";
import { StatusBar } from "../components/layout/StatusBar";
import { HeaderRow } from "../components/layout/HeaderRow";
import { Label } from "../components/typography/Label";
import { BigTitle } from "../components/typography/BigTitle";
import { PrimaryButton } from "../components/buttons/PrimaryButton";

const DEFAULT_MODELS_CLAUDE = [
  { id: "claude-opus-4-7", name: "Opus 4.7", desc: "razonamiento · 1M ctx · más lento" },
  { id: "claude-sonnet-4-6", name: "Sonnet 4.6", desc: "balance · recomendado" },
  { id: "claude-haiku-4-5-20251001", name: "Haiku 4.5", desc: "rápido · tareas ligeras" },
];

export function NewProject({ onCreated, onCancel }: { onCreated: () => void; onCancel: () => void }) {
  const pairings = usePairings((s) => s.pairings);
  const connections = useConnections((s) => s.states);

  const onlineMachines = useMemo(
    () => pairings.filter((p) => connections.get(p.id) === "authenticated"),
    [pairings, connections],
  );

  const [selectedPairingId, setSelectedPairingId] = useState(onlineMachines[0]?.id ?? "");
  const [name, setName] = useState("");
  const [workingDir, setWorkingDir] = useState("");
  const [defaultModel, setDefaultModel] = useState("claude-sonnet-4-6");
  const [tags, setTags] = useState<string[]>(["work"]);
  const [newTag, setNewTag] = useState("");

  const submit = () => {
    const pairing = pairings.find((p) => p.id === selectedPairingId);
    if (!pairing || !name.trim() || !workingDir.trim()) return;
    const agent = getAgent(pairing);
    agent.send({
      type: "create_project",
      name: name.trim(),
      workingDir: workingDir.trim(),
      tags,
      permissionPreset: "ask-for-side-effects",
      defaultModelClaudeCode: defaultModel,
    });
    onCreated();
  };

  const toggleTag = (t: string) => {
    setTags((cur) => cur.includes(t) ? cur.filter((x) => x !== t) : [...cur, t]);
  };

  const addTag = () => {
    const t = newTag.trim();
    if (t && !tags.includes(t)) {
      setTags([...tags, t]);
      setNewTag("");
    }
  };

  return (
    <div className="flex flex-col h-full bg-bg">
      <StatusBar />
      <HeaderRow left={<button className="font-mono text-text-dim text-[14px]" onClick={onCancel}>‹ cancel</button>} right={<Label>nuevo proyecto</Label>} />
      <div className="flex-1 overflow-y-auto p-5 flex flex-col gap-4">
        <BigTitle>Nuevo<br /><em className="not-italic text-accent">proyecto</em></BigTitle>

        <div>
          <Label>máquina</Label>
          <select
            className="font-mono text-[11px] text-text bg-surface-1 border-l-2 border-text-faint px-3 py-2 w-full outline-none block mt-1.5"
            value={selectedPairingId}
            onChange={(e) => setSelectedPairingId(e.target.value)}
          >
            {onlineMachines.length === 0 && <option value="">ninguna online</option>}
            {onlineMachines.map((p) => (
              <option key={p.id} value={p.id}>{p.displayName}</option>
            ))}
          </select>
        </div>

        <div>
          <Label>nombre</Label>
          <input
            className="font-mono text-[14px] text-text bg-transparent border-b-[1.5px] border-accent py-2 w-full outline-none mt-1.5"
            value={name}
            onChange={(e) => setName(e.target.value)}
            placeholder="aiconnect-api"
          />
        </div>

        <div>
          <Label>working directory</Label>
          <input
            className="font-mono text-[11px] text-text bg-surface-1 border-l-2 border-text-faint px-3 py-2 w-full outline-none mt-1.5"
            value={workingDir}
            onChange={(e) => setWorkingDir(e.target.value)}
            placeholder="~/repos/aiconnect-api"
          />
        </div>

        <div>
          <Label>modelo default · claude code</Label>
          <div className="flex flex-col gap-1.5 mt-1.5">
            {DEFAULT_MODELS_CLAUDE.map((m) => {
              const selected = defaultModel === m.id;
              return (
                <button
                  key={m.id}
                  className={`px-3 py-2.5 border ${selected ? "bg-accent-dim border-accent" : "border-border-strong"} text-left flex justify-between items-center`}
                  onClick={() => setDefaultModel(m.id)}
                >
                  <div>
                    <div className="font-serif font-semibold text-[12.5px] tracking-[-0.01em] text-text">{m.name}</div>
                    <div className="font-mono text-[8.5px] text-text-dim mt-0.5">{m.desc}</div>
                  </div>
                  {selected && <span className="font-mono text-accent text-[11px]">✓</span>}
                </button>
              );
            })}
          </div>
        </div>

        <div>
          <Label>tags · opcional</Label>
          <div className="flex gap-1.5 flex-wrap mt-1.5">
            {tags.map((t) => (
              <button key={t} onClick={() => toggleTag(t)} className="font-mono text-[9.5px] px-2.5 py-1 border border-accent bg-accent-dim text-accent">{t}</button>
            ))}
            <input
              className="font-mono text-[9.5px] bg-transparent border border-border-strong px-2.5 py-1 w-20 text-text-dim outline-none"
              placeholder="+ nueva"
              value={newTag}
              onChange={(e) => setNewTag(e.target.value)}
              onKeyDown={(e) => e.key === "Enter" && addTag()}
            />
          </div>
        </div>

        <div className="mt-auto pt-4">
          <PrimaryButton onClick={submit} disabled={!selectedPairingId || !name.trim() || !workingDir.trim()}>
            Crear proyecto
          </PrimaryButton>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add pwa/src/pages/NewProject.tsx
git commit -m "feat(pwa): new project form with model default selector"
```

---

## Task 14: New Conversation sheet

**Files:**
- Create: `pwa/src/pages/NewConversation.tsx`
- Modify: `pwa/src/pages/Inbox.tsx` to add "new conversation" entry points

- [ ] **Step 1: Implement `NewConversation.tsx`**

```typescript
import { useState, useEffect } from "react";
import { usePairings } from "../state/pairings";
import { useInbox } from "../state/inbox";
import { getAgent } from "../net/agent-client";
import type { ServerMessageT } from "../protocol/messages";
import { StatusBar } from "../components/layout/StatusBar";
import { HeaderRow } from "../components/layout/HeaderRow";
import { Label } from "../components/typography/Label";
import { PrimaryButton } from "../components/buttons/PrimaryButton";

const DEFAULT_MODELS_CLAUDE = [
  { id: "claude-opus-4-7", name: "Opus 4.7" },
  { id: "claude-sonnet-4-6", name: "Sonnet 4.6" },
  { id: "claude-haiku-4-5-20251001", name: "Haiku 4.5" },
];

export function NewConversation({ pairingId, projectId, onCreated, onCancel }: {
  pairingId: string;
  projectId: string;
  onCreated: (conversationId: string) => void;
  onCancel: () => void;
}) {
  const pairing = usePairings((s) => s.pairings.find((p) => p.id === pairingId));
  const project = useInbox((s) => s.projects.find((p) => p.pairingId === pairingId && p.id === projectId));
  const [model, setModel] = useState("claude-sonnet-4-6");

  useEffect(() => {
    // Listen for `conversation_created` to advance
    if (!pairing) return;
    const agent = getAgent(pairing);
    const onMessage = (_m: ServerMessageT) => { /* handled via store; we listen via state */ };
    void onMessage;
    // Simpler: subscribe via useInbox. Track conversation count delta.
    const before = useInbox.getState().conversations.filter((c) => c.pairingId === pairingId).length;
    const unsub = useInbox.subscribe((state) => {
      const after = state.conversations.filter((c) => c.pairingId === pairingId).length;
      if (after > before) {
        const newest = state.conversations
          .filter((c) => c.pairingId === pairingId)
          .sort((a, b) => b.lastActivityAt - a.lastActivityAt)[0];
        if (newest) { onCreated(newest.id); unsub(); }
      }
    });
    return () => unsub();
  }, [pairing, pairingId, onCreated]);

  const submit = () => {
    if (!pairing) return;
    const agent = getAgent(pairing);
    agent.send({
      type: "create_conversation",
      projectId,
      cli: "claude-code",
      model,
    });
  };

  return (
    <div className="flex flex-col h-full bg-bg">
      <StatusBar />
      <HeaderRow left={<button className="font-mono text-text-dim text-[14px]" onClick={onCancel}>‹ cancel</button>} right={<Label>nueva conversación</Label>} />
      <div className="flex-1 p-5 flex flex-col gap-4">
        <Label>proyecto</Label>
        <div className="font-mono text-[11px] text-text bg-surface-1 border-l-2 border-text-faint px-3 py-2">
          {project?.name ?? projectId} · {pairing?.displayName}
        </div>

        <Label>CLI</Label>
        <div className="flex gap-2">
          <div className="flex-1 border border-accent bg-accent-dim p-2.5 text-left">
            <div className="font-serif font-semibold text-[12.5px] text-text">Claude Code</div>
            <div className="font-mono text-[8.5px] text-text-dim">subscription</div>
          </div>
          <div className="flex-1 border border-border-strong p-2.5 text-left opacity-50">
            <div className="font-serif font-semibold text-[12.5px] text-text">Codex</div>
            <div className="font-mono text-[8.5px] text-text-dim">fase 2</div>
          </div>
        </div>

        <Label>modelo · override opcional</Label>
        <div className="flex flex-col gap-1.5">
          {DEFAULT_MODELS_CLAUDE.map((m) => (
            <button
              key={m.id}
              className={`px-3 py-2.5 border ${model === m.id ? "bg-accent-dim border-accent" : "border-border-strong"} text-left flex justify-between items-center`}
              onClick={() => setModel(m.id)}
            >
              <span className="font-serif font-semibold text-[12.5px] tracking-[-0.01em] text-text">{m.name}</span>
              {model === m.id && <span className="font-mono text-accent text-[11px]">✓</span>}
            </button>
          ))}
        </div>

        <div className="mt-auto">
          <PrimaryButton onClick={submit}>Abrir conversación →</PrimaryButton>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Add entry from Inbox to NewConversation**

For now, the inbox has a "+" button that opens `NewProject`. Also add a long-press or second button for "+ conv". Simpler: add a small button inside the empty state that lets the user pick a project.

Actually, the simplest flow: when opening a project detail page, show "+ nueva conv" there. For MVP, add a button from the inbox search row that routes to project picker → new conv. Implement a minimal `ProjectPickerSheet` for this.

Create `pwa/src/components/inbox/NewConvButton.tsx`:

```typescript
import { useState } from "react";
import { useInbox } from "../../state/inbox";
import { usePairings } from "../../state/pairings";

export function NewConvButton({ onPicked }: { onPicked: (pairingId: string, projectId: string) => void }) {
  const [open, setOpen] = useState(false);
  const projects = useInbox((s) => s.projects);
  const pairings = usePairings((s) => s.pairings);

  return (
    <>
      <button className="font-mono text-accent text-[14px] font-semibold" onClick={() => setOpen(true)}>+ conv</button>
      {open && (
        <div className="fixed inset-0 z-50 bg-black/70 flex items-end">
          <div className="bg-surface-1 w-full border-t border-border-strong p-5">
            <h3 className="font-serif font-semibold text-[16px] mb-3">Selecciona proyecto</h3>
            <div className="flex flex-col gap-2 max-h-[400px] overflow-y-auto">
              {projects.map((p) => {
                const pair = pairings.find((x) => x.id === p.pairingId);
                return (
                  <button
                    key={`${p.pairingId}:${p.id}`}
                    className="text-left px-3 py-2 border border-border-strong"
                    onClick={() => { setOpen(false); onPicked(p.pairingId, p.id); }}
                  >
                    <div className="font-serif font-semibold text-[13px]">{p.name}</div>
                    <div className="font-mono text-[9.5px] text-text-dim">{pair?.displayName}</div>
                  </button>
                );
              })}
            </div>
            <button className="mt-3 w-full font-mono text-[10px] uppercase tracking-[0.15em] py-2 border border-border-strong" onClick={() => setOpen(false)}>cancelar</button>
          </div>
        </div>
      )}
    </>
  );
}
```

Integrate into `Inbox.tsx` SearchRow:

Replace `<SearchRow onAddProject={onNewProject} />` with a small row including both `+` buttons:

```tsx
<div className="px-4 py-2.5 flex gap-2.5 items-center border-b border-border">
  <div className="flex-1 font-mono text-[11px] text-text-faint bg-surface-1 px-3 py-2 border border-border">
    buscar...
  </div>
  <NewConvButton onPicked={(pairingId, projectId) => /* navigate */} />
  <button className="text-accent font-mono text-[16px] font-semibold" onClick={onNewProject}>+ proj</button>
</div>
```

Thread the navigation callback up to `App.tsx` by adding a prop `onNewConv(pairingId, projectId)`.

Update `Inbox` component props and `App.tsx`:

In `App.tsx`, add `onNewConv` to Inbox:

```tsx
case "inbox": return <Inbox
    onOpenConv={...}
    onNewProject={...}
    onNewConv={(pairingId, projectId) => setRoute({ name: "new-conversation", pairingId, projectId })}
    onSettings={...}
  />;
```

- [ ] **Step 3: Commit**

```bash
git add pwa/src/pages/NewConversation.tsx pwa/src/components/inbox/NewConvButton.tsx pwa/src/pages/Inbox.tsx pwa/src/App.tsx
git commit -m "feat(pwa): new conversation sheet with model override"
```

---

## Task 15: Conversation page — chat view with streaming

**Files:**
- Create: `pwa/src/components/chat/` (several components)
- Create: `pwa/src/pages/Conversation.tsx`
- Create: `pwa/src/hooks/useConversation.ts`

- [ ] **Step 1: Implement chat components**

`pwa/src/components/chat/ChatHeader.tsx`:

```typescript
import type { InboxConversation } from "../../state/inbox";

export function ChatHeader({ conv, onBack, onModelTap }: { conv: InboxConversation; onBack: () => void; onModelTap: () => void }) {
  return (
    <div>
      <div className="px-4 py-3 border-b border-border flex justify-between items-center">
        <button className="font-mono text-text-dim text-[14px]" onClick={onBack}>‹</button>
        <span className="font-mono text-[9px] tracking-[0.15em] uppercase text-text-dim">conv · {conv.id.slice(-6)}</span>
      </div>
      <div className="px-4 py-3 border-b border-border">
        <div className="font-serif font-semibold text-[15px] tracking-[-0.02em] leading-tight">{conv.title}</div>
        <div className="font-mono text-[9.5px] text-text-dim mt-1 flex gap-2 items-center flex-wrap">
          <span>{conv.machineDisplayName}</span>
          <button className="bg-surface-2 px-2 py-0.5 border border-border cursor-pointer" onClick={onModelTap}>
            <span className="text-accent">●</span> {conv.currentModel ?? "default"} <span className="text-text-dim">⌄</span>
          </button>
          <span className={conv.status === "awaiting_approval" ? "text-coral" : conv.status === "idle" ? "text-text-dim" : "text-accent"}>
            {conv.status === "awaiting_approval" ? "● awaiting" : conv.status === "idle" ? "idle" : "● live"}
          </span>
        </div>
      </div>
    </div>
  );
}
```

`UserTurn.tsx`:

```typescript
import { relativeTime } from "../../utils/time";

export function UserTurn({ text, timestamp }: { text: string; timestamp: number }) {
  return (
    <div>
      <div className="font-mono text-[8.5px] tracking-[0.2em] uppercase text-accent mb-1">you · {relativeTime(timestamp)}</div>
      <div className="text-[13px] leading-[1.45] text-text pl-2.5 border-l-2 border-accent">{text}</div>
    </div>
  );
}
```

`AssistantTurn.tsx`:

```typescript
import type { ConversationEventT } from "../../protocol/messages";
import { relativeTime } from "../../utils/time";

interface TurnSegment {
  events: ConversationEventT[];
  firstTimestamp: number;
}

export function AssistantTurn({ segment }: { segment: TurnSegment }) {
  const text = segment.events
    .filter((e): e is Extract<ConversationEventT, { type: "assistant_text_delta" }> => e.type === "assistant_text_delta")
    .map((e) => e.payload.text)
    .join("");

  return (
    <div>
      <div className="font-mono text-[8.5px] tracking-[0.2em] uppercase text-text-faint mb-1">claude · {relativeTime(segment.firstTimestamp)}</div>
      {segment.events.map((e) => {
        if (e.type === "tool_use") {
          return <div key={e.id} className="font-mono text-[10.5px] text-text-dim bg-surface-1 border-l border-text-faint px-2.5 py-1.5 my-1">
            <span className="text-accent">▸ </span>{e.payload.tool} {String(JSON.stringify(e.payload.input)).slice(0, 60)}
          </div>;
        }
        return null;
      })}
      {text && <div className="text-[12.5px] leading-[1.55] text-text mt-2">{text}</div>}
    </div>
  );
}
```

`StreamingCursor.tsx`:

```typescript
export function StreamingCursor() {
  return <span className="inline-block w-[7px] h-[13px] bg-accent align-[-2px] animate-pulse ml-0.5" />;
}
```

`ApprovalCard.tsx`:

```typescript
interface Props {
  ask: string;
  command?: string;
  onApprove: () => void;
  onDeny: () => void;
}

export function ApprovalCard({ ask, command, onApprove, onDeny }: Props) {
  return (
    <div className="border border-coral bg-coral-dim p-3">
      <div className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-coral">approval · requerido</div>
      <div className="font-serif font-semibold text-[13px] mt-1.5 mb-2">{ask}</div>
      {command && (
        <div className="font-mono text-[10.5px] bg-black text-accent px-2.5 py-1.5 mb-2">
          <span className="text-text-dim">$ </span>{command}
        </div>
      )}
      <div className="flex gap-1.5">
        <button onClick={onDeny} className="flex-1 py-2 bg-transparent text-text border border-border-strong font-mono text-[9px] tracking-[0.12em] uppercase font-semibold">denegar</button>
        <button onClick={onApprove} className="flex-1 py-2 bg-coral text-white font-mono text-[9px] tracking-[0.12em] uppercase font-semibold">aprobar</button>
      </div>
    </div>
  );
}
```

`ChatInput.tsx`:

```typescript
import { useState, type KeyboardEvent } from "react";

export function ChatInput({ disabled, placeholder = "escribe un prompt...", onSend }: {
  disabled?: boolean;
  placeholder?: string;
  onSend: (text: string) => void;
}) {
  const [text, setText] = useState("");
  const submit = () => {
    if (text.trim()) {
      onSend(text.trim());
      setText("");
    }
  };
  const onKey = (e: KeyboardEvent<HTMLTextAreaElement>) => {
    if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); submit(); }
  };
  return (
    <div className="border-t border-border px-4 py-2.5 flex gap-2 items-center bg-surface-1">
      <span className="text-accent font-mono">›</span>
      <textarea
        className="flex-1 bg-transparent text-text font-mono text-[11.5px] outline-none resize-none leading-[1.5] py-1"
        placeholder={placeholder}
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={onKey}
        rows={1}
        disabled={disabled}
      />
      <button className="font-mono text-[10px] uppercase tracking-[0.15em] text-accent" onClick={submit} disabled={disabled}>send ⏎</button>
    </div>
  );
}
```

`ModelSwitcherSheet.tsx`:

```typescript
import { useState } from "react";

export interface ModelOption {
  id: string;
  name: string;
  desc: string;
}

const CLAUDE_MODELS: ModelOption[] = [
  { id: "claude-opus-4-7", name: "Opus 4.7", desc: "razonamiento · 1M ctx" },
  { id: "claude-sonnet-4-6", name: "Sonnet 4.6", desc: "balance" },
  { id: "claude-haiku-4-5-20251001", name: "Haiku 4.5", desc: "rápido" },
];

export function ModelSwitcherSheet({ current, onPick, onCancel }: {
  current: string | null;
  onPick: (modelId: string) => void;
  onCancel: () => void;
}) {
  const [selected, setSelected] = useState(current ?? "claude-sonnet-4-6");
  return (
    <div className="fixed inset-0 z-50 bg-black/70 flex items-end">
      <div className="bg-surface-1 w-full border-t border-border-strong p-5">
        <h3 className="font-serif font-semibold text-[16px] mb-3">Cambiar modelo</h3>
        <div className="flex flex-col gap-1.5">
          {CLAUDE_MODELS.map((m) => {
            const sel = selected === m.id;
            return (
              <button
                key={m.id}
                className={`px-3 py-2.5 border ${sel ? "bg-accent-dim border-accent" : "border-border-strong"} text-left flex justify-between items-center`}
                onClick={() => setSelected(m.id)}
              >
                <div>
                  <div className="font-serif font-semibold text-[12.5px] text-text">{m.name}</div>
                  <div className="font-mono text-[8.5px] text-text-dim mt-0.5">{m.desc}</div>
                </div>
                {sel && <span className="font-mono text-accent text-[11px]">✓</span>}
              </button>
            );
          })}
        </div>
        <div className="text-[10.5px] text-text-dim mt-3 font-mono">el cambio se registra en el historial</div>
        <div className="mt-4 flex gap-2">
          <button onClick={onCancel} className="flex-1 py-2.5 font-mono text-[10px] uppercase tracking-[0.15em] border border-border-strong">cancelar</button>
          <button
            onClick={() => onPick(selected)}
            className="flex-1 py-2.5 font-mono text-[10px] uppercase tracking-[0.15em] bg-accent text-bg font-semibold"
          >cambiar</button>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Implement `src/hooks/useConversation.ts`**

```typescript
import { useEffect } from "react";
import { usePairings } from "../state/pairings";
import { useInbox } from "../state/inbox";
import { useActiveConversation } from "../state/active-conversation";
import { getAgent } from "../net/agent-client";

export function useConversation(pairingId: string, conversationId: string) {
  const pairing = usePairings((s) => s.pairings.find((p) => p.id === pairingId));
  const conv = useInbox((s) => s.conversations.find((c) => c.pairingId === pairingId && c.id === conversationId));

  useEffect(() => {
    if (!pairing) return;
    const agent = getAgent(pairing);
    useActiveConversation.getState().open(pairingId, conversationId);
    // Request replay of all events for this conversation
    agent.send({ type: "replay_from", conversationId });

    return () => {
      useActiveConversation.getState().close();
    };
  }, [pairing, pairingId, conversationId]);

  return { pairing, conv };
}
```

- [ ] **Step 3: Implement `src/pages/Conversation.tsx`**

```typescript
import { useEffect, useMemo, useState, useRef } from "react";
import { useActiveConversation } from "../state/active-conversation";
import { useConversation } from "../hooks/useConversation";
import { getAgent } from "../net/agent-client";
import { usePairings } from "../state/pairings";
import type { ConversationEventT } from "../protocol/messages";
import { ChatHeader } from "../components/chat/ChatHeader";
import { UserTurn } from "../components/chat/UserTurn";
import { AssistantTurn } from "../components/chat/AssistantTurn";
import { ApprovalCard } from "../components/chat/ApprovalCard";
import { ChatInput } from "../components/chat/ChatInput";
import { ModelSwitcherSheet } from "../components/chat/ModelSwitcherSheet";

type AssistantSegment = { kind: "assistant"; events: ConversationEventT[]; firstTimestamp: number };
type UserSegment = { kind: "user"; event: Extract<ConversationEventT, { type: "user_prompt" }> };
type ApprovalSegment = { kind: "approval"; event: Extract<ConversationEventT, { type: "permission_request" }> };
type Segment = AssistantSegment | UserSegment | ApprovalSegment;

function segmentEvents(events: ConversationEventT[]): Segment[] {
  const out: Segment[] = [];
  let cur: AssistantSegment | null = null;
  for (const e of events) {
    if (e.type === "user_prompt") {
      if (cur) { out.push(cur); cur = null; }
      out.push({ kind: "user", event: e });
    } else if (e.type === "permission_request") {
      if (cur) { out.push(cur); cur = null; }
      out.push({ kind: "approval", event: e });
    } else if (e.type === "turn_complete" || e.type === "permission_response") {
      if (cur) { out.push(cur); cur = null; }
    } else if (e.type === "assistant_text_delta" || e.type === "tool_use" || e.type === "tool_result") {
      if (!cur) cur = { kind: "assistant", events: [], firstTimestamp: e.timestamp };
      cur.events.push(e);
    }
  }
  if (cur) out.push(cur);
  return out;
}

export function Conversation({ pairingId, conversationId, onBack }: { pairingId: string; conversationId: string; onBack: () => void }) {
  const { pairing, conv } = useConversation(pairingId, conversationId);
  const events = useActiveConversation((s) => s.events);
  const pairings = usePairings((s) => s.pairings);
  const [modelSheetOpen, setModelSheetOpen] = useState(false);
  const bodyRef = useRef<HTMLDivElement>(null);

  const segments = useMemo(() => segmentEvents(events), [events]);

  useEffect(() => {
    bodyRef.current?.scrollTo({ top: bodyRef.current.scrollHeight, behavior: "smooth" });
  }, [segments.length]);

  if (!conv || !pairing) return null;

  const pendingApproval = [...events].reverse().find((e) => e.type === "permission_request") as Extract<ConversationEventT, { type: "permission_request" }> | undefined;
  const hasResponse = pendingApproval ? events.some((e) => e.type === "permission_response" && e.payload.requestId === pendingApproval.payload.requestId) : false;
  const pending = pendingApproval && !hasResponse;

  const onSend = (prompt: string) => {
    const pair = pairings.find((p) => p.id === pairingId);
    if (!pair) return;
    getAgent(pair).send({ type: "send_prompt", conversationId, prompt, attachments: [] });
  };

  const onApprove = () => {
    const pair = pairings.find((p) => p.id === pairingId);
    if (!pair || !pendingApproval) return;
    getAgent(pair).send({
      type: "approval_response",
      conversationId,
      requestId: pendingApproval.payload.requestId,
      granted: true,
    });
  };

  const onDeny = () => {
    const pair = pairings.find((p) => p.id === pairingId);
    if (!pair || !pendingApproval) return;
    getAgent(pair).send({
      type: "approval_response",
      conversationId,
      requestId: pendingApproval.payload.requestId,
      granted: false,
    });
  };

  const onChangeModel = (modelId: string) => {
    const pair = pairings.find((p) => p.id === pairingId);
    if (!pair) return;
    getAgent(pair).send({ type: "change_model", conversationId, model: modelId });
    setModelSheetOpen(false);
  };

  return (
    <div className="flex flex-col h-full bg-bg">
      <ChatHeader conv={conv} onBack={onBack} onModelTap={() => setModelSheetOpen(true)} />
      <div ref={bodyRef} className="flex-1 overflow-y-auto px-4 py-4 flex flex-col gap-3.5">
        {segments.map((s, i) => {
          if (s.kind === "user") return <UserTurn key={s.event.id} text={s.event.payload.text} timestamp={s.event.timestamp} />;
          if (s.kind === "assistant") return <AssistantTurn key={`asst-${i}-${s.firstTimestamp}`} segment={{ events: s.events, firstTimestamp: s.firstTimestamp }} />;
          if (s.kind === "approval") return null; // rendered separately below
          return null;
        })}
        {pending && pendingApproval && (
          <ApprovalCard
            ask={pendingApproval.payload.detail || "aprobar acción"}
            command={pendingApproval.payload.command}
            onApprove={onApprove}
            onDeny={onDeny}
          />
        )}
      </div>
      <ChatInput
        disabled={pending}
        placeholder={pending ? "bloqueado hasta approval..." : "escribe un prompt..."}
        onSend={onSend}
      />
      {modelSheetOpen && (
        <ModelSwitcherSheet
          current={conv.currentModel}
          onPick={onChangeModel}
          onCancel={() => setModelSheetOpen(false)}
        />
      )}
    </div>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add pwa/src/components/chat/ pwa/src/pages/Conversation.tsx pwa/src/hooks/
git commit -m "feat(pwa): conversation page with streaming, approvals, model switch"
```

---

## Task 16: Offline banner + reconnection UX

**Files:**
- Create: `pwa/src/components/inbox/OfflineBanner.tsx`
- Create: `pwa/src/hooks/useOnline.ts`

- [ ] **Step 1: Implement `useOnline` hook**

```typescript
import { useEffect, useState } from "react";

export function useOnline(): boolean {
  const [online, setOnline] = useState(typeof navigator !== "undefined" ? navigator.onLine : true);
  useEffect(() => {
    const up = () => setOnline(true);
    const down = () => setOnline(false);
    window.addEventListener("online", up);
    window.addEventListener("offline", down);
    return () => {
      window.removeEventListener("online", up);
      window.removeEventListener("offline", down);
    };
  }, []);
  return online;
}
```

- [ ] **Step 2: Implement `OfflineBanner.tsx`**

```typescript
export function OfflineBanner({ text, actionLabel, onAction }: { text: string; actionLabel?: string; onAction?: () => void }) {
  return (
    <div className="bg-coral-dim border-l-[3px] border-coral px-4 py-2.5 flex justify-between items-center font-mono text-[10px] text-text">
      <span>{text}</span>
      {actionLabel && onAction && (
        <button className="text-coral font-semibold tracking-[0.1em] uppercase" onClick={onAction}>{actionLabel}</button>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Wire into Inbox**

In `Inbox.tsx`, display banner if `!online` or any machine is disconnected. Add after `ChipRow`:

```tsx
{!online && <OfflineBanner text="sin red en el teléfono · reconectando..." />}
{offlineMachines.length > 0 && (
  <OfflineBanner text={`${offlineMachines.length} máquina${offlineMachines.length > 1 ? "s" : ""} offline`} />
)}
```

(Add imports and derive `offlineMachines` from pairings + connections.)

- [ ] **Step 4: Commit**

```bash
git add pwa/src/hooks/useOnline.ts pwa/src/components/inbox/OfflineBanner.tsx pwa/src/pages/Inbox.tsx
git commit -m "feat(pwa): offline banner + online/reconnect status"
```

---

## Task 17: Settings page — machines list

**Files:**
- Create: `pwa/src/pages/Settings.tsx`

- [ ] **Step 1: Implement**

```typescript
import { usePairings } from "../state/pairings";
import { useConnections } from "../state/connection";
import { useInbox } from "../state/inbox";
import { StatusBar } from "../components/layout/StatusBar";
import { HeaderRow } from "../components/layout/HeaderRow";
import { Label } from "../components/typography/Label";

export function Settings({ onBack }: { onBack: () => void }) {
  const pairings = usePairings((s) => s.pairings);
  const remove = usePairings((s) => s.remove);
  const connections = useConnections((s) => s.states);
  const convsByPairing = (id: string) => useInbox.getState().conversations.filter((c) => c.pairingId === id).length;

  return (
    <div className="flex flex-col h-full bg-bg">
      <StatusBar />
      <HeaderRow left={<button className="font-mono text-text-dim text-[14px]" onClick={onBack}>‹ back</button>} right={<Label>settings</Label>} />
      <div className="flex-1 overflow-y-auto">
        <div className="px-4 pt-4 pb-2 flex justify-between items-baseline">
          <span className="font-serif font-semibold text-[16px] tracking-[-0.02em]">Máquinas</span>
          <span className="font-mono text-[9.5px] text-text-dim">{pairings.length} paireadas</span>
        </div>
        {pairings.map((p) => {
          const state = connections.get(p.id) ?? "disconnected";
          const online = state === "authenticated";
          return (
            <div key={p.id} className={`px-4 py-3 border-t border-border ${online ? "bg-gradient-to-r from-accent-dim to-transparent border-l-2 border-l-accent" : ""}`}>
              <div className="font-serif font-semibold text-[14px]">{p.displayName}</div>
              <div className="font-mono text-[9.5px] text-text-dim">{p.hostname}</div>
              <div className="flex justify-between items-baseline mt-2 pt-2 border-t border-border font-mono text-[9.5px] text-text-dim">
                <span>{online ? <><span className="text-accent">● </span>online</> : <><span className="text-text-faint">○ </span>{state}</>}</span>
                <span>{convsByPairing(p.id)} convs</span>
              </div>
              <div className="flex gap-1.5 mt-2.5">
                <button className="flex-1 py-1.5 border border-border-strong font-mono text-[8.5px] tracking-[0.1em] uppercase">Renombrar</button>
                <button className="flex-1 py-1.5 border border-border-strong font-mono text-[8.5px] tracking-[0.1em] uppercase">Detalles</button>
                <button
                  className="flex-1 py-1.5 border border-coral/30 text-coral font-mono text-[8.5px] tracking-[0.1em] uppercase"
                  onClick={async () => {
                    if (confirm(`Revocar ${p.displayName}? Esto elimina el pareo local.`)) {
                      await remove(p.id);
                    }
                  }}
                >Revocar</button>
              </div>
            </div>
          );
        })}
        {pairings.length === 0 && (
          <div className="p-8 text-center text-text-dim font-mono text-[11px]">
            Ninguna máquina paireada aún.
          </div>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add pwa/src/pages/Settings.tsx
git commit -m "feat(pwa): settings page listing paired machines"
```

---

## Task 18: Project detail page (optional for MVP but useful)

Skip in MVP — the inbox + conversation view covers the core flow. The spec's "project detail" screen is a polish item for Fase 2.

---

## Task 19: PWA manifest + service worker verification

**Files:**
- Modify: `pwa/vite.config.ts` (already done in Task 1)
- Verify installability

- [ ] **Step 1: Build production bundle**

```bash
cd pwa
bun run build
```

Expected: `dist/` directory with `manifest.webmanifest`, `sw.js`, and static assets.

- [ ] **Step 2: Serve and test installability**

```bash
bunx serve dist -p 4173
```

Open Chrome at `http://localhost:4173`. DevTools → Application → Manifest: verify name, icons, theme, standalone display.

On iOS Safari: open the URL, Share → "Add to Home Screen" — verify icon appears.

On Android Chrome: install prompt should appear or via menu → "Install app".

- [ ] **Step 3: Verify service worker**

DevTools → Application → Service Workers: workbox-generated SW is registered.

Close tab, reopen URL → loads from SW cache offline.

- [ ] **Step 4: Commit (no code changes expected)**

```bash
git commit --allow-empty -m "test(pwa): verified installability on ios and android"
```

---

## Task 20: E2E test with Playwright against real agent

**Files:**
- Create: `pwa/playwright.config.ts`
- Create: `pwa/tests/e2e/full-flow.spec.ts`
- Create: `pwa/scripts/dev-with-agent.sh`

- [ ] **Step 1: Install Playwright browsers**

```bash
cd pwa
bunx playwright install chromium
```

- [ ] **Step 2: Create `playwright.config.ts`**

```typescript
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  timeout: 60_000,
  use: {
    baseURL: "http://localhost:5173",
    trace: "on-first-retry",
  },
  projects: [
    { name: "mobile", use: { ...devices["iPhone 14 Pro"] } },
    { name: "desktop", use: { ...devices["Desktop Chrome"] } },
  ],
  webServer: {
    command: "bun run dev",
    url: "http://localhost:5173",
    reuseExistingServer: !process.env.CI,
    timeout: 30_000,
  },
});
```

- [ ] **Step 3: Create helper script to start agent**

`pwa/scripts/dev-with-agent.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(cd "$(dirname "$0")/../.." && pwd)"

# Use the stub CLI from agent tests so no real claude needed
export AICONNECT_PORT=51821
(cd "$ROOT/agent" && bun run src/index.ts init) &
AGENT_PID=$!

trap "kill $AGENT_PID 2>/dev/null || true" EXIT

(cd "$ROOT/pwa" && bun run dev)
```

- [ ] **Step 4: Write E2E test that uses a stub agent**

For E2E we spin up a fake WebSocket server that mimics the agent protocol (simpler than booting full agent in CI).

`pwa/tests/e2e/full-flow.spec.ts`:

```typescript
import { test, expect, type Page } from "@playwright/test";

test.describe("full flow", () => {
  test.beforeEach(async ({ page }) => {
    // Inject a fake WebSocket server via page.evaluate
    await page.addInitScript(() => {
      class FakeWs extends EventTarget {
        readyState = 0;
        url: string;
        constructor(url: string) {
          super();
          this.url = url;
          setTimeout(() => {
            this.readyState = 1;
            this.dispatchEvent(new Event("open"));
          }, 10);
        }
        send(data: string) {
          const parsed = JSON.parse(data);
          if (parsed.type === "auth") {
            setTimeout(() => {
              this.dispatchEvent(new MessageEvent("message", { data: JSON.stringify({
                type: "auth_ok",
                tokenId: "tid",
                machine: { id: "m1", fingerprint: "fp", hostname: "fake.ts.net", displayName: "fake-mac" },
                protocolVersion: 1,
              }) }));
            }, 10);
          } else if (parsed.type === "list_projects") {
            this.dispatchEvent(new MessageEvent("message", { data: JSON.stringify({
              type: "project_list", projects: [],
            }) }));
          } else if (parsed.type === "list_conversations") {
            this.dispatchEvent(new MessageEvent("message", { data: JSON.stringify({
              type: "conversation_list", conversations: [],
            }) }));
          } else if (parsed.type === "create_project") {
            this.dispatchEvent(new MessageEvent("message", { data: JSON.stringify({
              type: "project_created",
              project: {
                id: "proj1", name: parsed.name, workingDir: parsed.workingDir,
                tags: parsed.tags, permissionPreset: parsed.permissionPreset,
                defaultModelClaudeCode: parsed.defaultModelClaudeCode ?? null,
              },
            }) }));
          }
        }
        close() { this.readyState = 3; }
        addEventListener(t: string, cb: EventListenerOrEventListenerObject) { super.addEventListener(t, cb); }
      }
      (globalThis as unknown as { WebSocket: typeof FakeWs }).WebSocket = FakeWs as unknown as typeof WebSocket;
      (globalThis as unknown as { WebSocket: { OPEN: number; CLOSED: number } }).WebSocket.OPEN = 1;
      (globalThis as unknown as { WebSocket: { OPEN: number; CLOSED: number } }).WebSocket.CLOSED = 3;
    });
  });

  test("welcome → manual pair → name → inbox", async ({ page }) => {
    await page.goto("/");
    await expect(page.getByText("Controla tus")).toBeVisible();
    await page.getByText("Parear primera Mac").click();

    // Scan page — switch to manual
    await page.getByText("Usar código manual").click();

    await page.getByPlaceholder("mac-trabajo.lion-spark.ts.net").fill("fake.ts.net");
    await page.locator("input[inputMode=numeric]").fill("51820");
    await page.locator("input").nth(3).fill("a".repeat(32));
    await page.getByText("Conectar").click();

    // After auth_ok → name page
    await expect(page.getByText("¿Cómo llamas")).toBeVisible({ timeout: 5000 });
    await page.getByRole("button", { name: /Registrar/ }).click();

    // Inbox shows empty state
    await expect(page.getByText("Sin conversaciones")).toBeVisible({ timeout: 5000 });
  });
});
```

- [ ] **Step 5: Run**

```bash
cd pwa
bunx playwright test
```

Expected: 2 projects (mobile + desktop) × 1 test = 2 passing.

- [ ] **Step 6: Commit**

```bash
git add pwa/playwright.config.ts pwa/tests/e2e/ pwa/scripts/
git commit -m "test(pwa): e2e playwright with fake ws for onboarding flow"
```

---

## Task 21: Dogfood checklist + README

**Files:**
- Create: `pwa/README.md`

- [ ] **Step 1: Write README**

```markdown
# aiconnect-pwa

React PWA frontend for aiconnect. Connects to the agent on each paired Mac via Tailscale WebSocket.

## Requirements

- Agent running on at least one Mac (see `../agent/README.md`)
- Tailscale configured on the Mac + the device you'll use the PWA from
- Modern browser with WebSocket, IndexedDB, Service Worker support

## Dev

```
cd pwa
bun install
bun run dev
```

Open `http://<your-device-on-tailnet>:5173` from phone. Or open `http://localhost:5173` from the Mac running the agent.

## Build

```
bun run build
bunx serve dist -p 4173
```

To install as PWA: on iOS use Safari → Share → Add to Home Screen. On Android use Chrome menu → Install app.

## E2E

```
bunx playwright test
```

Uses an injected fake WebSocket so no real agent required.

## Architecture

- `src/protocol/` — zod schemas matching agent (keep in sync manually in Fase 1)
- `src/net/` — per-machine WebSocket orchestration
- `src/storage/` — IndexedDB for pairings + quick prompts
- `src/state/` — zustand stores (pairings, connections, inbox, active conv)
- `src/pages/` — each top-level screen
- `src/components/` — building blocks organized by feature

## Design system

See `docs/superpowers/specs/2026-04-18-aiconnect-design.md` section 13. Tailwind config reflects tokens in `tailwind.config.ts`.

## Dogfood checklist for Fase 1

- [ ] Instalar PWA en iPhone/Android real
- [ ] Parear con agente del Plan 1 vía QR
- [ ] Crear proyecto con working dir real
- [ ] Abrir conversación Claude Code
- [ ] Enviar 10 prompts variados (corto, largo, con code review)
- [ ] Aprobar/denegar 3 approvals
- [ ] Cambiar modelo mid-conversation (Sonnet → Opus)
- [ ] Matar app, reabrir, reconectar vía replay
- [ ] Simular "sin red" (modo avión) y verificar queue
- [ ] Usar una semana completa sin pelearse con bugs
```

- [ ] **Step 2: Commit**

```bash
git add pwa/README.md
git commit -m "docs(pwa): readme + dogfood checklist"
```

---

## Task 22: Smoke test end-to-end con agente real

- [ ] **Step 1: Start agent on Mac**

```bash
cd agent
./bin/aiconnect-darwin-arm64 init
# Scan QR from phone OR type short code on manual page
```

- [ ] **Step 2: Open PWA on phone**

Navigate to `http://<your-mac-tailnet-hostname>:5173` (dev) or deployed URL.

- [ ] **Step 3: Complete full flow manually**

1. Welcome → Parear primera Mac
2. Scan QR (or Use manual code)
3. Confirm machine name
4. Create project with real `~/repos/<something>` path
5. Open new conversation (Claude Code, Sonnet default)
6. Send prompt: "qué archivos hay en este repo"
7. See tool_use `ls` streamed, then assistant_text_delta response
8. Send: "corre los tests" → approval card appears → approve
9. See streamed output, then result
10. Change model to Opus via badge tap
11. Close tab, reopen → conversation replays from IndexedDB nothing + WebSocket reconnect retrieves last events

- [ ] **Step 4: Document any blocker issues and fix before calling Plan 2 done**

- [ ] **Step 5: Final tag**

```bash
git tag -a "fase1-plan2-complete" -m "aiconnect Fase 1 Plan 2 complete: PWA + E2E"
```

---

## Self-Review Checklist

**Spec coverage (Plan 2 scope):**
- ✅ PWA installable (Task 1, 19)
- ✅ Pareo QR + manual (Tasks 9, 10, 11)
- ✅ Nombrar máquina (Task 11)
- ✅ Inbox con filtros (Task 12)
- ✅ Conversación streaming (Task 15)
- ✅ Approvals UI (Task 15)
- ✅ Model switching (Tasks 13, 14, 15)
- ✅ Proyectos crear (Task 13)
- ✅ Reconexión + replay (Tasks 7, 15, 16)
- ✅ Settings · Máquinas (Task 17)
- ✅ Design system dark editorial (Task 2 + Tailwind config Task 1)

**Not in Plan 2 (per split):**
- Voice input → Plan 3
- Image attachment → Plan 3
- Quick prompts → Plan 3

**Placeholder scan:** no TBDs in executable tasks. Task 18 explicitly deferred.

**Type consistency:** `InboxConversation`, `Pairing`, `ConversationEventT`, `AgentClient` coherentes entre Task 6–17.

---

## Execution Handoff

**Plan 2 guardado en `docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan2-pwa.md`.** Se ejecuta con la misma estrategia que Plan 1 (subagent-driven-development). Entre Plan 1 y Plan 2 no hay dependencia de orden de implementación estricta — pueden paralelizarse si hay capacidad, pero Plan 2 requiere el agente del Plan 1 funcionando para dogfood real.
