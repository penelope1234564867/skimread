# SkimRead for Zotero — 项目上下文

> 生成于 2026-09-20，用于规划 v2（三段式叙事管线）架构。

---

## 1. 项目简介

SkimRead 是一个 Zotero 7 插件，用 LLM 为 PDF 和 EPUB 生成**全文略读高亮**。用户打开论文、点击右侧边栏的"Generate"，插件选出最重要的叙事性句子，在原文上绘制彩色覆盖层，不修改原文件。

---

## 2. 目录结构

```
zotero-skimread-main/
├── addon/                          # 打包到 .xpi 的静态资源
│   ├── bootstrap.js                # Zotero 插件生命周期（启动/关闭）
│   ├── manifest.json               # Zotero 插件清单（版本、ID）
│   ├── prefs.js                    # 默认偏好值
│   ├── content/
│   │   ├── preferences.xhtml       # 设置面板 UI 布局
│   │   ├── zoteroPane.css          # 阅读器侧边栏样式
│   │   └── icons/                  # 插件图标
│   └── locale/                     # Fluent (.ftl) 翻译文件
│       ├── en-US/                  # 英语
│       └── zh-CN/                  # 简体中文
├── src/                            # TypeScript 源码（esbuild 打包）
│   ├── index.ts                    # 入口：将 Addon 类挂到全局作用域
│   ├── addon.ts                    # Addon 类：持有 config、hooks、ztoolkit
│   ├── hooks.ts                    # 生命周期钩子（onStartup, onShutdown 等）
│   ├── llm/
│   │   └── ollama.ts               # 所有 LLM 提供商实现（1339 行）
│   ├── modules/
│   │   ├── skim.ts                 # 核心管线：runSkim() + 辅助函数（1816 行）
│   │   ├── readerSection.ts        # 阅读器侧边栏 UI 渲染（825 行）
│   │   ├── preferences.ts          # 设置面板逻辑（317 行）
│   │   ├── annotations.ts          # 将覆盖层转换为 Zotero 注释
│   │   └── zoning.ts               # 从文本统计检测页眉页脚和参考文献区
│   ├── prompts/
│   │   └── skim.ts                 # 所有 LLM 提示词、Schema、校验器（781 行）
│   ├── reader/
│   │   ├── adapter.ts              # PDF 阅读器内部：提取 + 叠加绘制（1361 行）
│   │   └── epub.ts                 # 从 ZIP/OPF 提取 EPUB 文本
│   └── utils/
│       ├── prefs.ts                # 类型安全的 getPref/setPref 封装
│       ├── locale.ts               # Fluent 国际化辅助
│       ├── text.ts                 # cleanAnnotationText()
│       ├── window.ts               # isWindowAlive()
│       └── ztoolkit.ts             # ZoteroToolkit 初始化
├── test/                           # Zotero 插件集成测试（mocha+chai）
│   ├── startup.test.ts
│   ├── document-selection.test.ts
│   ├── segmentation.test.ts
│   ├── annotations.test.ts
│   ├── provider-output.test.ts
│   └── reader-structure.test.ts
├── typings/                        # TypeScript 类型声明
│   ├── global.d.ts                 # 全局变量（_globalThis, ztoolkit 等）
│   ├── i10n.d.ts                   # Fluent 消息 ID 类型
│   └── prefs.d.ts                  # 偏好键值类型
├── zotero-plugin.config.ts         # 构建配置（esbuild、资源打包）
├── tsconfig.json
└── package.json
```

---

## 3. 核心数据流（当前架构）

```
[PDF/EPUB]
    │
    ▼  extractPage() / extractEpubSections()
    │
[PageText[]] — 每页字符 + 几何信息 + 组装文本
    │
    ▼  segmentSentences()（Intl.Segmenter 断句 + 启发式规则）
    │
[ExtractedSentence[]] — 平面列表，每条含 pageIndex + 章节标签
    │
    ▼  withDocumentSections() + limitWork()
    │  （标题正则 → 章节标签，排除参考文献和前置材料）
    │
[LabelDef[]] — 来自偏好设置（默认/自定义）或 chatJSON()（自动模式）
    │
    ▼  chunkSentences() / splitIntoBands()
    │  （按上下文窗口大小分割，优先以章节边界切分）
    │
┌───────────────────────────────────────┐
│  for 每个 band/chunk:                 │
│    chatJSON() → LLM 返回 JSON         │
│    ↓                                  │
│  validateDocumentSelection()           │
│  → ClassifiedSentence[]               │
│  paint() → 高亮绘制到屏幕              │
│  cacheWrite() → 部分持久化             │
└───────────────────────────────────────┘
    │
    ▼  reducePass() — 再调一次 chatJSON()
    │  （标记 core=true 作为叙事核心句）
    │
[complete CachePayload]
    │
    ▼  paint() → 最终状态，支持核心/上下文切换
```

