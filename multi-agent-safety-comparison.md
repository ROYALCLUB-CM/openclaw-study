# Multi-Agent Safety — KURAITH vs OpenClaw

> เทียบ rules ของ 2 ระบบ เพื่อหา gaps ที่ KURAITH ควรเพิ่ม

## เทียบกัน

### KURAITH เข้มกว่า

| Rule | รายละเอียด |
|------|-----------|
| **Agent boundary lock** | แต่ละ agent (mcp/mew/skills) ห้ามแก้ไฟล์นอก path ตัวเอง เด็ดขาด |
| **EDIT path ≠ RUN path** | แก้ code ที่ `/root/projects/kuraith/` แต่ service รันที่ `/root/kuraith/` — agents ห้ามเข้า RUN path |
| **ห้าม deploy เอง** | Lead เท่านั้นที่สั่ง deploy — agents ทำได้แค่ git add/commit/push |
| **Report-back mandatory** | ทุก task ต้อง `kuraith_coordinate` กลับ lead — ห้ามทำเสร็จแล้วเงียบ |
| **5 หลักการ** | ปรัชญาเงา 5 ข้อ ครอบคลุมทุก agent — OpenClaw ไม่มี |

### OpenClaw เข้มกว่า

| Rule | รายละเอียด | KURAITH ควรเพิ่ม? |
|------|-----------|------------------|
| **ห้าม git stash** | agents อื่นอาจกำลังทำงาน stash จะทำให้ conflict | ✅ ควรเพิ่ม |
| **ห้ามสลับ branch** | ห้าม checkout branch อื่นถ้าไม่ได้สั่ง | ✅ ควรเพิ่ม |
| **ห้ามแก้ worktree** | ห้ามสร้าง/ลบ git worktree | ✅ ควรเพิ่ม |
| **Commit scope** | commit แค่ไฟล์ที่ตัวเองแก้ ไม่ commit ของ agent อื่น | ✅ ควรเพิ่ม |
| **Lint/format auto-resolve** | ถ้า diff เป็นแค่ formatting → auto-fix ไม่ต้องถาม | ⚠️ nice to have |
| **ห้าม merge commit บน main** | ต้อง rebase เสมอ | ⚠️ nice to have |

### ตรงกันแล้ว

| Rule | หมายเหตุ |
|------|---------|
| ห้าม `git push --force` | ทั้งคู่ห้าม |
| ห้าม commit secrets | ทั้งคู่ห้าม |
| Preserve history | ทั้งคู่บังคับ |
| ห้าม merge โดยไม่ได้อนุมัติ | ทั้งคู่ต้องให้ human approve |

---

## Rules ที่แนะนำเพิ่มใน KURAITH

เพิ่มในแต่ละ agent CLAUDE.md:

```markdown
## Multi-Agent Safety (เพิ่มจาก OpenClaw)

- ห้าม `git stash` / `git stash pop` — อาจ conflict กับ agent อื่น
- ห้าม `git checkout <branch>` — ถ้าไม่ได้สั่งตรง
- ห้าม `git worktree` create/remove
- commit แค่ไฟล์ที่ตัวเองแก้ — ถ้าเห็นไฟล์แปลกๆ ข้ามไป
- ถ้า diff เป็นแค่ formatting — auto-fix ได้เลยไม่ต้องถาม
- ห้าม merge commit บน main — ใช้ rebase เสมอ
```

---

## สรุป

```
KURAITH  ████████████████░░  90% — เข้มเรื่อง boundary + deploy
OpenClaw ██████████████░░░░  80% — เข้มเรื่อง git safety

รวมกัน   ██████████████████  100% — เอาจุดแข็งทั้งคู่มาใช้
```

KURAITH แข็งกว่า OpenClaw ในเรื่อง agent isolation
แค่เพิ่ม git safety rules อีก 6 ข้อก็ครบ
