# AI 辩论聊天室 v1 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 做出 v1 最小闭环——扔一个话题，主 agent 自动配齐 2-4 个角色，在网页群聊里由自研调度器轮流辩论，真人可插话，主持人收敛出结论卡。

**Architecture:** 混合底座。自研「辩论调度器」(纯逻辑状态机) 管发言权/轮次/真人插队；角色/主持人 agent 借力 Claude Agent SDK；前端网页群聊经 WebSocket 实时流式。

**Tech Stack:** TypeScript 全栈 · Next.js(前端) · Node + Fastify(后端) · ws(WebSocket) · Drizzle + SQLite(状态) · `@anthropic-ai/claude-agent-sdk`(角色层) · Vitest(测试) · 打分用 Haiku / 生成用 Sonnet

> 配套设计文档：`docs/plans/2026-06-26-ai-debate-chatroom-design.md`

---

## 里程碑总览

| 里程碑 | 内容 | 完成即可验证 |
|---|---|---|
| **M0** | 详细选型 + UI 设计系统 + 脚手架 + SDK spike | 空壳跑通、设计 token 落地、SDK 能起一个 agent |
| **M1** | 辩论调度器（举手→仲裁→单人发言，纯逻辑） | 喂模拟消息，调度器正确选出下一个发言者 |
| **M2** | 角色层（persona 注入 + 联网搜索 + 组队器） | 给话题能生成角色、角色能产出一条发言 |
| **M3** | 真人介入（最高优先级中断 + @点名） | 真人发言能打断调度、被相关角色优先回应 |
| **M4** | 主持人收敛（控场 + 结论卡） | 一场辩论能产出结构化结论卡 |
| **M5** | 前端群聊 UI（实时流式 + 真人输入） | 浏览器里完整跑一场可插话的辩论 |

> 每个里程碑结束 = 一次可运行、可验证的软件。建议按里程碑分批执行、之间 review。

---

## 文件结构（决定边界，先锁定）

```
chatroom/
├─ packages/
│  ├─ shared/                      # 前后端共享类型
│  │  └─ src/types.ts              # Room/Role/Message/ConclusionCard/打分结构
│  ├─ server/
│  │  ├─ src/
│  │  │  ├─ scheduler/
│  │  │  │  ├─ scheduler.ts        # 调度器状态机（M1 核心）
│  │  │  │  ├─ constraints.ts      # 硬约束（连发/冷却/配额/上限）
│  │  │  │  └─ scoring.ts          # 举手打分（调 Haiku）
│  │  │  ├─ agents/
│  │  │  │  ├─ roster.ts           # 组队器：话题→角色（M2）
│  │  │  │  ├─ role-agent.ts       # 角色发言（调 Sonnet + 搜索）
│  │  │  │  ├─ moderator.ts        # 主持人 + 结论卡（M4）
│  │  │  │  └─ persona-template.ts # persona 文档模板
│  │  │  ├─ db/schema.ts           # Drizzle schema
│  │  │  ├─ room.ts                # 房间编排：串起调度器+agent+广播
│  │  │  └─ ws.ts                  # WebSocket 服务
│  │  └─ test/                     # Vitest 测试
│  └─ web/                         # Next.js 前端（M5）
│     └─ src/
│        ├─ design/tokens.ts       # UI 设计系统 token（M0）
│        ├─ components/            # 消息气泡/角色侧栏/结论卡/输入框
│        └─ app/room/[id]/page.tsx
├─ docs/
│  └─ design-system.md             # UI 设计系统规范（M0 产出）
└─ docs/plans/...
```

---

## M0 — 地基：选型 + UI 设计系统 + 脚手架

### Task 0.1: monorepo 脚手架 + 工具链

**Files:**
- Create: `package.json`(根, pnpm workspace)、`pnpm-workspace.yaml`、`tsconfig.base.json`、`.gitignore`、`.env.example`
- Create: `packages/shared/package.json`、`packages/server/package.json`、`packages/web/package.json`

