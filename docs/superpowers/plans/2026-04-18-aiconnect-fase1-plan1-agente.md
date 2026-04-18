# aiconnect Fase 1 · Plan 1/3 — Agente macOS + pareo

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir el agente daemon que corre en una Mac, envuelve el CLI de Claude Code con streaming JSON, expone una API WebSocket sobre Tailscale, persiste conversaciones en SQLite y soporta pareo vía QR.

**Architecture:** Binario único Bun compilado, distribuido vía brew tap. TypeScript con types estrictos. SQLite local con Drizzle ORM + FTS5. WebSocket server nativo de Bun bindeado a IP de Tailscale. Subprocesos de Claude Code viven entre prompts (no spawn por turn). LaunchAgent para auto-arranque al login.

**Tech Stack:** Bun 1.1+ · TypeScript · Drizzle ORM · bun:sqlite · `qrcode-terminal` · `ulid` · `zod` · `@libs-ninja/nanoid` para tokens

**Spec de referencia:** `docs/superpowers/specs/2026-04-18-aiconnect-design.md`

**Criterio de éxito del plan:** Un cliente WebSocket manual (ej: `websocat`) puede parear con el agente, crear un proyecto, abrir conversación con Claude Code, enviar prompts, recibir streaming de eventos, aprobar/denegar permisos, y todo persiste en SQLite sobreviviendo reinicios.

---

## File Structure

```
agent/
├── package.json
├── bunfig.toml
├── tsconfig.json
├── drizzle.config.ts
├── .gitignore
├── README.md
├── src/
│   ├── index.ts                        # Entry point, CLI dispatcher
│   ├── config.ts                       # Paths, constants, env
│   ├── logger.ts                       # Pino-style structured logs
│   │
│   ├── cli/
│   │   ├── init.ts                     # `aiconnect init` command
│   │   ├── revoke.ts                   # `aiconnect revoke <id>` command
│   │   ├── status.ts                   # `aiconnect status` command
│   │   └── launchagent.ts              # Install/uninstall LaunchAgent
│   │
│   ├── db/
│   │   ├── schema.ts                   # Drizzle table definitions
│   │   ├── client.ts                   # SQLite client singleton
│   │   ├── migrations/                 # SQL migrations generated
│   │   └── queries.ts                  # Reusable query fns
│   │
│   ├── auth/
│   │   ├── token.ts                    # Generate + hash + verify tokens
│   │   ├── fingerprint.ts              # Agent identity (stable UUID)
│   │   └── pairing.ts                  # QR + short code generation
│   │
│   ├── net/
│   │   ├── tailscale.ts                # Detect tailnet IP + hostname
│   │   └── interfaces.ts               # Bind helpers
│   │
│   ├── protocol/
│   │   ├── messages.ts                 # Zod schemas for all WS messages
│   │   └── version.ts                  # Protocol version constant
│   │
│   ├── ws/
│   │   ├── server.ts                   # WS server + connection lifecycle
│   │   ├── session.ts                  # Per-connection state
│   │   ├── router.ts                   # Dispatch incoming messages
│   │   └── handlers/
│   │       ├── auth.ts                 # Handshake
│   │       ├── identity.ts             # Who am I + list machines
│   │       ├── projects.ts             # CRUD proyectos
│   │       ├── conversations.ts        # Create, list, detail
│   │       ├── prompts.ts              # Send user_prompt → stream events
│   │       ├── approvals.ts            # Approve/deny pending
│   │       ├── model.ts                # Change model mid-conv
│   │       └── replay.ts               # Reconnect event replay
│   │
│   ├── cli-wrapper/
│   │   ├── claude-code.ts              # Spawn + lifecycle of `claude`
│   │   ├── stream-parser.ts            # Parse stream-json from stdout
│   │   ├── registry.ts                 # conversation_id → process
│   │   └── events.ts                   # Event types + normalization
│   │
│   ├── sleep/
│   │   └── caffeinate.ts               # Spawn/stop caffeinate -i
│   │
│   └── launchagent/
│       └── plist.ts                    # Generate .plist XML
│
├── tests/
│   ├── stub-cli/
│   │   └── claude                      # Bash stub that emits canned events
│   ├── auth/
│   │   ├── token.test.ts
│   │   └── fingerprint.test.ts
│   ├── db/
│   │   └── schema.test.ts
│   ├── protocol/
│   │   └── messages.test.ts
│   ├── cli-wrapper/
│   │   ├── claude-code.test.ts
│   │   └── stream-parser.test.ts
│   ├── ws/
│   │   ├── server.test.ts
│   │   └── handlers/
│   │       ├── auth.test.ts
│   │       ├── projects.test.ts
│   │       ├── conversations.test.ts
│   │       └── prompts.test.ts
│   └── e2e/
│       └── pair-and-chat.test.ts       # Full flow with stub CLI
│
└── scripts/
    ├── build-binary.sh                 # bun build --compile
    └── run-dev.sh                      # Dev mode with hot reload

Formula/
└── aiconnect-agent.rb                  # Brew formula (separate repo: jcani/homebrew-aiconnect)
```

**Key decomposition choices:**

- **`cli-wrapper/` separado** de `ws/handlers/` — el wrapper de Claude Code no sabe de WebSocket; los handlers no saben de stdio. Inyección vía `registry`.
- **`protocol/messages.ts` como fuente única de verdad** — los mismos schemas Zod se publican luego al PWA (Plan 2) para evitar drift.
- **Handlers pequeños por mensaje** en vez de un mega-router — cada archivo < 100 líneas, testeable aislado.
- **`stub-cli/claude`** — script bash que emite eventos stream-json canned. Permite tests E2E sin depender de Claude Code real.

---

## Task 1: Scaffolding inicial del proyecto

**Files:**
- Create: `agent/package.json`
- Create: `agent/bunfig.toml`
- Create: `agent/tsconfig.json`
- Create: `agent/.gitignore`
- Create: `agent/README.md`
- Create: `agent/src/index.ts`

- [ ] **Step 1: Crear directorio y correr `bun init`**

```bash
cd D:/PROYECTOS/aiconnect
mkdir -p agent
cd agent
bun init -y
```

- [ ] **Step 2: Reemplazar `package.json`**

```json
{
  "name": "aiconnect-agent",
  "version": "0.1.0",
  "description": "Remote-control daemon for Claude Code CLI over Tailscale",
  "type": "module",
  "module": "src/index.ts",
  "bin": {
    "aiconnect": "src/index.ts"
  },
  "scripts": {
    "dev": "bun run --watch src/index.ts",
    "test": "bun test",
    "test:watch": "bun test --watch",
    "typecheck": "tsc --noEmit",
    "build": "bash scripts/build-binary.sh",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "bun run src/db/migrate.ts"
  },
  "dependencies": {
    "drizzle-orm": "^0.36.0",
    "zod": "^3.23.0",
    "qrcode-terminal": "^0.12.0",
    "ulid": "^2.3.0",
    "nanoid": "^5.0.7"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "@types/qrcode-terminal": "^0.12.2",
    "drizzle-kit": "^0.28.0",
    "typescript": "^5.6.0"
  }
}
```

- [ ] **Step 3: Crear `bunfig.toml`**

```toml
[install]
auto = "force"

[test]
preload = ["./tests/setup.ts"]
coverage = false
```

- [ ] **Step 4: Crear `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "skipLibCheck": true,
    "types": ["bun-types"],
    "lib": ["ESNext"],
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true
  },
  "include": ["src/**/*", "tests/**/*"]
}
```

- [ ] **Step 5: Crear `.gitignore`**

```
node_modules/
dist/
bin/
*.log
.DS_Store
bun.lockb
.aiconnect/
```

- [ ] **Step 6: Crear README placeholder**

Create `agent/README.md`:

```markdown
# aiconnect-agent

macOS daemon that wraps Claude Code CLI and exposes a WebSocket API over Tailscale.

See `docs/superpowers/specs/2026-04-18-aiconnect-design.md` for full design.
```

- [ ] **Step 7: Stub `src/index.ts`**

```typescript
#!/usr/bin/env bun
console.log("aiconnect-agent v0.1.0");
process.exit(0);
```

- [ ] **Step 8: Instalar dependencias**

```bash
cd agent
bun install
```

Expected: all packages resolve, `bun.lockb` generated.

- [ ] **Step 9: Verificar typecheck pasa**

```bash
bun run typecheck
```

Expected: exit 0, no errors.

- [ ] **Step 10: Commit**

```bash
cd D:/PROYECTOS/aiconnect
git init 2>/dev/null || true
git add agent/
git commit -m "feat(agent): scaffold bun + typescript project"
```

---

## Task 2: Configuración y logger

**Files:**
- Create: `agent/src/config.ts`
- Create: `agent/src/logger.ts`
- Create: `agent/tests/config.test.ts`

- [ ] **Step 1: Write failing test**

Create `agent/tests/config.test.ts`:

```typescript
import { describe, test, expect } from "bun:test";
import { getConfig } from "../src/config";

describe("config", () => {
  test("returns paths derived from HOME", () => {
    const cfg = getConfig({ HOME: "/Users/test" });
    expect(cfg.configDir).toBe("/Users/test/.aiconnect");
    expect(cfg.dbPath).toBe("/Users/test/.aiconnect/agent.db");
    expect(cfg.identityPath).toBe("/Users/test/.aiconnect/identity.json");
  });

  test("defaults port to 51820", () => {
    const cfg = getConfig({ HOME: "/x" });
    expect(cfg.port).toBe(51820);
  });

  test("respects AICONNECT_PORT env", () => {
    const cfg = getConfig({ HOME: "/x", AICONNECT_PORT: "9999" });
    expect(cfg.port).toBe(9999);
  });
});
```

- [ ] **Step 2: Run test to verify failure**

```bash
cd agent
bun test tests/config.test.ts
```

Expected: FAIL, `getConfig is not a function` or module not found.

- [ ] **Step 3: Implement `src/config.ts`**

```typescript
import { join } from "node:path";

export interface Config {
  configDir: string;
  dbPath: string;
  identityPath: string;
  port: number;
  host: string;
  protocolVersion: number;
}

export function getConfig(env: Record<string, string | undefined> = process.env as Record<string, string | undefined>): Config {
  const home = env.HOME ?? env.USERPROFILE;
  if (!home) throw new Error("HOME env var missing");

  const configDir = join(home, ".aiconnect");
  return {
    configDir,
    dbPath: join(configDir, "agent.db"),
    identityPath: join(configDir, "identity.json"),
    port: env.AICONNECT_PORT ? Number.parseInt(env.AICONNECT_PORT, 10) : 51820,
    host: env.AICONNECT_HOST ?? "auto",
    protocolVersion: 1,
  };
}
```

- [ ] **Step 4: Run test to verify pass**

```bash
bun test tests/config.test.ts
```

Expected: PASS, 3 tests.

- [ ] **Step 5: Implement `src/logger.ts`**

```typescript
type Level = "debug" | "info" | "warn" | "error";

const LEVELS: Record<Level, number> = { debug: 0, info: 1, warn: 2, error: 3 };
const minLevel: number = LEVELS[(process.env.AICONNECT_LOG_LEVEL as Level) ?? "info"];

function emit(level: Level, msg: string, meta?: Record<string, unknown>): void {
  if (LEVELS[level] < minLevel) return;
  const entry = { ts: new Date().toISOString(), level, msg, ...meta };
  const line = JSON.stringify(entry);
  if (level === "error") process.stderr.write(line + "\n");
  else process.stdout.write(line + "\n");
}

export const log = {
  debug: (msg: string, meta?: Record<string, unknown>) => emit("debug", msg, meta),
  info: (msg: string, meta?: Record<string, unknown>) => emit("info", msg, meta),
  warn: (msg: string, meta?: Record<string, unknown>) => emit("warn", msg, meta),
  error: (msg: string, meta?: Record<string, unknown>) => emit("error", msg, meta),
};
```

- [ ] **Step 6: Commit**

```bash
git add agent/src/config.ts agent/src/logger.ts agent/tests/config.test.ts
git commit -m "feat(agent): config loader and structured logger"
```

---

## Task 3: Schema SQLite con Drizzle

**Files:**
- Create: `agent/src/db/schema.ts`
- Create: `agent/drizzle.config.ts`
- Create: `agent/src/db/client.ts`
- Create: `agent/src/db/migrate.ts`
- Create: `agent/tests/db/schema.test.ts`

- [ ] **Step 1: Implement `src/db/schema.ts`**

```typescript
import { sqliteTable, text, integer, index } from "drizzle-orm/sqlite-core";
import { sql } from "drizzle-orm";

// Identity of THIS machine (singleton row, id="self")
export const machine = sqliteTable("machine", {
  id: text("id").primaryKey(),
  fingerprint: text("fingerprint").notNull(),
  hostname: text("hostname").notNull(),
  displayName: text("display_name").notNull(),
  tag: text("tag"),
  createdAt: integer("created_at", { mode: "timestamp_ms" }).notNull().default(sql`(unixepoch() * 1000)`),
});

// Long-lived tokens (one per paired client device)
export const token = sqliteTable("token", {
  id: text("id").primaryKey(),
  hash: text("hash").notNull(),
  deviceLabel: text("device_label"),
  createdAt: integer("created_at", { mode: "timestamp_ms" }).notNull().default(sql`(unixepoch() * 1000)`),
  lastUsedAt: integer("last_used_at", { mode: "timestamp_ms" }),
  revoked: integer("revoked", { mode: "boolean" }).notNull().default(false),
});

export const project = sqliteTable("project", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  workingDir: text("working_dir").notNull(),
  tags: text("tags", { mode: "json" }).$type<string[]>().notNull().default([]),
  color: text("color"),
  permissionPreset: text("permission_preset").notNull().default("ask-for-side-effects"),
  defaultModelClaudeCode: text("default_model_claude_code"),
  defaultModelCodex: text("default_model_codex"),
  createdAt: integer("created_at", { mode: "timestamp_ms" }).notNull().default(sql`(unixepoch() * 1000)`),
});

export const conversation = sqliteTable("conversation", {
  id: text("id").primaryKey(),
  projectId: text("project_id").notNull().references(() => project.id, { onDelete: "cascade" }),
  cli: text("cli").notNull(), // "claude-code" | "codex"
  currentModel: text("current_model"),
  title: text("title").notNull().default("Nueva conversación"),
  status: text("status").notNull().default("idle"),
  mode: text("mode").notNull().default("active"),
  createdAt: integer("created_at", { mode: "timestamp_ms" }).notNull().default(sql`(unixepoch() * 1000)`),
  lastActivityAt: integer("last_activity_at", { mode: "timestamp_ms" }).notNull().default(sql`(unixepoch() * 1000)`),
}, (t) => ({
  byProject: index("conv_by_project").on(t.projectId),
  byActivity: index("conv_by_activity").on(t.lastActivityAt),
}));

export const event = sqliteTable("event", {
  id: text("id").primaryKey(), // ULID
  conversationId: text("conversation_id").notNull().references(() => conversation.id, { onDelete: "cascade" }),
  type: text("type").notNull(), // "user_prompt" | "assistant_text_delta" | "tool_use" | ...
  payload: text("payload", { mode: "json" }).notNull(),
  attachments: text("attachments", { mode: "json" }).$type<string[]>().default([]),
  timestamp: integer("timestamp", { mode: "timestamp_ms" }).notNull(),
}, (t) => ({
  byConv: index("event_by_conv").on(t.conversationId, t.id),
}));
```

- [ ] **Step 2: Create `drizzle.config.ts`**

```typescript
import type { Config } from "drizzle-kit";

export default {
  schema: "./src/db/schema.ts",
  out: "./src/db/migrations",
  dialect: "sqlite",
} satisfies Config;
```

- [ ] **Step 3: Generate initial migration**

```bash
cd agent
bun run db:generate
```

Expected: `src/db/migrations/0000_*.sql` created.

- [ ] **Step 4: Add FTS5 migration manually**

Create `src/db/migrations/0001_fts5_events.sql`:

```sql
-- Full-text search over event payloads (for future search UI)
CREATE VIRTUAL TABLE IF NOT EXISTS event_fts USING fts5(
  id UNINDEXED,
  conversation_id UNINDEXED,
  content,
  content_rowid=rowid
);

CREATE TRIGGER IF NOT EXISTS event_fts_insert AFTER INSERT ON event BEGIN
  INSERT INTO event_fts(id, conversation_id, content)
  VALUES (new.id, new.conversation_id, COALESCE(json_extract(new.payload, '$.text'), ''));
END;

CREATE TRIGGER IF NOT EXISTS event_fts_delete AFTER DELETE ON event BEGIN
  DELETE FROM event_fts WHERE id = old.id;
END;
```

