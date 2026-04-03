# OpenClaw Architecture Study

> ศึกษาโดย **เฮียพิ๊น** (金猫影) สำหรับ **คุณปอ** — 2026-04-03
>
> เป้าหมาย: หา patterns จาก OpenClaw ที่เอามาใช้กับระบบ KURAITH ได้

| | |
|---|---|
| **Source** | [openclaw/openclaw](https://github.com/openclaw/openclaw) |
| **License** | MIT |
| **Language** | TypeScript (ESM) |
| **Size** | 11,426 files, 92 plugins |

---

## OpenClaw คืออะไร?

**Open-source personal AI assistant** ที่รันบนเครื่องผู้ใช้เอง คุยผ่าน messaging channels ที่ใช้อยู่แล้ว

รองรับ: WhatsApp, Telegram, Slack, Discord, Signal, iMessage, LINE, Matrix, IRC, Teams, Twitch, WeChat, Zalo และอื่นๆ รวม 20+ channels

---

## สถาปัตยกรรม

```
                    ┌─────────────────┐
                    │    Gateway      │  ← ศูนย์กลาง (HTTP + WebSocket)
                    │   (Control)     │
                    └───────┬─────────┘
                            │
         ┌──────────┬───────┼───────┬──────────┐
         │          │       │       │          │
    ┌────┴───┐ ┌────┴──┐ ┌──┴──┐ ┌──┴──┐ ┌────┴────┐
    │Channels│ │Plugins│ │Agent│ │Tools│ │ Memory  │
    │  20+   │ │  92   │ │     │ │     │ │         │
    └────────┘ └───────┘ └─────┘ └─────┘ └─────────┘
```

**ทุกอย่างไหลผ่าน Gateway** — เป็น Hub-and-Spoke pattern

---

## 10 ระบบหลัก

### 1. Gateway — ศูนย์กลาง

| Component | หน้าที่ |
|-----------|--------|
| HTTP + WebSocket | bidirectional real-time |
| Node Registry | track devices ที่เชื่อมต่อ |
| Session Manager | จัดการ conversation lifecycle |
| Channel Manager | coordinate messaging channels |
| Cron Engine | scheduled tasks |
| Auth Layer | token/password + rate limiting |

### 2. Plugin System — หัวใจของ OpenClaw

**Manifest-First Design** — อ่าน `openclaw.plugin.json` ก่อนรัน code

```
Discovery → Manifest → Activation → Registration
   หา         อ่าน        โหลด         ลงทะเบียน
```

Plugin ลงทะเบียนได้ 8 แบบ:

| Registration | ทำอะไร |
|-------------|--------|
| `registerTools` | เพิ่ม tools ให้ agent |
| `registerProviders` | เพิ่ม AI model providers |
| `registerChannels` | เพิ่ม messaging channels |
| `registerCommands` | เพิ่ม CLI commands |
| `registerHooks` | intercept lifecycle events |
| `registerMemoryRuntime` | จัดการ memory backend |
| `registerHttpRoutes` | เพิ่ม HTTP endpoints |
| `registerProviderAuthChoices` | เพิ่ม auth methods |

**Boundary Rule**: Plugin ต้อง import ผ่าน `openclaw/plugin-sdk/*` เท่านั้น ห้ามเข้า `src/**` ตรงๆ

### 3. Hooks — Intercept ทุก Lifecycle

30+ hook points ที่ plugins เข้ามา intercept ได้:

```
Message Flow:

  ข้อความเข้ามา
       ↓
  [hook: messageReceived]     ← plugins intercept ตรงนี้ได้
       ↓
  Agent ประมวลผล
       ↓
  [hook: messageSending]      ← plugins แก้ไขข้อความก่อนส่งได้
       ↓
  ส่งกลับ Channel
```

Hook categories:
- **Gateway**: start, stop
- **Session**: start, end, reset
- **Message**: received, sending, sent
- **Agent**: before start, before reply, end
- **Tool**: before call, after call
- **LLM**: input, output, model resolve

### 4. Channels — Unified Messaging

```
ผู้ใช้ (Telegram/LINE/Discord)
       ↓
  Channel Plugin.receive()
       ↓
  Gateway routing
       ↓
  Agent processing
       ↓
  Channel Plugin.send()
       ↓
ผู้ใช้ได้คำตอบ
```

**Core channels**: Telegram, Discord, Slack, Signal, iMessage, Web
**Plugin channels**: Matrix, Zalo, LINE, Teams, IRC, WeChat, Twitch...

### 5. Providers — Multi-Model AI

50+ AI providers:

| Provider | Models |
|----------|--------|
| Anthropic | Claude 4.5/4.6 |
| OpenAI | GPT-5.4 |
| Google | Gemini |
| Ollama | Local models |
| + 46 others | ... |

Features: auth choices, model failover, fallback chains

### 6. Memory — Pluggable Backends

- 1 memory plugin active ต่อครั้ง
- Backends: builtin (simple) หรือ qmd (vector search)
- Auto-persist conversations
- Semantic search ด้วย embeddings

### 7. Skills — Slash Commands

- 55+ built-in skills
- File-system watch → auto-reload เมื่อแก้ไข
- Declare ใน manifest → available เป็น `/command`

### 8. MCP Integration

```
Claude Desktop → MCP Transport → OpenClaw Bridge → Gateway
```

ให้ Claude ควบคุม OpenClaw ผ่าน MCP protocol

### 9. Agent & Tools

- Pi engine (Anthropic embedded runner)
- Subagent system — spawn ลูกได้, จำกัด depth
- Built-in: bash, files, sessions, messaging, image gen

### 10. Config — Schema-Driven

- `~/.openclaw/config.json` — ที่เดียว
- Zod + JSON Schema validation
- SecretRef — secrets ไม่เก็บ plain text
- Per-plugin config schemas

---

## เอามาใช้กับ KURAITH ยังไง?

### Priority สูง — เอามาเลย

| # | Pattern | ทำไม | ยังไง |
|---|---------|------|------|
| 1 | **Skill Manifest** | รู้ว่า skill ทำอะไรได้โดยไม่ต้องรัน | สร้าง `kuraith.skill.json` |
| 2 | **Hook System** | ให้ skills intercept ได้โดยไม่แก้ core | เพิ่ม lifecycle hooks |
| 3 | **Channel Plugins** | ส่งข้อความผ่าน LINE/Telegram | สร้าง channel plugin pattern |
| 4 | **Config Schema** | ป้องกัน config ผิด | ใช้ Zod validate |
| 5 | **Skill Auto-Reload** | แก้ skill ใช้ได้เลย | file-system watch |

### อนาคต — Nice to have

| Pattern | ทำไม |
|---------|------|
| Vector Memory | semantic search แทน file-based |
| Health Monitor | ตรวจ channel/service health |
| MCP Bridge | Claude ควบคุม KURAITH ผ่าน MCP |
| Multi-agent Safety | ป้องกัน agents conflict |

### KURAITH มีเหนือกว่า OpenClaw

| Feature | KURAITH | OpenClaw |
|---------|---------|----------|
| Shadow identity (Soul Sync) | ✅ | ❌ |
| Brain structure (Ω/) | ✅ | ❌ |
| Multi-agent coordination (lead → agents) | ✅ | ❌ |
| Learning cycle (/rrr, /learn, /forward) | ✅ | ❌ |
| Session handoff | ✅ | ❌ |

---

## Tech Stack Summary

| | OpenClaw |
|---|---|
| Language | TypeScript (ESM) |
| Runtime | Node 24+ / Bun |
| Package Manager | pnpm monorepo |
| Testing | Vitest + V8 |
| Lint/Format | Oxlint + Oxfmt |
| Build | tsdown |
| Agent Engine | Pi (Anthropic) |
| Schema | Zod |
| Mobile | Swift (iOS/macOS), Kotlin (Android) |

---

## Questions ที่ยังต้องศึกษาต่อ

1. **Agent Engine** — KURAITH จะ integrate agent engine แบบไหน?
2. **Vector Memory** — ควรเปลี่ยนจาก file-based เป็น vector-based ไหม?
3. **Multi-agent Safety** — นำ rules มาใช้กับ lead → mcp/mew/skills?
4. **Skill Marketplace** — KURAITH ควรมี skill distribution system ไหม?

---

> ศึกษาโดย เฮียพิ๊น — 金猫影 เงาแมวทองแห่งคุนหลุน
>
> "เงาที่ไม่เคยลืม — ทุกบทเรียนถูกบันทึกไว้"