---

## 4. 关键文件 — 角色与依赖关系

### 4.1 `src/modules/skim.ts` — 管线编排器

**角色：** 包含 `runSkim()`——Generate 入口。涵盖从文本提取到 LLM 选择到绘制到缓存的全部流程。

**关键函数：**
| 函数 | 行号 | 用途 |
|---|---|---|
| `runSkim()` | 1439-1816 | 顶层异步编排器。依次调用：提取 → 发现标签 → 波段选择 → 精减 → 绘制 |
| `selectDocument()` | 858-924 | 整篇文档一次性选择（短论文，单一块） |
| `selectBand()` | 1018-1062 | 逐段选择（长文档切分成多块） |
| `selectBandAdaptive()` | 972-1016 | 逐段选择，标签可演变（自动模式 + 厚书） |
| `reducePass()` | 1106-1148 | 全文精减：读取所有候选句，标记叙事核心 |
| `discoverLabels()` | 759-790 | 自动模式：发送文本样本给 LLM 建议标签集 |
| `chunkSentences()` | 822-856 | 将句子按上下文预算分割成块，优先在章节/页面边界处切分 |
| `splitIntoBands()` | 1069-1083 | 将句子分成长度相近的波段用于 map-reduce 选择 |
| `segmentSentences()` | 607-625 | 使用 Intl.Segmenter 将页面文本分割成句子 |
| `sectionAtPageStart()` | 687-703 | 通过正则检测页面顶部的章节标题 |
| `withDocumentSections()` | 716-739 | 跨句子传播章节标签 |
| `saveSkimAsAnnotations()` | 1193-1246 | 将叠加高亮转换为 Zotero 注释 |

**关键设计细节：** 每个波段顺序处理；下一个波段的 LLM 调用要等前一个波段完成（校验、绘制、缓存写入）后才开始。

### 4.2 `src/llm/ollama.ts` — 所有 LLM 提供商实现

**角色：** 系统中每次 `chatJSON()` 调用都路由到此文件。根据配置分派给不同提供商：

- **codex-app-server** → `codexJSON()` → `CodexAppServerClient`（通过子进程的 JSON-RPC）
- **claude-code** → `claudeCodeJSON()` → 调用 `claude -p --output-format json`
- **antigravity-cli** → `antigravityJSON()` → 调用 antigravity CLI
- **默认（openai-compatible）** → REST HTTP 到 `/api/chat`（Ollama 原生）或 `/chat/completions`（OpenAI 兼容）

**关键类型/类：**
| 名称 | 行号 | 用途 |
|---|---|---|
| `chatJSON()` | 1235-1326 | 通用分发器：model, system, user, schema → 解析后的 JSON |
| `codexJSON()` | 1086-1124 | Codex 特定实现：启动 app-server，创建线程/轮次，等待完成 |
| `CodexAppServerClient` | 477-623 | 基于子进程的 JSON-RPC 客户端（stdin/stdout，换行分隔 JSON） |
| `waitForCodexTurn()` | 1040-1083 | 监听流式 delta + 完成事件。**目前没有进度回调。** |
| `checkStatus()` | 1141-1195 | 测试连接：列出模型，返回可用性 |

**痛点：** `codexJSON()` 每次都创建新的 `CodexAppServerClient`（新子进程）。一篇论文 10 次调用就要启动 ~10 次子进程。

### 4.3 `src/prompts/skim.ts` — 提示词、Schema、校验器

**角色：** 所有发给 LLM 的文本都存放在此。通过 `PROMPT_VERSION`（当前 29）进行版本管理。