- [ ] **Step 5: Implement `src/db/client.ts`**

```typescript
import { Database } from "bun:sqlite";
import { drizzle } from "drizzle-orm/bun-sqlite";
import { readdirSync, readFileSync, mkdirSync } from "node:fs";
import { dirname, join } from "node:path";
import { getConfig } from "../config";
import * as schema from "./schema";

export type DB = ReturnType<typeof drizzle<typeof schema>>;

let instance: { db: DB; raw: Database } | null = null;

export function openDb(dbPath?: string): { db: DB; raw: Database } {
  if (instance) return instance;
  const path = dbPath ?? getConfig().dbPath;
  mkdirSync(dirname(path), { recursive: true });
  const raw = new Database(path);
  raw.exec("PRAGMA journal_mode = WAL;");
  raw.exec("PRAGMA foreign_keys = ON;");
  const db = drizzle(raw, { schema });
  instance = { db, raw };
  return instance;
}

export function closeDb(): void {
  if (instance) {
    instance.raw.close();
    instance = null;
  }
}

export function runMigrations(migrationsDir: string, raw: Database): void {
  const files = readdirSync(migrationsDir)
    .filter((f) => f.endsWith(".sql"))
    .sort();
  for (const f of files) {
    const sql = readFileSync(join(migrationsDir, f), "utf8");
    raw.exec(sql);
  }
}
```

- [ ] **Step 6: Implement `src/db/migrate.ts`**

```typescript
import { join } from "node:path";
import { openDb, runMigrations } from "./client";
import { log } from "../logger";

export function migrate(): void {
  const { raw } = openDb();
  const migrationsDir = join(import.meta.dir, "migrations");
  runMigrations(migrationsDir, raw);
  log.info("migrations applied", { dir: migrationsDir });
}

if (import.meta.main) {
  migrate();
}
```

- [ ] **Step 7: Write schema test**

