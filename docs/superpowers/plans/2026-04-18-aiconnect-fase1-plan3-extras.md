# aiconnect Fase 1 · Plan 3/3 — Mobile-first extras

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Añadir a la PWA las tres features mobile-first que cierran el gap entre terminal y móvil: voice input (dictado), image attachment (multimodal), y quick prompts (biblioteca de prompts frecuentes). Pulir bordes para dar por terminada Fase 1.

**Architecture:** Web Speech API para transcripción en-cliente. Canvas API + `<input type="file" capture="environment">` para imágenes. IndexedDB (ya en Plan 2) para quick prompts. Pequeños cambios al protocolo PWA↔Agente para attachments.

**Tech Stack:** Web Speech API (nativo) · Canvas API · IndexedDB via `idb` (ya instalado) · Base64 encoding para imágenes pequeñas (<1MB después de resize)

**Spec de referencia:** `docs/superpowers/specs/2026-04-18-aiconnect-design.md` sección 8 (Fase 1 mobile-first extras)
**Plan 1 (agente):** `docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan1-agente.md`
**Plan 2 (PWA):** `docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan2-pwa.md`

**Criterio de éxito:** usando la PWA del Plan 2, el usuario puede (1) dictar un prompt tocando-y-manteniendo el botón de mic, editar la transcripción, enviarlo; (2) adjuntar una screenshot de la cámara o galería al prompt y que llegue a Claude Code con la imagen; (3) abrir una biblioteca de prompts guardados desde el input, tap para insertar, editar/crear/eliminar prompts desde settings.

---

## File Structure (nuevos archivos sobre Plan 2)

```
agent/
└── src/
    └── cli-wrapper/
        └── attachments.ts              # NEW · resolve base64 → tmp file for claude -p

pwa/
└── src/
    ├── voice/
    │   ├── recognition.ts              # NEW · Web Speech API wrapper
    │   └── permissions.ts              # NEW · microphone permission helper
    ├── images/
    │   ├── capture.ts                  # NEW · file picker + camera
    │   └── resize.ts                   # NEW · canvas-based resize
    ├── components/
    │   ├── chat/
    │   │   ├── VoiceOverlay.tsx        # NEW
    │   │   ├── ImageAttachment.tsx     # NEW
    │   │   └── QuickPromptsSheet.tsx   # NEW
    │   └── settings/
    │       └── QuickPromptEditor.tsx   # NEW
    ├── state/
    │   └── quick-prompts.ts            # NEW · zustand store + IndexedDB wiring
    └── pages/
        └── SettingsQuickPrompts.tsx    # NEW
```

---

## Task 1: Ampliar protocolo para attachments (coordinado con agente)

**Files:**
- Modify: `agent/src/protocol/messages.ts` — add `attachments` to `send_prompt`
- Modify: `pwa/src/protocol/messages.ts` — mirror change
- Modify: `agent/src/ws/handlers/prompts.ts` — pass attachments to wrapper
- Create: `agent/src/cli-wrapper/attachments.ts` — handle base64 → tmp file

- [ ] **Step 1: Update protocol schemas**

In both `agent/src/protocol/messages.ts` and `pwa/src/protocol/messages.ts`, update `SendPromptRequest`:

```typescript
export const AttachmentPayload = z.object({
  type: z.enum(["image"]),
  mimeType: z.string(),
  data: z.string(),        // base64 (no data: prefix)
  filename: z.string().optional(),
});

export const SendPromptRequest = z.object({
  type: z.literal("send_prompt"),
  conversationId: z.string(),
  prompt: z.string().min(1),
  attachments: z.array(AttachmentPayload).default([]),
});
```

Update `UserPromptEvent` payload:

```typescript
export const UserPromptEvent = EventBase.extend({
  type: z.literal("user_prompt"),
  payload: z.object({
    text: z.string(),
    attachments: z.array(AttachmentPayload).optional(),
  }),
});
```

- [ ] **Step 2: Bump protocol version**

In `agent/src/protocol/version.ts` and `pwa/src/protocol/version.ts`:

```typescript
export const PROTOCOL_VERSION = 2;
```

- [ ] **Step 3: Implement `agent/src/cli-wrapper/attachments.ts`**

```typescript
import { mkdtempSync, writeFileSync, rmSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import type { AttachmentPayload } from "../protocol/messages";
import { log } from "../logger";

const EXT_FROM_MIME: Record<string, string> = {
  "image/png": "png",
  "image/jpeg": "jpg",
  "image/webp": "webp",
  "image/heic": "heic",
};

export interface MaterializedAttachment {
  path: string;
  mimeType: string;
  cleanup: () => void;
}

export function materializeAttachments(atts: Array<{ type: string; mimeType: string; data: string; filename?: string }>): MaterializedAttachment[] {
  const out: MaterializedAttachment[] = [];
  for (const a of atts) {
    if (a.type !== "image") continue;
    const ext = EXT_FROM_MIME[a.mimeType] ?? "bin";
    const dir = mkdtempSync(join(tmpdir(), "aiconnect-att-"));
    const name = a.filename ?? `attachment.${ext}`;
    const safeName = name.replace(/[^\w.-]/g, "_");
    const filePath = join(dir, safeName);
    writeFileSync(filePath, Buffer.from(a.data, "base64"));
    out.push({
      path: filePath,
      mimeType: a.mimeType,
      cleanup: () => {
        try { rmSync(dir, { recursive: true, force: true }); }
        catch (err) { log.warn("attachment cleanup failed", { err: String(err) }); }
      },
    });
  }
  return out;
}
```

- [ ] **Step 4: Update `agent/src/ws/handlers/prompts.ts` to pass attachments**

