---
name: consolidate-memory
description: "Compact Hermes long-term memory (MEMORY.md / USER.md) by merging semantic duplicates while preserving every unique fact. Mandatory backup, mandatory diff proposal, mandatory user confirmation — lossless by design. Trigger when the user asks to 'consolidate memory', 'clean up memory', '整理记忆', '压缩记忆'; or proactively suggest (do not auto-run) when the memory tool returns a hard-limit error."
version: 1.0.0
author: Junlin Zhang (@MaketheFoxRush)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [memory, consolidation, maintenance, lossless, housekeeping]
    homepage: https://github.com/MaketheFoxRush/hermes-consolidate-memory
    related_skills: [consolidate-skills]
---

# Consolidate Memory

Hermes' built-in `memory` tool has a hard cap — **2,200 chars for `MEMORY.md`, 1,375 for `USER.md`**. Once it hits the cap, the agent silently fails to record new lessons. There's no built-in consolidation flow: `memory` only supports `add` / `replace` / `remove`, and the model overwhelmingly biases toward `add`.

This skill closes that gap. It reviews the current memory, finds semantic duplicates and over-specific lessons that can be abstracted, and proposes a compacted version. **It never deletes unique information** — the job is to merge, not to forget.

## Design principles (re-read before every run)

**Lossless is a hard constraint.** Every distinct fact, preference, or lesson present in the original must still be derivable from the result.

- ✅ Five concrete lessons abstracted into one general rule — *if* the new rule covers all the concrete details
- ✅ Removing one of two entries with 100% identical semantics
- ❌ Removing an entry because "it looks no longer relevant" — requires explicit user OK
- ❌ Removing partially-overlapping entries — preserve the unique part

**Two stores, two postures:**
- `MEMORY.md` (environment facts, lessons learned): aggressive merging is fine
- `USER.md` (user identity, communication style): **be extremely conservative**. Prefer redundancy over loss — every line here is part of who the user is

## Mandatory workflow (no step-skipping)

### Step 1 — Backup (mandatory; abort on failure)

```bash
TS=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR=/root/.hermes/memories/.backups   # or ~/.hermes/memories/.backups
mkdir -p "$BACKUP_DIR"
cp /root/.hermes/memories/MEMORY.md "$BACKUP_DIR/MEMORY-$TS.md"
cp /root/.hermes/memories/USER.md   "$BACKUP_DIR/USER-$TS.md"
ls -la "$BACKUP_DIR/MEMORY-$TS.md" "$BACKUP_DIR/USER-$TS.md"
```

If any command errors, **stop immediately**, report the failure, **do not continue**.

### Step 2 — Read current state

```bash
wc -c /root/.hermes/memories/MEMORY.md /root/.hermes/memories/USER.md
cat /root/.hermes/memories/MEMORY.md
cat /root/.hermes/memories/USER.md
```

Record both character counts for the post-merge comparison.

### Step 3 — Analyse (model task)

In order:

1. **100% semantic duplicates** — different wording, identical fact
2. **Abstractable groups** — multiple concrete rules that generalize losslessly into one
3. **Complementary merges** — e.g. "Mark X with label Y" + "Label Y ID is Z" → "Mark X with Y (ID Z)"
4. **Obsolete entries** — flag ONLY when there is explicit evidence (user said it no longer applies, configuration changed). Default = keep.

### Step 4 — Present diff proposal (no writes yet)

Output to the user in this exact shape:

```
## Consolidation Proposal

### MEMORY.md (current X chars → proposed Y chars, saved Z)

#### Merge group 1: [short topic name]
Original entries:
  - "[excerpt]"
  - "[excerpt]"
Reason: [100% duplicate / abstractable / complementary]
Merged into:
  "[new entry text]"

#### Merge group 2: ...

### USER.md (current X chars → proposed Y chars, saved Z)
[same format]

### Summary
- Merge groups: N
- Unique facts dropped: 0   (skill invariant)
- Backup: $BACKUP_DIR/*-$TS.md
```