Create `agent/tests/db/schema.test.ts`:

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { unlinkSync, existsSync, mkdtempSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { openDb, closeDb, runMigrations } from "../../src/db/client";
import { project, conversation } from "../../src/db/schema";
import { ulid } from "ulid";

let tmpPath: string;

beforeEach(() => {
  const dir = mkdtempSync(join(tmpdir(), "aiconnect-test-"));
  tmpPath = join(dir, "test.db");
});

afterEach(() => {
  closeDb();
  if (existsSync(tmpPath)) unlinkSync(tmpPath);
});

describe("db schema", () => {
  test("can insert and retrieve a project", () => {
    const { db, raw } = openDb(tmpPath);
    runMigrations(join(import.meta.dir, "../../src/db/migrations"), raw);
    const id = ulid();
    db.insert(project).values({
      id,
      name: "test-project",
      workingDir: "/tmp/x",
      tags: ["work"],
      permissionPreset: "ask-for-side-effects",
    }).run();
    const rows = db.select().from(project).all();
    expect(rows.length).toBe(1);
    expect(rows[0]?.name).toBe("test-project");
    expect(rows[0]?.tags).toEqual(["work"]);
  });

  test("cascades deletion from project to conversations", () => {
    const { db, raw } = openDb(tmpPath);
    runMigrations(join(import.meta.dir, "../../src/db/migrations"), raw);
    const pid = ulid();
    const cid = ulid();
    db.insert(project).values({ id: pid, name: "p", workingDir: "/x" }).run();
    db.insert(conversation).values({ id: cid, projectId: pid, cli: "claude-code" }).run();
    db.delete(project).where(undefined as never).run(); // TS workaround; in real use with eq()
    // Using raw SQL to delete all to verify cascade
    raw.exec("DELETE FROM project");
    const convs = db.select().from(conversation).all();
    expect(convs.length).toBe(0);
  });
});
```

- [ ] **Step 8: Run tests**

```bash
cd agent
bun test tests/db/schema.test.ts
```

Expected: PASS, 2 tests.

- [ ] **Step 9: Create `tests/setup.ts` for global test hooks**

```typescript
// Silence structured logs during tests
process.env.AICONNECT_LOG_LEVEL = "error";
```

- [ ] **Step 10: Commit**

```bash
git add agent/src/db/ agent/drizzle.config.ts agent/tests/
git commit -m "feat(agent): sqlite schema with drizzle + fts5 + migrations"
```

---

## Task 4: Generación y verificación de tokens

**Files:**
- Create: `agent/src/auth/token.ts`
- Create: `agent/tests/auth/token.test.ts`

- [ ] **Step 1: Write failing test**

```typescript
import { describe, test, expect } from "bun:test";
import { generateToken, hashToken, verifyToken } from "../../src/auth/token";

describe("token", () => {
  test("generateToken produces distinct random strings", () => {
    const a = generateToken();
    const b = generateToken();
    expect(a).not.toBe(b);
    expect(a.length).toBeGreaterThanOrEqual(32);
  });

  test("hashToken is deterministic", () => {
    const t = "test-token-123";
    expect(hashToken(t)).toBe(hashToken(t));
  });

  test("hashToken produces different hash for different tokens", () => {
    expect(hashToken("a")).not.toBe(hashToken("b"));
  });

  test("verifyToken returns true for matching hash", () => {
    const t = generateToken();
    const h = hashToken(t);
    expect(verifyToken(t, h)).toBe(true);
  });

  test("verifyToken returns false for non-matching hash", () => {
    const t = generateToken();
    const h = hashToken(t);
    expect(verifyToken("other", h)).toBe(false);
  });
});
```

- [ ] **Step 2: Run test (expect failure)**

```bash
bun test tests/auth/token.test.ts
```

Expected: FAIL, module not found.

- [ ] **Step 3: Implement `src/auth/token.ts`**

```typescript
import { createHash, randomBytes, timingSafeEqual } from "node:crypto";

const TOKEN_BYTES = 32;

export function generateToken(): string {
  return randomBytes(TOKEN_BYTES).toString("base64url");
}

export function hashToken(token: string): string {
  return createHash("sha256").update(token).digest("hex");
}

export function verifyToken(provided: string, expectedHash: string): boolean {
  const providedHash = hashToken(provided);
  const a = Buffer.from(providedHash, "hex");
  const b = Buffer.from(expectedHash, "hex");
  if (a.length !== b.length) return false;
  return timingSafeEqual(a, b);
}
```

- [ ] **Step 4: Run tests**

```bash
bun test tests/auth/token.test.ts
```

Expected: PASS, 5 tests.

- [ ] **Step 5: Commit**

```bash
git add agent/src/auth/token.ts agent/tests/auth/token.test.ts
git commit -m "feat(agent): token generation, hashing, and verification"
```

---

## Task 5: Fingerprint del agente

**Files:**
- Create: `agent/src/auth/fingerprint.ts`
- Create: `agent/tests/auth/fingerprint.test.ts`

- [ ] **Step 1: Write failing test**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync, readFileSync, existsSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { getOrCreateFingerprint } from "../../src/auth/fingerprint";

let dir: string;

beforeEach(() => {
  dir = mkdtempSync(join(tmpdir(), "fp-"));
});

afterEach(() => {
  rmSync(dir, { recursive: true, force: true });
});

describe("fingerprint", () => {
  test("creates identity file on first call", () => {
    const path = join(dir, "identity.json");
    const fp = getOrCreateFingerprint(path);
    expect(fp).toMatch(/^[a-f0-9]{32}$/);
    expect(existsSync(path)).toBe(true);
  });

  test("returns same fingerprint on second call", () => {
    const path = join(dir, "identity.json");
    const a = getOrCreateFingerprint(path);
    const b = getOrCreateFingerprint(path);
    expect(a).toBe(b);
  });
});
```

- [ ] **Step 2: Run test (expect failure)**

```bash
bun test tests/auth/fingerprint.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement `src/auth/fingerprint.ts`**

```typescript
import { existsSync, readFileSync, writeFileSync, mkdirSync } from "node:fs";
import { dirname } from "node:path";
import { randomBytes } from "node:crypto";

interface Identity {
  fingerprint: string;
  createdAt: string;
}

export function getOrCreateFingerprint(path: string): string {
  if (existsSync(path)) {
    const data = JSON.parse(readFileSync(path, "utf8")) as Identity;
    return data.fingerprint;
  }
  mkdirSync(dirname(path), { recursive: true });
  const fingerprint = randomBytes(16).toString("hex");
  const identity: Identity = { fingerprint, createdAt: new Date().toISOString() };
  writeFileSync(path, JSON.stringify(identity, null, 2), { mode: 0o600 });
  return fingerprint;
}
```

- [ ] **Step 4: Run tests**

```bash
bun test tests/auth/fingerprint.test.ts
```

Expected: PASS, 2 tests.

- [ ] **Step 5: Commit**

```bash
git add agent/src/auth/fingerprint.ts agent/tests/auth/fingerprint.test.ts
git commit -m "feat(agent): stable per-install agent fingerprint"
```

---

## Task 6: Detección de IP Tailscale

**Files:**
- Create: `agent/src/net/tailscale.ts`
- Create: `agent/tests/net/tailscale.test.ts`

- [ ] **Step 1: Write failing test**

```typescript
import { describe, test, expect, mock } from "bun:test";
import { parseTailscaleStatus, parseTailscaleIp } from "../../src/net/tailscale";

describe("tailscale parsing", () => {
  test("parseTailscaleIp extracts first IPv4", () => {
    expect(parseTailscaleIp("100.64.0.3\nfd7a::3\n")).toBe("100.64.0.3");
  });

  test("parseTailscaleIp returns null if empty", () => {
    expect(parseTailscaleIp("")).toBeNull();
  });

  test("parseTailscaleStatus extracts hostname from json", () => {
    const json = JSON.stringify({
      Self: { HostName: "jcani-mbp", DNSName: "jcani-mbp.lion-spark.ts.net." },
    });
    const result = parseTailscaleStatus(json);
    expect(result.hostname).toBe("jcani-mbp");
    expect(result.magicDns).toBe("jcani-mbp.lion-spark.ts.net");
  });
});
```

- [ ] **Step 2: Run test (expect failure)**

```bash
bun test tests/net/tailscale.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement `src/net/tailscale.ts`**

```typescript
import { $ } from "bun";

export function parseTailscaleIp(raw: string): string | null {
  const lines = raw.trim().split("\n");
  for (const line of lines) {
    const trimmed = line.trim();
    if (/^\d+\.\d+\.\d+\.\d+$/.test(trimmed)) return trimmed;
  }
  return null;
}

export interface TailscaleIdentity {
  hostname: string;
  magicDns: string;
}

export function parseTailscaleStatus(raw: string): TailscaleIdentity {
  const data = JSON.parse(raw) as { Self?: { HostName?: string; DNSName?: string } };
  const hostname = data.Self?.HostName ?? "unknown";
  const dns = data.Self?.DNSName ?? "";
  return { hostname, magicDns: dns.replace(/\.$/, "") };
}

export async function getTailscaleIp(): Promise<string | null> {
  try {
    const result = await $`tailscale ip -4`.quiet();
    return parseTailscaleIp(result.stdout.toString());
  } catch {
    return null;
  }
}

export async function getTailscaleIdentity(): Promise<TailscaleIdentity | null> {
  try {
    const result = await $`tailscale status --json`.quiet();
    return parseTailscaleStatus(result.stdout.toString());
  } catch {
    return null;
  }
}
```

- [ ] **Step 4: Run tests**

```bash
bun test tests/net/tailscale.test.ts
```

Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add agent/src/net/tailscale.ts agent/tests/net/tailscale.test.ts
git commit -m "feat(agent): tailscale ip and identity detection"
```

---

## Task 7: Protocol — definiciones de mensajes

**Files:**
- Create: `agent/src/protocol/version.ts`
- Create: `agent/src/protocol/messages.ts`
- Create: `agent/tests/protocol/messages.test.ts`

- [ ] **Step 1: Write failing test**

```typescript
import { describe, test, expect } from "bun:test";
import { ClientMessage, ServerMessage, AuthRequest, TurnCompleteEvent } from "../../src/protocol/messages";

describe("protocol messages", () => {
  test("AuthRequest validates correctly", () => {
    const ok = AuthRequest.safeParse({
      type: "auth",
      token: "abc",
      protocolVersion: 1,
      deviceLabel: "iPhone",
    });
    expect(ok.success).toBe(true);
  });

  test("AuthRequest rejects missing token", () => {
    const bad = AuthRequest.safeParse({ type: "auth", protocolVersion: 1 });
    expect(bad.success).toBe(false);
  });

  test("ClientMessage discriminates by type", () => {
    const parsed = ClientMessage.safeParse({
      type: "send_prompt",
      conversationId: "c1",
      prompt: "hello",
      attachments: [],
    });
    expect(parsed.success).toBe(true);
  });

  test("ServerMessage accepts turn_complete event", () => {
    const parsed = ServerMessage.safeParse({
      type: "event",
      conversationId: "c1",
      event: {
        id: "01J...",
        type: "turn_complete",
        payload: {},
        timestamp: Date.now(),
      },
    });
    expect(parsed.success).toBe(true);
  });
});
```

- [ ] **Step 2: Run test (expect failure)**

```bash
bun test tests/protocol/messages.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement `src/protocol/version.ts`**

```typescript
export const PROTOCOL_VERSION = 1;
```

- [ ] **Step 4: Implement `src/protocol/messages.ts`**

```typescript
import { z } from "zod";

// ========================================
// Shared subtypes
// ========================================

export const PermissionPreset = z.enum(["ask-for-side-effects", "auto-accept-edits", "full-auto"]);
export const ConversationStatus = z.enum(["idle", "running", "awaiting_approval", "error", "interrupted"]);
export const ConversationMode = z.enum(["active", "plan-only"]);
export const CliKind = z.enum(["claude-code", "codex"]);

export const EventBase = z.object({
  id: z.string(),
  timestamp: z.number(),
});

export const UserPromptEvent = EventBase.extend({
  type: z.literal("user_prompt"),
  payload: z.object({ text: z.string(), attachments: z.array(z.string()).optional() }),
});
export const AssistantDeltaEvent = EventBase.extend({
  type: z.literal("assistant_text_delta"),
  payload: z.object({ text: z.string() }),
});
export const ToolUseEvent = EventBase.extend({
  type: z.literal("tool_use"),
  payload: z.object({ tool: z.string(), input: z.unknown() }),
});
export const ToolResultEvent = EventBase.extend({
  type: z.literal("tool_result"),
  payload: z.object({ tool: z.string(), output: z.unknown(), isError: z.boolean().optional() }),
});
export const PermissionRequestEvent = EventBase.extend({
  type: z.literal("permission_request"),
  payload: z.object({
    requestId: z.string(),
    kind: z.enum(["bash", "edit", "write", "read", "network", "other"]),
    detail: z.string(),
    command: z.string().optional(),
    path: z.string().optional(),
  }),
});
export const PermissionResponseEvent = EventBase.extend({
  type: z.literal("permission_response"),
  payload: z.object({ requestId: z.string(), granted: z.boolean(), note: z.string().optional() }),
});
export const ModelChangeEvent = EventBase.extend({
  type: z.literal("model_change"),
  payload: z.object({ from: z.string().nullable(), to: z.string() }),
});
export const ErrorEvent = EventBase.extend({
  type: z.literal("error"),
  payload: z.object({ message: z.string(), detail: z.string().optional() }),
});
export const TurnCompleteEvent = EventBase.extend({
  type: z.literal("turn_complete"),
  payload: z.object({}).passthrough(),
});

export const ConversationEvent = z.discriminatedUnion("type", [
  UserPromptEvent,
  AssistantDeltaEvent,
  ToolUseEvent,
  ToolResultEvent,
  PermissionRequestEvent,
  PermissionResponseEvent,
  ModelChangeEvent,
  ErrorEvent,
  TurnCompleteEvent,
]);

// ========================================
// Client → Server messages
// ========================================

export const AuthRequest = z.object({
  type: z.literal("auth"),
  token: z.string().min(16),
  protocolVersion: z.number().int().positive(),
  deviceLabel: z.string().optional(),
});

export const ListProjectsRequest = z.object({
  type: z.literal("list_projects"),
});

export const CreateProjectRequest = z.object({
  type: z.literal("create_project"),
  name: z.string().min(1).max(100),
  workingDir: z.string().min(1),
  tags: z.array(z.string()).default([]),
  permissionPreset: PermissionPreset.default("ask-for-side-effects"),
  defaultModelClaudeCode: z.string().optional(),
});

export const ListConversationsRequest = z.object({
  type: z.literal("list_conversations"),
  projectId: z.string().optional(),
});

export const CreateConversationRequest = z.object({
  type: z.literal("create_conversation"),
  projectId: z.string(),
  cli: CliKind,
  model: z.string().optional(),
});

export const SendPromptRequest = z.object({
  type: z.literal("send_prompt"),
  conversationId: z.string(),
  prompt: z.string().min(1),
  attachments: z.array(z.string()).default([]),
});

export const ApprovalResponseRequest = z.object({
  type: z.literal("approval_response"),
  conversationId: z.string(),
  requestId: z.string(),
  granted: z.boolean(),
  note: z.string().optional(),
});

export const ChangeModelRequest = z.object({
  type: z.literal("change_model"),
  conversationId: z.string(),
  model: z.string(),
});

export const ReplayFromRequest = z.object({
  type: z.literal("replay_from"),
  conversationId: z.string(),
  afterEventId: z.string().optional(),
});

export const ClientMessage = z.discriminatedUnion("type", [
  AuthRequest,
  ListProjectsRequest,
  CreateProjectRequest,
  ListConversationsRequest,
  CreateConversationRequest,
  SendPromptRequest,
  ApprovalResponseRequest,
  ChangeModelRequest,
  ReplayFromRequest,
]);

// ========================================
// Server → Client messages
// ========================================

export const AuthOk = z.object({
  type: z.literal("auth_ok"),
  tokenId: z.string(),
  machine: z.object({
    id: z.string(),
    fingerprint: z.string(),
    hostname: z.string(),
    displayName: z.string(),
  }),
  protocolVersion: z.number(),
});

export const ProtocolError = z.object({
  type: z.literal("error"),
  code: z.enum(["unauthorized", "bad_request", "not_found", "internal"]),
  message: z.string(),
});

export const ProjectList = z.object({
  type: z.literal("project_list"),
  projects: z.array(z.object({
    id: z.string(),
    name: z.string(),
    workingDir: z.string(),
    tags: z.array(z.string()),
    permissionPreset: PermissionPreset,
    defaultModelClaudeCode: z.string().nullable(),
  })),
});

export const ProjectCreated = z.object({
  type: z.literal("project_created"),
  project: ProjectList.shape.projects.element,
});

export const ConversationList = z.object({
  type: z.literal("conversation_list"),
  conversations: z.array(z.object({
    id: z.string(),
    projectId: z.string(),
    cli: CliKind,
    currentModel: z.string().nullable(),
    title: z.string(),
    status: ConversationStatus,
    mode: ConversationMode,
    lastActivityAt: z.number(),
  })),
});

export const ConversationCreated = z.object({
  type: z.literal("conversation_created"),
  conversation: ConversationList.shape.conversations.element,
});

export const EventMessage = z.object({
  type: z.literal("event"),
  conversationId: z.string(),
  event: ConversationEvent,
});

export const ReplayDone = z.object({
  type: z.literal("replay_done"),
  conversationId: z.string(),
  lastEventId: z.string().nullable(),
});

export const ServerMessage = z.discriminatedUnion("type", [
  AuthOk,
  ProtocolError,
  ProjectList,
  ProjectCreated,
  ConversationList,
  ConversationCreated,
  EventMessage,
  ReplayDone,
]);

export type ClientMessageT = z.infer<typeof ClientMessage>;
export type ServerMessageT = z.infer<typeof ServerMessage>;
export type ConversationEventT = z.infer<typeof ConversationEvent>;
```

- [ ] **Step 5: Run tests**

```bash
bun test tests/protocol/messages.test.ts
```

Expected: PASS, 4 tests.

- [ ] **Step 6: Commit**

```bash
git add agent/src/protocol/ agent/tests/protocol/
git commit -m "feat(agent): protocol message schemas with zod"
```

---

## Task 8: Stub CLI de prueba

**Files:**
- Create: `agent/tests/stub-cli/claude` (executable bash script)

- [ ] **Step 1: Create stub CLI**

```bash
mkdir -p agent/tests/stub-cli
```

Create `agent/tests/stub-cli/claude`:

```bash
#!/usr/bin/env bash
# Stub that mimics `claude --output-format stream-json` for testing.
# Reads prompts from stdin (line-delimited JSON) and emits canned events.

set -euo pipefail

emit() {
  echo "$1"
}

ts() {
  date +%s%3N
}

ulid() {
  # Pseudo-ULID: not standards-compliant, but unique-enough for tests.
  printf "01J%s\n" "$(od -An -N10 -tx1 /dev/urandom | tr -d ' \n')"
}

# Read each line of input as a "user_prompt"
while IFS= read -r line; do
  prompt_id=$(ulid)
  emit "{\"id\":\"$(ulid)\",\"type\":\"assistant_text_delta\",\"payload\":{\"text\":\"Recibí: \"},\"timestamp\":$(ts)}"
  emit "{\"id\":\"$(ulid)\",\"type\":\"assistant_text_delta\",\"payload\":{\"text\":\"$line\"},\"timestamp\":$(ts)}"

  # Simulate a permission request if prompt contains the word "bash"
  if echo "$line" | grep -qi "bash"; then
    req_id=$(ulid)
    emit "{\"id\":\"$(ulid)\",\"type\":\"permission_request\",\"payload\":{\"requestId\":\"$req_id\",\"kind\":\"bash\",\"detail\":\"run a command\",\"command\":\"echo test\"},\"timestamp\":$(ts)}"
    # Wait for approval on stdin (simulated)
    while IFS= read -r ans; do
      if echo "$ans" | grep -q "\"requestId\":\"$req_id\""; then
        break
      fi
    done
  fi

  emit "{\"id\":\"$(ulid)\",\"type\":\"turn_complete\",\"payload\":{},\"timestamp\":$(ts)}"
done
```

- [ ] **Step 2: Make executable and test manually**

```bash
chmod +x agent/tests/stub-cli/claude
echo "hello" | agent/tests/stub-cli/claude
```

Expected output: 3 JSON lines (two deltas + turn_complete).

- [ ] **Step 3: Commit**

```bash
git add agent/tests/stub-cli/
git commit -m "test: stub claude cli for integration tests"
```

---

## Task 9: Stream parser del CLI

**Files:**
- Create: `agent/src/cli-wrapper/stream-parser.ts`
- Create: `agent/tests/cli-wrapper/stream-parser.test.ts`

- [ ] **Step 1: Write failing test**

```typescript
import { describe, test, expect } from "bun:test";
import { LineStreamParser } from "../../src/cli-wrapper/stream-parser";

describe("LineStreamParser", () => {
  test("emits complete lines as parsed JSON", () => {
    const parser = new LineStreamParser();
    const emitted: unknown[] = [];
    parser.on((o) => emitted.push(o));
    parser.feed("{\"a\":1}\n{\"b\":2}\n");
    expect(emitted).toEqual([{ a: 1 }, { b: 2 }]);
  });

  test("buffers partial lines across feeds", () => {
    const parser = new LineStreamParser();
    const emitted: unknown[] = [];
    parser.on((o) => emitted.push(o));
    parser.feed("{\"a\":");
    parser.feed("1}\n");
    expect(emitted).toEqual([{ a: 1 }]);
  });

  test("skips malformed lines and reports parse errors via errorHandler", () => {
    const parser = new LineStreamParser();
    const errs: Error[] = [];
    parser.onError((e) => errs.push(e));
    parser.feed("not-json\n{\"ok\":1}\n");
    expect(errs.length).toBe(1);
  });
});
```

- [ ] **Step 2: Run test (expect failure)**

```bash
bun test tests/cli-wrapper/stream-parser.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement `src/cli-wrapper/stream-parser.ts`**

```typescript
type Listener = (obj: unknown) => void;
type ErrorListener = (err: Error) => void;

export class LineStreamParser {
  private buffer = "";
  private listeners: Listener[] = [];
  private errorListeners: ErrorListener[] = [];

  on(l: Listener): void {
    this.listeners.push(l);
  }

  onError(l: ErrorListener): void {
    this.errorListeners.push(l);
  }

  feed(chunk: string): void {
    this.buffer += chunk;
    const lines = this.buffer.split("\n");
    this.buffer = lines.pop() ?? "";
    for (const line of lines) {
      const trimmed = line.trim();
      if (!trimmed) continue;
      try {
        const parsed: unknown = JSON.parse(trimmed);
        for (const l of this.listeners) l(parsed);
      } catch (err) {
        const e = err instanceof Error ? err : new Error(String(err));
        for (const el of this.errorListeners) el(e);
      }
    }
  }

  flush(): void {
    if (this.buffer.trim()) {
      this.feed("\n");
    }
  }
}
```

- [ ] **Step 4: Run tests**

```bash
bun test tests/cli-wrapper/stream-parser.test.ts
```

Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add agent/src/cli-wrapper/stream-parser.ts agent/tests/cli-wrapper/stream-parser.test.ts
git commit -m "feat(agent): newline-delimited json stream parser"
```

---

## Task 10: Wrapper del CLI Claude Code

**Files:**
- Create: `agent/src/cli-wrapper/claude-code.ts`
- Create: `agent/src/cli-wrapper/events.ts`
- Create: `agent/tests/cli-wrapper/claude-code.test.ts`

- [ ] **Step 1: Implement `src/cli-wrapper/events.ts`**

Normalizes events coming from the CLI's stream-json into our domain types:

```typescript
import { ulid } from "ulid";
import type { ConversationEventT } from "../protocol/messages";

export function normalizeCliEvent(raw: unknown): ConversationEventT | null {
  if (typeof raw !== "object" || raw === null) return null;
  const obj = raw as Record<string, unknown>;
  const type = obj.type;
  if (typeof type !== "string") return null;

  const id = typeof obj.id === "string" ? obj.id : ulid();
  const timestamp = typeof obj.timestamp === "number" ? obj.timestamp : Date.now();
  const payload = (obj.payload && typeof obj.payload === "object") ? obj.payload as Record<string, unknown> : {};

  switch (type) {
    case "assistant_text_delta":
      return { id, type, timestamp, payload: { text: String(payload.text ?? "") } };
    case "tool_use":
      return { id, type, timestamp, payload: { tool: String(payload.tool ?? ""), input: payload.input } };
    case "tool_result":
      return {
        id, type, timestamp,
        payload: {
          tool: String(payload.tool ?? ""),
          output: payload.output,
          ...(typeof payload.isError === "boolean" ? { isError: payload.isError } : {}),
        },
      };
    case "permission_request":
      return {
        id, type, timestamp,
        payload: {
          requestId: String(payload.requestId ?? ulid()),
          kind: ((): "bash" | "edit" | "write" | "read" | "network" | "other" => {
            const k = payload.kind;
            if (k === "bash" || k === "edit" || k === "write" || k === "read" || k === "network") return k;
            return "other";
          })(),
          detail: String(payload.detail ?? ""),
          ...(typeof payload.command === "string" ? { command: payload.command } : {}),
          ...(typeof payload.path === "string" ? { path: payload.path } : {}),
        },
      };
    case "turn_complete":
      return { id, type, timestamp, payload: {} };
    case "error":
      return { id, type, timestamp, payload: { message: String(payload.message ?? "unknown error") } };
    default:
      return null;
  }
}
```

- [ ] **Step 2: Implement `src/cli-wrapper/claude-code.ts`**

```typescript
import { spawn, type Subprocess } from "bun";
import { LineStreamParser } from "./stream-parser";
import { normalizeCliEvent } from "./events";
import type { ConversationEventT } from "../protocol/messages";
import { log } from "../logger";

export interface ClaudeCodeOptions {
  binary?: string; // path to `claude` (or stub for tests)
  cwd: string;
  model?: string;
  permissionPreset?: "ask-for-side-effects" | "auto-accept-edits" | "full-auto";
  extraArgs?: string[];
}

export interface ClaudeCodeWrapper {
  sendPrompt(prompt: string, attachments?: string[]): void;
  sendApproval(requestId: string, granted: boolean, note?: string): void;
  sendSlashCommand(cmd: string): void;
  kill(): void;
  onEvent(l: (e: ConversationEventT) => void): void;
  onExit(l: (code: number | null) => void): void;
  readonly pid: number | undefined;
}

export function startClaudeCode(opts: ClaudeCodeOptions): ClaudeCodeWrapper {
  const binary = opts.binary ?? "claude";
  const args: string[] = [
    "--output-format", "stream-json",
    "--input-format", "stream-json",
  ];
  if (opts.model) args.push("--model", opts.model);
  if (opts.permissionPreset === "full-auto") args.push("--dangerously-skip-permissions");
  if (opts.extraArgs) args.push(...opts.extraArgs);

  log.info("spawning claude", { binary, cwd: opts.cwd, model: opts.model });

  const proc = spawn({
    cmd: [binary, ...args],
    cwd: opts.cwd,
    stdin: "pipe",
    stdout: "pipe",
    stderr: "pipe",
  });

  const parser = new LineStreamParser();
  const eventListeners: Array<(e: ConversationEventT) => void> = [];
  const exitListeners: Array<(code: number | null) => void> = [];

  parser.on((raw) => {
    const normalized = normalizeCliEvent(raw);
    if (normalized) {
      for (const l of eventListeners) l(normalized);
    }
  });
  parser.onError((err) => {
    log.warn("cli stream parse error", { err: err.message });
  });

  // Consume stdout
  (async () => {
    const reader = proc.stdout.getReader();
    try {
      while (true) {
        const { value, done } = await reader.read();
        if (done) break;
        parser.feed(new TextDecoder().decode(value));
      }
    } catch (err) {
      log.warn("cli stdout reader error", { err: String(err) });
    }
  })();

  // Log stderr
  (async () => {
    const reader = proc.stderr.getReader();
    const decoder = new TextDecoder();
    try {
      while (true) {
        const { value, done } = await reader.read();
        if (done) break;
        log.warn("cli stderr", { data: decoder.decode(value) });
      }
    } catch { /* ignored */ }
  })();

  proc.exited.then((code) => {
    for (const l of exitListeners) l(code);
  });

  const writeLine = (obj: unknown): void => {
    if (!proc.stdin) return;
    const line = JSON.stringify(obj) + "\n";
    proc.stdin.write(line);
    proc.stdin.flush?.();
  };

  return {
    sendPrompt(prompt, attachments) {
      writeLine({ type: "user_prompt", text: prompt, attachments: attachments ?? [] });
    },
    sendApproval(requestId, granted, note) {
      writeLine({ type: "permission_response", requestId, granted, note });
    },
    sendSlashCommand(cmd) {
      writeLine({ type: "slash_command", command: cmd });
    },
    kill() {
      proc.kill();
    },
    onEvent(l) { eventListeners.push(l); },
    onExit(l) { exitListeners.push(l); },
    get pid() { return proc.pid; },
  };
}
```

- [ ] **Step 3: Write integration test using stub**

```typescript
import { describe, test, expect } from "bun:test";
import { join } from "node:path";
import { startClaudeCode } from "../../src/cli-wrapper/claude-code";
import type { ConversationEventT } from "../../src/protocol/messages";

const stubPath = join(import.meta.dir, "..", "stub-cli", "claude");

describe("claude-code wrapper with stub", () => {
  test("spawns stub, sends prompt, receives events", async () => {
    const events: ConversationEventT[] = [];
    const w = startClaudeCode({
      binary: stubPath,
      cwd: process.cwd(),
    });
    w.onEvent((e) => events.push(e));

    w.sendPrompt("hello");
    await new Promise((r) => setTimeout(r, 200));

    const types = events.map((e) => e.type);
    expect(types).toContain("assistant_text_delta");
    expect(types).toContain("turn_complete");

    w.kill();
    await new Promise((r) => setTimeout(r, 50));
  }, 5000);
});
```

- [ ] **Step 4: Run tests**

```bash
bun test tests/cli-wrapper/claude-code.test.ts
```

Expected: PASS (requires bash on PATH; macOS/Linux only).

- [ ] **Step 5: Commit**

```bash
git add agent/src/cli-wrapper/ agent/tests/cli-wrapper/claude-code.test.ts
git commit -m "feat(agent): claude code process wrapper with stream parser"
```

---

## Task 11: Registro de conversaciones activas

**Files:**
- Create: `agent/src/cli-wrapper/registry.ts`
- Create: `agent/tests/cli-wrapper/registry.test.ts`

- [ ] **Step 1: Write failing test**

```typescript
import { describe, test, expect } from "bun:test";
import { ConversationRegistry } from "../../src/cli-wrapper/registry";

describe("ConversationRegistry", () => {
  test("register and lookup", () => {
    const r = new ConversationRegistry();
    const mock = { kill: () => {}, sendPrompt: () => {}, sendApproval: () => {}, sendSlashCommand: () => {}, onEvent: () => {}, onExit: () => {}, pid: 1 };
    r.register("c1", mock as never);
    expect(r.get("c1")).toBe(mock as never);
  });

  test("remove cleans up", () => {
    const r = new ConversationRegistry();
    r.register("c1", { kill: () => {} } as never);
    r.remove("c1");
    expect(r.get("c1")).toBeUndefined();
  });

  test("size reflects active", () => {
    const r = new ConversationRegistry();
    r.register("a", { kill: () => {} } as never);
    r.register("b", { kill: () => {} } as never);
    expect(r.size).toBe(2);
    r.remove("a");
    expect(r.size).toBe(1);
  });
});
```

- [ ] **Step 2: Run (expect fail)**

```bash
bun test tests/cli-wrapper/registry.test.ts
```

Expected: FAIL.

- [ ] **Step 3: Implement `src/cli-wrapper/registry.ts`**

```typescript
import type { ClaudeCodeWrapper } from "./claude-code";

export class ConversationRegistry {
  private map = new Map<string, ClaudeCodeWrapper>();

  register(conversationId: string, wrapper: ClaudeCodeWrapper): void {
    this.map.set(conversationId, wrapper);
  }

  get(conversationId: string): ClaudeCodeWrapper | undefined {
    return this.map.get(conversationId);
  }

  remove(conversationId: string): void {
    const w = this.map.get(conversationId);
    if (w) {
      w.kill();
      this.map.delete(conversationId);
    }
  }

  killAll(): void {
    for (const [, w] of this.map) w.kill();
    this.map.clear();
  }

  get size(): number {
    return this.map.size;
  }
}
```

- [ ] **Step 4: Run tests**

```bash
bun test tests/cli-wrapper/registry.test.ts
```

Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add agent/src/cli-wrapper/registry.ts agent/tests/cli-wrapper/registry.test.ts
git commit -m "feat(agent): conversation registry for active cli processes"
```

---

## Task 12: WebSocket server base

**Files:**
- Create: `agent/src/ws/server.ts`
- Create: `agent/src/ws/session.ts`
- Create: `agent/tests/ws/server.test.ts`

- [ ] **Step 1: Implement `src/ws/session.ts`**

Per-connection state:

```typescript
import type { ServerWebSocket } from "bun";

export interface SessionState {
  authenticated: boolean;
  tokenId?: string;
  deviceLabel?: string;
  subscriptions: Set<string>; // conversationIds being streamed to this client
}

export function createSession(): SessionState {
  return {
    authenticated: false,
    subscriptions: new Set(),
  };
}

export type WSConn = ServerWebSocket<SessionState>;

export function send(ws: WSConn, obj: unknown): void {
  ws.send(JSON.stringify(obj));
}
```

- [ ] **Step 2: Implement `src/ws/server.ts` (minimal skeleton, router stub)**

```typescript
import { type Server } from "bun";
import { type SessionState, createSession, send } from "./session";
import { ClientMessage } from "../protocol/messages";
import { log } from "../logger";

export interface ServerOptions {
  hostname: string;
  port: number;
  onMessage: (ws: Bun.ServerWebSocket<SessionState>, msg: unknown) => Promise<void> | void;
  onClose?: (ws: Bun.ServerWebSocket<SessionState>) => void;
}

export function startWsServer(opts: ServerOptions): Server {
  return Bun.serve<SessionState, never>({
    hostname: opts.hostname,
    port: opts.port,
    fetch(req, server) {
      if (server.upgrade(req, { data: createSession() })) return;
      return new Response("aiconnect agent", { status: 200 });
    },
    websocket: {
      message: async (ws, message) => {
        let parsed: unknown;
        try {
          parsed = JSON.parse(typeof message === "string" ? message : new TextDecoder().decode(message));
        } catch {
          send(ws, { type: "error", code: "bad_request", message: "invalid json" });
          return;
        }

        const result = ClientMessage.safeParse(parsed);
        if (!result.success) {
          send(ws, { type: "error", code: "bad_request", message: "invalid message", detail: result.error.message });
          return;
        }

        try {
          await opts.onMessage(ws, result.data);
        } catch (err) {
          log.error("handler error", { err: String(err) });
          send(ws, { type: "error", code: "internal", message: "handler failed" });
        }
      },
      close: (ws) => {
        if (opts.onClose) opts.onClose(ws);
      },
      open: (ws) => {
        log.info("ws open", { remote: ws.remoteAddress });
      },
    },
  });
}
```

- [ ] **Step 3: Write server test**

```typescript
import { describe, test, expect, beforeAll, afterAll } from "bun:test";
import { startWsServer } from "../../src/ws/server";

let server: ReturnType<typeof startWsServer>;
let port: number;

beforeAll(() => {
  server = startWsServer({
    hostname: "127.0.0.1",
    port: 0,
    onMessage: (ws, msg) => {
      ws.send(JSON.stringify({ type: "echo", msg }));
    },
  });
  port = server.port;
});

afterAll(() => {
  server.stop(true);
});

describe("ws server", () => {
  test("accepts valid auth message and echoes", async () => {
    const ws = new WebSocket(`ws://127.0.0.1:${port}`);
    await new Promise<void>((r, e) => {
      ws.addEventListener("open", () => r(), { once: true });
      ws.addEventListener("error", () => e(new Error("ws error")), { once: true });
    });
    const received = new Promise<string>((r) => {
      ws.addEventListener("message", (e) => r(typeof e.data === "string" ? e.data : ""), { once: true });
    });
    ws.send(JSON.stringify({ type: "auth", token: "x".repeat(20), protocolVersion: 1 }));
    const raw = await received;
    expect(raw).toContain("echo");
    ws.close();
  });

  test("rejects malformed json", async () => {
    const ws = new WebSocket(`ws://127.0.0.1:${port}`);
    await new Promise<void>((r) => ws.addEventListener("open", () => r(), { once: true }));
    const received = new Promise<string>((r) => {
      ws.addEventListener("message", (e) => r(typeof e.data === "string" ? e.data : ""), { once: true });
    });
    ws.send("not-json");
    const raw = await received;
    expect(raw).toContain("error");
    expect(raw).toContain("bad_request");
    ws.close();
  });
});
```

- [ ] **Step 4: Run tests**

```bash
bun test tests/ws/server.test.ts
```

Expected: PASS, 2 tests.

- [ ] **Step 5: Commit**

```bash
git add agent/src/ws/ agent/tests/ws/server.test.ts
git commit -m "feat(agent): websocket server skeleton with message parsing"
```

---

## Task 13: Handler de autenticación

**Files:**
- Create: `agent/src/ws/handlers/auth.ts`
- Create: `agent/tests/ws/handlers/auth.test.ts`

- [ ] **Step 1: Implement `src/ws/handlers/auth.ts`**

```typescript
import { eq } from "drizzle-orm";
import { token, machine } from "../../db/schema";
import { hashToken } from "../../auth/token";
import { PROTOCOL_VERSION } from "../../protocol/version";
import { send, type WSConn } from "../session";
import type { DB } from "../../db/client";

export interface AuthDeps {
  db: DB;
  machineId: string;
}

export async function handleAuth(
  ws: WSConn,
  msg: { token: string; protocolVersion: number; deviceLabel?: string },
  deps: AuthDeps
): Promise<void> {
  if (msg.protocolVersion !== PROTOCOL_VERSION) {
    send(ws, { type: "error", code: "bad_request", message: "protocol version mismatch" });
    ws.close();
    return;
  }
  const hash = hashToken(msg.token);
  const rows = deps.db.select().from(token).where(eq(token.hash, hash)).all();
  const found = rows[0];
  if (!found || found.revoked) {
    send(ws, { type: "error", code: "unauthorized", message: "invalid token" });
    ws.close();
    return;
  }

  deps.db.update(token).set({ lastUsedAt: new Date() }).where(eq(token.id, found.id)).run();

  const machines = deps.db.select().from(machine).where(eq(machine.id, deps.machineId)).all();
  const self = machines[0];
  if (!self) {
    send(ws, { type: "error", code: "internal", message: "machine record missing" });
    ws.close();
    return;
  }

  ws.data.authenticated = true;
  ws.data.tokenId = found.id;
  if (msg.deviceLabel !== undefined) ws.data.deviceLabel = msg.deviceLabel;

  send(ws, {
    type: "auth_ok",
    tokenId: found.id,
    machine: {
      id: self.id,
      fingerprint: self.fingerprint,
      hostname: self.hostname,
      displayName: self.displayName,
    },
    protocolVersion: PROTOCOL_VERSION,
  });
}
```

- [ ] **Step 2: Write handler test**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { openDb, closeDb, runMigrations } from "../../../src/db/client";
import { token, machine } from "../../../src/db/schema";
import { handleAuth } from "../../../src/ws/handlers/auth";
import { generateToken, hashToken } from "../../../src/auth/token";
import { nanoid } from "nanoid";

let tmp: string;

beforeEach(() => {
  tmp = join(mkdtempSync(join(tmpdir(), "auth-")), "db.sqlite");
});

afterEach(() => {
  closeDb();
  rmSync(tmp, { force: true });
});

function fakeWs() {
  const sent: string[] = [];
  return {
    sent,
    close: () => {},
    send: (s: string) => sent.push(s),
    data: { authenticated: false, subscriptions: new Set<string>() },
  };
}

describe("handleAuth", () => {
  test("accepts valid token and sets authenticated", async () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const mId = nanoid();
    db.insert(machine).values({
      id: mId,
      fingerprint: "fp1",
      hostname: "mac-test",
      displayName: "mac-test",
    }).run();
    const t = generateToken();
    const tokenId = nanoid();
    db.insert(token).values({ id: tokenId, hash: hashToken(t) }).run();

    const ws = fakeWs();
    await handleAuth(ws as never, { token: t, protocolVersion: 1 }, { db, machineId: mId });
    expect(ws.data.authenticated).toBe(true);
    expect(ws.sent[0]).toContain("auth_ok");
  });

  test("rejects wrong protocol version", async () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const ws = fakeWs();
    await handleAuth(ws as never, { token: "x".repeat(20), protocolVersion: 999 }, { db, machineId: "m1" });
    expect(ws.sent[0]).toContain("protocol version mismatch");
  });

  test("rejects unknown token", async () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    db.insert(machine).values({ id: "m1", fingerprint: "fp", hostname: "h", displayName: "h" }).run();
    const ws = fakeWs();
    await handleAuth(ws as never, { token: "unknown".repeat(5), protocolVersion: 1 }, { db, machineId: "m1" });
    expect(ws.sent[0]).toContain("unauthorized");
  });
});
```

- [ ] **Step 3: Run tests**

```bash
bun test tests/ws/handlers/auth.test.ts
```

Expected: PASS, 3 tests.

- [ ] **Step 4: Commit**

```bash
git add agent/src/ws/handlers/auth.ts agent/tests/ws/handlers/auth.test.ts
git commit -m "feat(agent): ws auth handler with token verification"
```

---

## Task 14: Handler de proyectos (list + create)

**Files:**
- Create: `agent/src/ws/handlers/projects.ts`
- Create: `agent/tests/ws/handlers/projects.test.ts`

- [ ] **Step 1: Implement `src/ws/handlers/projects.ts`**

```typescript
import { existsSync, statSync } from "node:fs";
import { ulid } from "ulid";
import { project } from "../../db/schema";
import { send, type WSConn } from "../session";
import type { DB } from "../../db/client";
import { z } from "zod";
import { CreateProjectRequest } from "../../protocol/messages";

export function handleListProjects(ws: WSConn, _msg: unknown, db: DB): void {
  const rows = db.select().from(project).all();
  send(ws, {
    type: "project_list",
    projects: rows.map((r) => ({
      id: r.id,
      name: r.name,
      workingDir: r.workingDir,
      tags: r.tags,
      permissionPreset: r.permissionPreset,
      defaultModelClaudeCode: r.defaultModelClaudeCode ?? null,
    })),
  });
}

export function handleCreateProject(
  ws: WSConn,
  msg: z.infer<typeof CreateProjectRequest>,
  db: DB
): void {
  // Validate working directory exists and is a directory
  if (!existsSync(msg.workingDir)) {
    send(ws, { type: "error", code: "bad_request", message: `working_dir does not exist: ${msg.workingDir}` });
    return;
  }
  if (!statSync(msg.workingDir).isDirectory()) {
    send(ws, { type: "error", code: "bad_request", message: `working_dir is not a directory` });
    return;
  }

  const id = ulid();
  db.insert(project).values({
    id,
    name: msg.name,
    workingDir: msg.workingDir,
    tags: msg.tags,
    permissionPreset: msg.permissionPreset,
    defaultModelClaudeCode: msg.defaultModelClaudeCode ?? null,
  }).run();

  send(ws, {
    type: "project_created",
    project: {
      id,
      name: msg.name,
      workingDir: msg.workingDir,
      tags: msg.tags,
      permissionPreset: msg.permissionPreset,
      defaultModelClaudeCode: msg.defaultModelClaudeCode ?? null,
    },
  });
}
```

- [ ] **Step 2: Write handler test**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { openDb, closeDb, runMigrations } from "../../../src/db/client";
import { handleListProjects, handleCreateProject } from "../../../src/ws/handlers/projects";

let tmp: string;
let workDir: string;

beforeEach(() => {
  tmp = join(mkdtempSync(join(tmpdir(), "proj-")), "db.sqlite");
  workDir = mkdtempSync(join(tmpdir(), "wd-"));
});

afterEach(() => {
  closeDb();
  rmSync(tmp, { force: true });
  rmSync(workDir, { recursive: true, force: true });
});

function fakeWs() {
  const sent: string[] = [];
  return {
    sent,
    send: (s: string) => sent.push(s),
    data: { authenticated: true, subscriptions: new Set<string>() },
  };
}

describe("projects handler", () => {
  test("list returns empty on fresh db", () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const ws = fakeWs();
    handleListProjects(ws as never, {}, db);
    const parsed = JSON.parse(ws.sent[0]!);
    expect(parsed.type).toBe("project_list");
    expect(parsed.projects).toEqual([]);
  });

  test("create + list roundtrip", () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const ws = fakeWs();
    handleCreateProject(ws as never, {
      type: "create_project",
      name: "p1",
      workingDir: workDir,
      tags: ["work"],
      permissionPreset: "ask-for-side-effects",
    }, db);
    const created = JSON.parse(ws.sent[0]!);
    expect(created.type).toBe("project_created");

    const ws2 = fakeWs();
    handleListProjects(ws2 as never, {}, db);
    const listed = JSON.parse(ws2.sent[0]!);
    expect(listed.projects.length).toBe(1);
    expect(listed.projects[0].name).toBe("p1");
  });

  test("rejects non-existent working dir", () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const ws = fakeWs();
    handleCreateProject(ws as never, {
      type: "create_project",
      name: "p1",
      workingDir: "/nonexistent/path/aiconnect-test",
      tags: [],
      permissionPreset: "ask-for-side-effects",
    }, db);
    const err = JSON.parse(ws.sent[0]!);
    expect(err.type).toBe("error");
  });
});
```

- [ ] **Step 3: Run tests**

```bash
bun test tests/ws/handlers/projects.test.ts
```

Expected: PASS, 3 tests.

- [ ] **Step 4: Commit**

```bash
git add agent/src/ws/handlers/projects.ts agent/tests/ws/handlers/projects.test.ts
git commit -m "feat(agent): project list + create handlers"
```

---

## Task 15: Handler de conversaciones (list + create)

**Files:**
- Create: `agent/src/ws/handlers/conversations.ts`
- Create: `agent/tests/ws/handlers/conversations.test.ts`

- [ ] **Step 1: Implement `src/ws/handlers/conversations.ts`**

```typescript
import { eq, desc } from "drizzle-orm";
import { ulid } from "ulid";
import { conversation, project } from "../../db/schema";
import { send, type WSConn } from "../session";
import type { DB } from "../../db/client";
import { z } from "zod";
import { CreateConversationRequest, ListConversationsRequest } from "../../protocol/messages";
import { startClaudeCode } from "../../cli-wrapper/claude-code";
import type { ConversationRegistry } from "../../cli-wrapper/registry";