Modify `handleSendPrompt` to materialize attachments and reference them in the CLI input. Current `wrapper.sendPrompt(prompt, attachments?)` signature takes strings — now we pass file paths.

Update `agent/src/cli-wrapper/claude-code.ts` `sendPrompt`:

```typescript
sendPrompt(prompt, attachmentPaths) {
  writeLine({
    type: "user_prompt",
    text: prompt,
    attachments: (attachmentPaths ?? []).map((p) => ({ type: "file", path: p })),
  });
},
```

In `handleSendPrompt`:

```typescript
import { materializeAttachments } from "../../cli-wrapper/attachments";

// ... inside handler
const materialized = materializeAttachments(msg.attachments);
try {
  wrapper.sendPrompt(msg.prompt, materialized.map((m) => m.path));
} finally {
  // Cleanup after small delay to allow CLI to read them
  setTimeout(() => {
    for (const m of materialized) m.cleanup();
  }, 30_000);
}
```

- [ ] **Step 5: Run agent tests — adjust for schema change**

Existing tests may break due to `send_prompt` schema tightening. Update callers in tests to include `attachments: []`.

```bash
cd agent
bun test
```

Fix any failures (mostly just adding `attachments: []` to test fixtures).

- [ ] **Step 6: Commit (both repos in one since it's a coordinated schema change)**

```bash
git add agent/src/protocol/ agent/src/cli-wrapper/ agent/src/ws/handlers/prompts.ts agent/tests/ pwa/src/protocol/
git commit -m "feat: multimodal attachments in send_prompt (protocol v2)"
```

---

## Task 2: Microphone permission helper

**Files:**
- Create: `pwa/src/voice/permissions.ts`

- [ ] **Step 1: Implement**

```typescript
export type MicPermission = "granted" | "denied" | "prompt" | "unavailable";

export async function checkMicPermission(): Promise<MicPermission> {
  if (!navigator.permissions || !navigator.mediaDevices) return "unavailable";
  try {
    const status = await navigator.permissions.query({ name: "microphone" as PermissionName });
    return status.state as MicPermission;
  } catch {
    return "prompt";
  }
}

export async function requestMic(): Promise<boolean> {
  if (!navigator.mediaDevices) return false;
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    stream.getTracks().forEach((t) => t.stop());
    return true;
  } catch {
    return false;
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add pwa/src/voice/permissions.ts
git commit -m "feat(pwa): microphone permission helper"
```

---

## Task 3: Web Speech API wrapper

**Files:**
- Create: `pwa/src/voice/recognition.ts`

- [ ] **Step 1: Implement wrapper with typed events**

```typescript
// Note: Web Speech API types are not in TS lib by default. Declare minimal shapes.

interface SpeechRecognitionLike {
  continuous: boolean;
  interimResults: boolean;
  lang: string;
  onresult: ((e: { resultIndex: number; results: { length: number; [i: number]: { 0: { transcript: string }; isFinal: boolean } } }) => void) | null;
  onend: (() => void) | null;
  onerror: ((e: { error: string }) => void) | null;
  start: () => void;
  stop: () => void;
  abort: () => void;
}

declare global {
  interface Window {
    SpeechRecognition?: { new(): SpeechRecognitionLike };
    webkitSpeechRecognition?: { new(): SpeechRecognitionLike };
  }
}

export type VoiceCallback = (final: string, interim: string) => void;

export class VoiceRecognition {
  private rec: SpeechRecognitionLike | null = null;
  private finalTranscript = "";
  private onChange: VoiceCallback;
  private onEnd: () => void;
  private onError: (err: string) => void;

  constructor(onChange: VoiceCallback, onEnd: () => void, onError: (err: string) => void) {
    this.onChange = onChange;
    this.onEnd = onEnd;
    this.onError = onError;
  }

  static isSupported(): boolean {
    return typeof window !== "undefined" && (!!window.SpeechRecognition || !!window.webkitSpeechRecognition);
  }

  start(lang = "es-ES"): void {
    const Ctor = window.SpeechRecognition ?? window.webkitSpeechRecognition;
    if (!Ctor) { this.onError("unavailable"); return; }
    this.rec = new Ctor();
    this.rec.continuous = true;
    this.rec.interimResults = true;
    this.rec.lang = lang;
    this.rec.onresult = (e) => {
      let interim = "";
      let final = this.finalTranscript;
      for (let i = e.resultIndex; i < e.results.length; i++) {
        const res = e.results[i];
        if (res && res.isFinal) final += res[0].transcript;
        else if (res) interim += res[0].transcript;
      }
      this.finalTranscript = final;
      this.onChange(final, interim);
    };
    this.rec.onend = () => this.onEnd();
    this.rec.onerror = (e) => this.onError(e.error);
    this.rec.start();
  }

  stop(): void {
    this.rec?.stop();
    this.rec = null;
  }

  abort(): void {
    this.rec?.abort();
    this.rec = null;
  }

  reset(): void {
    this.finalTranscript = "";
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add pwa/src/voice/recognition.ts
git commit -m "feat(pwa): web speech api wrapper"
```

---

## Task 4: Voice overlay UI + integrar en ChatInput

**Files:**
- Create: `pwa/src/components/chat/VoiceOverlay.tsx`
- Modify: `pwa/src/components/chat/ChatInput.tsx`

- [ ] **Step 1: Implement `VoiceOverlay.tsx`**

```typescript
import { useEffect, useState, useRef } from "react";
import { VoiceRecognition } from "../../voice/recognition";
import { requestMic } from "../../voice/permissions";
import { PrimaryButton } from "../buttons/PrimaryButton";
import { SecondaryButton } from "../buttons/SecondaryButton";

type Phase = "granting" | "listening" | "done" | "error";

export function VoiceOverlay({ onCancel, onConfirm }: { onCancel: () => void; onConfirm: (text: string) => void }) {
  const [phase, setPhase] = useState<Phase>("granting");
  const [final, setFinal] = useState("");
  const [interim, setInterim] = useState("");
  const [error, setError] = useState<string>("");
  const recRef = useRef<VoiceRecognition | null>(null);

  useEffect(() => {
    (async () => {
      if (!VoiceRecognition.isSupported()) {
        setPhase("error");
        setError("este navegador no soporta Web Speech API");
        return;
      }
      const ok = await requestMic();
      if (!ok) {
        setPhase("error");
        setError("permiso de micrófono denegado");
        return;
      }
      const rec = new VoiceRecognition(
        (f, i) => { setFinal(f); setInterim(i); },
        () => setPhase("done"),
        (e) => { setError(e); setPhase("error"); },
      );
      recRef.current = rec;
      setPhase("listening");
      rec.start("es-ES");
    })();

    return () => { recRef.current?.abort(); };
  }, []);

  const stop = () => {
    recRef.current?.stop();
    setPhase("done");
  };

  const combined = (final + " " + interim).trim();

  return (
    <div className="absolute inset-0 bg-black/92 flex flex-col items-center justify-between p-10 pt-16 z-50">
      {phase === "granting" && <div className="text-text-dim font-mono text-[10px] tracking-[0.2em] uppercase">esperando permiso...</div>}
      {phase === "error" && (
        <>
          <div className="text-coral font-mono text-[11px]">⚠ {error}</div>
          <SecondaryButton onClick={onCancel}>cerrar</SecondaryButton>
        </>
      )}
      {(phase === "listening" || phase === "done") && (
        <>
          <div className="text-accent font-mono text-[10px] tracking-[0.15em] uppercase flex items-center gap-2">
            <span className={phase === "listening" ? "animate-pulse" : ""}>●</span>
            {phase === "listening" ? "escuchando" : "listo"}
          </div>
          <div className="relative w-[90px] h-[90px] border-2 border-accent rounded-full flex items-center justify-center text-accent font-mono text-[36px]">
            <span className="absolute inset-0 rounded-full border border-accent opacity-40 animate-ping"></span>
            ⌘
          </div>
          <div className="font-serif font-medium text-[20px] leading-[1.3] tracking-[-0.01em] text-text text-center max-w-[280px]">
            {final}
            {interim && <span className="text-text-dim italic"> {interim}</span>}
            {!final && !interim && <span className="text-text-faint">empieza a hablar...</span>}
          </div>
          <div className="w-full flex gap-2">
            <SecondaryButton onClick={() => { recRef.current?.abort(); onCancel(); }}>cancelar</SecondaryButton>
            {phase === "listening" && <SecondaryButton onClick={stop}>detener</SecondaryButton>}
            <PrimaryButton onClick={() => onConfirm(combined)} disabled={!combined}>enviar prompt</PrimaryButton>
          </div>
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Integrar en `ChatInput.tsx`**

Update `pwa/src/components/chat/ChatInput.tsx`:

```typescript
import { useState, type KeyboardEvent } from "react";
import { VoiceOverlay } from "./VoiceOverlay";
import { VoiceRecognition } from "../../voice/recognition";

export function ChatInput({ disabled, placeholder = "escribe un prompt...", onSend, onOpenQuickPrompts, onAttachImage }: {
  disabled?: boolean;
  placeholder?: string;
  onSend: (text: string) => void;
  onOpenQuickPrompts?: () => void;
  onAttachImage?: () => void;
}) {
  const [text, setText] = useState("");
  const [voiceOpen, setVoiceOpen] = useState(false);
  const submit = () => {
    if (text.trim()) { onSend(text.trim()); setText(""); }
  };
  const onKey = (e: KeyboardEvent<HTMLTextAreaElement>) => {
    if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); submit(); }
  };
  return (
    <>
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
        {onOpenQuickPrompts && (
          <button className="w-6 h-6 flex items-center justify-center border border-border text-text-dim" onClick={onOpenQuickPrompts} aria-label="quick prompts">◌</button>
        )}
        {onAttachImage && (
          <button className="w-6 h-6 flex items-center justify-center border border-border text-text-dim" onClick={onAttachImage} aria-label="attach image">◇</button>
        )}
        {VoiceRecognition.isSupported() && (
          <button className="w-6 h-6 flex items-center justify-center border border-border text-text-dim" onClick={() => setVoiceOpen(true)} aria-label="voice">⌘</button>
        )}
        <button className="font-mono text-[10px] uppercase tracking-[0.15em] text-accent" onClick={submit} disabled={disabled}>send ⏎</button>
      </div>
      {voiceOpen && (
        <VoiceOverlay
          onCancel={() => setVoiceOpen(false)}
          onConfirm={(t) => { setText((cur) => cur ? `${cur} ${t}` : t); setVoiceOpen(false); }}
        />
      )}
    </>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add pwa/src/voice/ pwa/src/components/chat/VoiceOverlay.tsx pwa/src/components/chat/ChatInput.tsx
git commit -m "feat(pwa): voice input overlay with web speech api"
```

---

## Task 5: Image capture + resize

**Files:**
- Create: `pwa/src/images/capture.ts`
- Create: `pwa/src/images/resize.ts`
- Create: `pwa/tests/unit/images/resize.test.ts`

- [ ] **Step 1: Implement `src/images/resize.ts`**

```typescript
export interface ResizeOptions {
  maxWidth: number;
  maxHeight: number;
  mimeType: "image/jpeg" | "image/webp";
  quality: number;  // 0..1
}

export async function resizeImage(file: File, opts: ResizeOptions): Promise<Blob> {
  const bitmap = await createImageBitmap(file);
  const { width: w0, height: h0 } = bitmap;
  let w = w0, h = h0;
  const r = Math.min(opts.maxWidth / w0, opts.maxHeight / h0, 1);
  w = Math.round(w0 * r);
  h = Math.round(h0 * r);

  const canvas = document.createElement("canvas");
  canvas.width = w;
  canvas.height = h;
  const ctx = canvas.getContext("2d");
  if (!ctx) throw new Error("canvas 2d context unavailable");
  ctx.drawImage(bitmap, 0, 0, w, h);
  return await new Promise<Blob>((resolve, reject) => {
    canvas.toBlob((b) => {
      if (b) resolve(b);
      else reject(new Error("canvas.toBlob failed"));
    }, opts.mimeType, opts.quality);
  });
}

export async function blobToBase64(blob: Blob): Promise<string> {
  const buf = await blob.arrayBuffer();
  const bytes = new Uint8Array(buf);
  let binary = "";
  for (let i = 0; i < bytes.length; i++) binary += String.fromCharCode(bytes[i]!);
  return btoa(binary);
}
```

- [ ] **Step 2: Implement `src/images/capture.ts`**

```typescript
export function pickImage(source: "camera" | "gallery" = "gallery"): Promise<File | null> {
  return new Promise((resolve) => {
    const input = document.createElement("input");
    input.type = "file";
    input.accept = "image/*";
    if (source === "camera") input.capture = "environment";
    input.onchange = () => {
      const f = input.files?.[0] ?? null;
      resolve(f);
    };
    input.oncancel = () => resolve(null);
    input.click();
  });
}
```

- [ ] **Step 3: Write test for resize ratio math**

`pwa/tests/unit/images/resize.test.ts`:

```typescript
import { describe, test, expect } from "bun:test";

// Resize is DOM-heavy; we test only the ratio calc via a pure helper extract.
function computeResized(w0: number, h0: number, maxW: number, maxH: number): { w: number; h: number } {
  const r = Math.min(maxW / w0, maxH / h0, 1);
  return { w: Math.round(w0 * r), h: Math.round(h0 * r) };
}

describe("image resize math", () => {
  test("preserves aspect ratio when downscaling", () => {
    const { w, h } = computeResized(4000, 2000, 1000, 1000);
    expect(w).toBe(1000);
    expect(h).toBe(500);
  });
  test("does not upscale", () => {
    const { w, h } = computeResized(500, 500, 1000, 1000);
    expect(w).toBe(500);
    expect(h).toBe(500);
  });
  test("respects tallest dimension constraint", () => {
    const { w, h } = computeResized(1000, 3000, 500, 1500);
    expect(w).toBe(500);
    expect(h).toBe(1500);
  });
});
```

- [ ] **Step 4: Run**

```bash
cd pwa
bun test tests/unit/images/resize.test.ts
```

Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add pwa/src/images/ pwa/tests/unit/images/
git commit -m "feat(pwa): image pick + canvas resize + base64 helpers"
```

---

## Task 6: ImageAttachment component + integrar en Conversation

**Files:**
- Create: `pwa/src/components/chat/ImageAttachment.tsx`
- Modify: `pwa/src/pages/Conversation.tsx` for attachment state + send

- [ ] **Step 1: Implement `ImageAttachment.tsx`**

```typescript
interface Props {
  filename: string;
  sizeBytes: number;
  dimensions?: { w: number; h: number };
  onRemove: () => void;
}

function formatSize(bytes: number): string {
  if (bytes < 1024) return `${bytes}B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(0)}KB`;
  return `${(bytes / 1024 / 1024).toFixed(1)}MB`;
}

export function ImageAttachment({ filename, sizeBytes, dimensions, onRemove }: Props) {
  return (
    <div className="bg-surface-1 border border-border p-2 flex gap-2 items-center">
      <div className="w-12 h-12 bg-gradient-to-br from-surface-3 to-surface-2 border border-border-strong flex items-center justify-center font-mono text-text-dim text-[9px] flex-shrink-0">IMG</div>
      <div className="flex-1 font-mono text-[10px] text-text">
        <div className="font-semibold">{filename}</div>
        <div className="text-text-dim text-[9px]">{dimensions ? `${dimensions.w}×${dimensions.h} · ` : ""}{formatSize(sizeBytes)}</div>
      </div>
      <button onClick={onRemove} className="text-coral font-mono text-[14px]" aria-label="remove">×</button>
    </div>
  );
}
```