- [ ] **Step 1: 初始化 workspace**

```bash
corepack enable && pnpm init
printf 'packages:\n  - "packages/*"\n' > pnpm-workspace.yaml
mkdir -p packages/shared/src packages/server/src packages/web/src
```

- [ ] **Step 2: 装核心依赖**

```bash
pnpm add -w -D typescript vitest @types/node tsx
pnpm --filter server add fastify ws drizzle-orm better-sqlite3 @anthropic-ai/claude-agent-sdk
pnpm --filter server add -D @types/ws drizzle-kit
```

- [ ] **Step 3: 写 `.env.example`**

```
ANTHROPIC_API_KEY=sk-ant-xxx
SCORING_MODEL=claude-haiku-4-5-20251001
GENERATION_MODEL=claude-sonnet-4-6
DATABASE_URL=./data/chatroom.db
```

- [ ] **Step 4: 提交**

```bash
git add -A && git commit -m "chore: monorepo 脚手架 + 工具链"
```

### Task 0.2: 共享类型（锁定全局契约）

**Files:**
- Create: `packages/shared/src/types.ts`

- [ ] **Step 1: 写类型定义**（后续所有任务以此为准）

```typescript
export type RoomStatus = 'forming' | 'debating' | 'converged';
export interface Room { id: string; topic: string; status: RoomStatus; roleIds: string[]; createdAt: number; }
export interface Role { id: string; roomId: string; name: string; domain: string; stance: string; persona: string; }
export interface Message { id: string; roomId: string; round: number; speaker: { kind: 'role'; roleId: string } | { kind: 'human'; userId: string }; content: string; mentions: string[]; createdAt: number; }
// 举手打分
export interface SpeakIntent { roleId: string; want: boolean; urgency: number; relevance: number; }
export interface ConclusionCard { consensus: string[]; conflicts: { sides: string; about: string }[]; strongestArgs: { roleId: string; arg: string }[]; openQuestions: string[]; }
```

- [ ] **Step 2: 提交**

```bash
git add packages/shared && git commit -m "feat: 共享类型契约"
```

### Task 0.3: UI 设计系统规范（本里程碑核心产出）

> 用刚装的 taste-skill 思路：先定「设计 Read」，反默认审美，token 化。聊天室定位为「克制、可信、像认真讨论的圆桌」，不是花哨социал app。

**Files:**
- Create: `docs/design-system.md`、`packages/web/src/design/tokens.ts`

- [ ] **Step 1: 写 `docs/design-system.md`** —— 至少覆盖：
  - **设计 Read**：一句话定调（如「严肃话题的圆桌辩论，面向想认真思考的人，克制 editorial 语言，深色可读优先」）
  - **色彩 token**：背景/前景/各角色区分色（角色色板要可扩展到 4 个、彼此可辨、符合无障碍对比）/主持人色/真人色
  - **字体**：正文/角色名/结论卡标题的字号与字重阶梯
  - **间距/圆角/密度**：消息气泡密度、留白
  - **动效约束**：「正在输入…」圆点 + 逐字流式；角色一个接一个登场不同时弹；可调停顿
  - **反模板硬检查清单**：不许用默认紫色渐变、不许居中堆 emoji、角色头像要有真实区分

- [ ] **Step 2: 把 token 落到 `tokens.ts`**

```typescript
export const tokens = {
  color: { bg: '#0f1115', fg: '#e6e8eb', moderator: '#9aa0a6', human: '#4ea1ff',
           roles: ['#e8a13a', '#6fbf73', '#d96f6f', '#8a7fd6'] },
  font: { body: 15, roleName: 13, cardTitle: 18 },
  space: { bubbleY: 8, bubbleX: 12, gap: 16 }, radius: 12,
} as const;
```

- [ ] **Step 3: 提交**