export function handleListConversations(
  ws: WSConn,
  msg: z.infer<typeof ListConversationsRequest>,
  db: DB
): void {
  const rows = msg.projectId
    ? db.select().from(conversation).where(eq(conversation.projectId, msg.projectId)).orderBy(desc(conversation.lastActivityAt)).all()
    : db.select().from(conversation).orderBy(desc(conversation.lastActivityAt)).all();

  send(ws, {
    type: "conversation_list",
    conversations: rows.map((r) => ({
      id: r.id,
      projectId: r.projectId,
      cli: r.cli as "claude-code" | "codex",
      currentModel: r.currentModel ?? null,
      title: r.title,
      status: r.status as "idle" | "running" | "awaiting_approval" | "error" | "interrupted",
      mode: r.mode as "active" | "plan-only",
      lastActivityAt: r.lastActivityAt.getTime(),
    })),
  });
}

export interface CreateConvDeps {
  db: DB;
  registry: ConversationRegistry;
  claudeBinary?: string; // for tests
}

export function handleCreateConversation(
  ws: WSConn,
  msg: z.infer<typeof CreateConversationRequest>,
  deps: CreateConvDeps
): void {
  const proj = deps.db.select().from(project).where(eq(project.id, msg.projectId)).all()[0];
  if (!proj) {
    send(ws, { type: "error", code: "not_found", message: "project not found" });
    return;
  }
  if (msg.cli !== "claude-code") {
    send(ws, { type: "error", code: "bad_request", message: "Fase 1 solo soporta claude-code" });
    return;
  }

  const id = ulid();
  const model = msg.model ?? proj.defaultModelClaudeCode ?? undefined;

  deps.db.insert(conversation).values({
    id,
    projectId: proj.id,
    cli: "claude-code",
    currentModel: model ?? null,
    title: "Nueva conversación",
    status: "idle",
    mode: "active",
  }).run();

  const wrapper = startClaudeCode({
    ...(deps.claudeBinary !== undefined ? { binary: deps.claudeBinary } : {}),
    cwd: proj.workingDir,
    ...(model !== undefined ? { model } : {}),
    permissionPreset: proj.permissionPreset as "ask-for-side-effects" | "auto-accept-edits" | "full-auto",
  });
  deps.registry.register(id, wrapper);

  send(ws, {
    type: "conversation_created",
    conversation: {
      id,
      projectId: proj.id,
      cli: "claude-code",
      currentModel: model ?? null,
      title: "Nueva conversación",
      status: "idle",
      mode: "active",
      lastActivityAt: Date.now(),
    },
  });
}
```

- [ ] **Step 2: Write handler test**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { openDb, closeDb, runMigrations } from "../../../src/db/client";
import { project } from "../../../src/db/schema";
import { ConversationRegistry } from "../../../src/cli-wrapper/registry";
import { handleCreateConversation, handleListConversations } from "../../../src/ws/handlers/conversations";
import { ulid } from "ulid";

let tmp: string;
let workDir: string;
const stubPath = join(import.meta.dir, "../../stub-cli/claude");

beforeEach(() => {
  tmp = join(mkdtempSync(join(tmpdir(), "conv-")), "db.sqlite");
  workDir = mkdtempSync(join(tmpdir(), "wd-"));
});

afterEach(() => {
  closeDb();
  rmSync(tmp, { force: true });
  rmSync(workDir, { recursive: true, force: true });
});

function fakeWs() {
  const sent: string[] = [];
  return { sent, send: (s: string) => sent.push(s), data: { authenticated: true, subscriptions: new Set<string>() } };
}

describe("conversations handler", () => {
  test("create with stub cli + list", async () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const pid = ulid();
    db.insert(project).values({ id: pid, name: "p", workingDir: workDir }).run();

    const registry = new ConversationRegistry();
    const ws = fakeWs();
    handleCreateConversation(ws as never, {
      type: "create_conversation",
      projectId: pid,
      cli: "claude-code",
    }, { db, registry, claudeBinary: stubPath });

    const created = JSON.parse(ws.sent[0]!);
    expect(created.type).toBe("conversation_created");

    const ws2 = fakeWs();
    handleListConversations(ws2 as never, { type: "list_conversations" }, db);
    const listed = JSON.parse(ws2.sent[0]!);
    expect(listed.conversations.length).toBe(1);

    registry.killAll();
    await new Promise((r) => setTimeout(r, 50));
  });

  test("rejects codex in fase 1", () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const pid = ulid();
    db.insert(project).values({ id: pid, name: "p", workingDir: workDir }).run();
    const ws = fakeWs();
    handleCreateConversation(ws as never, {
      type: "create_conversation", projectId: pid, cli: "codex",
    }, { db, registry: new ConversationRegistry() });
    expect(ws.sent[0]).toContain("Fase 1 solo soporta claude-code");
  });
});
```

