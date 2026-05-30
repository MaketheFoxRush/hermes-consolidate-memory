# hermes-consolidate-memory

A [Hermes Agent](https://github.com/NousResearch/hermes-agent) skill that compacts long-term memory (`MEMORY.md` / `USER.md`) by merging semantic duplicates — **without ever losing unique information**.

[中文说明见下方 ↓](#中文说明)

---

## Why this exists

Hermes' built-in `memory` tool has a hard cap:

- `MEMORY.md` (lessons + environment facts): **2,200 chars**
- `USER.md` (user identity + preferences): **1,375 chars**

When you hit the cap, the agent **silently fails** to record new lessons — it just keeps repeating the same mistakes you thought it had learned from.

There is **no built-in consolidation flow**. The `memory` tool only exposes `add` / `replace` / `remove`, and LLMs overwhelmingly bias toward `add` (cheap, no judgement required) over `remove` (requires deciding what is safe to drop). So memory only grows. Eventually it caps. Then learning stops.

This skill is the missing consolidation pass.

## Design: lossless by construction

The skill **merges** entries; it does not **delete unique information**. The invariant is enforced by a three-layer guard:

1. **Mandatory backup** — every run starts with `cp MEMORY.md USER.md → .backups/*-<timestamp>.md`. Never auto-cleaned.
2. **Mandatory diff proposal** — the agent must list each merge group (originals → merged) with a reason, and stop. No silent writes.
3. **Mandatory user confirmation** — apply only after the user replies `yes`, `no`, or `partial: [groups to skip]`. Vague replies get re-asked.

Writes go through the `memory` tool API (not raw `cat > file`) so Hermes' atomic-write and limit-check guarantees still apply.

## Install

```bash
hermes skills install MaketheFoxRush/hermes-consolidate-memory/consolidate-memory
```

The identifier follows Hermes' `owner/repo/skill-name` convention. Hermes fetches the SKILL.md, runs a security scan, and installs it into `~/.hermes/.hub/`.

Or clone manually if you prefer:

```bash
git clone https://github.com/MaketheFoxRush/hermes-consolidate-memory.git
cp -r hermes-consolidate-memory/skills/consolidate-memory \
  ~/.hermes/skills/maintenance/
```

Verify it loaded:

```bash
hermes skills list | grep consolidate-memory
```

## Usage

Just ask Hermes:

> consolidate memory
> 整理一下你的长期记忆
> compact your memory

The agent will:

1. Back up `MEMORY.md` and `USER.md` to `.backups/*-<TS>.md`
2. Read both files
3. Produce a diff proposal with every merge group, reason, and before/after sizes
4. Wait for your explicit `yes` / `no` / `partial`
5. Apply via the `memory` tool, abort on any failure
6. Verify final sizes, append to `.consolidation.log`

## Rollback

```bash
ls -lt ~/.hermes/memories/.backups/ | head
cp ~/.hermes/memories/.backups/MEMORY-<TS>.md ~/.hermes/memories/MEMORY.md
cp ~/.hermes/memories/.backups/USER-<TS>.md   ~/.hermes/memories/USER.md
systemctl restart hermes-gateway   # if running as a service
```

## Roadmap

- [ ] Optional `hermes cron` integration: weekly proposal pushed to your messaging channel (Telegram / WeChat / Slack), apply only on user reply
- [ ] `--dry-run` flag that writes the proposal to a file without expecting user reply (for CI)
- [ ] Multi-store support if Hermes adds more persistent memory files in future versions

## Contributing

Issues and PRs welcome. The skill is intentionally small — please keep additions in line with the lossless principle.

## License

MIT — see [LICENSE](./LICENSE).

---

## 中文说明

一个 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 的 skill，用于整理它的长期记忆（`MEMORY.md` / `USER.md`），合并语义重复条目、压缩冗余，**绝不丢失独特信息**。

### 为什么需要

Hermes 内置 `memory` 工具有硬上限：

- `MEMORY.md`（教训 + 环境事实）：**2,200 字符**
- `USER.md`（用户身份 + 偏好）：**1,375 字符**

触顶后 agent **静默失败**——它会继续重复犯你以为它早就学过的错。

而 hermes upstream **没有任何 consolidate 机制**。`memory` 工具只有 `add` / `replace` / `remove`，LLM 又强烈偏向 `add`（便宜、无判断成本）而非 `remove`（需要判断什么能扔）。所以记忆只增不减。最终撞墙，学习停止。

这个 skill 就是那个缺失的整理 pass。

### 设计：构造上保证信息不丢失

skill 做的是**合并**，不是**删除独特信息**。三层守护：

1. **强制备份**：每次执行先 `cp MEMORY.md USER.md → .backups/*-<时间戳>.md`，永不自动清理
2. **强制 diff 提案**：模型必须列出每个合并组（原条目 → 合并后 + 理由），停下。**禁止静默写入**
3. **强制用户确认**：等你明确 `yes` / `no` / `部分应用：[不要合并的组]` 才执行。模糊回复要追问

写入走 `memory` 工具 API（不是直接 `cat > file`），保留 hermes 的原子写入和限额检查。

### 安装

```bash
hermes skills install MaketheFoxRush/hermes-consolidate-memory/consolidate-memory
```

标识符格式是 hermes 的 `owner/repo/skill-name` 约定。hermes 会 fetch SKILL.md、跑安全扫描、装到 `~/.hermes/.hub/`。

或手动 clone：

```bash
git clone https://github.com/MaketheFoxRush/hermes-consolidate-memory.git
cp -r hermes-consolidate-memory/skills/consolidate-memory \
  ~/.hermes/skills/maintenance/
```

### 用法

直接对 Hermes 说：

> 整理一下你的长期记忆
> 压缩记忆
> consolidate memory

它会按 Step 1–7 走流程：备份 → 读取 → 分析 → 提案 → 等你确认 → 应用 → 验证 + 日志。

### 回滚

```bash
ls -lt ~/.hermes/memories/.backups/ | head
cp ~/.hermes/memories/.backups/MEMORY-<TS>.md ~/.hermes/memories/MEMORY.md
cp ~/.hermes/memories/.backups/USER-<TS>.md   ~/.hermes/memories/USER.md
systemctl restart hermes-gateway
```

### Roadmap

- [ ] 加 `hermes cron` 周期触发：每周生成提案推到你的消息通道，回 yes 才应用
- [ ] `--dry-run` 模式，写提案到文件不等用户回复（CI 用）
- [ ] 未来如果 hermes 加更多持久记忆文件，扩展支持

### License

MIT — 见 [LICENSE](./LICENSE)
