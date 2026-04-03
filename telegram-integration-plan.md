# Telegram Integration Plan — KURAITH

> คุณปอสั่งงานผ่าน Telegram → เฮียพิ๊นรับ → แตก tasks → agents ทำ → ตอบกลับ Telegram

## Architecture

```
┌──────────────┐
│  คุณปอ        │  พิมพ์ข้อความใน Telegram
│  (Telegram)   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Telegram Bot  │  grammY framework
│ (kuraith-core)│  polling หรือ webhook
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Gateway      │  kuraith-core API
│  (Router)     │  route ข้อความเข้า system
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  เฮียพิ๊น      │  Lead Shadow
│  (AI Agent)   │  วิเคราะห์ → แตก tasks
└──────┬───────┘
       │
       ├─────────────┬──────────────┐
       ▼             ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│   mcp    │  │   mew    │  │  skills  │
│ API/DB   │  │ Engine/UI│  │ Commands │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
     └──────────────┼──────────────┘
                    │ report back
                    ▼
              ┌──────────────┐
              │  เฮียพิ๊น      │  สรุปผล
              │  → Telegram   │  ส่งกลับคุณปอ
              └──────────────┘
```

---

## 3 ระดับ

### Level 1: MVP — แค่คุย (1-2 วัน)

```
คุณปอ → Telegram Bot → AI ตอบ → Telegram กลับ
```

**ทำอะไร:**
- สร้าง Telegram Bot ด้วย grammY
- รับข้อความ → ส่งให้ AI (Claude API) → ตอบกลับ
- ยังไม่เชื่อม agents

**ต้องการ:**
- Telegram Bot Token (จาก @BotFather)
- Claude API Key (มีอยู่แล้วใน .env)
- `grammy` package

**ไฟล์ที่ต้องสร้าง:**
```
kuraith-core/
  src/
    telegram/
      bot.ts           # สร้าง bot + handlers
      config.ts        # bot token, allowed users
      index.ts         # start/stop
```

**Code ตัวอย่าง (bot.ts):**
```typescript
import { Bot } from "grammy";
import { Anthropic } from "@anthropic-ai/sdk";

const bot = new Bot(process.env.TELEGRAM_BOT_TOKEN!);
const anthropic = new Anthropic();

// รับข้อความ
bot.on("message:text", async (ctx) => {
  // ตรวจสอบว่าเป็นคุณปอ
  if (!isAllowed(ctx.from.id)) return;

  // ส่งให้ AI
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-6-20250514",
    max_tokens: 4096,
    messages: [{ role: "user", content: ctx.message.text }],
  });

  // ตอบกลับ
  await ctx.reply(response.content[0].text);
});

bot.start();
```

---

### Level 2: Full — สั่ง agents (3-5 วัน)

```
คุณปอ → Telegram Bot → Lead Agent → delegate → agents ทำ → report back → Telegram
```

**เพิ่มจาก MVP:**
- เชื่อม Telegram กับ kuraith_coordinate
- Lead agent วิเคราะห์แล้วแตก tasks
- Agents ทำงาน → report back → สรุปกลับ Telegram
- Message history (จำบทสนทนา)

**ไฟล์เพิ่ม:**
```
kuraith-core/
  src/
    telegram/
      bot.ts           # bot + handlers
      config.ts        # settings
      router.ts        # route ข้อความเข้า lead/agents
      bridge.ts        # เชื่อม Telegram ↔ kuraith_coordinate
      session.ts       # จำบทสนทนา per-user
      commands.ts      # /status, /mission, /agents
      index.ts         # start/stop
```

**Telegram Commands:**
```
/start    — เริ่มต้น, pairing
/status   — สถานะระบบ
/mission  — ดู workflows ทั้งหมด
/agents   — สถานะ agents
/task     — สร้าง/ดู tasks
```