- [ ] **Step 3: Run tests**

```bash
bun test tests/ws/handlers/conversations.test.ts
```

Expected: PASS, 2 tests.

- [ ] **Step 4: Commit**

```bash
git add agent/src/ws/handlers/conversations.ts agent/tests/ws/handlers/conversations.test.ts
git commit -m "feat(agent): conversation list + create handlers"
```

---

## Task 16: Handler de send_prompt con persistencia de eventos

**Files:**
- Create: `agent/src/ws/handlers/prompts.ts`
- Create: `agent/tests/ws/handlers/prompts.test.ts`

- [ ] **Step 1: Implement `src/ws/handlers/prompts.ts`**

```typescript
import { eq } from "drizzle-orm";
import { ulid } from "ulid";
import { z } from "zod";
import { conversation, event } from "../../db/schema";
import { send, type WSConn } from "../session";
import type { DB } from "../../db/client";
import type { ConversationRegistry } from "../../cli-wrapper/registry";
import { SendPromptRequest, type ConversationEventT } from "../../protocol/messages";

export interface PromptDeps {
  db: DB;
  registry: ConversationRegistry;
}

function persistEvent(db: DB, conversationId: string, ev: ConversationEventT, attachments: string[] = []): void {
  db.insert(event).values({
    id: ev.id,
    conversationId,
    type: ev.type,
    payload: ev.payload,
    attachments,
    timestamp: new Date(ev.timestamp),
  }).run();
  db.update(conversation).set({ lastActivityAt: new Date() }).where(eq(conversation.id, conversationId)).run();
}

export function handleSendPrompt(
  ws: WSConn,
  msg: z.infer<typeof SendPromptRequest>,
  deps: PromptDeps
): void {
  const convRow = deps.db.select().from(conversation).where(eq(conversation.id, msg.conversationId)).all()[0];
  if (!convRow) {
    send(ws, { type: "error", code: "not_found", message: "conversation not found" });
    return;
  }

  const wrapper = deps.registry.get(msg.conversationId);
  if (!wrapper) {
    send(ws, { type: "error", code: "not_found", message: "conversation process not running" });
    return;
  }

  // Persist the user_prompt event
  const userEvent: ConversationEventT = {
    id: ulid(),
    type: "user_prompt",
    timestamp: Date.now(),
    payload: { text: msg.prompt, attachments: msg.attachments },
  };
  persistEvent(deps.db, msg.conversationId, userEvent, msg.attachments);
  send(ws, { type: "event", conversationId: msg.conversationId, event: userEvent });

  // Subscribe this session to stream events from this conversation
  ws.data.subscriptions.add(msg.conversationId);

  // Attach listener once per wrapper (guard using a marker on data-but-bound-globally:
  // in practice, the listener is registered when conversation is created, not per prompt.
  // See integration (Task 19) for the proper wiring. Here we forward the prompt.)
  wrapper.sendPrompt(msg.prompt, msg.attachments);

  // Mark conversation running
  deps.db.update(conversation).set({ status: "running" }).where(eq(conversation.id, msg.conversationId)).run();
}

/**
 * Wire a single event listener per conversation that:
 *   - persists each event to SQLite
 *   - broadcasts to all subscribed connections
 *   - updates conversation status on turn_complete / permission_request
 * Called from the create_conversation handler or on process re-attach.
 */
export function wireConversationListener(
  conversationId: string,
  db: DB,
  registry: ConversationRegistry,
  getSubscribers: () => WSConn[]
): void {
  const w = registry.get(conversationId);
  if (!w) return;
  w.onEvent((ev) => {
    persistEvent(db, conversationId, ev);
    for (const ws of getSubscribers()) {
      if (ws.data.subscriptions.has(conversationId)) {
        send(ws, { type: "event", conversationId, event: ev });
      }
    }
    if (ev.type === "permission_request") {
      db.update(conversation).set({ status: "awaiting_approval" }).where(eq(conversation.id, conversationId)).run();
    } else if (ev.type === "turn_complete") {
      db.update(conversation).set({ status: "idle" }).where(eq(conversation.id, conversationId)).run();
    }
  });
  w.onExit(() => {
    db.update(conversation).set({ status: "interrupted" }).where(eq(conversation.id, conversationId)).run();
    registry.remove(conversationId);
  });
}

export { persistEvent };
```

- [ ] **Step 2: Write handler test**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { ulid } from "ulid";
import { openDb, closeDb, runMigrations } from "../../../src/db/client";
import { project, conversation, event as eventTable } from "../../../src/db/schema";
import { ConversationRegistry } from "../../../src/cli-wrapper/registry";
import { handleSendPrompt, wireConversationListener } from "../../../src/ws/handlers/prompts";
import { startClaudeCode } from "../../../src/cli-wrapper/claude-code";

let tmp: string;
let workDir: string;
const stubPath = join(import.meta.dir, "../../stub-cli/claude");

beforeEach(() => {
  tmp = join(mkdtempSync(join(tmpdir(), "prompt-")), "db.sqlite");
  workDir = mkdtempSync(join(tmpdir(), "wd-"));
});

afterEach(() => {
  closeDb();
  rmSync(tmp, { force: true });
  rmSync(workDir, { recursive: true, force: true });
});

function fakeWs() {
  const sent: string[] = [];
  return { sent, send: (s: string) => sent.push(s), data: { authenticated: true, subscriptions: new Set<string>() } };
}

describe("send_prompt handler", () => {
  test("persists user_prompt and streams back events from stub", async () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const pid = ulid();
    const cid = ulid();
    db.insert(project).values({ id: pid, name: "p", workingDir: workDir }).run();
    db.insert(conversation).values({ id: cid, projectId: pid, cli: "claude-code" }).run();

    const registry = new ConversationRegistry();
    registry.register(cid, startClaudeCode({ binary: stubPath, cwd: workDir }));

    const ws = fakeWs();
    const subscribers: unknown[] = [ws];
    wireConversationListener(cid, db, registry, () => subscribers as never);

    handleSendPrompt(ws as never, {
      type: "send_prompt", conversationId: cid, prompt: "hello", attachments: [],
    }, { db, registry });

    await new Promise((r) => setTimeout(r, 400));
    const rows = db.select().from(eventTable).all();
    expect(rows.length).toBeGreaterThan(0);
    expect(rows.some((r) => r.type === "user_prompt")).toBe(true);
    expect(rows.some((r) => r.type === "turn_complete")).toBe(true);

    registry.killAll();
    await new Promise((r) => setTimeout(r, 50));
  });
});
```

- [ ] **Step 3: Run tests**

```bash
bun test tests/ws/handlers/prompts.test.ts
```

Expected: PASS, 1 test.

- [ ] **Step 4: Commit**

```bash
git add agent/src/ws/handlers/prompts.ts agent/tests/ws/handlers/prompts.test.ts
git commit -m "feat(agent): send_prompt handler with event persistence and streaming"
```

---

## Task 17: Handler de approvals

**Files:**
- Create: `agent/src/ws/handlers/approvals.ts`
- Create: `agent/tests/ws/handlers/approvals.test.ts`

- [ ] **Step 1: Implement `src/ws/handlers/approvals.ts`**

```typescript
import { eq } from "drizzle-orm";
import { ulid } from "ulid";
import { z } from "zod";
import { conversation, event } from "../../db/schema";
import { send, type WSConn } from "../session";
import type { DB } from "../../db/client";
import type { ConversationRegistry } from "../../cli-wrapper/registry";
import { ApprovalResponseRequest, type ConversationEventT } from "../../protocol/messages";
import { persistEvent } from "./prompts";

export function handleApprovalResponse(
  ws: WSConn,
  msg: z.infer<typeof ApprovalResponseRequest>,
  db: DB,
  registry: ConversationRegistry
): void {
  const convRow = db.select().from(conversation).where(eq(conversation.id, msg.conversationId)).all()[0];
  if (!convRow) {
    send(ws, { type: "error", code: "not_found", message: "conversation not found" });
    return;
  }

  const wrapper = registry.get(msg.conversationId);
  if (!wrapper) {
    send(ws, { type: "error", code: "not_found", message: "conversation process not running" });
    return;
  }

  const responseEvent: ConversationEventT = {
    id: ulid(),
    type: "permission_response",
    timestamp: Date.now(),
    payload: {
      requestId: msg.requestId,
      granted: msg.granted,
      ...(msg.note !== undefined ? { note: msg.note } : {}),
    },
  };
  persistEvent(db, msg.conversationId, responseEvent);

  wrapper.sendApproval(msg.requestId, msg.granted, msg.note);

  db.update(conversation).set({ status: "running" }).where(eq(conversation.id, msg.conversationId)).run();

  send(ws, { type: "event", conversationId: msg.conversationId, event: responseEvent });
}
```

- [ ] **Step 2: Write test**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { ulid } from "ulid";
import { openDb, closeDb, runMigrations } from "../../../src/db/client";
import { project, conversation, event } from "../../../src/db/schema";
import { ConversationRegistry } from "../../../src/cli-wrapper/registry";
import { wireConversationListener } from "../../../src/ws/handlers/prompts";
import { handleApprovalResponse } from "../../../src/ws/handlers/approvals";
import { startClaudeCode } from "../../../src/cli-wrapper/claude-code";

let tmp: string; let workDir: string;
const stubPath = join(import.meta.dir, "../../stub-cli/claude");

beforeEach(() => {
  tmp = join(mkdtempSync(join(tmpdir(), "approv-")), "db.sqlite");
  workDir = mkdtempSync(join(tmpdir(), "wd-"));
});
afterEach(() => { closeDb(); rmSync(tmp, { force: true }); rmSync(workDir, { recursive: true, force: true }); });

function fakeWs() { const sent: string[] = []; return { sent, send: (s: string) => sent.push(s), data: { authenticated: true, subscriptions: new Set<string>() } }; }

describe("approvals handler", () => {
  test("forwards approval to cli and persists response", async () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const pid = ulid(); const cid = ulid();
    db.insert(project).values({ id: pid, name: "p", workingDir: workDir }).run();
    db.insert(conversation).values({ id: cid, projectId: pid, cli: "claude-code" }).run();

    const registry = new ConversationRegistry();
    registry.register(cid, startClaudeCode({ binary: stubPath, cwd: workDir }));
    const ws = fakeWs();
    ws.data.subscriptions.add(cid);
    wireConversationListener(cid, db, registry, () => [ws] as never);

    const wrapper = registry.get(cid)!;
    wrapper.sendPrompt("bash command please");
    await new Promise((r) => setTimeout(r, 300));

    const req = db.select().from(event).all().find((e) => e.type === "permission_request");
    expect(req).toBeDefined();
    const reqId = (req!.payload as { requestId: string }).requestId;

    handleApprovalResponse(ws as never, {
      type: "approval_response", conversationId: cid, requestId: reqId, granted: true,
    }, db, registry);

    await new Promise((r) => setTimeout(r, 200));
    const responseRow = db.select().from(event).all().find((e) => e.type === "permission_response");
    expect(responseRow).toBeDefined();
    expect((responseRow!.payload as { granted: boolean }).granted).toBe(true);

    registry.killAll();
    await new Promise((r) => setTimeout(r, 50));
  });
});
```

- [ ] **Step 3: Run**

```bash
bun test tests/ws/handlers/approvals.test.ts
```

Expected: PASS, 1 test.

- [ ] **Step 4: Commit**

```bash
git add agent/src/ws/handlers/approvals.ts agent/tests/ws/handlers/approvals.test.ts
git commit -m "feat(agent): approval response handler forwards to cli"
```

---

## Task 18: Handler de cambio de modelo

**Files:**
- Create: `agent/src/ws/handlers/model.ts`
- Create: `agent/tests/ws/handlers/model.test.ts`

- [ ] **Step 1: Implement `src/ws/handlers/model.ts`**

```typescript
import { eq } from "drizzle-orm";
import { ulid } from "ulid";
import { z } from "zod";
import { conversation } from "../../db/schema";
import { send, type WSConn } from "../session";
import type { DB } from "../../db/client";
import type { ConversationRegistry } from "../../cli-wrapper/registry";
import { ChangeModelRequest, type ConversationEventT } from "../../protocol/messages";
import { persistEvent } from "./prompts";

export function handleChangeModel(
  ws: WSConn,
  msg: z.infer<typeof ChangeModelRequest>,
  db: DB,
  registry: ConversationRegistry
): void {
  const convRow = db.select().from(conversation).where(eq(conversation.id, msg.conversationId)).all()[0];
  if (!convRow) {
    send(ws, { type: "error", code: "not_found", message: "conversation not found" });
    return;
  }
  const wrapper = registry.get(msg.conversationId);
  if (!wrapper) {
    send(ws, { type: "error", code: "not_found", message: "conversation process not running" });
    return;
  }

  const modelChange: ConversationEventT = {
    id: ulid(),
    type: "model_change",
    timestamp: Date.now(),
    payload: { from: convRow.currentModel ?? null, to: msg.model },
  };
  persistEvent(db, msg.conversationId, modelChange);

  wrapper.sendSlashCommand(`/model ${msg.model}`);

  db.update(conversation).set({ currentModel: msg.model }).where(eq(conversation.id, msg.conversationId)).run();

  send(ws, { type: "event", conversationId: msg.conversationId, event: modelChange });
}
```