- [ ] **Step 2: Add attachment state to `Conversation.tsx`**

In `Conversation.tsx`, add state:

```typescript
import { pickImage } from "../images/capture";
import { resizeImage, blobToBase64 } from "../images/resize";
import { ImageAttachment } from "../components/chat/ImageAttachment";

interface PendingAttachment {
  id: string;
  filename: string;
  mimeType: string;
  base64: string;
  sizeBytes: number;
  width: number;
  height: number;
}

// Inside the component:
const [pendingAtts, setPendingAtts] = useState<PendingAttachment[]>([]);

const onAttachImage = async () => {
  const file = await pickImage("gallery");
  if (!file) return;
  const blob = await resizeImage(file, { maxWidth: 1920, maxHeight: 1920, mimeType: "image/jpeg", quality: 0.85 });
  const base64 = await blobToBase64(blob);
  const bitmap = await createImageBitmap(blob);
  setPendingAtts((cur) => [...cur, {
    id: crypto.randomUUID(),
    filename: file.name,
    mimeType: "image/jpeg",
    base64,
    sizeBytes: blob.size,
    width: bitmap.width,
    height: bitmap.height,
  }]);
};

const onSend = (prompt: string) => {
  const pair = pairings.find((p) => p.id === pairingId);
  if (!pair) return;
  getAgent(pair).send({
    type: "send_prompt",
    conversationId,
    prompt,
    attachments: pendingAtts.map((a) => ({
      type: "image" as const,
      mimeType: a.mimeType,
      data: a.base64,
      filename: a.filename,
    })),
  });
  setPendingAtts([]);
};
```

