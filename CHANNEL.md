# 渠道契约 —— 让任何渠道接上 persona-hub

> persona-hub 是**渠道无关的人格层**：它管「谁在演哪个人格、记得什么、怎么说话」，
> **不管**「消息怎么从平台上收上来、怎么发出去」。收发是渠道插件的事。
>
> 所以你不必自己写连接层：选一个现成的渠道插件，按下面三件事把线连上即可。
> 本机现在用它的是 `dsh-onebot-qq`（QQ/OneBot 11 + NapCat）。

---

## 总览

```
        渠道插件（任何平台）                        persona-hub（人格层）
   ┌────────────────────────────┐          ┌────────────────────────────────┐
   │ 收消息 → 找到/新建会话      │          │                                │
   │ 会话 id = <渠道>-private-<号>│  ────▶   │ 按会话 id 认人：注入人格正文、  │
   │        = <渠道>-group-<号>   │  契约 1   │ 四层记忆、情绪、作息、身份档位  │
   ├────────────────────────────┤  契约 2   ├────────────────────────────────┤
   │ 要发的东西先问"怎么发"      │          │ planSpeech()：去旁白 / 分行 /   │
   │ 拿到 bubbles 再逐条发出     │  ◀────   │ 最多 3 条 / 每条等多久（契约 3）│
   └────────────────────────────┘          └────────────────────────────────┘
```

三件事：**① 会话怎么命名**、**② 消息怎么进来**、**③ 话怎么发出去**。

---

## 契约 1 · 会话 id（必须遵守）

渠道给每个聊天建一个**持久会话**，id 必须是：

```
<渠道名>-private-<对方标识>      单聊
<渠道名>-group-<群标识>          群聊
```

正则（persona-hub 内部就是这条）：`/^(.*?)-(private|group)-(.+)$/`

| 会认 ✓ | 不认 ✗ |
|---|---|
| `onebot-private-10001` | `session-1234`（没有 private/group 段） |
| `wechat-private-alice` | `onebot-private-`（标识为空） |
| `discord-group-1234567890` | `private-123`（缺渠道名） |
| `my-channel-private-中文昵称` | `onebot-channel-9`（channel 不是 private/group） |

- `<渠道名>` **随便叫什么**（`onebot` / `wechat` / `discord` / 你自己的名字都行），persona-hub 不挑。
- `<标识>` 里可以有下划线、中文，**不要有 `:`**（`:` 是 persona-hub 内部 chatId 的分隔符）。
- 一个聊天**永远用同一个 id**：重启、换天、换模型都不许新建，否则她会"失忆"。
- DSH 的 `/new` 会给会话文件加后缀（`<id>-2.json`），persona-hub 找会话时会自动带上——渠道不用管。
- 已验证：7 种渠道前缀 + 6 个反例全过；真实文件系统里 `wechat-private-alice.json` 也能被准确定位
  （`node tools/test-channel-agnostic.mjs` → **24 项全过**）。

## 契约 2 · 入站（渠道几乎不用做）

渠道只要把用户消息投递给**契约 1 那个会话**（DSH 的 agent/session），就完事了：
人格注入、记忆召回、情绪、作息、身份档位，persona-hub 会在 agent 组装时**按会话 id 自己完成**。

渠道唯一要注意的：**群聊里"要不要理这条"由渠道决定**（被 @、被引用、白名单…），
persona-hub 不参与唤醒判断——它只负责"醒了之后她是谁、她记得什么"。

## 契约 3 · 出站说话（推荐接，接了就获得"她的说话样子"）

渠道在**发送正文前**问 persona-hub 一句：分几条、每条什么内容、每条之前等多久。

```js
// 渠道插件里（发送路径上，必须在 async 函数内）
const hub = this.ctx?.personaHub;                        // 同进程插件服务
let plan = null;
if (hub !== undefined && typeof hub.planSpeech === 'function') {
  try {
    const got = await hub.planSpeech({ text: plainText });   // ⚠ 必须 await（它是 async）
    if (got?.bubbles?.length > 0) plan = got;
  } catch { /* 忽略：下面会退回你自己的逻辑 */ }
}

if (plan !== null) {
  for (let i = 0; i < plan.bubbles.length; i++) {
    await sleep(plan.bubbles[i].delayMs);                 // 每条之前等这么久（她在"打字"）
    await send(chatId, plan.bubbles[i].text);             // 发这一条
  }
} else {
  await send(chatId, plainText);                          // 退回你的老逻辑
}
```

返回结构：

```js
{
  plain: '去掉旁白后的整段',
  bubbles: [ { text: '第一条', delayMs: 2510 }, { text: '第二条', delayMs: 2750 } ],
  dropped: 0        // 因超过气泡上限被并进最后一条的行数
}
```

