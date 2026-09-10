---
name: chat-to-tutorial
description: Use this skill when the boss gives raw material (WeChat/DingTalk chat history zip, transcript, customer quotes, screenshots) and asks to "整理成教程" / "按我的思路整理" / "整理一下" / "写进钉钉文档". The output is a structured DingTalk AI Doc that **preserves the boss's original words, original images, and original narrative order** — never abstracted into a generic methodology. Output language is Chinese regardless of source language. Trigger on phrases like "聊天记录整理", "原话整理", "按我的思路", "保留原话", "原图原话", "原样整理", "整理成笔记", "整理成教程", "写进钉钉". Do NOT use for: generic writing without original material, summarization, rewriting, translation, or polishing — those have their own skills.
description_zh: 老板给原始素材（微信/钉钉聊天 zip、录音转写、客户原话、截图），要求"按原话/原图/原顺序"整理成结构化文档（默认落钉钉 AI 文档）
version: 0.2.0
display_name: chat-to-tutorial
display_name_zh: 素材整理成教程
visibility: public
---

# Chat-to-Tutorial

把老板的真实素材（聊天记录 / 录音转写 / 客户原话 / 截图）整理成结构化教程，**忠于原话 + 原图 + 原顺序**，默认落钉钉 AI 文档。

## 一句话定位

> **老板发素材 → AI 按原貌整理 → 钉钉 AI 文档（含原话 + 原图 + 图说）**
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

## 已确认的开发决策（2026-09-10）

| 决策项 | 默认值 |
|--------|--------|
| 输出平台 | **钉钉 AI 文档**（`dws doc`） |
| 输出语言 | **中文**（英文素材也输出中文） |
| 图说粒度 | **一行**（≤80 字） |
| 简短导语 | 默认有（≤100 字，老板说"原样复制粘贴"时跳过） |
| 多平台并行 | 默认只一份（老板明确说"同时给飞书一份"才双发） |
| Skill 名称 | `chat-to-tutorial` |

---

## 核心硬规则（必须死守）

1. **保留原话**：老板素材里的每句话都要保留，包括口语笔误、错别字、不通顺的地方
2. **保留原图**：截图原样嵌入，**不替老板截图、不生成假图说、不在缺图时编图**
3. **保留原顺序**：按素材原本的叙事顺序整理（聊天按消息时间、转写按时间顺序）
4. **不发散**：不抽象成"通用方法论"、不替老板提炼金句、不重新结构化
5. **AI 只做排版**：排版美化 + 图说 + 简短导语/总结（不超过原文 1/3 篇幅）

> **核心原则**：不要过度发散；按素材原有思路组织，并让图片和文字对应。

---

## 执行流程（按顺序执行，不要跳步）

### Step 1：识别素材类型
- 收到 zip → 在运行环境提供的解压目录中查找 `*.jpg`、`*.png`、`*.txt`，确认结构
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

### Step 5：上传到钉钉 AI 文档
```bash
cat "{workspace}/outputs/{slug}.md" | dws doc +create \
  --name "教程：{主题}" \
  --content - \
  --doc-format markdown \
  --format json
```

记录返回的 `nodeId` 和 `docUrl`，供 Step 6 用。

### Step 6：插入图片（仅当 Step 2 有图时）
1. 把图复制到 cwd 下相对路径：
   ```bash
   cp "{源图路径}" "{cwd}/images/{NN}-{slug}.jpg"
   ```
2. `dws doc +fetch --node {nodeId} --detail full` 拿到所有 block 的 uuid，找到对应图说段落 uuid
3. 对每张图依次插入：
   ```bash
   dws doc +media-insert \
     --node {nodeId} \
     --file "images/{NN}-{slug}.jpg" \
     --ref-block {图说段落 uuid} \
     --where before \
     --name "图{N}：{≤30字图说}" \
     --mime-type "image/jpeg" \
     --yes \
     --format json
   ```

注意：
- 每张图插入到对应图说的**前面**（`--where before`），让图说自然成为 caption
- 每张图新生成的 uuid 会被返回（`data.blockId`），但下次插入用下一个图说的 uuid 作为 ref-block（链式锚定）
- 写操作必须 `--yes` 跳过确认门禁（老板明确要求附图）

### Step 7：交付
1. `present_files` 展示文档链接（`docUrl`）和本地源 md 文件路径
2. 给老板的最终回复必须包含：
   - 文档链接
   - 简短结构说明（让老板快速核对是否符合预期）
   - 图说是否准确的提示（让老板二次校对）
   - 踩坑备查（如有）

---

## 钉钉通道踩坑备查（必须记忆）

| 踩坑 | 解决方案 |
|------|----------|
| `dws doc +create --content @绝对路径` 报错 "只接受工作目录内相对路径" | 用 `cat ... \| dws doc +create --content -` 从 stdin 传 |
| `dws doc +media-insert --file` 只接受 cwd 内的相对路径 | 先 `cp` 到 cwd 下，用相对路径引用 |
| 插入图片后想加章节标题 | 用旧最后一块的 `--ref-block` 精确定位，新图的 uuid 从返回 `data.blockId` 拿 |
| `doc +create` 回读校验失败（`doc_write_verification_failed`） | 用 `doc +fetch --detail full` 实际确认内容，API 端比对延迟，不影响落地 |
| `media-insert` / `+update` 是写操作需要 `--yes` | 老板明确要求附图 / 修改时直接 `--yes` 跳过确认 |
| `doc +update --content @绝对路径` 报错 | 同上，stdin 解决 |
| 老板微信 zip 不一定附带图片 | 每次先 `Glob` 解压目录确认，不能看到文本里有 `[图片]` 就假设有图 |

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
- **可选附加**：目标平台（默认钉钉）、文档标题（默认按主题生成）、是否跳过导语（默认否）

### 输出
- **主交付物**：钉钉 AI 文档（含标题 + 导语 + 原话分段 + 原图 + 图说）
- **附属交付物**：本地源 md 文件（`{workspace}/outputs/{slug}.md`）

---

## 当前边界

- 默认输出端是钉钉 AI 文档；若运行环境未提供 `dws` 命令，应改为先生成本地 Markdown，或征求用户指定的文档平台。
- 只有用户明确要求且运行环境已授权时，才执行创建文档、上传图片等外部写操作。
