# PaperOrchestra AI 架构文档套件

> 源码快照：`paper-orchestra` @ `d9bae997e793a015ed38bb1605be9eb58c00284f`（`d9bae99`）。以下只描述该快照可验证的实现。

## 1. 项目定位

PaperOrchestra 是 Google Cloud AI Research 的多智能体论文写作框架。源码 README 将其定义为：把“不受约束的写作前材料”（研究想法与实验日志）转换为可投稿 LaTeX 稿件，并自动生成文献综合与可视化图表。仓库采用 Apache-2.0，README 明确声明“不是 Google 官方支持产品”。

它解决的问题是 **写作阶段的自动化**：给定已有 idea 与 experimental log，生成 outline、Introduction/Related Work、正文、图表和精炼后的 PDF。它不负责提出研究想法、生成实验代码或执行真实实验；这些需要由用户提供或由其他系统（例如同工作区的 AutoResearchClaw）完成。

一句话职责边界：

```text
输入：idea.md + experimental_log.md + 已有/生成图片 + 会议 LaTeX 模板
输出：outline.json、citations/BibTeX、LaTeX 源码、最终 PDF、评审与精炼记录
```

源码支持的典型应用场景：

- 已有方法与实验日志，需要快速生成符合会议模板的初稿；
- 需要从 outline 自动展开文献检索、Introduction 与 Related Work，并严格限制文献时间；
- 需要为论文生成统计图或概念图，并让 VLM 参与图表/版面审查；
- 需要通过 Streamlit 界面交互式提供材料、查看进度和下载 PDF；
- 需要通过 `run_batch_writeup()` 批量处理多篇论文并利用 log 断点续跑。

## 2. 与“全自动科研体系”的关系

在端到端科研体系里，PaperOrchestra 位于 **实验完成后、投稿前的论文生产层**：

| 能力 | PaperOrchestra | AutoResearchClaw（仅作对照） |
|---|---|---|
| 研究主题/假设生成 | 不负责，消费 `idea.md` | 23 阶段流水线的 Stage 1/8 |
| 实验设计与执行 | 不负责，消费 `experimental_log.md` | Stage 9–15 |
| Outline / Related Work / 正文 | 核心能力 | Stage 16–19 也有类似职责 |
| 文献检索 | Google Search + Semantic Scholar，按 outline 任务检索并强制时间 cutoff | OpenAlex/S2/arXiv 等独立检索阶段 |
| 图表 | PlottingAgent：统计图走本地 matplotlib 代码执行，概念图走 Gemini 图像生成 | FigureAgent（代码图/Nano Banana）在 Stage 14 |
| 精炼 | AgentReview 风格审稿 + 多轮内容/格式精炼 | Stage 18–20 评审/质量门 |
| 最终格式 | 直接复制并填充会议 LaTeX 模板 | Markdown→LaTeX 转换器 + 会议模板注册表 |

两个仓库没有代码级依赖或共享模块；此对照只帮助零上下文 AI 理解职责分工。

## 3. 核心执行路径

```text
paper_writing_cli.py
  ├─ 无 plotting → methods/paper_writer.py:write_single_paper
  │    OutlineAgent → HybridLiteratureAgent → SectionWritingAgent → ContentRefinementAgent
  └─ 有 plotting → methods/paper_writer_with_plotting.py:write_single_paper
       OutlineAgent → (HybridLiteratureAgent ∥ PlottingAgent)
                   → SectionWritingAgent → ContentRefinementAgent
```

管线整体包一层 `max_n_tries = 3` 的异常重试。每一步以磁盘文件为交接点，便于 Streamlit 前端展示中间产物与失败状态。

## 4. 文档导航

| 文档 | 回答的问题 |
|---|---|
| `architecture.md` | 多 Agent 乐队式协作：大纲、综述、正文、绘图、精炼各 agent 的职责与依赖 |
| `pipeline_and_dataflow.md` | 原始日志/笔记如何变成 LaTeX、图表、PDF，以及批处理日志结构 |
| `modules_reference.md` | 核心模块、Prompt 设计、函数/类接口与 LLM backend 选择 |
| `execution_guide.md` | 为什么无需 GPU、环境变量、CLI/Streamlit/批处理运行方法 |
| `agent_orchestration_deep_dive.md` | 五大 Agent 内部执行流、代码调用链、`outline.json`/`citation_map.json`/`figures/info.json` 等数据传递对象 |
| `prompts_and_templates_reference.md` | 各 Agent 完整 System/User Prompt、CVPR/ICLR 插值契约、自定义实验日志与 LaTeX 模板二开教程 |
| `troubleshooting.md` | API 报错与重试、matplotlib `exec()` 失败和沙箱边界、缺失宏包、图表引用修复 |

## 5. 源码地图

| 路径 | 作用 |
|---|---|
| `paper_writing_cli.py`, `paper_writing_cli.sh` | 单篇 CLI 与示例批处理启动脚本 |
| `methods/paper_writer.py` | 无绘图路径的单篇编排与 `run_batch_writeup()` |
| `methods/paper_writer_with_plotting.py` | 有绘图路径，文献与绘图并行执行 |
| `methods/agents/outline_agent.py` | 结构化大纲/图表计划生成 |
| `methods/agents/literature_review_agent.py` | 检索、去重、BibTeX、Introduction/Related Work 写入 |
| `methods/agents/section_writing_agent.py` | 多模态补全剩余 LaTeX 章节 |
| `methods/agents/plotting_agent.py` | 图表计划并行生成 |
| `methods/agents/content_refinement_agent.py` | 内容评审、格式评审、修订循环 |
| `methods/prompts/*`, `utils/prompt_utils.py` | Prompt 模板与匿名化约束 |
| `utils/llm_backend_utils.py`, `utils/gemini_utils.py`, `utils/openai_utils.py` | LLM 路由、重试、解析 |
| `utils/pdf_utils.py`, `utils/paper_banana_utils.py`, `utils/scholar_utils.py` | PDF 编译/截图、绘图流水线、Semantic Scholar 查询 |
| `autoraters/*` | AgentReview 风格评审及若干自动评分脚本 |
| `templates/cvpr2025`, `templates/iclr2025` | 当前随仓库提供的会议模板 |
| `frontend/app.py`, `frontend/frontend_utils.py` | Streamlit 交互 Demo |

## 6. 维护约定

- 所有路径相对仓库根目录；配置/函数引用写成 `path:symbol`。
- 用户任务描述中提到 NeurIPS/ICML 输出，但**当前仓库实际只内置 CVPR2025 与 ICLR2025 模板目录**；neurips/icml 仅在 Prompt 文本中作为会议风格示例出现。要生成对应格式必须自备模板目录，`docs/pipeline_and_dataflow.md` 说明了自定义模板要求。
- CLI 帮助写着“generate figures from code”，但源码中只有“统计图”走本地 matplotlib 代码执行，概念图走 Gemini 图像生成；文档按源码描述。