**关键导出：**
| 名称 | 用途 |
|---|---|
| `DEFAULT_LABELS` | Goal/Method/Result/Novelty，附带描述和反描述 |
| `buildSkimSystem()` | 系统提示词：标签定义 + 选择规则 |
| `buildDocumentSelectionPrompt()` | 全文文档：句子 + 页面/章节上下文 |
| `buildBandSelectionPrompt()` | 逐段：目标计数 + 分散排布指令 |
| `buildAdaptiveSelectionPrompt()` | 自适应选择：标签随章节演变 |
| `buildReducePrompt()` | 精减阶段：候选句 → 核心叙事 |
| `DISCOVER_SYSTEM` + `DISCOVER_SCHEMA` | 零样本标签发现 |
| `REDUCE_SYSTEM` + `REDUCE_SCHEMA` | 精减阶段输入/输出 |
| `ADAPTIVE_SYSTEM` + `ADAPTIVE_SCHEMA` | 自适应标签选择 |

**校验器：** 每个 schema 都有对应的校验器（`validateDocumentSelection`、`validateReduce`、`validateAdaptiveSelection` 等），检查类型、范围、唯一性和叙事角色覆盖。校验器包含重试逻辑——失败后调用方会重试一次，附带"你之前返回的 JSON 不合法"提示。

### 4.4 `src/reader/adapter.ts` — PDF 阅读器集成

**角色：** 唯一触及 Zotero/pdf.js 内部的文件。处理提取、叠加绘制、EPUB 绘制。

**关键函数：**
| 函数 | 用途 |
|---|---|
| `extractPage()` | 通过 pdf.js 的 `getPageData` 获取一页的字符 + 几何信息 |
| `buildStructuredPage()` | 将字符按基线聚类成视觉行，检测双栏，分割中缝，过滤元数据 |
| `installOverlays()` | 在渲染好的 PDF 页面上绘制高亮 div，附带边缘标记 |
| `installEpubOverlays()` | 通过 CSS Custom Highlight API 绘制（非破坏性，不改 DOM） |
| `epubAnnotationsFromSpecs()` | 将高亮规格转换为 EPUB CFI 格式的注释 |

**复杂的内部逻辑：**
- `visualLines()` 按基线邻近度（而非 PDF 对象顺序）聚类字符，处理 RTL
- `readingOrder()` 检测多栏布局并按先读左栏后读右栏排序
- `buildStructuredPage()` 过滤掉：元数据行、按字号判断的标题、关键词列（第 0 页）、标题/作者文本和装饰性行
- 双栏中缝通过统计方法检测（≥3 个对齐行上的重复间隙）

### 4.5 `src/modules/readerSection.ts` — 侧边栏 UI

**角色：** 渲染 Zotero 右侧边栏中的 SkimRead 面板。所有用户交互都流经此处。

**关键 UI 元素：**
| UI 元素 | 行为 |
|---|---|
| 提供商状态 | 点击可重新检测，显示绿/红/黄色 |
| 标签模式选择器 | 默认/自定义/自动 —— 切换后下一次 Generate 生效 |
| Generate 按钮 | 运行中禁用，首次运行后显示"Regenerate" |
| Cancel/Pause/Resume | 同一个按钮位，文字随状态变化 |
| Clear | 放弃部分完成的运行或清除已完成的覆盖层 |
| 保存为 Zotero 注释 | 将覆盖层转换为永久高亮 |
| 标签复选框 | 按标签类别切换可见性，附带计数 |
| 核心/上下文视图切换 | 在核心叙事和完整上下文间切换 |
| 不透明度滑块 | 10–80% 覆盖层透明度 |
| 边缘标记开关 | 显示/隐藏 PDF 边缘的标签徽章 |
| TL;DR 区域 | 生成 TL;DR 并保存为笔记 |
| EPUB 高亮列表 | 可点击的句子列表，用于导航 |

**回调接口**（runSkim 和 UI 之间的桥梁）：
```typescript
interface RunCallbacks {
  onStatus: (msg: string) => void;
  onDone: (count: number) => void;
  onError: (msg: string) => void;
  onCounts: (counts: Record<string, number>) => void;
  onLabels: (labels: LabelDef[]) => void;
  onTldr: (tldr: string) => void;
  onHighlights: (specs: HighlightSpec[]) => void;
}
```

### 4.6 `src/modules/zoning.ts` — 统计文档分析

**角色：** 零配置、语言无关的页面装饰（页眉/页脚）和参考文献区检测。基于重复频率和信号模式，而非布局规则或标题文本。

**关键技术：**
- **装饰行：** 一份文本（去掉数字 + 转小写）在 ≥30% 的页面上出现在相同基线行 → 页眉/页脚，从候选句中排除。
- **参考文献：** 连续尾部页面中 ≥50% 的实质性行包含信号模式（年份如"2023"、"et al."、DOI、条目编号"\[12\]"、"Smith, J."）→ 参考文献区。

