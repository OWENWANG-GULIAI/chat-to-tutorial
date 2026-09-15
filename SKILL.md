---
name: chat-to-tutorial
description: Use this skill when the boss gives raw material (WeChat/DingTalk chat history zip, transcript, customer quotes, screenshots) and asks to "整理成教程" / "按我的思路整理" / "整理一下" / "写进钉钉文档". The output is a structured DingTalk AI Doc that **preserves the boss's original words, original images, and original narrative order** — never abstracted into a generic methodology. Output language is Chinese regardless of source language. Trigger on phrases like "聊天记录整理", "原话整理", "按我的思路", "保留原话", "原图原话", "原样整理", "整理成笔记", "整理成教程", "写进钉钉". Do NOT use for: generic writing without original material, summarization, rewriting, translation, or polishing — those have their own skills.
description_zh: 老板给原始素材（微信/钉钉聊天 zip、录音转写、客户原话、截图），要求"按原话/原图/原顺序"整理成结构化记录（默认落老板指定的 AI 文档作为子页面）
version: 0.4.0
display_name: chat-to-tutorial
display_name_zh: 素材整理成教程
visibility: public
---

# Chat-to-Tutorial

把老板的真实素材（聊天记录 / 录音转写 / 客户原话 / 截图）整理成结构化教程，**忠于原话 + 原图 + 原顺序**，默认落老板指定的钉钉 AI 文档（作为该文档的下一层子页面）。

## 一句话定位

> **老板发素材 → AI 按原貌整理 → 钉钉 AI 文档（作为父文档的子页面，含原话 + 原图 + 图说）**
>
> 不是"AI 自由发挥写教程"，而是"AI 给老板的素材做排版和图说"。

---

## 何时触发（When to Use）

老板发出以下任一指令时**立即 load 本 skill**：

- "把这个聊天记录整理成教程"
- "按我的聊天整理一下" / "按我分享给你的内容整理"
- "把这段录音转写整理成笔记"
- "把这些客户原话整理出来"
- "把这份素材写进钉钉文档 / 飞书 / Obsidian"
- "原样整理" / "保留原话" / "不要发散"
- "整理成笔记" / "整理成教程"

**伴随特征**：老板会附上素材文件（zip / md / txt / 截图），且明确要求"按她的思路"。

## 不适用（明确告知老板走别的 skill）

- 老板要求"写一份关于 X 的教程"（没有原素材，让 AI 自由发挥）→ 通用写作 skill
- 老板要求"总结 / 摘要"（只要要点，不要原话）→ 通用摘要 skill
- 老板要求"改写 / 翻译 / 润色"（明确要 AI 加工原话）→ 通用润色 skill
- 老板要的不是文档而是 PPT / Word / Excel → `tencent-pptx` / `tencent-docx` / `tencent-docs-sheet-generation`

---

## 已确认的开发决策（2026-09-10；2026-09-15 多次更新）

| 决策项 | 默认值 |
|--------|--------|
| 输出平台 | **钉钉 AI 文档**（`dws doc`） |
| 输出语言 | **中文**（英文素材也输出中文） |
| 图说粒度 | **一行**（≤80 字） |
| 简短导语 | 默认有（≤100 字，老板说"原样复制粘贴"时跳过） |
| 多平台并行 | 默认只一份（老板明确说"同时给飞书一份"才双发） |
| 图片嵌入方式 | **base64 data URI**（不再用 `dws doc +media-insert`，见 Step 6） |
| 父 AI 文档 nodeId | `dQPGYqjpJYNMadqwSKYPAq2D8akx1Z5N`（标题"王跃平 OWENWANG 的日常分享"） |
| 子页面挂载方式 | 在父 doc 下创建子文档作为教程，详见 Step 5 |
| Skill 名称 | `chat-to-tutorial` |

---

## 核心硬规则（必须死守）

1. **保留原话**：老板素材里的每句话都要保留，包括口语笔误、错别字、不通顺的地方
2. **保留原图**：截图原样嵌入，**不替老板截图、不生成假图说、不在缺图时编图**
3. **保留原顺序**：按素材原本的叙事顺序整理（聊天按消息时间、转写按时间顺序）
4. **不发散**：不抽象成"通用方法论"、不替老板提炼金句、不重新结构化
5. **AI 只做排版**：排版美化 + 图说 + 简短导语/总结（不超过原文 1/3 篇幅）

> **老板原话（铁律来源）**：
> "不要去进行太多的发散，要按照我分享给你的内容，去按照我的思路去整理教程出来。图片跟文字需要搭配上。"

---

## 执行流程（按顺序执行，不要跳步）