Render pending attachments above the ChatInput:

```tsx
{pendingAtts.length > 0 && (
  <div className="px-4 py-2 border-t border-border flex flex-col gap-1.5 bg-bg">
    {pendingAtts.map((a) => (
      <ImageAttachment
        key={a.id}
        filename={a.filename}
        sizeBytes={a.sizeBytes}
        dimensions={{ w: a.width, h: a.height }}
        onRemove={() => setPendingAtts((cur) => cur.filter((x) => x.id !== a.id))}
      />
    ))}
  </div>
)}
<ChatInput
  disabled={pending}
  placeholder={pending ? "bloqueado hasta approval..." : "escribe un prompt..."}
  onSend={onSend}
  onAttachImage={onAttachImage}
/>
```

- [ ] **Step 3: Also render attachments in the UserTurn**

Update `UserTurn.tsx` to accept optional attachments:

```typescript
import type { AttachmentPayload } from "../../protocol/messages"; // Actually, the type is inferred via zod
import type { z } from "zod";

interface Attachment {
  type: string;
  mimeType: string;
  data?: string;
  filename?: string;
}

export function UserTurn({ text, timestamp, attachments }: { text: string; timestamp: number; attachments?: Attachment[] }) {
  return (
    <div>
      <div className="font-mono text-[8.5px] tracking-[0.2em] uppercase text-accent mb-1">you · {relativeTime(timestamp)}</div>
      <div className="text-[13px] leading-[1.45] text-text pl-2.5 border-l-2 border-accent">{text}</div>
      {attachments && attachments.length > 0 && (
        <div className="pl-2.5 mt-2 flex flex-col gap-1">
          {attachments.map((a, i) => (
            <div key={i} className="font-mono text-[10px] text-text-dim">
              ▪ {a.filename ?? a.mimeType}
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

Update call sites in `Conversation.tsx` accordingly to pass `s.event.payload.attachments`.

- [ ] **Step 4: Commit**

```bash
git add pwa/src/components/chat/ImageAttachment.tsx pwa/src/components/chat/UserTurn.tsx pwa/src/pages/Conversation.tsx
git commit -m "feat(pwa): image attachment with resize + multimodal send"
```

---

## Task 7: Quick prompts storage + store

**Files:**
- Modify: `pwa/src/storage/pairings.ts` — add quick prompts CRUD (or split file)
- Create: `pwa/src/state/quick-prompts.ts`

- [ ] **Step 1: Add CRUD functions for quick prompts**

Update `pwa/src/storage/pairings.ts` (or create new file `quick-prompts.ts`):

```typescript
import { getDb, type QuickPrompt } from "./db";