```bash
git add docs/design-system.md packages/web/src/design && git commit -m "feat: UI 设计系统规范 + 设计 token"
```

### Task 0.4: Agent SDK Spike（验证可行性，别纸上谈兵）

**Files:**
- Create: `packages/server/src/agents/_spike.ts`

- [ ] **Step 1: 写一个最小 spike**：用 SDK 起一个 agent，给定 system prompt 与一个工具（web search 或 mock），让它回一句话；打印结果。

- [ ] **Step 2: 跑通**

Run: `pnpm --filter server tsx src/agents/_spike.ts`
Expected: 终端打印出 agent 的一句话回复，确认 SDK + API key 工作、能挂 system prompt（= persona）与工具。

- [ ] **Step 3: 在 `docs/design-system.md` 同目录记一条 `docs/sdk-notes.md`**：写下 SDK 实测能力与限制（能否自定义并发、流式回调形态、子 agent 起法）。后续 M2/M3 据此实现。

- [ ] **Step 4: 提交**

```bash
git add packages/server docs/sdk-notes.md && git commit -m "chore: Agent SDK 可行性 spike + 实测笔记"
```

**M0 验收**：`pnpm -r build` 通过；spike 能起 agent；设计 token 与规范文档落盘。

---

## M1 — 辩论调度器（核心，重点 TDD）

> 调度器是纯逻辑、不调 LLM 的部分要全 TDD。打分(scoring)调 LLM 的部分用接口隔离、测试里 mock。

### Task 1.1: 硬约束模块

**Files:**
- Create: `packages/server/src/scheduler/constraints.ts`
- Test: `packages/server/test/constraints.test.ts`

- [ ] **Step 1: 写失败测试**

```typescript
import { describe, it, expect } from 'vitest';
import { eligibleSpeakers } from '../src/scheduler/constraints';

const intents = [
  { roleId: 'a', want: true, urgency: 0.9, relevance: 0.9 },
  { roleId: 'b', want: true, urgency: 0.5, relevance: 0.8 },
];

describe('eligibleSpeakers', () => {
  it('排除刚发言的角色（防连发）', () => {
    const out = eligibleSpeakers(intents, { lastSpeaker: 'a', recent: ['a'], quota: {} });
    expect(out.map(i => i.roleId)).toEqual(['b']);
  });
  it('排除 want=false', () => {
    const out = eligibleSpeakers([{ roleId: 'c', want: false, urgency: 1, relevance: 1 }], { lastSpeaker: null, recent: [], quota: {} });
    expect(out).toHaveLength(0);
  });
  it('排除超配额角色', () => {
    const out = eligibleSpeakers(intents, { lastSpeaker: null, recent: [], quota: { a: 99 }, maxPerRole: 10 });
    expect(out.map(i => i.roleId)).toEqual(['b']);
  });
});
```

- [ ] **Step 2: 跑测试确认失败**

Run: `pnpm vitest run packages/server/test/constraints.test.ts`
Expected: FAIL（`eligibleSpeakers` is not defined）

- [ ] **Step 3: 最小实现**

```typescript
import type { SpeakIntent } from '@shared/types';
export interface SchedState { lastSpeaker: string | null; recent: string[]; quota: Record<string, number>; maxPerRole?: number; }
export function eligibleSpeakers(intents: SpeakIntent[], s: SchedState): SpeakIntent[] {
  const max = s.maxPerRole ?? 10;
  return intents.filter(i =>
    i.want &&
    i.roleId !== s.lastSpeaker &&
    (s.quota[i.roleId] ?? 0) < max
  );
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `pnpm vitest run packages/server/test/constraints.test.ts`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add packages/server/src/scheduler/constraints.ts packages/server/test/constraints.test.ts
git commit -m "feat(scheduler): 发言硬约束（防连发/配额/want）"
```

### Task 1.2: 仲裁选人（带冷却权重 + 确定性随机）