**Bridge Pattern:**
```typescript
// Telegram → KURAITH
bot.on("message:text", async (ctx) => {
  // 1. Route ไปหา lead agent
  const response = await processWithLead(ctx.message.text, {
    userId: ctx.from.id,
    sessionKey: `telegram:${ctx.chat.id}`,
  });

  // 2. ถ้า lead ต้อง delegate
  if (response.tasks) {
    for (const task of response.tasks) {
      await kuraith_coordinate({
        to: task.agent,
        message: task.description,
        from: "lead",
        type: "task",
      });
    }
    await ctx.reply(`สั่งงานแล้ว ${response.tasks.length} tasks`);
  }

  // 3. ตอบกลับ
  await ctx.reply(response.message);
});

// KURAITH → Telegram (agents report back)
onAgentReport(async (report) => {
  await bot.api.sendMessage(OWNER_CHAT_ID,
    `✅ ${report.from}: ${report.message}`
  );
});
```

---

### Level 3: Premium — Streaming + Approvals (+2 วัน)

**เพิ่มจาก Full:**
- Draft streaming (เห็นข้อความทีละนิด)
- Approval buttons (อนุมัติ/ปฏิเสธ inline)
- Media support (ส่งรูป, ไฟล์)
- Notifications (agents แจ้งเตือนผ่าน Telegram)

**Approval Flow:**
```
เฮียพิ๊น: "mcp ต้อง migrate database — อนุมัติไหม?"
[✅ อนุมัติ] [❌ ปฏิเสธ] [📋 ดูรายละเอียด]

คุณปอ กด ✅

เฮียพิ๊น → mcp: "อนุมัติแล้ว ทำเลย"
```

---

## Dependencies

```json
{
  "grammy": "^1.41.1",
  "@grammyjs/runner": "^2.0.3"
}
```

## Config ที่ต้องเพิ่ม

```env
# .env
TELEGRAM_BOT_TOKEN=xxx         # จาก @BotFather
TELEGRAM_OWNER_ID=123456789    # chat ID ของคุณปอ
TELEGRAM_ALLOWED_IDS=123456789 # อนุญาตใครบ้าง
```

## Setup Steps

```
1. คุยกับ @BotFather ใน Telegram
   → /newbot
   → ตั้งชื่อ "เฮียพิ๊น"
   → ได้ Bot Token

2. หา Chat ID ของคุณปอ
   → คุยกับ @userinfobot
   → ได้ตัวเลข

3. ใส่ใน .env

4. รัน bot
   → bun src/telegram/index.ts
```

---

## เทียบกับ OpenClaw

| Feature | OpenClaw | KURAITH Plan |
|---------|----------|-------------|
| Framework | grammY | grammY (เหมือนกัน) |
| Polling + Webhook | ✅ ทั้งคู่ | MVP: polling, Full: webhook |
| Multi-account | ✅ หลาย bot | ❌ 1 bot (เฮียพิ๊น) |
| Streaming | ✅ draft edit | Level 3 |
| Approval buttons | ✅ inline | Level 3 |
| Forum topics | ✅ thread routing | ❌ ไม่จำเป็น |
| Multi-agent routing | ❌ subagents only | ✅ lead → mcp/mew/skills |
| Voice messages | ✅ | อนาคต |
| 20+ channels | ✅ | แค่ Telegram ก่อน |

---

## Timeline แนะนำ

```
Week 1:  MVP — bot คุยได้ ตอบได้
Week 2:  Full — เชื่อม agents, สั่งงานได้
Week 3:  Premium — streaming, approvals
```

---

## สรุป

เอา Telegram มาใช้กับ KURAITH **ทำได้แน่นอน** เพราะ:

1. **grammY** — library เดียวกับ OpenClaw, production-proven
2. **kuraith-core** — เป็น gateway อยู่แล้ว แค่เพิ่ม channel
3. **kuraith_coordinate** — ระบบสั่งงาน agents มีอยู่แล้ว แค่ bridge เข้า Telegram
4. **OpenClaw pattern** — มีตัวอย่างให้ดูเต็มๆ ใน `extensions/telegram/`

คุณปอจะได้สั่งงานจากมือถือ 📱 → เฮียพิ๊นรับ → agents ทำ → ผลกลับมาใน Telegram