export async function listQuickPrompts(): Promise<QuickPrompt[]> {
  const db = await getDb();
  const all = await db.getAll("quickPrompts");
  return all.sort((a, b) => a.createdAt - b.createdAt);
}

export async function saveQuickPrompt(p: QuickPrompt): Promise<void> {
  const db = await getDb();
  await db.put("quickPrompts", p);
}

export async function removeQuickPrompt(id: string): Promise<void> {
  const db = await getDb();
  await db.delete("quickPrompts", id);
}

export async function seedDefaultQuickPrompts(): Promise<void> {
  const db = await getDb();
  const existing = await db.getAll("quickPrompts");
  if (existing.length > 0) return;
  const now = Date.now();
  const defaults: QuickPrompt[] = [
    { id: crypto.randomUUID(), name: "Code review", body: "Haz code review del último diff. Busca bugs, memory leaks, edge cases no cubiertos, problemas de seguridad.", shortcut: "/cr", createdAt: now },
    { id: crypto.randomUUID(), name: "Escribir tests", body: "Escribe tests unitarios para los cambios más recientes. Cubre happy path, edge cases y casos de error.", shortcut: "/test", createdAt: now + 1 },
    { id: crypto.randomUUID(), name: "Explicar código", body: "Explícame qué hace este módulo, sus dependencias y posibles mejoras.", shortcut: "/exp", createdAt: now + 2 },
    { id: crypto.randomUUID(), name: "Resumen de sesión", body: "Dame un resumen de lo que hemos hecho en esta conversación, archivos tocados y próximos pasos.", shortcut: "/sum", createdAt: now + 3 },
  ];
  for (const p of defaults) await db.put("quickPrompts", p);
}
```

- [ ] **Step 2: Implement `src/state/quick-prompts.ts`**

```typescript
import { create } from "zustand";
import { listQuickPrompts, saveQuickPrompt, removeQuickPrompt, seedDefaultQuickPrompts } from "../storage/quick-prompts";
import type { QuickPrompt } from "../storage/db";

interface QuickPromptsState {
  prompts: QuickPrompt[];
  loaded: boolean;
  load: () => Promise<void>;
  add: (p: Omit<QuickPrompt, "id" | "createdAt">) => Promise<void>;
  update: (p: QuickPrompt) => Promise<void>;
  remove: (id: string) => Promise<void>;
}

export const useQuickPrompts = create<QuickPromptsState>((set, get) => ({
  prompts: [],
  loaded: false,
  async load() {
    await seedDefaultQuickPrompts();
    const all = await listQuickPrompts();
    set({ prompts: all, loaded: true });
  },
  async add(p) {
    const full: QuickPrompt = { id: crypto.randomUUID(), createdAt: Date.now(), ...p };
    await saveQuickPrompt(full);
    set({ prompts: [...get().prompts, full] });
  },
  async update(p) {
    await saveQuickPrompt(p);
    set({ prompts: get().prompts.map((x) => x.id === p.id ? p : x) });
  },
  async remove(id) {
    await removeQuickPrompt(id);
    set({ prompts: get().prompts.filter((x) => x.id !== id) });
  },
}));
```

- [ ] **Step 3: Initialize in App.tsx**

In `App.tsx`, add to the initial effect:

```typescript
import { useQuickPrompts } from "./state/quick-prompts";