**Files:**
- Create: `packages/server/src/scheduler/scheduler.ts`
- Test: `packages/server/test/scheduler.test.ts`

- [ ] **Step 1: 写失败测试**

```typescript
import { describe, it, expect } from 'vitest';
import { pickSpeaker } from '../src/scheduler/scheduler';

describe('pickSpeaker', () => {
  it('选 (urgency×relevance) 最高的合格者', () => {
    const intents = [
      { roleId: 'a', want: true, urgency: 0.9, relevance: 0.9 }, // 0.81
      { roleId: 'b', want: true, urgency: 0.5, relevance: 0.5 }, // 0.25
    ];
    expect(pickSpeaker(intents, { lastSpeaker: null, recent: [], quota: {} }, () => 0).roleId).toBe('a');
  });
  it('全员不合格时返回 null（该停，交还真人）', () => {
    expect(pickSpeaker([{ roleId: 'a', want: false, urgency: 1, relevance: 1 }], { lastSpeaker: null, recent: [], quota: {} }, () => 0)).toBeNull();
  });
  it('冷却：recent 中的角色权重打折', () => {
    const intents = [
      { roleId: 'a', want: true, urgency: 0.8, relevance: 0.8 }, // 0.64 ×0.5 冷却 =0.32
      { roleId: 'b', want: true, urgency: 0.6, relevance: 0.7 }, // 0.42
    ];
    expect(pickSpeaker(intents, { lastSpeaker: null, recent: ['a'], quota: {} }, () => 0).roleId).toBe('b');
  });
});
```

- [ ] **Step 2: 跑测试确认失败**

Run: `pnpm vitest run packages/server/test/scheduler.test.ts`
Expected: FAIL

- [ ] **Step 3: 实现**（`rng` 注入以保证测试确定性）

```typescript
import type { SpeakIntent } from '@shared/types';
import { eligibleSpeakers, type SchedState } from './constraints';

export function pickSpeaker(intents: SpeakIntent[], s: SchedState, rng: () => number = Math.random): SpeakIntent | null {
  const elig = eligibleSpeakers(intents, s);
  if (elig.length === 0) return null;
  const scored = elig.map(i => {
    const base = i.urgency * i.relevance;
    const cooled = s.recent.includes(i.roleId) ? base * 0.5 : base;
    const jitter = cooled * 0.1 * rng(); // 高分接近时制造"抢话"
    return { i, score: cooled + jitter };
  });
  scored.sort((x, y) => y.score - x.score);
  return scored[0].i;
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `pnpm vitest run packages/server/test/scheduler.test.ts`
Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add packages/server/src/scheduler/scheduler.ts packages/server/test/scheduler.test.ts
git commit -m "feat(scheduler): 仲裁选人（urgency×relevance + 冷却 + 抢话随机）"
```

### Task 1.3: 连续 AI 上限 → 该停信号

**Files:**
- Modify: `packages/server/src/scheduler/scheduler.ts`
- Test: `packages/server/test/scheduler.test.ts`

- [ ] **Step 1: 追加失败测试**

```typescript
import { shouldYieldToHuman } from '../src/scheduler/scheduler';
it('连续 AI 发言达上限要交还真人', () => {
  expect(shouldYieldToHuman(4, 4)).toBe(true);
  expect(shouldYieldToHuman(2, 4)).toBe(false);
});
```

- [ ] **Step 2: 跑测试确认失败** — Run: `pnpm vitest run packages/server/test/scheduler.test.ts` → FAIL

- [ ] **Step 3: 实现**

```typescript
export function shouldYieldToHuman(consecutiveAi: number, maxConsecutiveAi: number): boolean {
  return consecutiveAi >= maxConsecutiveAi;
}
```

- [ ] **Step 4: 跑测试确认通过** — PASS

- [ ] **Step 5: 提交** — `git commit -m "feat(scheduler): 连续 AI 上限交还真人"`