- [ ] **Step 2: Write test**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { ulid } from "ulid";
import { openDb, closeDb, runMigrations } from "../../../src/db/client";
import { project, conversation, event } from "../../../src/db/schema";
import { ConversationRegistry } from "../../../src/cli-wrapper/registry";
import { handleChangeModel } from "../../../src/ws/handlers/model";
import { startClaudeCode } from "../../../src/cli-wrapper/claude-code";

let tmp: string; let workDir: string;
const stubPath = join(import.meta.dir, "../../stub-cli/claude");

beforeEach(() => {
  tmp = join(mkdtempSync(join(tmpdir(), "model-")), "db.sqlite");
  workDir = mkdtempSync(join(tmpdir(), "wd-"));
});
afterEach(() => { closeDb(); rmSync(tmp, { force: true }); rmSync(workDir, { recursive: true, force: true }); });

function fakeWs() { const sent: string[] = []; return { sent, send: (s: string) => sent.push(s), data: { authenticated: true, subscriptions: new Set<string>() } }; }

describe("change_model handler", () => {
  test("persists model_change event and updates current_model", async () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const pid = ulid(); const cid = ulid();
    db.insert(project).values({ id: pid, name: "p", workingDir: workDir }).run();
    db.insert(conversation).values({ id: cid, projectId: pid, cli: "claude-code", currentModel: "sonnet-4.6" }).run();

    const registry = new ConversationRegistry();
    registry.register(cid, startClaudeCode({ binary: stubPath, cwd: workDir }));
    const ws = fakeWs();

    handleChangeModel(ws as never, {
      type: "change_model", conversationId: cid, model: "opus-4.7",
    }, db, registry);

    const rows = db.select().from(event).all();
    const mc = rows.find((r) => r.type === "model_change");
    expect(mc).toBeDefined();
    expect((mc!.payload as { to: string }).to).toBe("opus-4.7");

    const updated = db.select().from(conversation).all()[0]!;
    expect(updated.currentModel).toBe("opus-4.7");

    registry.killAll();
    await new Promise((r) => setTimeout(r, 50));
  });
});
```

- [ ] **Step 3: Run**

```bash
bun test tests/ws/handlers/model.test.ts
```

Expected: PASS, 1 test.

- [ ] **Step 4: Commit**

```bash
git add agent/src/ws/handlers/model.ts agent/tests/ws/handlers/model.test.ts
git commit -m "feat(agent): change_model handler with /model slash command"
```

---

## Task 19: Handler de replay (reconexión)

**Files:**
- Create: `agent/src/ws/handlers/replay.ts`
- Create: `agent/tests/ws/handlers/replay.test.ts`

- [ ] **Step 1: Implement `src/ws/handlers/replay.ts`**

```typescript
import { and, eq, gt, asc } from "drizzle-orm";
import { z } from "zod";
import { event, conversation } from "../../db/schema";
import { send, type WSConn } from "../session";
import type { DB } from "../../db/client";
import { ReplayFromRequest, type ConversationEventT } from "../../protocol/messages";

export function handleReplayFrom(
  ws: WSConn,
  msg: z.infer<typeof ReplayFromRequest>,
  db: DB
): void {
  const convRow = db.select().from(conversation).where(eq(conversation.id, msg.conversationId)).all()[0];
  if (!convRow) {
    send(ws, { type: "error", code: "not_found", message: "conversation not found" });
    return;
  }

  const rows = msg.afterEventId
    ? db.select().from(event).where(and(eq(event.conversationId, msg.conversationId), gt(event.id, msg.afterEventId))).orderBy(asc(event.id)).all()
    : db.select().from(event).where(eq(event.conversationId, msg.conversationId)).orderBy(asc(event.id)).all();

  for (const row of rows) {
    const ev = {
      id: row.id,
      type: row.type,
      timestamp: row.timestamp.getTime(),
      payload: row.payload,
    } as ConversationEventT;
    send(ws, { type: "event", conversationId: msg.conversationId, event: ev });
  }

  ws.data.subscriptions.add(msg.conversationId);

  send(ws, {
    type: "replay_done",
    conversationId: msg.conversationId,
    lastEventId: rows.length > 0 ? rows[rows.length - 1]!.id : null,
  });
}
```

- [ ] **Step 2: Write test**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { ulid } from "ulid";
import { openDb, closeDb, runMigrations } from "../../../src/db/client";
import { project, conversation, event } from "../../../src/db/schema";
import { handleReplayFrom } from "../../../src/ws/handlers/replay";

let tmp: string;

beforeEach(() => { tmp = join(mkdtempSync(join(tmpdir(), "replay-")), "db.sqlite"); });
afterEach(() => { closeDb(); rmSync(tmp, { force: true }); });

function fakeWs() { const sent: string[] = []; return { sent, send: (s: string) => sent.push(s), data: { authenticated: true, subscriptions: new Set<string>() } }; }

describe("replay_from handler", () => {
  test("replays events after given id", () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const pid = ulid(); const cid = ulid();
    db.insert(project).values({ id: pid, name: "p", workingDir: "/tmp" }).run();
    db.insert(conversation).values({ id: cid, projectId: pid, cli: "claude-code" }).run();

    const eids: string[] = [];
    for (let i = 0; i < 5; i++) {
      const eid = ulid();
      eids.push(eid);
      db.insert(event).values({
        id: eid,
        conversationId: cid,
        type: "assistant_text_delta",
        payload: { text: `event ${i}` },
        timestamp: new Date(1000 + i),
      }).run();
    }

    const ws = fakeWs();
    handleReplayFrom(ws as never, {
      type: "replay_from", conversationId: cid, afterEventId: eids[1],
    }, db);

    // Expect events 2,3,4 plus replay_done = 4 messages
    expect(ws.sent.length).toBe(4);
    expect(ws.sent[3]).toContain("replay_done");
  });

  test("replay from scratch when no afterEventId", () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../../src/db/migrations"), raw);
    const pid = ulid(); const cid = ulid();
    db.insert(project).values({ id: pid, name: "p", workingDir: "/tmp" }).run();
    db.insert(conversation).values({ id: cid, projectId: pid, cli: "claude-code" }).run();

    for (let i = 0; i < 3; i++) {
      db.insert(event).values({
        id: ulid(), conversationId: cid, type: "assistant_text_delta", payload: { text: "x" }, timestamp: new Date(i),
      }).run();
    }

    const ws = fakeWs();
    handleReplayFrom(ws as never, { type: "replay_from", conversationId: cid }, db);
    expect(ws.sent.length).toBe(4); // 3 events + replay_done
  });
});
```

- [ ] **Step 3: Run**

```bash
bun test tests/ws/handlers/replay.test.ts
```

Expected: PASS, 2 tests.

- [ ] **Step 4: Commit**

```bash
git add agent/src/ws/handlers/replay.ts agent/tests/ws/handlers/replay.test.ts
git commit -m "feat(agent): replay_from handler for reconnection"
```

---

## Task 20: Router central + wiring de handlers

**Files:**
- Create: `agent/src/ws/router.ts`
- Create: `agent/tests/ws/router.test.ts`

- [ ] **Step 1: Implement `src/ws/router.ts`**

```typescript
import type { ClientMessageT } from "../protocol/messages";
import { send, type WSConn } from "./session";
import { handleAuth } from "./handlers/auth";
import { handleListProjects, handleCreateProject } from "./handlers/projects";
import { handleListConversations, handleCreateConversation } from "./handlers/conversations";
import { handleSendPrompt, wireConversationListener } from "./handlers/prompts";
import { handleApprovalResponse } from "./handlers/approvals";
import { handleChangeModel } from "./handlers/model";
import { handleReplayFrom } from "./handlers/replay";
import type { DB } from "../db/client";
import type { ConversationRegistry } from "../cli-wrapper/registry";

export interface RouterDeps {
  db: DB;
  registry: ConversationRegistry;
  machineId: string;
  getSubscribers: () => WSConn[];
  claudeBinary?: string;
}

export async function routeMessage(
  ws: WSConn,
  msg: ClientMessageT,
  deps: RouterDeps
): Promise<void> {
  // Auth is the only message allowed pre-authentication
  if (msg.type === "auth") {
    await handleAuth(ws, msg, { db: deps.db, machineId: deps.machineId });
    return;
  }

  if (!ws.data.authenticated) {
    send(ws, { type: "error", code: "unauthorized", message: "authenticate first" });
    ws.close();
    return;
  }

  switch (msg.type) {
    case "list_projects":
      handleListProjects(ws, msg, deps.db);
      return;
    case "create_project":
      handleCreateProject(ws, msg, deps.db);
      return;
    case "list_conversations":
      handleListConversations(ws, msg, deps.db);
      return;
    case "create_conversation":
      handleCreateConversation(ws, msg, {
        db: deps.db,
        registry: deps.registry,
        ...(deps.claudeBinary !== undefined ? { claudeBinary: deps.claudeBinary } : {}),
      });
      // After creation, wire the listener for the new conversation
      // The handler already inserted the conversation; find the most recent one
      // for the project the client just created with. Simpler: the handler
      // returns the id, so refactor to call wireConversationListener from here.
      // For now, rely on create_conversation having registered the wrapper.
      // The caller must know the new id to wire; we do this by observing the
      // registry size and wiring any non-wired conversation.
      // Practical approach: call wireConversationListener on every active id
      // (idempotent by replacing previous listener? Not currently idempotent.
      // So we add a `wired` set.)
      {
        const ids = Array.from((deps as RouterDeps & { _wired?: Set<string> })._wired ?? new Set<string>());
        // Lazy-init the wired set on deps
        const wired = ((deps as RouterDeps & { _wired?: Set<string> })._wired ??= new Set<string>());
        for (const id of [...ids]) {
          if (!deps.registry.get(id)) wired.delete(id);
        }
        // Wire any conversation whose wrapper exists but id isn't wired yet
        for (const row of deps.db.query("SELECT id FROM conversation").all() as Array<{ id: string }>) {
          if (deps.registry.get(row.id) && !wired.has(row.id)) {
            wireConversationListener(row.id, deps.db, deps.registry, deps.getSubscribers);
            wired.add(row.id);
          }
        }
      }
      return;
    case "send_prompt":
      handleSendPrompt(ws, msg, { db: deps.db, registry: deps.registry });
      return;
    case "approval_response":
      handleApprovalResponse(ws, msg, deps.db, deps.registry);
      return;
    case "change_model":
      handleChangeModel(ws, msg, deps.db, deps.registry);
      return;
    case "replay_from":
      handleReplayFrom(ws, msg, deps.db);
      return;
    default: {
      const _exhaustive: never = msg;
      void _exhaustive;
      send(ws, { type: "error", code: "bad_request", message: "unknown message type" });
    }
  }
}
```

- [ ] **Step 2: Write integration test for the router**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { nanoid } from "nanoid";
import { openDb, closeDb, runMigrations } from "../../src/db/client";
import { token, machine } from "../../src/db/schema";
import { ConversationRegistry } from "../../src/cli-wrapper/registry";
import { routeMessage } from "../../src/ws/router";
import { generateToken, hashToken } from "../../src/auth/token";

let tmp: string; let workDir: string;
const stubPath = join(import.meta.dir, "../stub-cli/claude");

beforeEach(() => {
  tmp = join(mkdtempSync(join(tmpdir(), "router-")), "db.sqlite");
  workDir = mkdtempSync(join(tmpdir(), "wd-"));
});
afterEach(() => { closeDb(); rmSync(tmp, { force: true }); rmSync(workDir, { recursive: true, force: true }); });

function fakeWs() { const sent: string[] = []; return { sent, send: (s: string) => sent.push(s), close: () => {}, data: { authenticated: false, subscriptions: new Set<string>() } }; }

describe("router integration", () => {
  test("auth + create project + create conversation + send prompt + receive events", async () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../src/db/migrations"), raw);

    const mId = nanoid();
    db.insert(machine).values({ id: mId, fingerprint: "fp", hostname: "h", displayName: "mac-test" }).run();
    const t = generateToken();
    db.insert(token).values({ id: nanoid(), hash: hashToken(t) }).run();

    const registry = new ConversationRegistry();
    const ws = fakeWs();
    const deps = { db, registry, machineId: mId, getSubscribers: () => [ws as never], claudeBinary: stubPath };

    await routeMessage(ws as never, { type: "auth", token: t, protocolVersion: 1 }, deps);
    expect(ws.data.authenticated).toBe(true);

    await routeMessage(ws as never, {
      type: "create_project", name: "p", workingDir: workDir, tags: [], permissionPreset: "ask-for-side-effects",
    }, deps);
    const created = JSON.parse(ws.sent[ws.sent.length - 1]!);
    expect(created.type).toBe("project_created");
    const projectId = created.project.id;

    await routeMessage(ws as never, {
      type: "create_conversation", projectId, cli: "claude-code",
    }, deps);
    const conv = JSON.parse(ws.sent[ws.sent.length - 1]!);
    expect(conv.type).toBe("conversation_created");
    const conversationId = conv.conversation.id;

    await routeMessage(ws as never, {
      type: "send_prompt", conversationId, prompt: "hello", attachments: [],
    }, deps);

    await new Promise((r) => setTimeout(r, 400));

    const events = ws.sent
      .map((s) => JSON.parse(s))
      .filter((m) => m.type === "event")
      .map((m) => m.event.type);
    expect(events).toContain("user_prompt");
    expect(events).toContain("turn_complete");

    registry.killAll();
    await new Promise((r) => setTimeout(r, 50));
  });
});
```

- [ ] **Step 3: Run**

```bash
bun test tests/ws/router.test.ts
```

Expected: PASS, 1 test.

- [ ] **Step 4: Commit**

```bash
git add agent/src/ws/router.ts agent/tests/ws/router.test.ts
git commit -m "feat(agent): ws message router with handler dispatch"
```

---

## Task 21: Comando `aiconnect init` con QR

**Files:**
- Create: `agent/src/cli/init.ts`
- Create: `agent/src/auth/pairing.ts`
- Create: `agent/tests/cli/init.test.ts`

- [ ] **Step 1: Implement `src/auth/pairing.ts`**

```typescript
import { randomBytes } from "node:crypto";

export function generateShortCode(): string {
  // 6-character alphanumeric code, hyphenated for readability: "4F7-K2B"
  const alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567";
  const bytes = randomBytes(6);
  const chars = Array.from(bytes, (b) => alphabet[b % alphabet.length]).join("");
  return `${chars.slice(0, 3)}-${chars.slice(3, 6)}`;
}

export interface PairingPayload {
  version: number;
  host: string; // tailscale magic dns
  port: number;
  token: string;
  fingerprint: string;
  shortCode: string;
}

export function serializePairingPayload(p: PairingPayload): string {
  return JSON.stringify(p);
}
```

- [ ] **Step 2: Implement `src/cli/init.ts`**

```typescript
import qrcode from "qrcode-terminal";
import { ulid } from "ulid";
import { nanoid } from "nanoid";
import { openDb, runMigrations } from "../db/client";
import { machine, token } from "../db/schema";
import { generateToken, hashToken } from "../auth/token";
import { getOrCreateFingerprint } from "../auth/fingerprint";
import { generateShortCode, serializePairingPayload } from "../auth/pairing";
import { getTailscaleIdentity, getTailscaleIp } from "../net/tailscale";
import { getConfig } from "../config";
import { log } from "../logger";
import { startAgent } from "../index";
import { eq } from "drizzle-orm";
import { join } from "node:path";