**You must stop here and wait for the user's reply. Do not enter Step 5 on your own.**

### Step 5 — User confirmation (mandatory)

Ask explicitly:

> Apply the proposal?
> - `yes` → apply all groups
> - `no`  → abort; backup retained
> - `partial: [groups to skip]` → re-generate proposal with those groups removed

If the user replies vaguely ("looks good", "ok", "嗯"), **re-ask for an explicit decision**.

### Step 6 — Apply (via the `memory` tool API)

For each confirmed group, in this order:

1. `memory(action="replace", target="memory"|"user", old_text="<unique substring of first entry>", content="<merged content>")`
2. `memory(action="remove",  target="memory"|"user", old_text="<unique substring of each other entry>")`

**Order matters.** Replace first (so the merged content lands), then remove the consumed entries. If you `remove` first, the subsequent `replace` won't find its anchor.

If any `memory` call returns `success=false`, **stop immediately** and report:
- which groups were applied
- which group failed and the exact error
- backup location for rollback

### Step 7 — Verify + log

```bash
wc -c /root/.hermes/memories/MEMORY.md /root/.hermes/memories/USER.md

LOG=/root/.hermes/memories/.consolidation.log
echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] backup=$BACKUP_DIR/*-$TS.md memory=$BEFORE_MEM->$AFTER_MEM user=$BEFORE_USER->$AFTER_USER groups=N" >> "$LOG"
```

Final report to the user: before/after sizes, groups merged, backup path, log path.

## Rollback

If the user notices something off afterwards:

```bash
ls -lt /root/.hermes/memories/.backups/ | head
cp /root/.hermes/memories/.backups/MEMORY-<TS>.md /root/.hermes/memories/MEMORY.md
cp /root/.hermes/memories/.backups/USER-<TS>.md   /root/.hermes/memories/USER.md
systemctl restart hermes-gateway   # if running under systemd; otherwise restart the agent
```

## Do not

- ❌ Skip Step 1 backup
- ❌ Skip Step 5 user confirmation
- ❌ Delete entries that "look unused"
- ❌ Touch `USER.md` entries about identity, preferred language, or communication channel (even if duplicated)
- ❌ Auto-trigger consolidation within the same turn that hit the memory cap. Tell the user the cap is reached, wait for them to invoke this skill on the next turn.

## When to trigger

- User says: `consolidate memory` / `clean up memory` / `compact memory` / `整理记忆` / `压缩记忆`
- The `memory` tool returns `Memory at X/Y chars` — surface this to the user and **suggest** invoking this skill (do not auto-invoke in the same turn)
- `MEMORY.md` > ~80% full — mention it the next time consolidation is a natural topic

## Backup retention

`.backups/` is **not auto-cleaned**. Files are small (~3 KB each); years of weekly backups stay under 1 MB. Clean only when the user explicitly asks.

---

# 中文说明（简版）

**用途**：整理 Hermes 长期记忆，合并重复条目、压缩冗余，**保证信息不丢失**。

**为什么需要**：Hermes 的 `memory` 工具有硬上限（MEMORY.md 2200 字符，USER.md 1375 字符），触顶后所有新教训会被静默丢弃。upstream **无 consolidate 机制**——`memory` 只有 add/replace/remove，模型默认偏向 add，最终一定撞墙。

**三层信息不丢失保护**：
1. **强制备份**：动手前 `cp` 到 `.backups/`，永不自动清理
2. **强制 diff 提案**：列出每个合并组的原条目 + 理由 + 合并后，禁止跳过直接写入
3. **强制用户确认**：等明确 `yes` / `no` / `部分` 才能进入写入

**触发**：用户说"整理记忆"、"压缩记忆"、"consolidate memory"；或 memory 工具返回触顶错误时由 hermes 主动**建议**（不自动执行）。

**完整流程**：见上方英文 Step 1–7。