### Task 1.4: 打分接口（隔离 LLM，便于 mock）

**Files:**
- Create: `packages/server/src/scheduler/scoring.ts`
- Test: `packages/server/test/scoring.test.ts`

- [ ] **Step 1: 写失败测试**（mock LLM 客户端）

```typescript
import { describe, it, expect, vi } from 'vitest';
import { scoreIntents } from '../src/scheduler/scoring';

it('对每个角色并行打分并归一化到 0-1', async () => {
  const fakeLlm = vi.fn().mockResolvedValue({ want: true, urgency: 0.8, relevance: 0.7 });
  const roles = [{ id: 'a' }, { id: 'b' }] as any;
  const out = await scoreIntents(roles, 'ctx', fakeLlm);
  expect(out).toHaveLength(2);
  expect(out[0]).toMatchObject({ roleId: 'a', want: true });
  expect(fakeLlm).toHaveBeenCalledTimes(2); // 并行各一次
});
```

- [ ] **Step 2: 跑测试确认失败** — FAIL

- [ ] **Step 3: 实现**（`scoreOne` 由外部注入，真实实现里调 Haiku）

```typescript
import type { Role, SpeakIntent } from '@shared/types';
export type ScoreFn = (role: Pick<Role, 'id'>, context: string) => Promise<Omit<SpeakIntent, 'roleId'>>;
export async function scoreIntents(roles: Pick<Role, 'id'>[], context: string, scoreOne: ScoreFn): Promise<SpeakIntent[]> {
  return Promise.all(roles.map(async r => ({ roleId: r.id, ...(await scoreOne(r, context)) })));
}
```

- [ ] **Step 4: 跑测试确认通过** — PASS

- [ ] **Step 5: 提交** — `git commit -m "feat(scheduler): 举手打分接口（LLM 可注入）"`

**M1 验收**：`pnpm vitest run` 全绿；调度器在给定意愿分下能确定性地选出下一个发言者或返回「该停」。

---

## M2 — 角色层（组队器 + persona + 发言）

### Task 2.1: persona 模板

**Files:**
- Create: `packages/server/src/agents/persona-template.ts`
- Test: `packages/server/test/persona.test.ts`

- [ ] **Step 1: 失败测试**

```typescript
import { buildPersona } from '../src/agents/persona-template';
it('persona 含身份/立场/语言风格/知识边界/不暴露AI', () => {
  const p = buildPersona({ name: '社会学家·宏观', domain: '社会学', stance: '生育是社会结构问题' } as any);
  for (const k of ['社会学', '生育是社会结构问题', '语言风格', '不知道', '不要说自己是 AI']) expect(p).toContain(k);
});
```

- [ ] **Step 2: 跑测试确认失败** — FAIL

- [ ] **Step 3: 实现**

```typescript
import type { Role } from '@shared/types';
export function buildPersona(r: Pick<Role, 'name' | 'domain' | 'stance'>): string {
  return [
    `你是「${r.name}」，从${r.domain}视角参与一场群聊辩论。`,
    `核心立场：${r.stance}。坚持你的学科立场，合理时反对他人，每次发言尽量带一个反论点。`,
    `语言风格：像真实的${r.domain}从业者，简洁、有据、不堆术语。`,
    `知识边界：明确你不知道什么，别不懂装懂。`,
    `规则：你是平等的参与者，不附和权威；先回应别人/真人的具体观点再表达；不要说自己是 AI；每条发言控制在 120 字内。`,
  ].join('\n');
}
```

- [ ] **Step 4: 跑测试确认通过** — PASS

- [ ] **Step 5: 提交** — `git commit -m "feat(agents): persona 模板"`

### Task 2.2: 组队器（话题→角色草案）

**Files:**
- Create: `packages/server/src/agents/roster.ts`
- Test: `packages/server/test/roster.test.ts`

- [ ] **Step 1: 失败测试**（LLM 注入 mock，校验数量裁剪 2-4）