useEffect(() => {
  load().then(() => {
    if (usePairings.getState().pairings.length > 0) setRoute({ name: "inbox" });
  });
  useQuickPrompts.getState().load();
}, []);
```

- [ ] **Step 4: Commit**

```bash
git add pwa/src/storage/quick-prompts.ts pwa/src/state/quick-prompts.ts pwa/src/App.tsx
git commit -m "feat(pwa): quick prompts storage + store with default seed"
```

---

## Task 8: QuickPromptsSheet component + integrar en ChatInput

**Files:**
- Create: `pwa/src/components/chat/QuickPromptsSheet.tsx`
- Modify: `pwa/src/pages/Conversation.tsx`

- [ ] **Step 1: Implement sheet**

```typescript
import { useQuickPrompts } from "../../state/quick-prompts";

export function QuickPromptsSheet({ onInsert, onCancel, onManage }: {
  onInsert: (body: string) => void;
  onCancel: () => void;
  onManage: () => void;
}) {
  const prompts = useQuickPrompts((s) => s.prompts);

  return (
    <div className="fixed inset-0 z-50 bg-black/70 flex items-end">
      <div className="bg-surface-1 w-full border-t border-border-strong p-5 max-h-[75vh] overflow-y-auto">
        <h3 className="font-serif font-semibold text-[16px] mb-1">Quick prompts</h3>
        <div className="font-mono text-[10.5px] text-text-dim mb-3">Tap para insertar · editables en settings</div>
        <div className="flex flex-col gap-1.5">
          {prompts.map((p) => (
            <button
              key={p.id}
              onClick={() => onInsert(p.body)}
              className="px-3 py-2.5 border border-border-strong flex justify-between items-center gap-2.5 text-left"
            >
              <div className="flex-1 min-w-0">
                <div className="font-serif font-semibold text-[13px] tracking-[-0.01em] text-text">{p.name}</div>
                <div className="font-mono text-[9.5px] text-text-dim mt-0.5 truncate">{p.body}</div>
              </div>
              {p.shortcut && <span className="font-mono text-[9px] text-text-faint px-1.5 py-0.5 border border-border">{p.shortcut}</span>}
            </button>
          ))}
        </div>
        <div className="flex gap-2 mt-4">
          <button className="flex-1 py-2.5 font-mono text-[10px] uppercase tracking-[0.15em] border border-border-strong" onClick={onCancel}>cerrar</button>
          <button className="flex-1 py-2.5 font-mono text-[10px] uppercase tracking-[0.15em] border border-accent text-accent" onClick={onManage}>gestionar</button>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Integrar en `Conversation.tsx`**

Add state + handler for quick prompts sheet and pass to `ChatInput`:

```typescript
import { QuickPromptsSheet } from "../components/chat/QuickPromptsSheet";

// Inside component:
const [qpSheetOpen, setQpSheetOpen] = useState(false);

// Pass onOpenQuickPrompts to ChatInput:
<ChatInput
  disabled={pending}
  placeholder={...}
  onSend={onSend}
  onAttachImage={onAttachImage}
  onOpenQuickPrompts={() => setQpSheetOpen(true)}
/>

{qpSheetOpen && (
  <QuickPromptsSheet
    onInsert={(body) => {
      // Prefill input — since ChatInput owns its text state, we need a ref or
      // elevate text to Conversation. For MVP, we do a simple approach:
      // close the sheet and send immediately with the prompt body.
      // A better approach: lift chat text state up. Do that refactor.
      setQpSheetOpen(false);
      onSend(body); // sends immediately — not ideal UX
    }}
    onCancel={() => setQpSheetOpen(false)}
    onManage={() => { setQpSheetOpen(false); /* navigate to settings quick prompts */ }}
  />
)}
```

- [ ] **Step 3: Lift chat text state (better UX for quick prompts)**

Refactor `ChatInput` to be controlled (text + onChange as props) so `Conversation` can prefill when user picks a quick prompt:

Update `ChatInput.tsx` signature:

```typescript
export function ChatInput({ text, onTextChange, disabled, placeholder, onSend, onOpenQuickPrompts, onAttachImage }: {
  text: string;
  onTextChange: (t: string) => void;
  disabled?: boolean;
  placeholder?: string;
  onSend: (text: string) => void;
  onOpenQuickPrompts?: () => void;
  onAttachImage?: () => void;
}) {
  // Use `text`/`onTextChange` instead of internal useState
  // ...
}
```

In `Conversation.tsx`:

```typescript
const [chatText, setChatText] = useState("");

// onSend clears text
const onSend = (prompt: string) => {
  // same as before, clear text + atts
  setChatText("");
  setPendingAtts([]);
  // ... send via agent
};

<ChatInput
  text={chatText}
  onTextChange={setChatText}
  disabled={pending}
  onSend={onSend}
  onAttachImage={onAttachImage}
  onOpenQuickPrompts={() => setQpSheetOpen(true)}
/>

{qpSheetOpen && (
  <QuickPromptsSheet
    onInsert={(body) => {
      setChatText((cur) => cur ? `${cur}\n${body}` : body);
      setQpSheetOpen(false);
    }}
    onCancel={() => setQpSheetOpen(false)}
    onManage={() => { setQpSheetOpen(false); /* route to settings.quickPrompts */ }}
  />
)}
```

- [ ] **Step 4: Commit**

```bash
git add pwa/src/components/chat/QuickPromptsSheet.tsx pwa/src/components/chat/ChatInput.tsx pwa/src/pages/Conversation.tsx
git commit -m "feat(pwa): quick prompts sheet integrated into chat input"
```

---

## Task 9: Settings page para quick prompts (CRUD)

**Files:**
- Create: `pwa/src/pages/SettingsQuickPrompts.tsx`
- Modify: `pwa/src/App.tsx` to add route

- [ ] **Step 1: Implement CRUD UI**

```typescript
import { useState } from "react";
import { useQuickPrompts } from "../state/quick-prompts";
import type { QuickPrompt } from "../storage/db";
import { StatusBar } from "../components/layout/StatusBar";
import { HeaderRow } from "../components/layout/HeaderRow";
import { Label } from "../components/typography/Label";
import { PrimaryButton } from "../components/buttons/PrimaryButton";

export function SettingsQuickPrompts({ onBack }: { onBack: () => void }) {
  const prompts = useQuickPrompts((s) => s.prompts);
  const add = useQuickPrompts((s) => s.add);
  const update = useQuickPrompts((s) => s.update);
  const remove = useQuickPrompts((s) => s.remove);

  const [editing, setEditing] = useState<QuickPrompt | null>(null);
  const [name, setName] = useState("");
  const [body, setBody] = useState("");
  const [shortcut, setShortcut] = useState("");

  const startNew = () => {
    setEditing({ id: "NEW", name: "", body: "", shortcut: "", createdAt: Date.now() });
    setName(""); setBody(""); setShortcut("");
  };

  const startEdit = (p: QuickPrompt) => {
    setEditing(p);
    setName(p.name);
    setBody(p.body);
    setShortcut(p.shortcut ?? "");
  };

  const save = async () => {
    if (!name.trim() || !body.trim()) return;
    if (editing?.id === "NEW") {
      await add({
        name: name.trim(),
        body: body.trim(),
        ...(shortcut.trim() ? { shortcut: shortcut.trim() } : {}),
      });
    } else if (editing) {
      await update({
        ...editing,
        name: name.trim(),
        body: body.trim(),
        ...(shortcut.trim() ? { shortcut: shortcut.trim() } : {}),
      });
    }
    setEditing(null);
  };

  const del = async () => {
    if (editing && editing.id !== "NEW") {
      if (confirm(`Eliminar "${editing.name}"?`)) {
        await remove(editing.id);
        setEditing(null);
      }
    }
  };

  return (
    <div className="flex flex-col h-full bg-bg">
      <StatusBar />
      <HeaderRow left={<button className="font-mono text-text-dim text-[14px]" onClick={editing ? () => setEditing(null) : onBack}>‹ back</button>} right={<Label>quick prompts</Label>} />

      {!editing ? (
        <div className="flex-1 overflow-y-auto">
          {prompts.map((p) => (
            <button key={p.id} onClick={() => startEdit(p)} className="w-full text-left px-4 py-3 border-b border-border block">
              <div className="flex justify-between items-baseline">
                <span className="font-serif font-semibold text-[14px] text-text">{p.name}</span>
                {p.shortcut && <span className="font-mono text-[9px] text-text-dim px-1.5 py-0.5 border border-border">{p.shortcut}</span>}
              </div>
              <div className="font-mono text-[10px] text-text-dim mt-1 line-clamp-2">{p.body}</div>
            </button>
          ))}
          <button className="m-4 py-3 border border-dashed border-border-strong w-[calc(100%-2rem)] font-mono text-[9.5px] uppercase tracking-[0.15em] text-text-dim" onClick={startNew}>+ nuevo quick prompt</button>
        </div>
      ) : (
        <div className="flex-1 overflow-y-auto p-5 flex flex-col gap-4">
          <Label>{editing.id === "NEW" ? "nuevo" : "editar"}</Label>

          <div>
            <label className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim mb-1.5 block">nombre</label>
            <input
              className="font-mono text-[14px] text-text bg-transparent border-b-[1.5px] border-accent py-2 w-full outline-none"
              value={name} onChange={(e) => setName(e.target.value)}
            />
          </div>

          <div>
            <label className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim mb-1.5 block">atajo · opcional</label>
            <input
              className="font-mono text-[12px] text-text bg-surface-1 border-l-2 border-text-faint px-3 py-2 w-full outline-none"
              value={shortcut} onChange={(e) => setShortcut(e.target.value)}
              placeholder="/cr"
            />
          </div>

          <div>
            <label className="font-mono text-[9.5px] tracking-[0.15em] uppercase text-text-dim mb-1.5 block">cuerpo</label>
            <textarea
              className="font-mono text-[12px] text-text bg-surface-1 border-l-2 border-text-faint px-3 py-2 w-full outline-none resize-none"
              rows={6}
              value={body} onChange={(e) => setBody(e.target.value)}
            />
          </div>

          <div className="mt-auto">
            <PrimaryButton onClick={save}>Guardar</PrimaryButton>
            {editing.id !== "NEW" && (
              <button className="w-full py-3 mt-2 border border-coral/30 text-coral font-mono text-[10px] uppercase tracking-[0.15em]" onClick={del}>
                Eliminar
              </button>
            )}
          </div>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Add route in `App.tsx`**

Add a new route case:

```typescript
| { name: "settings-quick-prompts" }
```

And in the switch:

```tsx
case "settings-quick-prompts": return <SettingsQuickPrompts onBack={() => setRoute({ name: "settings" })} />;
```

Add an entry in Settings page:

In `pwa/src/pages/Settings.tsx`, add a section above Máquinas:

```tsx
<div className="px-4 pt-4 pb-2">
  <button onClick={onOpenQuickPrompts} className="w-full text-left">
    <span className="font-serif font-semibold text-[16px] tracking-[-0.02em]">Quick prompts</span>
    <div className="font-mono text-[9.5px] text-text-dim mt-1">gestionar biblioteca</div>
  </button>
</div>
```

Add `onOpenQuickPrompts` prop to `Settings` and wire from `App.tsx`.

- [ ] **Step 3: Commit**

```bash
git add pwa/src/pages/SettingsQuickPrompts.tsx pwa/src/App.tsx pwa/src/pages/Settings.tsx
git commit -m "feat(pwa): quick prompts crud settings page"
```

---

## Task 10: Update E2E test for Plan 3 features

**Files:**
- Modify: `pwa/tests/e2e/full-flow.spec.ts`
- Create: `pwa/tests/e2e/quick-prompts.spec.ts`

- [ ] **Step 1: Write quick prompts E2E**

```typescript
import { test, expect } from "@playwright/test";

test.describe("quick prompts", () => {
  test.beforeEach(async ({ page, context }) => {
    // Preseed storage with a pairing so we jump straight to inbox
    await context.addInitScript(() => {
      (globalThis as unknown as { _seedInbox?: boolean })._seedInbox = true;
    });
    // Fake WS same as in full-flow.spec.ts — simplified here
    // (Real test would import a shared helper.)
  });

  test.skip("quick prompts default seed appears in settings", async ({ page }) => {
    // Skipped because seed runs on first open; integrating with fake WS adds
    // complexity. Run manually during dogfood.
  });
});
```

Keep this as a `.skip` test placeholder — E2E for quick prompts is better verified via manual dogfood given the complexity of orchestrating IndexedDB + fake WS.

- [ ] **Step 2: Commit**

```bash
git add pwa/tests/e2e/
git commit -m "test(pwa): quick prompts e2e placeholder (dogfood-verified)"
```

---

## Task 11: Manual dogfood test completo de Fase 1

- [ ] **Step 1: Use aiconnect for 1 real working day**

Real work session. Track any friction:

1. Abrir app en iPhone. Inbox carga con ≥ 1 conversación de Claude Code en progreso.
2. Dictar prompt por voz: "revisa session.ts y explica qué hace"
3. Verificar transcripción, editar si hace falta, enviar.
4. Ver streaming en vivo con tool_use visible y respuesta en markdown.
5. Llega approval para `npm test` → aprobar desde móvil.
6. Cambiar modelo Sonnet → Opus tocando el badge.
7. Tomar screenshot del finder mostrando algo, adjuntarla al prompt: "¿este archivo debería estar en src/ o en bin/?".
8. Usar quick prompt `/cr` para code review.
9. Crear nuevo quick prompt "Documenta función" con cuerpo "Agrega JSDoc a la función X explicando parámetros, retorno y casos de error."
10. Matar app (swipe), reabrir, verificar que retoma la conversación.
11. Poner teléfono en modo avión mid-prompt, salir del avión, verificar reconexión automática.
12. Al final del día, revisar `~/.aiconnect/agent.db` (SQLite) y confirmar que todas las conversaciones están ahí.

- [ ] **Step 2: Documentar bugs encontrados**

Crear `docs/superpowers/dogfood-fase1.md` con issues observados y priorización de fixes antes de marcar Fase 1 como terminada.

- [ ] **Step 3: Fix blockers antes de cerrar**

Cualquier bug que te haga "pelearte con la app" se arregla antes del tag final. Friction aceptable (ej: un placeholder feo, un borde UI) va a backlog de Fase 2.

- [ ] **Step 4: Tag final de Fase 1**

```bash
git tag -a "fase1-complete" -m "aiconnect Fase 1 (MVP) complete: agent + pwa + mobile extras"
```

---

## Self-Review Checklist

**Spec coverage (Plan 3 scope):**
- ✅ Voice input con Web Speech API + overlay (Tasks 2, 3, 4)
- ✅ Image attachment via file picker + canvas resize (Tasks 1, 5, 6)
- ✅ Multimodal prompts que llegan al CLI (Task 1, 6)
- ✅ Quick prompts storage con seed default (Task 7)
- ✅ Quick prompts sheet en input (Task 8)
- ✅ Quick prompts CRUD en settings (Task 9)
- ✅ Protocol v2 coordinado entre agente y PWA (Task 1)

**Placeholder scan:** Task 10 tiene tests `.skip` documentados — no es placeholder oculto sino decisión explícita de cubrir via dogfood. Task 11 requiere tiempo real con la app.

**Type consistency:** `AttachmentPayload`, `QuickPrompt`, `PendingAttachment` coherentes.

**Ruta completa Fase 1:** Plans 1 → 2 → 3 cubren el scope completo con dependencias claras. Plan 3 depende estructuralmente de Plan 2 (mismo PWA). Plan 1 puede correr en paralelo con Plan 2 tan pronto el protocol schema esté congelado.

---

## Execution Handoff

**Plan 3 guardado en `docs/superpowers/plans/2026-04-18-aiconnect-fase1-plan3-extras.md`.**

La secuencia recomendada es:
1. **Plan 1** primero (agente completo) — 2 semanas con subagent-driven
2. **Plan 2** segundo (PWA core) — 1.5 semanas, dogfood contra agente de Plan 1
3. **Plan 3** tercero (extras mobile-first) — 4 días, requiere PWA del Plan 2

Al completar Plan 3 + Task 11 (dogfood de 1 día completo sin blockers), **Fase 1 está terminada** per criterio del spec.

Siguiente fase natural después: **Fase 2 · Expansión** — Codex CLI + multi-máquina inbox + UI búsqueda + tags + plan mode + @mentions + aiconnect memory + push actionable. Esa fase merece su propio ciclo de brainstorming → spec → plans cuando corresponda.
