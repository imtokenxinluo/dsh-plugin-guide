# DSH 插件开发者守则

> **一句话**：写 DSH 插件，你的一次疏忽，代价由**别人的记忆**承担。
>
> 状态：draft v0.1 · 2026-09-13 · 本地草稿（未发布）
> 对象：任何给 DeepSeek Harness 写插件（cordis 插件 / bundle / 工具）的人
> 依据：DSH `0.1.0-rc.6` 安装版内核实读 + 我们亲历的三次事故 + 社区公开报告

---

## 0. 为什么要有这份文档

不是洁癖。以下代价都是真实发生过的：

| 事故 | 现象 | 谁受损 |
|---|---|---|
| `dsh-ssh-ops` 的 `sftp_*`/`tunnel_*` 工具 `render()` 返回裸字符串 | 多个会话报 `history unavailable … must contain one tool-result block`，**历史永久打不开** | 使用者的记忆 |
| 我们自己的 `dsh-session-doctor@0.2.0` 打包漏了 `cordis.patch.yml` | 安装后 **DSH 整个插件树拒绝启动**（bundle 机制 ENOENT） | 所有装它的人 |
| 第三方插件写入词汇表外的自定义事件类型 | 新版加载时 **整条日志被拒绝解释** | 使用者的记忆 |

共同点：**插件作者看不出问题**（本地跑得好好的），**用户看不出原因**（GUI 只报一句含糊的错），**伤害落在会话历史上，且当时不可逆**。

DSH 的架构决定了这个不对称：**写入路径宽松，读取路径严格，失败模式是"整条会话"而不是"这一条记录"。**

---

## 1. 五条硬契约（违反 = 损害他人数据）

### C1 · 工具的 `render()` 必须返回 `ContentBlock[]`

**症状**：会话某天突然报
`SessionPersistenceCorruptionError: session event at seq N message must contain one tool-result block`
→ 整条历史不可加载。

**根因**（已核验）：`@deepseek-ai/dsh-tools` 的 `createSuccessResult()` 里
`rendered = tool.output.render(...)` 的结果**被原样写入会话日志**，运行时**从不校验**它是不是数组。
而加载器要求 `tool-result` 块的 `content` 必须是数组。类型契约 `render(): ContentBlock[]`
只存在于 TypeScript 里——**内核自己的工具都遵守，所以内部永远测不出问题；第一次暴露必然来自第三方插件。**

**规矩**：
```js
// ✗ 会写坏别人的会话
output: { render: (args, value) => `已写入 ${value.bytes} 字节` }

// ✓
output: { render: (args, value) => [{ type: "text", text: `已写入 ${value.bytes} 字节` }] }
```
**每一个** `render()` 返回路径都要检查——包括提前 `return`、错误分支、`map`/`filter` 之后。
（`dsh-ssh-ops` 就是 9 个工具、15 处返回点里漏了几处。）

**自检**：`grep -n "render:" src/` 逐个看返回值，或跑一份静态审计（见 §5）。

---

### C2 · 消息事件的形状必须与加载器一致

**根因**（已核验）：加载路径 `assertMessageEventShape` 对消息类事件逐项校验；不匹配即拒载整条日志。
（写路径在未打补丁的内核上**不校验**——所以你本地怎么试都正常。）

| 事件 | 必须满足 |
|---|---|
| `user/message` | `data.content` 是数组；`data.role === "user"`；`data.source.kind` 是非空字符串；`data.id` 非空 |
| `assistant/message` | 同上但 `role === "assistant"`；**`source.kind === "model"` 且带非空 `provider` / `model`** |
| `tool/result` | `role === "user"`；`source.kind === "tool"` 且 `source.callId` 非空；**`message.content` 长度恰为 1**；`content[0].type === "tool-result"`；**`content[0].content` 必须是数组**；`content[0].toolCallId === source.callId` |
| `request/header` | `data.header.config` 带非空 `provider` / `model`；`reasoningEffort` 若存在须为非空字符串 |

**规矩**：**不要手搓消息事件**。要产出工具结果，就让 `render()` 干（C1）；要发消息，让框架干。

---

### C3 · 只写词汇表内的事件类型——绝不要自己发明事件名

**这是最容易被忽略、后果最重的一条。**

**根因**（已核验的内核事实）：
- 加载器只认识 `KNOWN_SESSION_EVENT_TYPES` 词汇表里的事件类型；遇到表外类型**直接拒绝解释整条日志**：
  > `session "…" contains event type "…" (seq N) unknown to this harness and not marked ignorable; refusing to interpret the log — it was likely written by a newer harness`
  判定就一行：`if (KNOWN_SESSION_EVENT_TYPES.has(event.type) || event.ignorable === true) continue;`
  （`dsh-session-persistence/lib/index.js`，0.1.0-rc.6 第 1119 行）