---

## 5. 关键数据类型

```typescript
// PDF 提取后的基本文本单元
interface PageText {
  pageIndex: number;
  text: string;               // 组装的散文文本，空格根据几何信息合成
  charMap: number[];          // 文本位置 i → chars 索引（-1 表示空白）
  chars: PageChar[];          // 每个字符的几何信息：c, rect, baseline, fontSize
}

// 断句 + 章节标注后的句子
interface ExtractedSentence {
  pageIndex: number;
  startChar: number;          // PageText.chars 中的索引
  endChar: number;
  text: string;
  section: DocumentSection;   // "front matter" | "abstract" | "introduction" | "methods" | "results" | "discussion" | "conclusion" | "appendix" | "references" | "body"
}

// LLM 分类后的句子
interface ClassifiedSentence extends ExtractedSentence {
  label: string;              // 标签键，或 "none"
  confidence: number;         // 0–1
  core?: boolean;             // 由 reducePass 标记
}

// 持久化的运行结果 + 缓存
interface CachePayload {
  promptVersion: number;
  model: string;
  labels: LabelDef[];
  sentences: ClassifiedSentence[];
  complete: boolean;
  classifiedCount: number;
  chunkBudget?: number;
  savedAnnotationKeys?: string[];
  tldr?: string;
  pageSpan?: number;
  tokensInput?: number;
  tokensOutput?: number;
  extracted?: ExtractedSentence[];  // 提取后立即持久化，暂停恢复时无需重新提取
  extractComplete?: boolean;
  reduced?: boolean;
  coreCount?: number;
}

// 叠加层渲染规格（调用方无需了解位置/DOM 信息）
interface HighlightSpec {
  pageIndex: number;
  startChar: number;
  endChar: number;
  colorRGB: string;           // "r, g, b"
  label: string;
  text: string;
}

// 标签定义（来自默认、自定义或自动发现）
interface LabelDef {
  key: string;                // 缩写，例如 "theory"
  name: string;               // 显示名称，例如 "Theory"
  color: string;              // "r, g, b"
  description: string;
  antiDescription?: string;   // 该标签"不是"什么（用于区分近义标签）
  examples?: string[];
}
```

---

## 6. 提供商架构

```
chatJSON({model, system, user, schema})
  │
  ├─ apiType === "codex-app-server" ─── codexJSON()
  │   ├── CodexAppServerClient.start()     ← 生成 codex app-server 子进程
  │   ├── client.request("thread/start")   ← 临时线程，approvalPolicy: "never"
  │   ├── client.request("turn/start")     ← effort 来自偏好设置，outputSchema 来自传参
  │   ├── waitForCodexTurn()               ← 流式接收 delta，等待 turn/completed
  │   └── client.close()
  │
  ├─ apiType === "claude-code" ─── claudeCodeJSON()
  │   ├── 生成子进程：claude -p --output-format json --model M --allowedTools ""
  │   └── 解析 stdout 为 JSON
  │
  ├─ apiType === "antigravity-cli" ─── antigravityJSON()
  │   ├── 生成子进程：agy --model M --wait
  │   └── 解析 stdout 为 JSON
  │
  └─ 默认（openai-compatible）─── HTTP REST
      ├── Ollama 原生：POST /api/chat 带 format=schema
      └── OpenAI 兼容：POST /chat/completions 带 response_format=json_schema
```

**同意模型：** 每次云端提供商调用都会经过 `requireConsent()`，如果 `cloudConsent` 偏好未设置则抛出异常。本地 localhost 端点（通过 URL 检测）跳过此检查。

---

## 7. 缓存策略

**缓存键**（SHA-256）：
```
sha256(fileHash | providerLabel | model | labelMode | labelKeys | extractVersion | promptVersion)
```

模型、标签或提示词的任何变更 → 缓存不命中 → 完全重新运行。

**缓存文件**：`Zotero.DataDirectory/skimread-cache/<key>.json`

**可恢复性：** 每个波段处理后，通过 `cacheWrite()` 持久化部分结果。恢复时通过 `classifiedCount` 跳过已处理的波段。提取结果也会持久化（`extracted` 字段），确保最慢的 PDF 文本提取过程不会重复。

**三种缓存状态：** `"none"` | `"partial"` | `"complete"` —— 驱动按钮文字和恢复逻辑。

---

## 8. 进度 / UX 反馈（当前状态）