```typescript
import { proposeRoster } from '../src/agents/roster';
it('裁剪到 2-4 个角色', async () => {
  const fake = async () => Array.from({ length: 9 }, (_, i) => ({ name: `r${i}`, domain: 'd', stance: 's' }));
  const out = await proposeRoster('女性该不该生孩子', { max: 4 }, fake);
  expect(out.length).toBeGreaterThanOrEqual(2);
  expect(out.length).toBeLessThanOrEqual(4);
});
```

- [ ] **Step 2: 跑测试确认失败** — FAIL

- [ ] **Step 3: 实现**

```typescript
export interface RoleDraft { name: string; domain: string; stance: string; }
export type RosterFn = (topic: string) => Promise<RoleDraft[]>;
export async function proposeRoster(topic: string, opts: { max: number }, gen: RosterFn): Promise<RoleDraft[]> {
  const drafts = await gen(topic);
  return drafts.slice(0, Math.max(2, Math.min(opts.max, 4)));
}
```

- [ ] **Step 4: 跑测试确认通过** — PASS

- [ ] **Step 5: 提交** — `git commit -m "feat(agents): 组队器（话题→角色，裁剪2-4）"`

### Task 2.3: 角色发言 + 联网搜索（接 SDK，集成测试）

**Files:**
- Create: `packages/server/src/agents/role-agent.ts`
- Test: `packages/server/test/role-agent.int.test.ts`（标 `it.skipIf(!process.env.ANTHROPIC_API_KEY)`）

- [ ] **Step 1: 实现 `generateUtterance(role, history)`**：用 Agent SDK 起角色 agent，system=persona，挂 web search 工具，输入最近 history，返回一条 ≤120 字发言。`scoreOne`(Haiku 打分) 也在此文件实现并导出，供 M1 scoring 注入。

- [ ] **Step 2: 集成测试**（有 key 才跑）

```typescript
import { generateUtterance } from '../src/agents/role-agent';
it.skipIf(!process.env.ANTHROPIC_API_KEY)('角色能产出一条发言', async () => {
  const role = { id: 'a', name: '经济学家', domain: '经济学', stance: '养育成本高', persona: '...' } as any;
  const text = await generateUtterance(role, []);
  expect(text.length).toBeGreaterThan(0);
}, 30_000);
```

- [ ] **Step 3: 跑** — Run: `ANTHROPIC_API_KEY=... pnpm vitest run packages/server/test/role-agent.int.test.ts` → PASS（无 key 则 skip）

- [ ] **Step 4: 提交** — `git commit -m "feat(agents): 角色发言 + Haiku 打分（接 SDK）"`

**M2 验收**：给定话题能生成 2-4 角色草案；单个角色能产出一条发言（有 key 时）。

---

## M3 — 真人介入 + 房间编排

### Task 3.1: @点名解析

**Files:**
- Create: `packages/server/src/agents/mentions.ts`
- Test: `packages/server/test/mentions.test.ts`

- [ ] **Step 1: 失败测试**

```typescript
import { parseMentions } from '../src/agents/mentions';
it('解析 @角色名', () => {
  expect(parseMentions('@经济学家 你怎么算成本', [{ id: 'a', name: '经济学家' }] as any)).toEqual(['a']);
  expect(parseMentions('随便说说', [{ id: 'a', name: '经济学家' }] as any)).toEqual([]);
});
```

- [ ] **Step 2: 跑测试确认失败** — FAIL
- [ ] **Step 3: 实现**

```typescript
import type { Role } from '@shared/types';
export function parseMentions(text: string, roles: Pick<Role, 'id' | 'name'>[]): string[] {
  return roles.filter(r => text.includes('@' + r.name)).map(r => r.id);
}
```

- [ ] **Step 4: 跑测试确认通过** — PASS
- [ ] **Step 5: 提交** — `git commit -m "feat(agents): @点名解析"`