### Step 1：识别素材类型
- 收到 zip → 读取系统已预解压的目录（一般 `/Users/owenwang/.workbuddy/app/tmp/wechat-chat-history/wechat-chat-history-{id}-{hash}/`）→ `Glob *.jpg` / `Glob *.png` / `Glob *.txt` 确认结构
- 收到 md / txt → 直接 `Read`
- 收到截图 → `Read` 工具识别内容

### Step 2：检查图片是否附带
执行 `Glob` 或 `find {dir} -name "微信图片_*" -o -name "*.jpg" -o -name "*.png"`。

- **有图** → 复制到 cwd 下 `images/{NN}-{slug}.jpg`，走 Step 6 的图片嵌入流程
- **没图** → 文档里用 `[图 N：待补 —— 描述图片应有的内容]` 占位，**明确告知老板**"本次 zip 没附带 X 张截图，请补图"，等补图后用 media-insert

### Step 3：按素材原本叙事顺序整理 markdown 源文件
- 聊天记录：按消息发送顺序逐条保留
- 录音转写：按时间顺序
- 截图：按截图顺序
- **绝不重新组织叙事顺序，不合并老板的多条消息**

### Step 4：写本地 markdown 源文件
- 路径：`{workspace}/outputs/{topic-slug}.md`（slug 用拼音或短英文短语）
- 结构：
  - H1 标题（按素材主题生成）
  - 简短导语（≤100 字，说明素材背景 + 来源 + 日期）
  - H2 分段（每段对应素材里的一个主题转折，如"起因 / 怎么做 / 结论"）
  - 每段内：老板原话（blockquote `> ...`）+ 图说占位/正式图说
- **不创建假图说，不杜撰老板没说的话，不润色原话**

### Step 5：上传到老板指定的父 AI 文档的下一层

**父 AI 文档**：
- nodeId: `dQPGYqjpJYNMadqwSKYPAq2D8akx1Z5N`
- 标题："王跃平 OWENWANG 的日常分享"
- 链接：https://alidocs.dingtalk.com/i/nodes/dQPGYqjpJYNMadqwSKYPAq2D8akx1Z5N

**核心操作（2026-09-15 老板截图确认 + 13:04 老板再截图反驳）**：
**必须**带 `--folder <父 docId>` 创建（或先 `+create` 后 `+move --folder`），否则 doc 会变成**顶层独立文档**，不在父 doc 子页面里。

```bash
cat "{workspace}/outputs/{slug}.md" | dws doc +create \
  --name "教程：{主题}" \
  --content - \
  --doc-format markdown \
  --folder "dQPGYqjpJYNMadqwSKYPAq2D8akx1Z5N" \
  --yes \
  --format json
```

记录返回的 `nodeId` 和 `docUrl`。

如果忘了带 `--folder`，doc 已经创建为顶层独立文档，**补救方法**：
```bash
dws doc +move --node <新 doc nodeId> --folder "dQPGYqjpJYNMadqwSKYPAq2D8akx1Z5N" --yes --format json
```

记录返回的 `nodeId` 和 `docUrl`。

**已有子页面示例（老板 2026-09-15 截图确认的 3 个教程文档）**：

| 截图位置 | 标题 | nodeId | 钉盘 path |
|---|---|---|---|
| 第 1 个 | 教程：用 WPS 直接打开 .md 文件 | `Qnp9zOoBVBn9Am1LTerdkrroV1DK0g6l` | `/6728263234/8249089790/8304239868/8304239869.adoc` |
| 第 2 个 | 教程：让 WorkBuddy 每天自动帮你点网页（agent-browser） | `N7dx2rn0JbnyroEgTNxgll27JMGjLRb3` | `/6728263234/8249089790/8259927734/8259927735.adoc` |
| 第 3 个 | 教程：用 WorkBuddy 一键把 HTML 发布成可分享网站 | `QOG9lyrgJPrlMGqwulDD3QE6WzN67Mw4` | `/6728263234/8249089790/8259975054/8259975055.adoc` |