- 唯一的豁免是在事件信封上标 `ignorable: true`。
- **但 `Session.append(type, data, ...opts)` 的 opts 只接受 `sourceEventSeqs` 和 `surfaceOp`**——事件信封由内核构造，只有 `type/seq/time/data` + surface 元数据，**插件没有设置 `ignorable` 的入口**。
- 更硬的一条（已核验）：**整个内核里不存在任何把 `ignorable` 置为 `true` 的赋值**。这个概念只有读取方在用，没有写入方。也就是说在 0.1.0-rc.6 上，**没有任何事件会被标成 ignorable**——表外类型 = 必然拒载。

**结论**：`ctx.session.append("my-plugin/whatever", …)` 会让**使用者的会话在新版 DSH 上无法加载**，而且你无法自救。

**规矩**：
1. 优先**不写自定义事件**——把状态存在自己的 `storageDomain` / 自己的文件里。
2. 确实需要写会话事件时，只用词汇表内的类型（本版本共 **44 个**，含 `todo/write`、`goal/change`、`session/title` 等）。
3. 不要指望"以后内核会加 `ignorable` 支持"——**在你亲手验证它存在之前，就当它不存在。**

<details>
<summary>词汇表全量（0.1.0-rc.6，用于核对）</summary>

`agent-preset/selected` `agent/inbox/spliced` `approval/asked` `approval/decided`
`approval/policy` `assistant/chunk` `assistant/message` `command/done` `command/run`
`compaction/end` `compaction/prune` `compaction/start` `compaction/summary`
`feedback/record` `goal/change` `hook/invoked` `hook/result` `llm/retry`
`llm/retry-started` `permission/preset` `plan/mode` `request/context` `request/header`
`sandbox/mode` `schedule/change` `session/end-seed` `session/title`
`session/title-llm-request` `step/end` `step/start` `subagent/descriptor` `todo/write`
`tool-workflow/agent-end` `tool-workflow/agent-start` `tool-workflow/run-end`
`tool-workflow/run-start` `tool/call` `tool/code-dispatch` `tool/code-dispatch-start`
`tool/result` `turn/end` `turn/start` `user/message` `web/deepseek-search-llm-request`

</details>

**自检**：`grep -rn "\.append(" src/` 看有没有非词汇表的第一参数。

---

### C4 · npm 打包白名单必须覆盖 `package.json` 声明的一切

**症状**：安装后 **DSH 整个插件树拒绝启动**（不是你这一个插件失效，是**全部插件**都起不来）。

**根因**（我们自己的事故）：`package.json` 声明了
```json
"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
```
但 `files` 白名单是 `["src","bin","README.md"]`——**漏了 `cordis.patch.yml`**。
npm 按白名单过滤打包 → 包里没这个文件 → bundle 机制启动时必须读它 → `ENOENT` → 插件树整体拒启。

**规矩**：`files` 是白名单，**声明了什么就必须打包什么**。逐一核对：

- [ ] `main` / `exports` 指向的文件
- [ ] `bin` 指向的文件
- [ ] `dsh.bundle.patch` 指向的文件 ← **最容易漏**
- [ ] `dsh.client` 的入口（若为 client 插件）
- [ ] 运行期真的会 `import`/`readFile` 的文件（含运行时模板、`cordis.patch.yml`、`skills/` 目录）

**自检（发布前必跑）**：
```bash
npm pack --dry-run          # 看清单里有没有上面每一项
```
再狠一点：解包后按 `package.json` 的声明逐个 `existsSync`。

---

### C5 · 不要杀宿主进程，也不要自杀式重启

**症状**：用户的会话中断、GUI 掉线，最坏情况是运行中的会话被腰斩（日志被截断）。

**我们亲历的真实案例**：一个会话为了"重启 harness 加载新代码"，自己拼了一条
`Get-CimInstance … | Where-Object { $_.CommandLine -match 'dsh' } | Stop-Process -Force` 再 `Start-Process`——
**它把自己所在的进程杀了**，会话死了 2 分 20 秒，`turn/end` 记的是 `interrupted`。

**规矩**：
- 插件**绝不**杀/重启宿主进程。要重启就提供**明确的、用户触发的**通道（按钮/命令），并且**先起新进程、再退旧进程**。
- 进程匹配**不要用宽泛关键词**（`dsh`、`node`）——那会连自己和无关进程一起杀。
- 工具作者注意：**给 agent 一个安全的重启工具**，比让它自己发挥要好得多（它不会总是选对）。没有安全通道时，agent 会退化成手搓 `Stop-Process`。

---

## 2. 其它已知坑

