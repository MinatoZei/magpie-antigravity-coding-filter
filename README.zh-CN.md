# magpie-antigravity-coding-filter

[English](README.md) · **简体中文**

> **这是 [NKBaa/magpie-antigravity-coding-filter](https://github.com/NKBaa/magpie-antigravity-coding-filter) 的 fork，**基于 `a0acaac`（v0.5.3），版本号 `0.5.3-agybridge.1`。原作者 NKBaa（MIT）；该项目移植自 [jellyfish-p/cpa-plugin-antigravity-coding-filter](https://github.com/jellyfish-p/cpa-plugin-antigravity-coding-filter)（MIT）。
>
> 这个 fork 改了一件事：让插件在 [Antigravity Bridge](https://github.com/DylanLIiii/opencode-antigravity-bridge) 上真正生效。原版对 Bridge 不起作用，而且不报任何错。

## 本 fork 修正了什么

| # | 原版的问题 | 修正 | 代码位置 |
|---|---|---|---|
| 1 | **对 Antigravity Bridge 不生效，也不报错。** 原版只看 `ctx.provider.account` 是不是 `antigravity`，而 magpie 0.1.4309 给所有插件 provider 的 account 都是 `"plugin"`，所以永远对不上 | 新增选项 `providers`，默认 `["antigravity-bridge"]`，按 provider id 匹配，不区分大小写 | `providerAllowed()`，`onProvider` |
| 2 | **和 Bridge 自带的 prompt-guard 冲突。** 本插件先执行，会改掉 Bridge 要原样匹配的两句 Claude 身份句。Bridge 认不出来，结果变成半遮半露，例如 `You are a C​l​aude agent, built on the Claude Agent SDK by Anthropic.` | 发往 Bridge 时这两句原样保留，交给 Bridge 处理 | `bridgeSentences`，`protectedSpans()` |
| 3 | Responses API 里字符串形式的 `input` 其实是用户这一轮的消息，原版也会改它 | 不再改它，只改系统提示词 | `input` 分支 |
| 4 | 预设缺了几个常见的编码 CLI | 加入 `Cursor Agent`、`Cursor CLI`、`Kiro CLI`、`Kimi CLI`、`iFlow CLI`，共 53 个名称。都是两个词，`cursor` 这样的普通单词不受影响 | preset，`page/filter.js` |
| 5 | `report` 只写进 `ctx.state`，magpie 从不读它，打开了也看不到结果 | 另外往 magpie 日志写一行，只记录命中的名字和次数，不含提示词，每分钟最多 60 行 | `report()` |
| 6 | 默认 `mode: "cloak"`，也就是插入零宽字符。据 magpie 官方 antigravity-cloak 的说明，2026-11-14 起 Google 的检查会忽略零宽字符 | 默认改为 `rewrite`，直接替换名字 | `package.json` |
| 7 | 新选项没有登记，manifest 测试会失败 | 测试的已知选项里加上 `providers` | `test/page-files.test.mjs` |

**验证环境**：magpie 0.1.4309 + Antigravity Bridge `ef58846`，Linux，请求经 magpie 网关发出。

| 测试 | 结果 |
|---|---|
| 发往 Antigravity 的请求 | 系统提示词里的名字被替换，日志里是 `changed:true` |
| 同样的请求发往 Claude 订阅 | 原样不动，没有日志行 |
| 用户消息里出现这个名字 | 不改，日志里是 `changed:false` |
| 系统提示词里有 Bridge 那两句身份句 | 原样交给 Bridge；本插件只替换了其中的 `Claude Code` |

`bun test` 跑了 unified-rules、page-files、page-smoke、features 和 middleware check，89 个全部通过。需要浏览器或 magpie 进程的测试没有跑。

## 推荐设置（页面里勾哪些）

装好后，打开 magpie 侧边栏的 **Plugins → Antigravity 编码过滤**，在「过滤设置」里设置。也可以用命令行：`magpie plugin options <插件名> '<json>'`。

**基本设置**

| 选项 | 勾不勾 | 说明 |
|---|---|---|
| 启用过滤 | ✅ 勾 | |
| 处理方式 | 选 **「替换名字」** | 不要选「保留原文」（零宽字符），见上面第 6 条 |
| 常用名称（53 个） | ✅ 勾 | |
| 同时处理模型名称 | ⬜ 不勾 | 会把 `claude-opus-5-5` 这类模型名一起改掉。名字都替换了还是 429，再试着打开 |

**高级设置**

| 选项 | 勾不勾 | 说明 |
|---|---|---|
| 只匹配完整名称 | ✅ 勾 | 避免误伤 `decline`、`Traefik` 这类普通单词 |
| 保护代码块和行内代码 | ✅ 勾 | 代码里的字符串保持原样 |
| 启用全部 11 个歧义词 | ⬜ 不勾 | 会误伤 cursor、goose、tabby 这类普通单词 |
| 歧义词区分大小写 | ⬜ 不勾 | 只有打开了歧义词才有用 |
| 仅检查，请求保持原样 | ⬜ 不勾 | 排查问题时临时勾上，要和下一项一起用 |
| 记录命中信息到请求状态 | ⬜ 不勾 | 排查问题时勾上，magpie 日志里会出现 `antigravity-coding-filter {…}` 一行 |
| 账号类型 | 填 `antigravity` | 保持默认 |

**页面上没有 `providers` 这一项**，默认值就是 `antigravity-bridge`。如果你的 Bridge 用的是别的 provider id，用命令行改：

```sh
magpie plugin options <插件名> '{"mode":"rewrite","providers":["<你的 provider id>"],"accounts":["antigravity"]}'
```

**装在哪台 magpie 上**：装在运行 Antigravity Bridge 的那台。如果另有一台网关通过 remote-magpie 连到它，**不要装在网关上**。网关看到的 provider 是那台远程 magpie，分不清哪些请求是发往 Antigravity 的，插件不会生效。

---

一个 [magpie](https://usemagpie.ai) 网关中间件：把系统提示词里其他编码 agent 的名字改写或伪装——只作用于发给 Antigravity 账号的请求，其他请求一概不动。

移植自 [jellyfish-p/cpa-plugin-antigravity-coding-filter](https://github.com/jellyfish-p/cpa-plugin-antigravity-coding-filter)（CLIProxyAPI v7 动态插件，MIT），改用 magpie 的网关中间件 API。内置词表、受保护的技术引用和改写语义沿用该插件；钩子和默认动作则来自 magpie。

这份代码还带有[上游移植版没有的新增功能](#nkbaa-版新增的功能)：整词匹配、代码保护、可选的模型名关键词、报告本会命中哪些名字的试运行模式；另外还在 magpie 窗口里加了[一个页面](#设置页面)，用表单设置选项，并能在 agent 发出系统提示词之前先试一试效果。

## 为什么需要它

如果请求的系统提示词看起来出自其他编码 agent，Antigravity 就会返回 `429 Resource has been exhausted`，不管剩余额度还有多少；magpie 会把它显示为 400，并在错误信息中写明这一点。上游插件的 issue [#4](https://github.com/jellyfish-p/cpa-plugin-antigravity-coding-filter/issues/4) 做过对照测试：账号、模型、User-Agent 和代理都相同，只改身份描述语句，429 就变成了 200。

magpie 在请求路由之后调用 `onProvider`，每次尝试调用一次，因此这个中间件可以只处理落到 Antigravity 账号上的那次尝试，而发给 Claude 订阅（或 API 密钥）的请求则完全保持 agent 写的原样——即使在先尝试 Antigravity 的路由组里也是如此。

## 安装

magpie 可以从 npm、GitHub 或本地文件夹安装插件。添加本仓库即可（它带有 `magpie-plugin` topic，「Plugins › Discover」能找到它），也可以让 magpie 指向本地检出的文件夹（「Plugins › Installed › Install from folder」）。

在「Plugins › Installed › Options」里设置选项，或者用 `magpie plugin options antigravity-coding-filter '<json>'`；也可以在[设置页面](#设置页面)里设置，那里既能改选项，又能直接看到效果。

## 选项

```json
{ "mode": "cloak",
  "useDefaultKeywords": true,
  "useModelKeywords": false,
  "useAmbiguousKeywords": false,
  "customMappings": {},
  "accounts": ["antigravity"],
  "boundaries": true,
  "protectCode": true,
  "strictCase": false,
  "dryRun": false,
  "report": false,
  "off": false }
```

- **`mode`**：`cloak`（默认）、`rewrite`，其他任何值都按 `cloak` 处理。
- **`useDefaultKeywords`**：`true`（默认）启用内置词表；`false` 则只用 `customMappings`。
- **`useModelKeywords`**：`false`（默认）不处理模型名。`true` 会加上一份简短列表——`Claude Sonnet`、`Claude Opus`、`Claude Haiku`、`Claude`、`GPT-5`、`GPT-4o`、`GPT-4.1`、`GPT-4`、`GPT-3.5`、`Gemini 3`、`Gemini 2.5`、`Gemini 2.0`——用于那些因模型名而不是 agent 名触发 429 的提示词。
- **`useAmbiguousKeywords`**：`false`（默认）时，Go 版插件自带、但同时也是普通单词的 11 个名字不在列表里，见[内置词表](#内置词表)。`true` 一次性把它们全部加回来。
- **`customMappings`**：额外的名字。可以是对象（`{"Cursor": "Antigravity"}`）、以逗号或换行分隔的字符串（`"Cursor: Antigravity, Windsurf: Antigravity"`），或者由这类字符串组成的数组。键会覆盖内置词表中拼写相同（忽略大小写）的名字。值为 `null` 时把该名字移出列表——想删掉一条你根本不会发送的内置词条，就写 `{"Windsurf": null}`；值为空则跳过。字符串里含换行时按每行一条解析，因此值里可以带逗号：`"Cursor: My IDE, Inc.\nWindsurf: My IDE, Inc."`。
- **`accounts`**：要处理的订阅类型，比较时忽略大小写。默认是 `antigravity`；走 API 密钥的尝试没有账号，永远不会被改动。
- **`boundaries`**：`true`（默认）只把名字当作整词匹配，所以 `Traefik`、`decline`、`Kirov`、`raider` 不会被当成 `Trae`、`Cline`、`Kiro`、`Aider`。`false` 即 Go 版插件的行为：名字出现在任何位置都会匹配，包括更长单词的内部。
- **`protectCode`**：`true`（默认）不改动 ` ``` ` 代码块和 `inline code` 里的名字，代码示例中的字符串字面量不会被改写。`false` 则像处理正文一样改写它们。
- **`strictCase`**：`false`（默认）忽略大小写匹配，因此小写的身份句（如 `you are cursor`）照样会被伪装，这正是这个中间件的用途。`true` 时，11 个歧义词（前提是它们在列表里）只按其原本的拼写匹配，于是 `move the cursor to the end`、`goose bumps`、`the tabby cat` 不会被改动——代价是小写的身份句会漏过去。
- **`dryRun`**：`true` 时只计算会匹配到什么，请求原样发出，`report` 仍照常填写。
- **`report`**：`true` 时把结果写到 `ctx.state["antigravity-coding-filter"]`（`{ account, mode, dryRun, changed, matches: [{ name, count }] }`），网关日志不用重放请求就能显示命中了什么。请求本身不受影响。
- **`off`**：所有请求原样放行。`true`、`"true"`、`"1"`、`"yes"`、`"on"` 都会关闭中间件；`"false"` 不会。

上游插件把其中两项写作 `use_default_keywords` 和 `custom_mappings`；两种写法都能识别，`use_model_keywords`、`protect_code`、`dry_run` 也一样，因此上游的配置可以直接搬过来。

## 模式

| 模式 | 作用 | 原因 |
|---|---|---|
| `cloak`（默认） | 在每个匹配到的名字的第一个和第二个字母后各插入一个零宽空格（U+200B），例如 `C\u200bu\u200brsor`。模型读到的词和原来一样。 | 句子保持通顺。`rewrite` 会把 “You are a Claude agent” 变成 “You are a Antigravity”。magpie 自带的 `antigravity-cloak` 中间件也是这么做的。 |
| `rewrite` | 匹配到的名字替换为映射值，通常是 `Antigravity`。 | 上游插件的 `rewrite` 模式，原样移植。 |

上游插件的第三种模式 `block`（返回 403 拒绝请求，并写明匹配到了什么）在这里没有对应实现，设置 `mode: "block"` 会按 `cloak` 处理。magpie 在请求路由之后才调用 `onProvider`，而 `onProvider` 无法拒绝请求（在这里调用 `ctx.reject` 不起作用，agent 照样会收到回复）。如果要在路由前拒绝请求，请另外安装 [@magpie-community/middleware-block-patterns](https://github.com/magpie-community/plugins/tree/main/packages/block-patterns)——它在 `onRequest` 阶段、尚未选定任何 provider 时运行，因此分不清 Antigravity 和 Claude 订阅。

## 处理范围

只处理系统提示词，无论请求所用的 API 以哪种结构存放它。请求中的其他内容一律不动：用户和助手的对话轮次、工具定义和工具结果、元数据、图片都不碰。

| API | 读取的内容 |
|---|---|
| Anthropic | 顶层 `system`（字符串或文本块），以及所有 `system` / `developer` 消息 |
| Chat Completions | 所有 `system` / `developer` 消息，`content` 可以是字符串、单个文本对象或文本块 |
| Responses | `instructions`、字符串形式的 `input`，以及所有 `system` / `developer` 输入项（包括 `input_text` 块） |
| Gemini | `systemInstruction` 和 `system_instruction`（字符串或 `{ parts }` 对象），以及 role 为 `system` 或 `developer` 的 `contents` 条目 |

包在 `request` 对象里的请求会先解包，再按同样的方式读取，与上游插件一致。

在系统提示词内部，属于技术引用的名字不会被改动：URL、`mcp__…` 标识符、Windows 和 POSIX 路径、`./` 和 `../`、`.codex`，以及 `*.md` 文件名。所以 `C:\work\Cursor\src\main.ts`、`https://cursor.com/docs`、`cursor.md` 和 `.codex/Cursor/rules.md` 里的 `Cursor` 会保留——它是路径的一部分，不是身份。开启 `protectCode` 后，代码块和行内代码也按同样方式保护。上游插件用相同的正则表达式保护前面这一组。

## 内置词表

共 48 个名字：Claude Agent、Claude Code、OpenAI Codex / Codex CLI、OpenCode、GitHub Copilot / CLI、Gemini Code Assist / Gemini CLI、Windsurf、Codeium、Roo Code、Kilo Code、Continue.dev、Amazon Q Developer / CodeWhisperer、JetBrains AI Assistant / Junie、Qoder / CLI、Qwen Code、Tabnine、Sourcegraph Cody、Augment Code、Replit Agent / Ghostwriter、OpenHands、SWE-agent、Zed AI、Void Editor、PearAI、Refact.ai、GitLab Duo、Visual Studio IntelliCode、CodeBuddy、Blackbox AI、Pieces for Developers、CodiumAI、Rovo Dev CLI、Factory Droid、OpenClaw / Clawdbot / Moltbot、Hermes Agent、WorkBuddy、Chatbox / Chatbox AI、Kelivo。

匹配时最长的名字优先，忽略大小写，所以 `GitHub Copilot CLI` 只会被替换一次，而不是先替换 `GitHub Copilot`、再剩下一个 ` CLI`。

Go 版插件自带的 11 个名字**不在**其中，因为它们同时也是普通英文单词或人名：**Aider、Cline、Codex、Cursor、Devin、Goose、Hermes、Kiro、Qodo、Tabby、Trae**。把它们放进列表时，`move the cursor to the end`、`goose bumps`、`the tabby cat` 和 `Kirov` 都被改坏了。如果提示词里用到其中某个名字，就单独加上它：

```json
{ "customMappings": { "Cursor": "Antigravity", "Codex": "Antigravity" } }
```

或者用 `"useAmbiguousKeywords": true` 把 11 个全部加回来，再配合 `"strictCase": true` 避免误伤 `move the cursor`。这份列表通过 `ambiguousNames` 导出；替换值设为 `null` 则彻底移除某个名字：`{"Goose": null}`。

已知的缺口（沿用上游列表，没有凭猜测去补）：

- 不含模型名。除非开启 `useModelKeywords`，否则写着 `You are Claude` 或 `based on GPT-5` 的提示词会原样通过。Issue #4 发现单凭模型名就可能再次触发 429，遇到这种情况，请开启 `useModelKeywords`，或者把你实际发送的那句加进去：`"customMappings": {"Claude Sonnet": "Antigravity", "GPT-5": "Antigravity"}`。
- `Amp`、`Manus`、`v0`、`Lovable`，以及单独出现的 `claude` 也不在其中。
- 11 个歧义词则是反方向的取舍：默认情况下，写着 `You are Cursor` 的提示词**不会**被过滤，除非你把它加进来。这是有意为之——会把 `move the cursor` 改掉的过滤器，比漏掉一个只需一行就能补上的名字更糟——但这也意味着，如果 agent 自称其中某个名字而 429 又出现了，首先要检查的就是 `customMappings`。

## NKBaa 版新增的功能

| 选项 | 默认值 | 解决的问题 |
|---|---|---|
| `boundaries` | `true` | 没有它时，`Traefik` 会变成 `Tr\u200bae\u200bfik`，`decline` 变成 `dec\u200bl\u200bine`，`Kirov` 变成 `K\u200bi\u200brov`，`raider` 变成 `ra\u200bi\u200bder`——提到这个 ingress 控制器或写了 “we decline” 的提示词都会被改坏。开启后，名字只在前后不紧贴其他字母、数字或下划线时才匹配。`boundaries: false` 可完全恢复 Go 版插件的匹配方式。 |
| `protectCode` | `true` | 代码块或行内代码里的名字以前也会被改写，`const who = "Cursor"` 会变成 `const who = "C\u200bu\u200brsor"`——模型本该照抄的示例里，字符串字面量被改了。 |
| `useAmbiguousKeywords` | `false` | Go 版插件自带、同时也是普通单词的 11 个名字已从内置词表中移除（见[内置词表](#内置词表)）；对于自称 Cursor、Codex 或 Goose 的 agent，用这个选项可以一步把它们加回来。 |
| `strictCase` | `false` | 针对这 11 个名字（在列表中时）。`boundaries` 能挡住 `goosebumps`，但挡不住 `move the cursor to the end` 或 `the tabby cat`；`strictCase` 要求按它们书写时的拼写匹配，其余名字仍忽略大小写。默认关闭，因为拦住小写的身份句比正文干净更重要。 |
| `useModelKeywords` | `false` | 加上内置词表没有收录的模型名；issue #4 表明，单凭模型名就可能再次触发 429。 |
| `dryRun` / `report` | `false` | 对于 429 又回来的情况，README 原本的建议是手动二分系统提示词。`dryRun` 加 `report` 无需改动任何发出的内容就能指出匹配到了什么：跑一遍，读取 `ctx.state["antigravity-coding-filter"].matches`，再添加一条 `customMappings`。 |

有两处改动看不见，但确实存在：

- **钩子更快了。** `protectedSpans` 现在会合并重叠的区间，`transform` 用一个游标顺序遍历，不再每次匹配都扫描整个列表。在一个含 5,000 个 URL、5,000 处匹配的 272 KB 提示词上，钩子原来耗时 316 ms——超出 250 ms 的预算，超时会导致中间件被丢弃——现在约 9 ms。移植版给出的 80 KB / 0.6 ms 数据依然成立。
- **选项读取更稳妥。** `accounts` 忽略大小写比较（以前 `["Antigravity"]` 什么都匹配不到，而且不报错），`off: "false"` 不再关闭中间件；四种 API 剩下的几种提示词结构——没有 `type` 的 `system` 块、`content` 为单个文本对象的消息、字符串形式的 `input`、role 为 `system` 的 Gemini `contents` 条目——现在都会读取，不再跳过。

还有些小改动，是读代码时发现的，而不是有人报告的：把名字映射成自己的条目不再计为改动（以前钩子会返回一个并未修改的请求体）；匹配器缓存保留最近 8 组选项，而不是 1 组，同时服务两组选项的网关不必再每个请求都重建缓存；`customMappings` 可以删除条目（`null`），值里也可以带逗号（每行一条）。

## 设置页面

这个包还会在 magpie 窗口里加一个页面，位于侧边栏「Plugins」下：「Antigravity 编码过滤」。

![设置页面：左侧是选项，右侧是预览](docs/screenshot.png)

页面跟随 magpie 的浅色和深色主题（[深色](docs/screenshot-dark.png)），窗口较窄时自动改为单栏布局。

页面调用 Magpie 官方的 `ui.button`、`ui.toggle` 和 `ui.row` 组件。社区页面的 bridge 不暴露宿主内核的 `magpie.ui`，所以插件内置了一份未经修改的组件子集，取自 [Magpie 内核](https://github.com/yetone/magpie/blob/f38ef9821b17cd08af1bca60081d03b00d6970dc/internal/gui/assets/kernel.js)，并附带相应的官方样式规则。源码版本、方法校验和以及 MIT 许可证都放在 `page/vendor/` 下。`page/ui.js` 提供宿主元素辅助函数，并把表单状态和 `--magpie-*` 样式 token 适配到沙箱中；它不会修改或访问父窗口。如果宿主暴露了 `magpie.ui`，优先使用宿主的。

设置行、所有开关和操作按钮都由这些组件工厂函数生成。`page/style.css` 保留页面布局和 token 别名。页面的基础排版和主题仍由 bridge 提供。浏览器测试会断言最终页面包含官方的开关按钮和设置行，且不再有任何复选框输入；`test/magpie-ui.test.mjs` 负责守住固定版本的组件方法和加载顺序。

- **常用设置在前。** 打开「启用过滤」，在「保留原文」和「替换名字」中选一个，按需开启「同时处理模型名称」。高级匹配开关、账号类型和 JSON 默认折叠。
- **统一的规则管理。** 内置名字和自定义名字合并成一个去重的列表，自定义和已修改的规则排在前面。可按全部规则、默认词表、模型名称、歧义词或自定义 / 已修改条目筛选，也可以按名称搜索；每行显示来源、替换值、状态和操作。
- **直接在列表里新增和修改。** 点「新增规则」或「修改」——编辑器以紧凑的横条形式出现在列表上方——修改要匹配的名称 / 替换值或排除状态，然后点「应用到列表」。点「取消」则规则保持不变。修改内置名字会保存为一条覆盖规则；「恢复默认」会删除这条覆盖。给内置名字改名时，会明确排除旧名字并新建新名字。纯自定义规则可以改名或删除。重复名称（忽略大小写）会被拒绝。
- **保存草稿。** 「应用到列表」会更新列表和预览；「保存更改」才会持久保存。Ctrl/Cmd+S 也会在保存前校验并应用正在编辑的内容。[编辑器截图](docs/screenshot-editor.png)。
- **窗口尺寸下依然易读。** 带「保存更改」按钮的标题栏固定在顶部，规则列表随页面一起滚动（没有嵌套滚动条，也不会截断行）；选中的处理方式用实色强调色显示，而不是悬停时的浅色。
- **如实的预览。** 粘贴一段系统提示词，或载入 Chatbox 示例。发出的内容和匹配数立即更新；关闭过滤或仅检查模式下原文保持不变。预览使用当前草稿，网关请求则使用已保存的设置。
- **显式保存。** 只有在做了有效修改之后，「保存更改」才可用。页面会报告读写失败，保留无关的选项，并把旧版映射格式转换成列表行，不会丢失名字。`Ctrl/Cmd+S` 同样可以保存，没有改动时什么也不做。保存时会合并其他窗口的独立修改；如果同一项设置或同一个名字有冲突的修改，草稿保持未保存，并给出说明。
- **兼容沙箱的复制。** 如果无法访问剪贴板，「复制 JSON」会展开配置并选中其内容，方便按 Ctrl/Cmd+C 复制。

页面通过 magpie bridge 读写这个中间件自己的选项。如果当作普通文件直接打开，它只会保存到浏览器存储（页面上有明确标注），不会改动网关。

`test/page.test.mjs` 在 10 组选项下对比预览用的匹配器和中间件。表单测试覆盖旧版配置、排除、重复名称，以及关闭 / 仅检查时的预览。运行 `node test/browser-ui.mjs` 可在真实 Chrome 中做交互测试，覆盖编辑、保存、重新加载、失败和窄屏布局（在其他机器上用 `CHROME_BIN` 指定 Chrome）。浏览器测试使用官方基础样式表和一个受控的 bridge 替身，并不算连接真实 magpie 的端到端测试。

## Chatbox

[Chatbox](https://chatboxai.app) 是一个支持 OpenAI、Claude、Gemini 和本地模型的桌面客户端。它的模型提供方设置可以填自定义的 OpenAI 兼容 API 地址，所以它的请求可以像 agent 的请求一样经过 magpie 网关、落到 Antigravity 账号上——而它会把自己的名字写进发出的系统提示词，在这条路由上，这就足以再次触发 429。因此 `Chatbox` 和 `Chatbox AI` 收录在内置词表中。

接入 magpie 的方法：在 Chatbox 里添加一个自定义的 OpenAI 兼容提供方，API 地址填 magpie 的网关地址，密钥填 magpie 网关密钥，再选一个 magpie 提供的模型。其他都不受影响：过滤只作用于发给 Antigravity 账号的请求，所以走 Claude 订阅的 Chatbox 会收到它原本写的提示词。

## Kelivo

[Kelivo](https://github.com/Chevey339/kelivo) 是一个用 Flutter 开发的开源跨平台聊天和 agent 客户端，支持 Android、iOS、macOS、Windows 和 Linux。它的模型提供方设置可以填自定义的 OpenAI 兼容 API 地址，所以它的请求可以像 agent 的请求一样经过 magpie 网关、落到 Antigravity 账号上——而在这条路由上，系统提示词里一旦出现这个应用的名字，就会再次触发 429。因此 `Kelivo` 和 Chatbox 一样收录在内置词表中。

接入 magpie 的方法：在 Kelivo 里添加一个提供方，Base URL 填 magpie 的网关地址（通常以 `/v1` 结尾），密钥填 magpie 网关密钥，然后获取模型列表并选一个 magpie 提供的模型。其他都不受影响：过滤只作用于发给 Antigravity 账号的请求，所以走 Claude 订阅的 Kelivo 会收到它原本写的提示词。

## 仍然出现 429 时

如果 magpie 显示还有额度、同一模型和账号上的其他请求也都能通过，却仍然返回 `429 Resource has been exhausted`，说明这个系统提示词里还有别的内容被匹配到了——往往不是 agent 自己的文字，而是自定义提示词或其他插件注入的内容。

排查方法：打开 `report`（如果想让请求原样发出，再打开 `dryRun`），查看匹配结果；或者把提示词粘贴到设置页面的预览里，看那里列出的名字；或者用一条简短的用户消息重放失败的请求，每次去掉一半系统提示词，直到 429 消失。然后从触发 429 的那部分里挑一句有辨识度的短语，加进 `customMappings`。这个中间件只处理给定的名字，无法隐藏任意提示词。

## 与 Go 版插件的区别

| | CLIProxyAPI 插件 | 本插件 |
|---|---|---|
| 钩子 | `request.intercept_after`，当 `ToFormat == "antigravity"` 时 | `onProvider`，当 `ctx.provider.account` 在 `accounts` 中时 |
| 默认模式 | `block`（403） | `cloak`（零宽空格） |
| `block` | 403 `blocked_by_antigravity_coding_filter` | 不可用；`mode: "block"` 按 `cloak` 处理 |
| 改写 | 名字 → `Antigravity` | 相同，在 `rewrite` 模式下 |
| 匹配 | 子串匹配，忽略大小写 | 默认整词匹配；`boundaries: false` 恢复原行为 |
| 受保护的引用 | URL、`mcp__`、路径、`.codex`、`*.md` | 同样的列表、同样的正则表达式，开启 `protectCode` 时还保护代码 |
| 选项 | `mode`、`use_default_keywords`、`custom_mappings` | camelCase，仍可识别 snake_case 写法 |
| 耗时预算 | 每个钩子 250 ms | 每个钩子 250 ms；80 KB 提示词约 0.6 ms，带 5,000 个保护区间的 272 KB 提示词约 9 ms |

## 启用前须知

- 它会改动发给 Google 的内容，让本该被拒绝的请求不被拒绝。这可能违反 Antigravity 的条款，Google 可能因此封号。请只在丢了也无所谓的账号上使用。
- Google 随时可能调整匹配规则。11-11 那次，首字母后只插一个零宽空格就失效了，得再插第二个；下一次调整可能让 `cloak` 再次失效，429 也会随之回来。真遇到了，可以试试 `rewrite`，或者改用 `customMappings` 里配置的名字，而不是伪装后的名字。
- 模型读到的词和原来一样，所以这不会改变 agent 被告知的身份，只改变字符串在传输中的样子。

## 测试

```sh
bun test
```

`cases.json` 按 magpie 网关的调用方式运行钩子，同样的用例也会在 magpie 自己的 JavaScript 引擎（moejs）里跑一遍。`test/middleware.mjs` 复制自 [magpie-community/plugins](https://github.com/magpie-community/plugins)（MIT），让这些用例可以独立运行。`test/features.test.mjs` 覆盖用例表达不了的部分：`ctx.state`、`dryRun`、第二遍处理不产生改动、内置词表的两部分，以及钩子的耗时预算。`test/page.test.mjs` 确保页面里那份匹配器与中间件的一致，`test/page-files.test.mjs` 检查页面没有内联脚本、只查找确实存在的 id，`test/page-smoke.test.mjs` 检查表单转换和预览行为。`node test/browser-ui.mjs` 通过受控的 bridge 在 Chrome 中实际操作页面。