### Task 3.2: 房间编排循环（真人中断 + 调度推进）

**Files:**
- Create: `packages/server/src/room.ts`、`packages/server/src/db/schema.ts`
- Test: `packages/server/test/room.test.ts`（调度/agent 全 mock，验证编排逻辑）

- [ ] **Step 1: 失败测试**——验证三条编排不变量：
  1. 真人发言入列后，下一轮打分上下文包含该发言、且相关角色 relevance 被抬高
  2. `pickSpeaker` 返回 null 或 `shouldYieldToHuman` 为真时，循环停下等真人
  3. 真人消息到达时正在进行的 AI 轮被标记中断、不再广播

```typescript
import { RoomEngine } from '../src/room';
it('真人发言打断 AI 轮并触发重新打分', async () => {
  const engine = new RoomEngine(/* 注入 mock scheduler/agents */);
  await engine.start();
  engine.onHumanMessage({ content: '@社会学家 我觉得是个人选择', userId: 'u1' });
  expect(engine.lastRescoreContext()).toContain('个人选择');
  expect(engine.interruptedCurrentAiTurn()).toBe(true);
});
```

- [ ] **Step 2: 跑测试确认失败** — FAIL
- [ ] **Step 3: 实现 `RoomEngine`**：维护 SchedState/consecutiveAi/round；`tick()` = 打分→pickSpeaker→(null/yield 则停)→生成→广播→自增；`onHumanMessage()` = 入列、置中断标志、清 consecutiveAi、相关角色优先、触发重新 tick。DB schema 用 Drizzle 定义 Room/Role/Message/ConclusionCard 表。
- [ ] **Step 4: 跑测试确认通过** — PASS
- [ ] **Step 5: 提交** — `git commit -m "feat(server): 房间编排循环 + 真人中断 + DB schema"`

**M3 验收**：单元层面，真人发言能打断 AI 轮、抬高相关角色、触发重打分；连续 AI 到上限会停。

---

## M4 — 主持人收敛（结论卡）

### Task 4.1: 结论卡生成

**Files:**
- Create: `packages/server/src/agents/moderator.ts`
- Test: `packages/server/test/moderator.test.ts`

- [ ] **Step 1: 失败测试**（LLM mock，校验结构完整）

```typescript
import { summarize } from '../src/agents/moderator';
it('产出含共识/分歧/最强论据/待验证的结论卡', async () => {
  const fake = async () => ({ consensus: ['x'], conflicts: [{ sides: 'a vs b', about: 'y' }], strongestArgs: [{ roleId: 'a', arg: 'z' }], openQuestions: ['q'] });
  const card = await summarize([], fake);
  for (const k of ['consensus', 'conflicts', 'strongestArgs', 'openQuestions']) expect(card).toHaveProperty(k);
});
```

- [ ] **Step 2: 跑测试确认失败** — FAIL
- [ ] **Step 3: 实现**：`summarize(history, gen)` 调主持人 agent（Sonnet），prompt 强调「找矛盾才是最有价值的输出、每个判断附反面论据、不输出正确的废话」，返回 `ConclusionCard`。`gen` 注入便于测试。
- [ ] **Step 4: 跑测试确认通过** — PASS
- [ ] **Step 5: 提交** — `git commit -m "feat(agents): 主持人结论卡（求同存异）"`

### Task 4.2: 收尾触发接入房间

**Files:**
- Modify: `packages/server/src/room.ts`
- Test: `packages/server/test/room.test.ts`

- [ ] **Step 1: 追加失败测试**：`engine.requestSummary()` 产出 ConclusionCard 并把房间状态置 `converged`。
- [ ] **Step 2: 跑测试确认失败** — FAIL
- [ ] **Step 3: 实现**：`requestSummary()` 调 `summarize`，写 DB，广播结论卡，状态→converged。
- [ ] **Step 4: 跑测试确认通过** — PASS
- [ ] **Step 5: 提交** — `git commit -m "feat(server): 收尾触发→结论卡→converged"`