| 坑 | 症状 | 规矩 |
|---|---|---|
| **消息配对** | 会话报 `400 INVALID_REQUEST` 崩溃 | 改写/注入消息时，`tool_calls` 与对应 tool 结果必须成对；不要只删一半（社区报告：`Leeminjing/dsh-messages-sanitizer`） |
| **typert 命名导出** | `typert-loader: … exports "./typert" but its module has no TYPERT manifest object` | 必须 `export { TYPERT }`（**命名导出**），不是 `export default TYPERT` |
| **profile patch 与 lockfile 不一致** | `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH` | 删/改 `patchedDependencies` 时必须同步改 `pnpm-lock.yaml`（配置里是路径，lockfile 里是 hash） |
| **供应链延迟** | 刚发布的版本装不上 | pnpm 有最小发布龄保护；新版本需在 `minimumReleaseAgeExclude` 里放行 |
| **`!!js` 配置求值** | 配置里带 `!!js` 会在加载时执行代码 | 你不是配置的**唯一**读者时，别在配置里塞可执行表达式 |
| **工具改名/删除** | 会话中断后恢复时，对已不存在的工具会报 `unknown tool "…"`（我们实测过）| 改名/卸载要当**破坏性变更**：正在执行的工具调用会失去落点；变更日志里写清楚 |

---

## 3. 信任姿态（写之前先想清楚）

**cordis 插件是进程内代码。** DSH 自带的"创造模式"讲得很直白：

> `cordis_mount` evaluates model-written JavaScript against the live runtime … **Treat a session on this preset as shell access.**

也就是说：**装一个插件 ≈ 给作者一个 shell**。所以：

- **默认只读**。写文件、发网络请求、起子进程，都要在 README 里明说。
- **零依赖优先**。每个依赖都是你替用户引入的新信任面。（我们的 doctor/backup 都是零运行时依赖。）
- **不要偷偷扩权**。新增一个网络端点、一个 shell 调用、一个后台定时器，都算能力扩张，要写在变更日志里。
- **你的插件会读到的内容，都是用户的隐私**（会话历史、路径、密钥所在的环境变量）。读之前先问"我真的需要吗"。

---

## 4. 发布前自检清单

```bash
# 1. 打包内容是否覆盖全部声明（C4）
npm pack --dry-run

# 2. render 返回值审计（C1）——对照下面这份脚本思路
grep -rn "render:" src/

# 3. 是否写了词汇表外的事件类型（C3）
grep -rn "\.append(" src/

# 4. 是否有杀进程 / 宽泛匹配（C5）
grep -rniE "taskkill|Stop-Process|pkill|killall|Stop-Service" src/

# 5. 是否有意外的网络/子进程（§3）
grep -rniE "fetch\(|https?://|child_process|spawn\(|exec\(" src/
```

- [ ] `files` 覆盖 `main`/`exports`/`bin`/`dsh.bundle.patch`/`dsh.client`（C4）
- [ ] 每个 `render()` 返回路径都是 `ContentBlock[]`（C1）
- [ ] 没有手搓消息事件（C2）
- [ ] 没有词汇表外的事件类型（C3）
- [ ] 没有杀宿主进程、没有宽泛进程匹配（C5）
- [ ] 测试能跑（`npm test`），且**测的是契约**，不只是 happy path
- [ ] README 写清：碰什么（文件/网络/进程）、不碰什么、边界在哪
- [ ] 装进一个干净 profile 验一遍：**能启动、能卸载、卸载后不留垃圾**

---

## 5. 范例：什么叫"做对了"

我们自己在维护的两个插件可以当参照（不是让你抄功能，是抄**姿态**）：

- **`dsh-session-doctor`** —— 零运行时依赖；**默认只读**；`repair` 只修**一种已确认**的损坏，其它一律报告不修；改文件前必备份、重写先写临时文件、产物复验通过才落盘；每条边界（比如"watch 不阻止写入"）都如实写在 README 里。
- **`dsh-session-backup`** —— 只做备份/还原，不做分析；manifest 带 sha256；还原前校验。

共同点：**能力小、边界清、可验证、不越权。**

---

## 6. 出处与状态（哪些是核验过的）

| 结论 | 来源 |
|---|---|
| render 契约不校验、写路径宽松读路径严格 | 内核实读（`dsh-tools` / `dsh-session` lib）+ 我们应用过的 P0-1/P0-3 补丁 |
| 消息事件形状各项要求 | 内核 `assertMessageEventShape` 的镜像实现（`dsh-session-doctor/src/validate.js`） |
| 事件类型词汇表 + 表外类型拒载 + `append` 无 `ignorable` | 内核实读（`KNOWN_SESSION_EVENT_TYPES`、persistence 报错点、`Session.append` 签名） |
| bundle patch 漏文件导致插件树拒启 | 我们自己的 0.2.0 事故 + 0.2.1 修复 |
| 自杀式重启 | 真实会话日志（2026-09-06，`scheduled-run-3fd818d1`） |
| 消息配对 / typert / 供应链延迟 | 社区公开报告与本机踩坑记录，**标注为社区经验，非内核核验** |

**版本敏感性**：以上针对 `0.1.0-rc.6`。**升级 DSH 后请重新核验**——词汇表、校验函数、`append` 签名都可能变。这份文档的价值在于"**去哪查、怎么查**"，而不只是结论。

---

## 附：为什么这份文档自己不推广

它不带你装任何东西、不指向任何第三方链接、不需要你关注谁。
**如果它有用，它会因为"能挡住一次事故"被引用；如果没用，推它也没用。**