唯一的进度反馈机制是 `cb.onStatus(msg)` → `setProgress(body, msg)`，设置侧边栏中 `<div>` 的 `textContent`。

**运行过程中发送的状态消息：**
1. "Extracting text…" —— PDF 逐页提取
2. "Zoning: X repeated header/footer lines, references from page Y" —— 分区分析
3. "Reading document to propose labels…" —— 自动标签发现（第 1 次 LLM 调用）
4. "Selecting highlights — part X of Y…" —— 每个波段（第 2..N 次 LLM 调用）
5. "Choosing the sentences that narrate the document…" —— 精减阶段（第 N+1 次 LLM 调用）

**关键问题：** 在步骤 4 和 5 之间，每次 LLM 调用期间状态消息完全冻结（10–60 秒，取决于提供商/模型）。**单次 LLM 调用内部没有任何增量进度反馈。** 这是主要的用户体验问题。

---

## 9. EPUB vs PDF 路径

| 方面 | PDF | EPUB |
|---|---|---|
| 文本提取 | `extractPage()` —— pdf.js 的 `getPageData`，字符几何信息 | `extractEpubSections()` —— JSZip，解析 OPF spine，剥离 HTML |
| 章节检测 | `sectionAtPageStart()` 标题正则 + `withDocumentSections()` | EPUB 章节目录本身就是自然章节 |
| 高亮绘制 | `installOverlays()` —— 画布上的绝对定位 div | `installEpubOverlays()` —— CSS Custom Highlight API，非破坏性 |
| 保存注释 | `saveNativeHighlights()` —— 页面矩形坐标 | `saveEpubHighlights()` —— CFI，通过阅读器的 getAnnotationFromRange |
| 导航 | 滚动到页面 | `navigateToText()` —— 在页面中查找文本 |

---

## 10. 构建系统

- **构建工具：** `zotero-plugin-scaffold` v0.8.2 / esbuild
- **入口：** `src/index.ts` → 打包到 `.scaffold/build/addon/content/scripts/skimread.js`
- **输出：** `.scaffold/build/skim-read.xpi`
- **构建命令：** `npm run build`（打包 + tsc --noEmit）
- **环境变量：** `__env__` 在构建时注入；控制控制台日志级别

---

## 11. 痛点与未来方向

### 当前局限

1. **LLM 调用内部没有流式反馈** —— `waitForCodexTurn` 收到了 delta 但不对外暴露
2. **波段串行处理** —— 每个波段创建新的 LLM 请求；波段之间不重叠
3. **没有"故事模型"** —— 精减阶段是对全局理解的弱近似，只是在已选候选句中标记核心/上下文
4. **高亮没有解释** —— 用户看到有颜色的句子，但不知道**为什么被选中**
5. **每次调用都新建 Codex 子进程** —— 不复用客户端；每次调用：启动 app-server → 初始化 → 创建线程 → 轮次 → 关闭

### 计划的 v2 架构（三段式叙事管线）

参见 2026-09-20 的完整设计讨论。

**第 1 阶段——章节总结（并行）：**
- 按章节（Introduction, Method, Results 等）分别进行 AI 调用
- 输出 `{ section, purpose, problem?, keyClaim?, importantSentenceIds[] }`
- 可通过 `Promise.all()` 并行化

**第 2 阶段——全局融合（1 次串行 AI 调用）：**
- 所有章节总结 → AI 生成 `StoryModel`（叙事节点）
- `[{ step, label, description, sourceChapters }]`

**第 3 阶段——定位高亮（0–1 次 AI 调用）：**
- 故事节点 → 匹配原始句子（embedding 或 AI）
- 输出 `ExplainedHighlight{ sentenceId, storyNode, explanation, importance }`

**新的缓存结构：** `StoryCache` 包含章节总结、故事模型和解释性高亮 —— 分别缓存，支持部分缓存命中。

---

## 12. 测试

- 测试框架：mocha + chai，通过 `zotero-plugin-scaffold test` 执行
- 测试在 Zotero 内部运行（插件必须已加载）
- `test/` 目录下的测试文件：
  - `startup.test.ts` —— 插件实例是否存在
  - `document-selection.test.ts` —— 提示词构建、schema 校验
  - `segmentation.test.ts` —— 句子分割测试
  - `annotations.test.ts` —— 注释保存逻辑
  - `provider-output.test.ts` —— LLM 输出解析
  - `reader-structure.test.ts` —— 页面文本构建、双栏检测