> **路径规律**：所有子页面都在父 doc 所在钉盘目录 `/6728263234/8249089790/` 下，路径风格都是 `/6728263234/8249089790/<额外子目录>/<文件>.adoc`（多一层）。3 个老教程和新建的"皮肤"教程都是这个风格。父 doc 本身在 `/6728263234/8249089790/8249089791.adoc`，没有额外子目录。
>
> **踩坑确认（2026-09-15 三次纠正）**：
> 1. 第一次（11:49）：我以为 `+create` 不带 `--folder` 会自动落到父 doc 所在目录 → **错**。`+create` 默认落到 dws profile 默认工作空间，不是父 doc 目录。创建后会成为**顶层独立文档**，不在父 doc 子页面里。
> 2. 第二次（12:00）：老板反驳"WPS 教程不就挂在父文档下面么？" → 我误判"UI 上能看到就没事"。但 WPS 教程当时**确实**挂在父 doc 下，是因为用了 `--folder` 创建，不是"默认就好"。
> 3. 第三次（13:04）：老板再截图反驳 —— **新建的"皮肤"教程在最外层，跟父 doc 平级**，三个老教程才是父 doc 子页面。这才暴露真相：`+create` 不带 `--folder` 创建的就是**顶层独立文档**。
>
> **正确做法**：
> - **`+create` 时必须带 `--folder <父 docId>`**，否则会成为顶层独立文档
> - 如果忘了带，可以用 `dws doc +move --node <新docId> --folder <父docId>` 补救（move 已验证有效，"皮肤"教程已通过 move 修复）
> - 钉盘路径多一层是**正常现象**，UI 上正常挂在父 doc 子页面，路径嵌套不影响功能

### Step 6：嵌入图片（base64 data URI，不再用 media-insert）

**核心结论**：**不要用 `dws doc +media-insert` 上传图片到钉钉服务端**。CLI 虽然返回 verified:true，但钉钉 web 端实际渲染时图片会"加载失败"（src 是相对路径 `/core/api/resources/{id}/detail`，web 端带 cookie 也加载不出来，外部直访 404）。

**正确做法**：把图片转 base64，用 `data:image/png;base64,...` 格式直接嵌入到 jsonml 的 img src 字段，浏览器解码渲染，不依赖任何外部 URL，绝对永久。

1. Python 脚本读取图片，转 base64：
   ```python
   import base64
   with open('images/01-slug.png', 'rb') as f:
       b64 = base64.b64encode(f.read()).decode('ascii')
   data_uri = f'data:image/{ext};base64,{b64}'
   ```
2. 在 jsonml 文本中找到对应图说段落前的 img 块，把 src 替换成 `data_uri`
3. 用 `dws doc +update --command overwrite --doc-format jsonml --content - --expected-revision <REV> --yes` 整体覆盖

注意：
- base64 嵌入会让 jsonml 体积变大（图片 190KB → base64 ~254KB → 整体 +250KB 左右）
- 不要再尝试：改 src 为绝对 URL（仍需 cookie）、attachment 块（不在 schema）、embed 块（API 拒绝 blockType=embed）

### Step 7：交付
1. `present_files` 展示行链接（recordUrl）和本地源 md 文件路径
2. 给老板的最终回复必须包含：
   - AI 文档子页面链接
   - 简短结构说明（让老板快速核对是否符合预期）
   - 图说是否准确的提示（让老板二次校对）
   - 踩坑备查（如有）

---

## 钉钉通道踩坑备查（必须记忆）

| 踩坑 | 解决方案 |
||----------|
| `dws doc +create --content @绝对路径` 报错 "只接受工作目录内相对路径" | 用 `cat ... \| dws doc +create --content -` 从 stdin 传 |
| 老板微信 zip 不一定附带图片 | 每次先 `Glob` 解压目录确认，不能看到文本里有 `[图片]` 就假设有图 |
| **（2026-09-15）** `dws doc +media-insert` 上传后图片在钉钉 web 端**加载失败** | **改用 base64 data URI 直接嵌 jsonml 的 img src**，CLI verified 但 web 端实际加载不出来，不要再走 media-insert 路径 |
| **（2026-09-15）** 改 src 为绝对 URL（`https://alidocs.dingtalk.com/...`）仍然 404 | URL 仍需 cookie 鉴权且服务端路径不对，不要尝试 |
| **（2026-09-15）** 用 `block insert --element '{blockType:"embed"...}'` 报 invalid block element type | API 拒绝 embed blockType，不要用 |
| **（2026-09-15 三次纠正最终版）** | `+create` **不带** `--folder` 创建出来是**顶层独立文档**（钉盘默认工作空间），**不在父 doc 子页面里**。老板 13:04 截图明确显示新建"皮肤"教程在最外层，跟父 doc 平级。**必须带** `--folder <父docId>`，否则需要用 `+move` 补救 |
| **（2026-09-15 三次老板反驳，最终结论）** | **必须带 `--folder <父 docId>` 才能挂到父 doc 子页面**！`+create` 不带 `--folder` 会变成**顶层独立文档**（钉盘默认工作空间），不在父 doc 子页面里。补救：`dws doc +move --node <新docId> --folder <父docId>` |

---

## 验收标准（老板核对清单）

执行完 Step 7 后，自检这 8 条：