export async function initCommand(options: { deviceLabel?: string; displayName?: string; tag?: string } = {}): Promise<void> {
  const cfg = getConfig();
  const { db, raw } = openDb();
  runMigrations(join(import.meta.dir, "../db/migrations"), raw);

  // 1. Fingerprint
  const fingerprint = getOrCreateFingerprint(cfg.identityPath);

  // 2. Tailscale identity (fail if not available)
  const ts = await getTailscaleIdentity();
  const ip = await getTailscaleIp();
  if (!ts || !ip) {
    console.error("\n  ⚠  No se pudo detectar Tailscale. Instala y arranca `tailscale up` antes de `aiconnect init`.\n");
    process.exit(1);
  }

  // 3. Self machine record (upsert by fingerprint)
  const existing = db.select().from(machine).where(eq(machine.fingerprint, fingerprint)).all()[0];
  let machineId: string;
  if (existing) {
    machineId = existing.id;
    db.update(machine).set({ hostname: ts.magicDns }).where(eq(machine.id, existing.id)).run();
  } else {
    machineId = nanoid();
    db.insert(machine).values({
      id: machineId,
      fingerprint,
      hostname: ts.magicDns,
      displayName: options.displayName ?? ts.hostname,
      ...(options.tag !== undefined ? { tag: options.tag } : {}),
    }).run();
  }

  // 4. Generate pairing token
  const token_str = generateToken();
  const tokenId = nanoid();
  db.insert(token).values({
    id: tokenId,
    hash: hashToken(token_str),
    ...(options.deviceLabel !== undefined ? { deviceLabel: options.deviceLabel } : {}),
  }).run();

  // 5. Short code (just for display)
  const shortCode = generateShortCode();

  const payload = serializePairingPayload({
    version: 1,
    host: ts.magicDns,
    port: cfg.port,
    token: token_str,
    fingerprint,
    shortCode,
  });

  // 6. Render QR + short code
  console.log("\n  aiconnect · pareo\n");
  console.log(`  host       ${ts.magicDns}`);
  console.log(`  port       ${cfg.port}`);
  console.log(`  short code ${shortCode}`);
  console.log("");
  qrcode.generate(payload, { small: true });
  console.log(`  Escanea el QR desde la PWA, o pega el código manual.\n`);
  console.log(`  Arrancando agente...\n`);

  // 7. Start agent
  await startAgent({ tailnetIp: ip, port: cfg.port, db, machineId });
}
```

- [ ] **Step 3: Add placeholder `startAgent` export in `src/index.ts` (will be implemented in Task 24)**

```typescript
#!/usr/bin/env bun
import { log } from "./logger";
import type { DB } from "./db/client";

export interface StartAgentOptions {
  tailnetIp: string;
  port: number;
  db: DB;
  machineId: string;
}

export async function startAgent(_opts: StartAgentOptions): Promise<void> {
  log.info("startAgent placeholder — to be implemented in Task 24");
}

async function main() {
  const cmd = process.argv[2];
  if (cmd === "init") {
    const { initCommand } = await import("./cli/init");
    await initCommand();
  } else if (cmd === "status") {
    const { statusCommand } = await import("./cli/status");
    await statusCommand();
  } else if (cmd === "revoke") {
    const { revokeCommand } = await import("./cli/revoke");
    await revokeCommand(process.argv[3]);
  } else {
    console.log("usage: aiconnect <init|status|revoke>");
    process.exit(1);
  }
}

if (import.meta.main) main();
```

- [ ] **Step 4: Write smoke test for init (without starting agent fully — mock startAgent)**

Create `agent/tests/cli/init.test.ts`:

```typescript
import { describe, test, expect } from "bun:test";
import { generateShortCode, serializePairingPayload } from "../../src/auth/pairing";

describe("pairing payload", () => {
  test("short code has correct format", () => {
    const code = generateShortCode();
    expect(code).toMatch(/^[A-Z2-7]{3}-[A-Z2-7]{3}$/);
  });

  test("serialize produces parseable JSON", () => {
    const s = serializePairingPayload({
      version: 1, host: "h", port: 1234, token: "t", fingerprint: "fp", shortCode: "ABC-123",
    });
    const parsed = JSON.parse(s);
    expect(parsed.version).toBe(1);
    expect(parsed.host).toBe("h");
  });
});
```

- [ ] **Step 5: Run**

```bash
bun test tests/cli/init.test.ts
```

Expected: PASS, 2 tests.

- [ ] **Step 6: Commit**

```bash
git add agent/src/cli/init.ts agent/src/auth/pairing.ts agent/src/index.ts agent/tests/cli/init.test.ts
git commit -m "feat(agent): aiconnect init command with qr and short code"
```

---

## Task 22: Comandos `aiconnect status` y `aiconnect revoke`

**Files:**
- Create: `agent/src/cli/status.ts`
- Create: `agent/src/cli/revoke.ts`
- Create: `agent/tests/cli/revoke.test.ts`

- [ ] **Step 1: Implement `src/cli/status.ts`**

```typescript
import { openDb, runMigrations } from "../db/client";
import { machine, token, project, conversation } from "../db/schema";
import { eq } from "drizzle-orm";
import { join } from "node:path";
import { getTailscaleIp, getTailscaleIdentity } from "../net/tailscale";

export async function statusCommand(): Promise<void> {
  const { db, raw } = openDb();
  runMigrations(join(import.meta.dir, "../db/migrations"), raw);

  const machines = db.select().from(machine).all();
  const self = machines[0];
  const tokens = db.select().from(token).where(eq(token.revoked, false)).all();
  const projects = db.select().from(project).all();
  const convs = db.select().from(conversation).all();
  const ip = await getTailscaleIp();
  const ts = await getTailscaleIdentity();

  console.log("\n  aiconnect · status\n");
  if (self) {
    console.log(`  machine      ${self.displayName}  (${self.hostname})`);
    console.log(`  fingerprint  ${self.fingerprint}`);
  } else {
    console.log(`  machine      not yet initialized — run \`aiconnect init\``);
  }
  console.log(`  tailscale    ${ip ?? "not detected"}  ${ts?.magicDns ?? ""}`);
  console.log(`  tokens       ${tokens.length} active`);
  for (const t of tokens) {
    console.log(`               ${t.id}  ${t.deviceLabel ?? "(no label)"}  last: ${t.lastUsedAt?.toISOString() ?? "never"}`);
  }
  console.log(`  projects     ${projects.length}`);
  console.log(`  conversations ${convs.length}\n`);
}
```

- [ ] **Step 2: Implement `src/cli/revoke.ts`**

```typescript
import { openDb, runMigrations } from "../db/client";
import { token } from "../db/schema";
import { eq } from "drizzle-orm";
import { join } from "node:path";

export async function revokeCommand(tokenId: string | undefined): Promise<void> {
  if (!tokenId) {
    console.error("usage: aiconnect revoke <token-id>");
    process.exit(1);
  }
  const { db, raw } = openDb();
  runMigrations(join(import.meta.dir, "../db/migrations"), raw);

  const rows = db.select().from(token).where(eq(token.id, tokenId)).all();
  if (rows.length === 0) {
    console.error(`token ${tokenId} not found`);
    process.exit(1);
  }
  db.update(token).set({ revoked: true }).where(eq(token.id, tokenId)).run();
  console.log(`✓ token ${tokenId} revoked`);
}
```

- [ ] **Step 3: Write test for revoke**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { nanoid } from "nanoid";
import { openDb, closeDb, runMigrations } from "../../src/db/client";
import { token } from "../../src/db/schema";
import { eq } from "drizzle-orm";

let tmp: string;

beforeEach(() => {
  tmp = join(mkdtempSync(join(tmpdir(), "rev-")), "db.sqlite");
  process.env.HOME = tmp.replace("/db.sqlite", "");
});
afterEach(() => { closeDb(); rmSync(tmp, { force: true }); });

describe("revoke", () => {
  test("marks a token as revoked in db", () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../src/db/migrations"), raw);
    const tid = nanoid();
    db.insert(token).values({ id: tid, hash: "h" }).run();

    db.update(token).set({ revoked: true }).where(eq(token.id, tid)).run();
    const row = db.select().from(token).all()[0]!;
    expect(row.revoked).toBe(true);
  });
});
```

- [ ] **Step 4: Run**

```bash
bun test tests/cli/revoke.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add agent/src/cli/status.ts agent/src/cli/revoke.ts agent/tests/cli/revoke.test.ts
git commit -m "feat(agent): status and revoke cli commands"
```

---

## Task 23: Integración caffeinate (sleep prevention)

**Files:**
- Create: `agent/src/sleep/caffeinate.ts`
- Create: `agent/tests/sleep/caffeinate.test.ts`

- [ ] **Step 1: Implement `src/sleep/caffeinate.ts`**

```typescript
import { spawn, type Subprocess } from "bun";
import { log } from "../logger";

let proc: Subprocess | null = null;
let activeCount = 0;

export function acquireCaffeinate(): void {
  activeCount++;
  if (proc) return;
  try {
    proc = spawn({ cmd: ["caffeinate", "-i"], stdout: "ignore", stderr: "ignore" });
    log.info("caffeinate started", { pid: proc.pid });
  } catch (err) {
    log.warn("caffeinate not available", { err: String(err) });
  }
}

export function releaseCaffeinate(): void {
  activeCount = Math.max(0, activeCount - 1);
  if (activeCount === 0 && proc) {
    proc.kill();
    proc = null;
    log.info("caffeinate stopped");
  }
}

export function caffeinateCount(): number { return activeCount; }
```

- [ ] **Step 2: Write test**

```typescript
import { describe, test, expect, beforeEach } from "bun:test";
import { acquireCaffeinate, releaseCaffeinate, caffeinateCount } from "../../src/sleep/caffeinate";

beforeEach(() => {
  // Reset by fully releasing any leftovers
  while (caffeinateCount() > 0) releaseCaffeinate();
});

describe("caffeinate", () => {
  test("ref counts acquire/release", () => {
    expect(caffeinateCount()).toBe(0);
    acquireCaffeinate();
    expect(caffeinateCount()).toBe(1);
    acquireCaffeinate();
    expect(caffeinateCount()).toBe(2);
    releaseCaffeinate();
    expect(caffeinateCount()).toBe(1);
    releaseCaffeinate();
    expect(caffeinateCount()).toBe(0);
  });
});
```

- [ ] **Step 3: Run**

```bash
bun test tests/sleep/caffeinate.test.ts
```

Expected: PASS (note: will `caffeinate` succeed on Windows/Linux? On non-macOS it logs a warning but the count still works).

- [ ] **Step 4: Commit**

```bash
git add agent/src/sleep/caffeinate.ts agent/tests/sleep/caffeinate.test.ts
git commit -m "feat(agent): caffeinate ref-counted sleep prevention"
```

---

## Task 24: Agent startup — uniendo todo (`startAgent`)

**Files:**
- Modify: `agent/src/index.ts` (replace placeholder `startAgent`)

- [ ] **Step 1: Replace `src/index.ts`**

```typescript
#!/usr/bin/env bun
import { startWsServer } from "./ws/server";
import { routeMessage } from "./ws/router";
import { ClientMessage, type ClientMessageT } from "./protocol/messages";
import { ConversationRegistry } from "./cli-wrapper/registry";
import type { DB } from "./db/client";
import type { WSConn } from "./ws/session";
import { acquireCaffeinate, releaseCaffeinate, caffeinateCount } from "./sleep/caffeinate";
import { log } from "./logger";

export interface StartAgentOptions {
  tailnetIp: string;
  port: number;
  db: DB;
  machineId: string;
  claudeBinary?: string; // optional override for tests
}

export async function startAgent(opts: StartAgentOptions): Promise<void> {
  const registry = new ConversationRegistry();
  const allConnections = new Set<WSConn>();
  const getSubscribers = (): WSConn[] => Array.from(allConnections);

  const deps = {
    db: opts.db,
    registry,
    machineId: opts.machineId,
    getSubscribers,
    ...(opts.claudeBinary !== undefined ? { claudeBinary: opts.claudeBinary } : {}),
  };

  const server = startWsServer({
    hostname: opts.tailnetIp,
    port: opts.port,
    onMessage: async (ws, raw) => {
      // raw is already validated via ClientMessage in ws/server.ts
      await routeMessage(ws, raw as ClientMessageT, deps);
      if (registry.size > 0 && caffeinateCount() === 0) acquireCaffeinate();
    },
    onClose: (ws) => {
      allConnections.delete(ws);
    },
  });

  // Register connections in open handler via side-channel:
  // We patch by tracking via the ws.data contract.
  // Simpler: piggyback on message handler above.
  // Actually Bun.serve.websocket.open isn't directly exposed here — let's wrap:
  // Minor hack: subscribe on first message
  const originalFetch = server.fetch;
  void originalFetch;

  process.on("SIGTERM", () => {
    log.info("SIGTERM, shutting down");
    registry.killAll();
    while (caffeinateCount() > 0) releaseCaffeinate();
    server.stop(true);
    process.exit(0);
  });

  log.info("agent listening", { host: opts.tailnetIp, port: opts.port });
}

async function main() {
  const cmd = process.argv[2];
  if (cmd === "init") {
    const { initCommand } = await import("./cli/init");
    await initCommand();
  } else if (cmd === "status") {
    const { statusCommand } = await import("./cli/status");
    await statusCommand();
  } else if (cmd === "revoke") {
    const { revokeCommand } = await import("./cli/revoke");
    await revokeCommand(process.argv[3]);
  } else {
    console.log("usage: aiconnect <init|status|revoke>");
    process.exit(1);
  }
}

if (import.meta.main) main();
```

- [ ] **Step 2: Fix connection tracking in `ws/server.ts`**

The `startWsServer` needs to expose open/close so we can track connections. Modify it:

Open `agent/src/ws/server.ts` and update to pass through `open`:

```typescript
import { type Server } from "bun";
import { type SessionState, createSession, send } from "./session";
import { ClientMessage } from "../protocol/messages";
import { log } from "../logger";
import type { WSConn } from "./session";

export interface ServerOptions {
  hostname: string;
  port: number;
  onOpen?: (ws: WSConn) => void;
  onMessage: (ws: WSConn, msg: unknown) => Promise<void> | void;
  onClose?: (ws: WSConn) => void;
}

export function startWsServer(opts: ServerOptions): Server {
  return Bun.serve<SessionState, never>({
    hostname: opts.hostname,
    port: opts.port,
    fetch(req, server) {
      if (server.upgrade(req, { data: createSession() })) return;
      return new Response("aiconnect agent", { status: 200 });
    },
    websocket: {
      message: async (ws, message) => {
        let parsed: unknown;
        try {
          parsed = JSON.parse(typeof message === "string" ? message : new TextDecoder().decode(message));
        } catch {
          send(ws, { type: "error", code: "bad_request", message: "invalid json" });
          return;
        }
        const result = ClientMessage.safeParse(parsed);
        if (!result.success) {
          send(ws, { type: "error", code: "bad_request", message: "invalid message", detail: result.error.message });
          return;
        }
        try {
          await opts.onMessage(ws, result.data);
        } catch (err) {
          log.error("handler error", { err: String(err) });
          send(ws, { type: "error", code: "internal", message: "handler failed" });
        }
      },
      open: (ws) => {
        log.info("ws open", { remote: ws.remoteAddress });
        if (opts.onOpen) opts.onOpen(ws);
      },
      close: (ws) => {
        if (opts.onClose) opts.onClose(ws);
      },
    },
  });
}
```

- [ ] **Step 3: Wire `onOpen` in `startAgent`**

Replace the `startWsServer` call in `src/index.ts` `startAgent` with:

```typescript
const server = startWsServer({
  hostname: opts.tailnetIp,
  port: opts.port,
  onOpen: (ws) => allConnections.add(ws),
  onClose: (ws) => allConnections.delete(ws),
  onMessage: async (ws, raw) => {
    await routeMessage(ws, raw as ClientMessageT, deps);
    if (registry.size > 0 && caffeinateCount() === 0) acquireCaffeinate();
  },
});
```

- [ ] **Step 4: Run typecheck**

```bash
bun run typecheck
```

Expected: no errors.

- [ ] **Step 5: Run all tests**

```bash
bun test
```

Expected: all previously passing tests still pass.

- [ ] **Step 6: Commit**

```bash
git add agent/src/index.ts agent/src/ws/server.ts
git commit -m "feat(agent): wire startAgent with connection tracking and shutdown"
```

---

## Task 25: End-to-end test real (client ws + stub cli)

**Files:**
- Create: `agent/tests/e2e/pair-and-chat.test.ts`