**要点**

- **必须 await**：`planSpeech` 是 async，忘了 await 会拿到 Promise → 看起来"接上了"但永远走回退分支
  （不报错，静默失效）。
- **fail-open**：persona-hub 没装/报错时，渠道要能自己发出去。**绝不能让她的消息被卡住**。
- **分工**：persona-hub 决定"内容与节奏"，渠道保留"markdown 剥离、正在输入提示、真正 sleep、
  被打断就停、图片卡片、合并转发"这些传输的事。
- **想改说话方式**：改 persona-hub 的偏好（`prefs.speechMaxBubbles` / `speechTypingMsPerChar` /
  `speechLineGapMs` / `speechMaxGapMs`），不必动第三方插件。
  默认值 = `3 / 120 / 1800 / 12000`（与 dsh-onebot-qq 线上配置一致，所以行为不变）。
- 与 onebot 原实现**逐条等价**（`node tools/test-speech.mjs` → 22 项全过）。

## 可选 · 「她能做的事」清单

人格提示词里有一段 `<你能做的事>`，写的是她**手里有哪些工具**（发图/语音/拍一拍/发动态…）。
默认那段是 QQ（`qq_*`）工具名。

- 渠道工具名不一样时：在 persona-hub 偏好里写 `handsText`，**整段替换**成你这个渠道的工具说明。
  ```json
  { "prefs": { "handsText": "· send_image —— 发图\n· send_voice —— 发语音" } }
  ```
- 只想**关掉**其中几项：面板里已有开关（`handsOff`）。
- 不改也不会坏：她只是"以为"自己有那几个工具；真调用了会失败。

## 绑定与身份（面板里配）

- **机器人账号**：一个 bot 号可以挂多个人格，人格可以绑到指定账号/群（面板「账号」与「绑定」）。
- 单聊里"谁是主人"：面板里设 `ownerQq`（或 `prefs.ownerQq`）；非主人时她会留在礼貌距离。
- 渠道不需要做任何事——只要会话 id（契约 1）对得上。

---

## 三步自查（接完渠道后跑）

```bash
# ① 人格层本身没问题
node test/run.mjs                       # 应打印 all suites passed（28 / 34 / 12）
# ② 渠道无关性
node tools/test-channel-agnostic.mjs    # 应打印 24 项通过
# ③ 说话接线（如果用 onebot）
node tools/patch-all.mjs                # 应打印 21 项全在
```

现象自查：

| 现象 | 说明什么 |
|---|---|
| 她会说话，但**带 `（笑）`这类旁白**、一口气发八条 | 契约 3 没接（或忘了 await）—— 走的是渠道自己的老逻辑 |
| 她**不说话** | 契约 1 或 2 没对上：会话 id 格式不对，或消息没投进那个会话 |
| 她**换了个名字/性格** | 会话 id 变了（新建了会话）→ 她"重新自我介绍"；检查 id 是否稳定 |
| 面板「补丁体检」红 | 那是 QQ 补丁的事，与渠道契约无关；见 `PATCHES.md` |

## 各渠道现状（2026-10-16）

| 渠道插件 | 会话 id 是否已符合契约 | 出站契约 3 | 备注 |
|---|---|---|---|
| **`dsh-onebot-qq`（本机在用）** | ✓ `onebot-private-<qq>` | ✓ 已接（`patch-onebot-speech.mjs`） | 21 项补丁，见 `PATCHES.md` |
| `dsh-1bot` | 需确认（它自称每聊天一个 agent+session） | ✗ 需自己加一段（5 行左右） | 工具面简（5 个） |
| `dsh-onebot-hub` | 需确认 | ✗ 需自己加一段 | 自带记忆层，会和 persona-hub 重叠，注意取舍 |
| `@xmanrui/dsh-im` | 需确认 | ✗ 需自己加一段 | QQ 走官方机器人（不能发动态/拍一拍） |

> 「需确认」= 我没读过它的源码，不能替它保证；按契约 1 对照它的 `sessionId` 生成代码即可。

## 边界（老实说）

- persona-hub **不提供**：平台连接、重连、媒体上传、图片卡片、命令解析——这些永远是渠道的事。
- **QQ 专属能力**（拍一拍 `qq_pat`、发动态 `qq_publish_moment`、群历史）只有提供这些工具的渠道才有；
  换平台时她的这段"手脚"说明要么替换（`handsText`），要么关掉（`handsOff`）。
- 出站钩子**不是 DSH 提供的**：DSH 只发 `session/event`（通知型，谁都能听、改不了别人发送）。
  所以"说话方式归人格层"靠的是渠道**主动调用** persona-hub —— 这就是为什么这份契约存在。