- [ ] 文档标题清晰反映老板素材的核心主题
- [ ] 老板原话全部出现在文档里，没有被改写 / 合并 / 删除
- [ ] 老板原话里的笔误 / 口语化表述原样保留
- [ ] 图片按原顺序插入，每张图有一行图说
- [ ] 老板截图原样嵌入，没有 AI 生成的截图
- [ ] 缺图时文档明确标注"待补"，没有编图
- [ ] 末尾简短导语 ≤100 字，没超过原文 1/3 篇幅
- [ ] 文档链接 + 本地源 md 同步存档并 present_files 展示

---

## 输入 / 输出

### 输入
- **素材本体**：聊天 zip / md / txt / 截图
- **可选附加**：目标平台（默认钉钉 AI 文档，挂在老板指定的父 doc 下）、文档标题（默认按主题生成）、是否跳过导语（默认否）

### 输出
- **主交付物**：钉钉 AI 文档新页面（父 doc: `dQPGYqjpJYNMadqwSKYPAq2D8akx1Z5N` 的下一层，含标题 + 导语 + 原话分段 + 原图 + 图说）
- **附属交付物**：本地源 md 文件（`{workspace}/outputs/{slug}.md`）

---

## 沉淀依据

| 时间 | 事件 | 学到 |
|------|------|------|
| 2026-09-10 13:08 | 整理"HTML 一键发布成网站"教程 → 初版（过度发散被老板反馈） | 不要替老板抽象 |
| 2026-09-10 13:13 | 按老板反馈重写 → 学到"保留原话 + 原图 + 原顺序" | 核心铁律成型 |
| 2026-09-10 13:44-13:48 | 整理"agent-browser 自动签到"教程 → 应用铁律，三图匹配成功 | 流程跑通 |
| 2026-09-10 15:03 | 老板要求沉淀成 skill PRD | 决定做 skill |
| 2026-09-10 15:07 | 老板确认 6 个开放问题（默认钉钉 / 全部中文 / 一行图说 / 其他默认） | 开始开发 |
| 2026-09-15 10:37 | 整理"WPS 打开 .md 文件"教程 → 用 media-insert 上传图片 | CLI verified 但 web 端实际加载失败（src 是相对路径） |
| 2026-09-15 10:46 | 老板质疑"这么复杂么"——我钻牛角尖改 src、改 attachment 块、改 embed 块，全失败 | 不要过度修复，回到最简方案 |
| 2026-09-15 10:52 | 老板发截图确认图片加载失败 | 必须用 base64 data URI 嵌入 jsonml，浏览器直解，不依赖外部 URL |
| 2026-09-15 11:46 | 老板新指令：教程输出位置改为 AI 表格（固定 nodeId） | skill 默认落点迁移；图片用 base64 嵌入策略固化 |
| 2026-09-15 11:49 | 老板澄清：那个 nodeId 是 AI 文档不是 AI 表格，教程要放"下一层" | skill 默认落点纠正为 AI 文档子页面；CLI 用 `--folder` 模拟创建子文档，UI 子页面需手动操作 |
| 2026-09-15 11:57 | 老板发截图：3 个 WorkBuddy 教程（HTML/agent-browser/WPS）已经作为子页面挂在"日常分享"下 | 当时我误以为"CLI 直接 `+create` 不加 `--folder` 就能落到这个目录" —— **13:04 证明这是错的**：3 个老教程之所以挂在父 doc 下，是因为它们都用了 `--folder` 创建，并不是默认行为 |
| 2026-09-15 12:00 | 老板反驳："WPS 教程不就挂在父文档下面么？" | **自我纠正**：我之前在 skill 里写"加了 `--folder` 会藏起来看不到"是错的判断。实际验证：加了 `--folder` 路径多一层但 UI 仍可见，**不影响功能**，WPS 教程（用 `--folder` 创建）UI 上正常挂着。已修订 skill 踩坑备查：改为"两种方式都能挂上，默认不加 `--folder` 是为了路径风格跟老教程一致，而不是因为加了会失败" |
| 2026-09-15 13:04 | 老板再发截图反驳："新建教程跟父 doc 是同一层级，不是父子关系" | **第二次自我纠正**：之前 12:00 的结论还是错的。`+create` 不带 `--folder` 创建出来是**顶层独立文档**（钉盘默认工作空间），不在父 doc 子页面里。WPS 教程能挂在父 doc 下是因为用了 `--folder`。**正确做法：`+create` 必须带 `--folder <父docId>`，或者先 `+create` 后 `+move --folder`**。已用 `+move` 把"皮肤"教程从顶层移到父 doc 下，UI 显示恢复正常。skill 第三次修订 |

完整 PRD 在：`{workspace}/outputs/skill-prd-chat-to-tutorial.md`（开发完成后可删）