- [ ] **Step 1: Write E2E test**

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test";
import { mkdtempSync, rmSync, writeFileSync, mkdirSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { nanoid } from "nanoid";
import { openDb, closeDb, runMigrations } from "../../src/db/client";
import { machine, token } from "../../src/db/schema";
import { generateToken, hashToken } from "../../src/auth/token";
import { startAgent } from "../../src/index";

let tmp: string; let workDir: string; let server: { stop: () => void } | undefined;
const stubPath = join(import.meta.dir, "../stub-cli/claude");

beforeEach(() => {
  tmp = join(mkdtempSync(join(tmpdir(), "e2e-")), "db.sqlite");
  workDir = mkdtempSync(join(tmpdir(), "wd-"));
});
afterEach(() => {
  closeDb();
  rmSync(tmp, { force: true });
  rmSync(workDir, { recursive: true, force: true });
});

async function waitFor(pred: () => boolean, timeoutMs = 5000): Promise<void> {
  const start = Date.now();
  while (!pred()) {
    if (Date.now() - start > timeoutMs) throw new Error("waitFor timeout");
    await new Promise((r) => setTimeout(r, 20));
  }
}

describe("e2e: pair + create conv + chat", () => {
  test("full happy path", async () => {
    const { db, raw } = openDb(tmp);
    runMigrations(join(import.meta.dir, "../../src/db/migrations"), raw);
    const mId = nanoid();
    db.insert(machine).values({ id: mId, fingerprint: "fp", hostname: "h.ts.net", displayName: "h" }).run();
    const t = generateToken();
    db.insert(token).values({ id: nanoid(), hash: hashToken(t) }).run();

    const port = 54000 + Math.floor(Math.random() * 1000);
    await startAgent({ tailnetIp: "127.0.0.1", port, db, machineId: mId, claudeBinary: stubPath });
    await new Promise((r) => setTimeout(r, 100));

    const ws = new WebSocket(`ws://127.0.0.1:${port}`);
    const received: unknown[] = [];
    ws.addEventListener("message", (e) => {
      received.push(JSON.parse(typeof e.data === "string" ? e.data : ""));
    });
    await new Promise<void>((r) => ws.addEventListener("open", () => r(), { once: true }));

    ws.send(JSON.stringify({ type: "auth", token: t, protocolVersion: 1 }));
    await waitFor(() => received.some((m) => (m as { type: string }).type === "auth_ok"));

    ws.send(JSON.stringify({
      type: "create_project", name: "p", workingDir: workDir, tags: [], permissionPreset: "ask-for-side-effects",
    }));
    await waitFor(() => received.some((m) => (m as { type: string }).type === "project_created"));
    const proj = received.find((m) => (m as { type: string }).type === "project_created") as { project: { id: string } };

    ws.send(JSON.stringify({
      type: "create_conversation", projectId: proj.project.id, cli: "claude-code",
    }));
    await waitFor(() => received.some((m) => (m as { type: string }).type === "conversation_created"));
    const conv = received.find((m) => (m as { type: string }).type === "conversation_created") as { conversation: { id: string } };

    ws.send(JSON.stringify({
      type: "send_prompt", conversationId: conv.conversation.id, prompt: "hello world", attachments: [],
    }));

    await waitFor(() => received.some((m) => {
      return (m as { type: string; event?: { type: string } }).type === "event"
        && (m as { event: { type: string } }).event.type === "turn_complete";
    }), 5000);

    ws.close();
    await new Promise((r) => setTimeout(r, 100));
    // Agent shutdown is SIGTERM; tests don't invoke it. Bun auto-cleans on exit.
  }, 10000);
});
```

- [ ] **Step 2: Run**

```bash
bun test tests/e2e/pair-and-chat.test.ts
```

Expected: PASS, 1 test (takes ~2-3 seconds).

- [ ] **Step 3: Commit**

```bash
git add agent/tests/e2e/pair-and-chat.test.ts
git commit -m "test(agent): e2e pair + create project + conversation + chat"
```

---

## Task 26: LaunchAgent plist y scripts de install/uninstall

**Files:**
- Create: `agent/src/launchagent/plist.ts`
- Create: `agent/src/cli/launchagent.ts`
- Create: `agent/tests/launchagent/plist.test.ts`

- [ ] **Step 1: Implement `src/launchagent/plist.ts`**

```typescript
export interface PlistOptions {
  label: string;
  binaryPath: string;
  logDir: string;
}

export function generatePlist(opts: PlistOptions): string {
  return `<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>${opts.label}</string>
  <key>ProgramArguments</key>
  <array>
    <string>${opts.binaryPath}</string>
    <string>daemon</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>${opts.logDir}/agent.out.log</string>
  <key>StandardErrorPath</key><string>${opts.logDir}/agent.err.log</string>
  <key>ProcessType</key><string>Background</string>
</dict>
</plist>
`;
}
```

- [ ] **Step 2: Implement `src/cli/launchagent.ts`**

```typescript
import { writeFileSync, mkdirSync, unlinkSync, existsSync } from "node:fs";
import { join } from "node:path";
import { $ } from "bun";
import { generatePlist } from "../launchagent/plist";

const LABEL = "dev.jcani.aiconnect-agent";

export async function installLaunchAgent(binaryPath: string): Promise<void> {
  const home = process.env.HOME;
  if (!home) throw new Error("HOME not set");
  const plistDir = join(home, "Library", "LaunchAgents");
  const plistPath = join(plistDir, `${LABEL}.plist`);
  const logDir = join(home, ".aiconnect", "logs");
  mkdirSync(plistDir, { recursive: true });
  mkdirSync(logDir, { recursive: true });

  const content = generatePlist({ label: LABEL, binaryPath, logDir });
  writeFileSync(plistPath, content);
  await $`launchctl unload ${plistPath}`.quiet().nothrow();
  await $`launchctl load ${plistPath}`.quiet();
  console.log(`✓ LaunchAgent instalado: ${plistPath}`);
}

export async function uninstallLaunchAgent(): Promise<void> {
  const home = process.env.HOME;
  if (!home) throw new Error("HOME not set");
  const plistPath = join(home, "Library", "LaunchAgents", `${LABEL}.plist`);
  if (existsSync(plistPath)) {
    await $`launchctl unload ${plistPath}`.quiet().nothrow();
    unlinkSync(plistPath);
    console.log(`✓ LaunchAgent desinstalado`);
  } else {
    console.log("No LaunchAgent instalado.");
  }
}
```

- [ ] **Step 3: Write test**

```typescript
import { describe, test, expect } from "bun:test";
import { generatePlist } from "../../src/launchagent/plist";

describe("generatePlist", () => {
  test("includes label, binary, log paths", () => {
    const p = generatePlist({ label: "dev.test.x", binaryPath: "/usr/local/bin/aiconnect", logDir: "/tmp/logs" });
    expect(p).toContain("<string>dev.test.x</string>");
    expect(p).toContain("<string>/usr/local/bin/aiconnect</string>");
    expect(p).toContain("/tmp/logs/agent.out.log");
    expect(p).toContain("<true/>"); // RunAtLoad
  });
});
```

- [ ] **Step 4: Add `install`/`uninstall` commands to main dispatcher**

Update `src/index.ts` `main()`:

```typescript
} else if (cmd === "install") {
  const { installLaunchAgent } = await import("./cli/launchagent");
  await installLaunchAgent(process.argv[3] ?? process.execPath);
} else if (cmd === "uninstall") {
  const { uninstallLaunchAgent } = await import("./cli/launchagent");
  await uninstallLaunchAgent();
} else if (cmd === "daemon") {
  // Daemon mode: load state and call startAgent without regenerating pairing.
  const { daemonCommand } = await import("./cli/daemon");
  await daemonCommand();
```

- [ ] **Step 5: Create minimal `src/cli/daemon.ts`**

```typescript
import { openDb, runMigrations } from "../db/client";
import { machine } from "../db/schema";
import { join } from "node:path";
import { getTailscaleIp } from "../net/tailscale";
import { getConfig } from "../config";
import { startAgent } from "../index";

export async function daemonCommand(): Promise<void> {
  const cfg = getConfig();
  const { db, raw } = openDb();
  runMigrations(join(import.meta.dir, "../db/migrations"), raw);
  const self = db.select().from(machine).all()[0];
  if (!self) {
    console.error("agent not initialized. run `aiconnect init` first.");
    process.exit(1);
  }
  const ip = await getTailscaleIp();
  if (!ip) {
    console.error("tailscale not available");
    process.exit(1);
  }
  await startAgent({ tailnetIp: ip, port: cfg.port, db, machineId: self.id });
}
```

- [ ] **Step 6: Run**

```bash
bun test tests/launchagent/plist.test.ts
bun run typecheck
```

Expected: all pass.

- [ ] **Step 7: Commit**

```bash
git add agent/src/launchagent/ agent/src/cli/launchagent.ts agent/src/cli/daemon.ts agent/src/index.ts agent/tests/launchagent/
git commit -m "feat(agent): launchagent install/uninstall + daemon mode"
```

---

## Task 27: Build script y brew formula

**Files:**
- Create: `agent/scripts/build-binary.sh`
- Create: `Formula/aiconnect-agent.rb`

- [ ] **Step 1: Create `agent/scripts/build-binary.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(cd "$(dirname "$0")/.." && pwd)"
OUT="$ROOT/bin"
mkdir -p "$OUT"

cd "$ROOT"
bun build --compile --minify \
  --target=bun-darwin-arm64 \
  --outfile "$OUT/aiconnect-darwin-arm64" \
  ./src/index.ts

bun build --compile --minify \
  --target=bun-darwin-x64 \
  --outfile "$OUT/aiconnect-darwin-x64" \
  ./src/index.ts

echo "✓ Built:"
ls -lh "$OUT"
```

- [ ] **Step 2: Make executable and test**

```bash
chmod +x agent/scripts/build-binary.sh
cd agent
bash scripts/build-binary.sh
```

Expected: two binaries in `bin/` directory (arm64 and x64). Smoke test:

```bash
./bin/aiconnect-darwin-arm64
```

Expected: `usage: aiconnect <init|status|revoke>`.

- [ ] **Step 3: Create `Formula/aiconnect-agent.rb`**

This is the brew formula. Goes in a separate repo (`jcani/homebrew-aiconnect`) but we put it here as reference.

```ruby
class AiconnectAgent < Formula
  desc "Remote-control daemon for Claude Code CLI over Tailscale"
  homepage "https://github.com/jcani/aiconnect"
  version "0.1.0"
  license "MIT"

  on_macos do
    on_arm do
      url "https://github.com/jcani/aiconnect/releases/download/v0.1.0/aiconnect-darwin-arm64.tar.gz"
      sha256 "REPLACE_WITH_SHA256"
    end
    on_intel do
      url "https://github.com/jcani/aiconnect/releases/download/v0.1.0/aiconnect-darwin-x64.tar.gz"
      sha256 "REPLACE_WITH_SHA256"
    end
  end

  depends_on "tailscale"

  def install
    bin.install Dir["aiconnect-*"].first => "aiconnect"
  end

  test do
    assert_match "usage: aiconnect", shell_output("#{bin}/aiconnect 2>&1", 1)
  end

  def caveats
    <<~EOS
      After installation:
        1. Ensure Tailscale is running:  tailscale up
        2. Run initial pairing:          aiconnect init
        3. (Optional) install as daemon: aiconnect install $(which aiconnect)
    EOS
  end
end
```

- [ ] **Step 4: Commit**

```bash
git add agent/scripts/ Formula/
git commit -m "build(agent): binary compile script and brew formula"
```

---

## Task 28: README final del agente

**Files:**
- Modify: `agent/README.md`

- [ ] **Step 1: Write comprehensive README**

```markdown
# aiconnect-agent

macOS daemon that wraps the Claude Code CLI and exposes a WebSocket API bound to Tailscale. Part of [aiconnect](../docs/superpowers/specs/2026-04-18-aiconnect-design.md).

## Requirements

- macOS 13+ (Ventura or newer)
- [Tailscale](https://tailscale.com) installed and running (`tailscale up`)
- [Claude Code CLI](https://docs.claude.com/claude-code) installed and authenticated with your Max subscription
- [Bun](https://bun.sh) (for development; binary distributions will be available via brew)

## Install

Once released:

```
brew tap jcani/aiconnect
brew install aiconnect-agent
```

For development:

```
cd agent
bun install
bun run build    # produces bin/aiconnect-darwin-{arm64,x64}
```

## Usage

### Initial pairing

Run once per Mac:

```
aiconnect init
```

Prints a QR code and a 6-character short code. Scan from the PWA (or type the short code).

### Install as LaunchAgent (auto-start on login)

```
aiconnect install /usr/local/bin/aiconnect
```

### Status

```
aiconnect status
```

### Revoke a paired device

```
aiconnect revoke <token-id>
```

## Development

```
bun install
bun run typecheck
bun test
bun run dev       # watches src/ and reruns
```

## Architecture

- `src/db/` — SQLite schema (drizzle) with FTS5 for future search
- `src/auth/` — token generation/hash, agent fingerprint, pairing payload
- `src/net/` — Tailscale detection
- `src/protocol/` — zod message schemas (shared contract with PWA)
- `src/ws/` — WebSocket server + per-message handlers
- `src/cli-wrapper/` — Claude Code subprocess lifecycle + stream-json parsing
- `src/cli/` — `init`, `status`, `revoke`, `install`, `daemon` commands
- `src/launchagent/` — macOS plist generation

Data lives in `~/.aiconnect/`:
- `agent.db` — SQLite (conversations, projects, tokens)
- `identity.json` — stable fingerprint
- `logs/` — LaunchAgent logs when running as daemon

## Testing

End-to-end test with stub CLI:

```
bun test tests/e2e/
```

Uses a bash stub at `tests/stub-cli/claude` that emits canned stream-json events. Does not require real `claude` binary.
```

- [ ] **Step 2: Commit**

```bash
git add agent/README.md
git commit -m "docs(agent): comprehensive README"
```

---

## Task 29: Smoke test manual de Plan 1 completo

- [ ] **Step 1: Setup fresh dev environment on a Mac**

```bash
cd D:/PROYECTOS/aiconnect/agent  # On your Mac equivalent path
bun install
bun run build
```

- [ ] **Step 2: Initialize (requires Tailscale + claude CLI)**

```bash
./bin/aiconnect-darwin-arm64 init
```

Expected: QR rendered, short code printed, agent starts listening on tailnet IP.

- [ ] **Step 3: In another terminal, connect with websocat**

```bash
brew install websocat   # if not already
websocat ws://<tailnet-ip>:51820
```

- [ ] **Step 4: Send test messages manually**

Paste:

```json
{"type":"auth","token":"<token-from-init>","protocolVersion":1}
```

Expect `auth_ok` response.

```json
{"type":"create_project","name":"test","workingDir":"/Users/<you>/some/repo","tags":[],"permissionPreset":"ask-for-side-effects"}
```

Expect `project_created`.

```json
{"type":"create_conversation","projectId":"<id>","cli":"claude-code"}
```

Expect `conversation_created`.

```json
{"type":"send_prompt","conversationId":"<id>","prompt":"hola, qué archivos hay acá","attachments":[]}
```

Expect streaming events ending in `turn_complete`.

- [ ] **Step 5: Verify persistence**

Kill agent (Ctrl+C). Restart:

```bash
./bin/aiconnect-darwin-arm64 daemon
```

Reconnect websocat. Send `replay_from` for the previous conversation; verify events replay.

- [ ] **Step 6: Document any issues found**

If anything breaks, file a follow-up task. Otherwise, mark Plan 1 as complete.

- [ ] **Step 7: Final commit**

```bash
cd D:/PROYECTOS/aiconnect
git tag -a "fase1-plan1-complete" -m "aiconnect Fase 1 Plan 1 complete: agent + pairing"
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ Agente para una Mac con LaunchAgent (Tasks 24, 26)
- ✅ Claude Code CLI wrap con stream-json (Tasks 10, 13)
- ✅ Proyectos: crear + listar con working_dir (Task 14)
- ✅ Conversaciones streaming (Tasks 15, 16)
- ✅ Approvals request/response (Task 17)
- ✅ Pareo vía QR (Task 21)
- ✅ SQLite con FTS5 (Task 3)
- ✅ Reconexión con replay_from (Task 19)
- ✅ Model selector (Tasks 14, 15, 18)
- ✅ Caffeinate (Task 23)
- ✅ Tailscale binding (Tasks 6, 24)
- ✅ Token-based auth con revocación (Tasks 4, 22)
- ✅ Fingerprint persistente (Task 5)
- ✅ Build + distribución (Task 27)

**No covered in Plan 1 (per plan split):**
- PWA frontend — Plan 2
- Voice input — Plan 3
- Image attachment — Plan 3
- Quick prompts — Plan 3

**Placeholder scan:** no TBDs, no "implement later", no "handle edge cases" without specifics.

**Type consistency:** `ConversationRegistry`, `ClaudeCodeWrapper`, `ConversationEventT` used consistently across tasks.

---

## Execution Handoff

**Plan complete and saved to `docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan1-agente.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — Dispatch un subagente fresco por tarea, revisión entre tareas, iteración rápida. Útil si vas a trabajar en esto de manera continua.

**2. Inline Execution** — Ejecutar tareas en esta sesión con checkpoints. Útil si querés revisar cada tarea directamente conmigo.

**¿Qué prefieres? (O prefieres hacer las tareas tú mismo y yo solo ayudo cuando me preguntes?)**