**M4 验收**：一段 mock 历史能产出结构化结论卡，房间正确收敛。

---

## M5 — 前端群聊 UI（端到端跑通）

### Task 5.1: WebSocket 服务 + 前端连接

**Files:**
- Create: `packages/server/src/ws.ts`、`packages/web/src/app/room/[id]/page.tsx`、`packages/web/src/lib/socket.ts`

- [ ] **Step 1: 实现 ws 服务**：房间事件（新消息/正在输入/结论卡）广播；接收真人消息转 `engine.onHumanMessage`。
- [ ] **Step 2: 前端建立连接**，订阅房间，收到消息追加到列表，逐字渲染流式增量。
- [ ] **Step 3: 手动验证** — Run: `pnpm --filter server dev` + `pnpm --filter web dev`，浏览器开房间，看到 AI 消息实时流入。
- [ ] **Step 4: 提交** — `git commit -m "feat: WebSocket 实时通道 + 前端订阅"`

### Task 5.2: 消息流 + 角色侧栏 + 真人输入（用 M0 设计 token）

**Files:**
- Create: `packages/web/src/components/{MessageBubble,RoleSidebar,Composer,TypingDots}.tsx`

- [ ] **Step 1: 实现组件**，严格用 `tokens.ts`：角色按色板区分、主持人/真人专色、「正在输入…」圆点、角色逐条登场不同时弹、Composer 支持 `@`。对照 `docs/design-system.md` 的反模板硬检查清单自查。
- [ ] **Step 2: 手动验证**：发一条真人消息，看到被相关角色优先回应、且回应先引用了你的话。
- [ ] **Step 3: 提交** — `git commit -m "feat(web): 消息流/角色侧栏/真人输入（应用设计系统）"`

### Task 5.3: 建房流程 + 结论卡渲染（闭环）

**Files:**
- Create: `packages/web/src/app/page.tsx`(输话题→看角色草案→微调→建房)、`packages/web/src/components/ConclusionCard.tsx`

- [ ] **Step 1: 实现建房页**：输话题 → 调组队器看草案 → 增删改 → 确认建房（盲写立场后进群）。
- [ ] **Step 2: 实现结论卡组件**：独立卡片渲染共识/分歧/最强论据/待验证；提供「总结一下」按钮触发 `requestSummary`。
- [ ] **Step 3: 端到端手动验证**：输入「女性该不该生孩子」→ 配齐 3-4 角色 → 看他们辩论 → 插一句话 → 点总结 → 拿到结论卡。
- [ ] **Step 4: 提交** — `git commit -m "feat(web): 建房流程 + 结论卡（v1 闭环）"`

**M5 验收（= v1 验收）**：浏览器里完整跑一场——自动组队、实时辩论、真人插话被搭理、求同存异结论卡。

---

## Self-Review 备注

- **Spec 覆盖**：调度器(M1)/角色+组队(M2)/真人介入(M3)/主持人收敛(M4)/前端(M5)/UI设计系统+选型(M0) 对应设计文档 §1-§5，全覆盖。
- **二期不在本计划**：find-skill 动态装配、私有 skill 库、跨会话记忆、事实核查、异构模型 —— 按设计文档 §5 推迟。
- **类型一致性**：`SpeakIntent`/`SchedState`/`Role`/`ConclusionCard` 在 M0 Task 0.2 锁定，后续任务引用同名签名（`pickSpeaker`/`eligibleSpeakers`/`scoreIntents`/`generateUtterance`/`summarize`/`onHumanMessage`/`requestSummary`）。
- **成本控制贯穿**：打分用 Haiku、生成用 Sonnet、每条发言 ≤120 字、只有中选者调大模型 —— 在 0.3 env、2.1 persona、2.3 实现处落实。
