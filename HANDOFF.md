# 接班报告

> 生成于 2026-09-20，会话结束时的项目状态快照。

---

## 已完成的工作

### 项目分析

- 完整阅读了所有 17 个 TypeScript 源文件
- 梳理了核心数据流：PDF/EPUB 提取 → 断句 → 章节标注 → band/chunk 分割 → LLM 逐段选择 → 精减 → 绘制
- 定位了所有关键模块的职责和接口边界
- 做了 Codex 并行实测（3 请求串行 68s vs 并行 39s，有部分并行能力）

### 已创建的文件

| 文件 | 用途 |
|---|---|
| `PROJECT_CONTEXT.md` | 完整的项目技术上下文（711 行，已翻译为中文） |
| `CHANGELOG.md` | 版本更新日志（空架子，后续填入） |
| `HANDOFF.md` | 本文件，当前接班报告 |

### 架构决策（待你审核）

确定了 v2 三段式管线方向：
1. **章节总结**（可并行）→ 每章输出小总结 + 重要句子编号
2. **全局融合**（1 次 AI 调用）→ 建故事模型
3. **定位高亮**（0–1 次 AI 调用）→ 按故事节点匹配句子 + 生成解释卡

三个待定的设计决策：
- ① 章节总结并行还是串行？（实测有部分并行能力，建议分 2 批）
- ② 搜索定位用 embedding 匹配还是 AI 再挑一次？
- ③ 解释卡放 Zotero 注释里还是右侧边栏？

---

## 代码现状

- 构建通过：`npm run build` → `.scaffold/build/skim-read.xpi`
- 当前版本：0.2.5
- 依赖已安装（`npm install` 完成）
- 已部署到桌面：`~/Desktop/skim-read.xpi`
- 主要输出目录：`.scaffold/build/`

---

## 已知问题

1. **Codex 模式下没有流式进度反馈** — `waitForCodexTurn` 收到 delta 但不对外暴露，用户每次 LLM 调用（10–60s）期间看不到任何进展
2. **每次 Codex 调用都重新创建子进程** — 无客户端复用，10 次调用 = 10 次 `codex app-server` 启动/关闭
3. **当前架构缺乏"论文理解"** — 只在句子层面挑重要句子，没有先建故事模型再定位，导致高亮质量天花板
4. **高亮没有解释** — 用户看到彩色句子但不知道为什么被选中

---

## 待完成

### 短期

- [ ] 确定三个设计决策（并行策略、搜索方法、解释卡展示方式）
- [ ] 拆解实施步骤，逐个审核

### 中期（v2 三段式架构）

- [ ] 实现第 1 阶段：章节切分 + 并行章节总结
- [ ] 实现第 2 阶段：全局融合建故事模型
- [ ] 实现第 3 阶段：搜索定位生成高亮 + 解释卡
- [ ] 重写缓存结构（StoryCache 替代 CachePayload）
- [ ] 重写侧边栏 UI（故事面板 + 解释卡列表）

### 基础设施

- [ ] `chatJSON` 增加 `onProgress` 回调（通用改进，无论新旧架构都需要）
- [ ] Codex 客户端复用（在 runSkim 层面共享 CodexAppServerClient）

---

## 关键文件索引

| 文件 | 行数 | 核心作用 |
|---|---|---|
| `src/modules/skim.ts` | 1816 | 核心管线编排，`runSkim()` 入口 |
| `src/llm/ollama.ts` | 1339 | 所有 LLM 提供商实现，`chatJSON()` 分发器 |
| `src/prompts/skim.ts` | 781 | 所有 LLM 提示词、Schema、校验器 |
| `src/reader/adapter.ts` | 1361 | PDF 提取 + 叠加绘制，EPUB 绘制 |
| `src/modules/readerSection.ts` | 825 | 右侧边栏 UI 渲染和交互逻辑 |
| `src/modules/zoning.ts` | 197 | 统计检测页眉页脚和参考文献区 |
| `src/modules/annotations.ts` | 185 | 覆盖层 → Zotero 注释的转换 |
| `src/modules/preferences.ts` | 317 | 设置面板逻辑 |
| `src/reader/epub.ts` | 133 | EPUB 文本提取（JSZip + OPF 解析） |

---

## 连接信息

- 本机 Codex CLI 已安装：`/usr/local/bin/codex`
- Codex 已登录：`~/.codex` 目录存在全局状态
- 实验用的测试脚本：`/tmp/test-codex-parallel.sh`（并行对比测试）
- 模型：`gpt-5.6-luna`（默认）、`medium` reasoning