# PaperOrchestra 排障指南

> 源码快照：`d9bae99`。本文按错误发生层次组织：API/解析 → 绘图执行 → LaTeX 编译 → 图表引用 → 批处理与模板。

## 0. 先看产物再猜原因

```text
base_dir/
├─ outline.json                         # OutlineAgent 是否产出合法 JSON
├─ literature_agent_output/
│  ├─ outline_v1.json                   # citation_candidates 是否回填
│  ├─ updated_template.tex              # Intro/Related Work 是否写入
│  ├─ references.bib                    # BibTeX 是否生成
│  └─ citation_map.json                 # 后续 Agent 的引用白名单
├─ plotting_results.json                # 绘图结果（仅 --use_plotting）
├─ latex_writeup/
│  ├─ template.tex                      # 已被 LiteratureAgent 覆盖
│  ├─ references.bib
│  ├─ raw_draft_paper.tex               # SectionWritingAgent 输出
│  ├─ figures/info.json                 # 绘图路径才有
│  └─ figures/*                         # 图像文件
├─ content_refinement_workdir/
│  ├─ initial_draft.pdf
│  ├ refined_paper_vN.tex/.pdf
│  ├ peer_reviews/review_vN.json
│  ├ content_refinement_worklog.json
│  ├ format_refinement_worklog.json
│  └ final_refined_paper.pdf/.tex
└─ final_paper.pdf                      # 最终复制产物
```

`write_single_paper()` 外层最多重试 3 次，每次打印 `[Try N/3] EXCEPTION in perform_writeup:` 和完整 traceback。多数 Agent 内部也会吞异常并继续，所以“没有 exception”不等于成功；必须检查最终 PDF 是否存在。

---

## 1. API 报错处理

### 1.1 凭模型名路由

`utils/llm_backend_utils.py:get_llm_parser()` 与 `call_llm_with_*()` 按 model name 字符串判断：

| 模型名包含 | 后端 | 客户端/凭据 |
|---|---|---|
| `gpt`, `o1`, `o3`（小写匹配） | OpenAI | `utils/openai_utils.py:get_openai_client()` 要求 `OPENAI_API_KEY` |
| 其他（默认 Gemini） | Gemini | `utils/gemini_utils.py:get_genai_client()` |

Gemini 客户端初始化优先级：

1. `VERTEX_AI_PROJECT` + `VERTEX_AI_LOCATION` 同时存在 → `genai.Client(vertexai=True, ...)`。
2. 否则 `GEMINI_API_KEY` → `genai.Client(api_key=...)`。
3. 都没有 → `ValueError("Either VERTEX_AI_PROJECT/VERTEX_AI_LOCATION or GEMINI_API_KEY must be set to use Gemini models.")`。

`.env` 从 `utils/../.env`（仓库根）加载。客户端是懒加载并加锁的单例，因此 OpenAI-only 环境不需要 Gemini key，反之亦然。

修复步骤：

```bash
# 仓库根目录 .env
GEMINI_API_KEY=...
# 或 Vertex AI
VERTEX_AI_PROJECT=...
VERTEX_AI_LOCATION=us-central1
# OpenAI 模型另行配置
OPENAI_API_KEY=...
```

注意：如果用 Vertex AI，需要额外配置 ADC（`gcloud auth application-default login` 或服务账号），源码只读取 project/location 两个变量。

### 1.2 重试与退避

`call_gemini_with_contents()`、`call_gemini_with_images()`、`call_gemini_with_text_prompt()`：

- `max_retries = 5`，`base_interval_sec = 5`。
- 首次失败延迟 0，之后按 `5 * 2^(attempt-1)` 退避（5s、10s、20s、40s）。
- 错误信息包含 `maximum number of tokens allowed`、`400 bad request`、`code: 400`、`400 invalid_argument` 时**立即抛出，不重试**，因为重试同样的输入无意义。
- 其他错误在第 5 次失败后抛出。
- `check_parsed_response_not_none=True` 时，解析失败会触发 `AssertionError` 并进入相同重试。

OpenAI 路径同样 5 次重试，但没有“400 不重试”的特判。

典型日志：

```text
Warning: Gemini API call failed on attempt 2 of 5. Error: ...
Detected input token count exceed. Not retrying.
Error: All 5 retry attempts failed.
```

处理：

- token 超限：减小 `experimental_log.md`、减少图数量、降低 `plotting_max_critic_rounds`，或换更大上下文模型。
- 解析失败：确认模型输出用 ```` ```json ```` / ```` ```latex ```` 包裹；`parse_gemini_json_results()` 会尝试去控制字符、尾逗号、非法反斜杠后重解析。
- 认证失败：检查 key/Vertex 配置；不要在模型名里混入空格。
- 429/5xx：等待退避；若持续失败，检查配额与区域。

### 1.3 Semantic Scholar / Google 搜索

`utils/scholar_utils.py`：

- 模块导入时读取 `SEMANTIC_SCHOLAR_API_KEY` 并直接打印（`Using S2 API Key: ...`）；日志中要注意不要外泄。
- `s2_title_search()` 单次请求 timeout 5s，非 200 返回 `None`；异常只打印警告。
- 标题相似度用 `thefuzz.fuzz.ratio`，阈值 >70；年份提示 +10。
- `cutoff_date` 在 S2 结果与 `HybridLiteratureAgent._is_paper_allowed()` 中各检查一次。

`HybridLiteratureAgent._process_search_task_google()` 对单个 search task 最多重试 3 次；失败仅打印 `Google Search failed for ...`。若大量文献缺失，先看 `outline.json.intro_related_work_plan` 的查询是否过窄，再检查 Google Search 工具配额。

### 1.4 AgentReview 与 PDF 读取

`ContentRefinementAgent._get_peer_review()` 中 `perform_review_agentreview()` 失败会返回 `{"Error": ..., "Overall": 0}`，循环会继续，但可能错误判定“分数下降”。看到 `Error generating review` 时先修 PDF/API，不要相信后续 worklog 的分数。

---

## 2. matplotlib 绘图代码 `exec()` 失败与沙箱安全

### 2.1 源码行为

`utils/paper_banana_utils.py:execute_plot_code_worker()`：

```python
match = re.search(r"```python(.*?)```", code_text, re.DOTALL)
code_clean = match.group(1).strip() if match else code_text.strip()

plt.switch_backend("Agg")
plt.close("all")
plt.rcdefaults()

try:
    exec_globals = {}
    exec(code_clean, exec_globals)
    if plt.get_fignums():
        buf = io.BytesIO()
        plt.savefig(buf, format="jpeg", bbox_inches="tight", dpi=300)
        ...
        return base64.b64encode(img_bytes).decode("utf-8")
    else:
        return None
except Exception as e:
    print(f"Error executing plot code: {e}")
    return None
```

因此失败拦截的含义是：

- 任何 Python 异常都被捕获，只打印一行错误；函数返回 `None`，不会中断 PlottingAgent。
- 代码没有创建 figure（`plt.get_fignums()` 为空）也返回 `None`。
- 返回 `None` 时，critic 阶段收到 `[SYSTEM NOTICE] The plot image could not be generated ...`，尝试根据描述生成更简单的版本；最多 `plotting_max_critic_rounds` 轮（默认 3）。
- 如果最终仍无图片，`PlottingAgent.run()` 不会写 `image_path`，writer 也就不会把它加入 `figures/info.json`。

### 2.2 排查步骤

1. 在终端搜索 `Error executing plot code:`；这是最直接的异常来源。
2. 检查 `plotting_results.json`：有 `image_path` 才算成功保存；只有 `base64_img` 的中间结果不会出现在最终结果中。
3. 常见错误：
   - `NameError`：描述里引用了 log 中不存在的列名/数组。
   - `KeyError`：JSON 数据字段与代码假设不一致。
   - `ValueError` / 空数据：原始数据选择错误；检查 `process_single_figure()` 的 `data_source` 与 raw material 文件名匹配。
   - 中文字体缺失导致方框：字体问题不会让 exec 失败，但影响可读图；安装字体或改用英文标签。
   - 没有 `plt.figure()/plt.subplots()`：LLM 只写了数据处理代码，critic 应要求重生成。
4. 用更明确的 `objective` 重写 `outline.json.plotting_plan` 后重跑；不要只重跑 SectionWriting，因为图是在它之前生成的。

### 2.3 安全边界（必须知道）

`exec(code_clean, exec_globals)` 在**当前 Python 进程**执行 LLM 生成代码：

- 没有沙箱、没有 syscall 过滤、没有文件系统/网络隔离。
- `exec_globals` 是空字典，但内置 `__builtins__` 会自动注入；代码可以打开文件、发网络请求、修改进程状态。
- `plt.switch_backend("Agg")` 只是关闭交互后端，不是安全边界。
- critic 失败保护只防“图表生成失败”，不防恶意代码。

建议在受信任研究环境中运行；若处理不可信输入，应在外部容器/VM 中运行整个 CLI，并限制网络、CPU、内存和文件系统挂载。不要误以为 `--use_plotting` 自带隔离。

---

## 3. LaTeX 编译与缺失宏包

### 3.1 编译流程

`utils/pdf_utils.py:compile_latex(cwd, pdf_file, texfile_name, timeout=30)`：

```text
删除 <texfile_name>.pdf（若存在）
清空 TEXINPUTS / TEXMFHOME / TEXMF / TEXMFVAR / TEXMFCONFIG
1. pdflatex -interaction=nonstopmode <texfile_name>.tex
2. bibtex <texfile_name>
3. pdflatex -interaction=nonstopmode <texfile_name>.tex
4. pdflatex -interaction=nonstopmode <texfile_name>.tex
若生成 PDF 且无 TIMEOUT → 移动到 pdf_file
否则用 rich 打印每条命令的 stdout/stderr
```

`ContentRefinementAgent.run()` 在初稿、每轮 refinement、formatting candidate 和最终 PDF 中反复调用它。初稿编译失败时直接返回错误字符串，顶层 writer 的 `assert exists(final_pdf)` 会触发 3 次整体重试。

### 3.2 常见错误

| 错误 | 原因 | 处理 |
|---|---|---|
| `! LaTeX Error: File 'cvpr.sty' not found.` | 模板目录缺样式文件，或 `latex_writeup/` 中没有复制过来 | 把 `.sty/.bst/.tex` 依赖完整放进 `latex_template_dir`；检查 `latex_writeup/` |
| `! LaTeX Error: File 'math_commands.tex' not found.` | ICLR 模板依赖未复制 | 同上；注意 `\input{}` 使用相对路径 |
| `! I couldn't open style file ieeenat_fullname.bst` | bib style 缺失 | 复制 `ieeenat_fullname.bst`；确认 `\bibliographystyle{}` 拼写 |
| `Citation ... undefined` | bibtex 步骤失败或 key 不在 `references.bib` | 先修 bibtex 错误；确认 `\cite{key}` 的 key 来自 `citation_map.json` |
| `There were undefined references` | 多遍编译未完成或 label/ref 不匹配 | 确认四步命令都执行；检查 `\label/\ref` |
| `TIMEOUT after 30s` | 图表过大、宏包过多或机器慢 | 简化图/表；在外部提高 timeout（需改源码） |
| `PDF Generation Failed (No PDF file produced)` | 汇总性失败 | 向上查看每条命令的 rich 输出，不要只看最后一行 |

`compile_latex()` 会清除 TeX 环境变量，因此本机 `TEXINPUTS` 中的自定义宏包不会被使用；需要把宏包放进项目目录或安装到 TeX 树。它也不会自动 `tlmgr install`，缺失宏包必须人工安装。

### 3.3 模板相关排障

- 自定义模板缺少 `%%%%%%%%% INTRODUCTION %%%%%%%%%` 等标记，LiteratureAgent 可能不知道在哪里写 Intro/Related Work。
- `guidelines.md` 缺失会触发 writer 的断言：`No guidelines file detected at ...`。
- `template.tex` 缺失触发 `No template file detected at ...`。
- `idea.md`/`experimental_log.md` 缺失触发对应断言；CLI 默认 idea 文件名是 `idea_sparse.md`，而 writer 函数默认值是 `idea_dense.md`，显式传 `--idea_filename` 可避免混淆。
- Prompt 明确禁止把 `cleveref` 改成 `cleverref`；没有 `cleverref.sty`。

---

## 4. 图表未按预期引用

### 4.1 源码里的三层约束

1. **Outline**：`plotting_plan` 决定生成哪些图；`figure_id` 不得含 `Figure`，建议语义化。
2. **SectionWriting prompt**：必须使用 `figures_list` 中所有图、用精确文件名、放在 `figures/` 下、正文要有 `\ref`/引用、不要合并图、caption 不写 `Figure x`。
3. **ContentRefinement**：VLM 审查 prompt 要求逐图逐表报告“near or before first reference”、不得在 Conclusion 之后，formatting loop 最多 1 轮且只改排版。

没有生产代码会自动扫描 LaTeX 并插入缺失引用；修复依赖重新生成或人工编辑。

### 4.2 修复路径

| 症状 | 检查 | 修复 |
|---|---|---|
| 生成的 LaTeX 完全没有图 | `plotting_results.json` 是否为空；`latex_writeup/figures/info.json` 是否存在 | 修 plotting_plan/数据后重跑；无绘图路径需手动放图并确保模板/prompt 提及 |
| 有图片文件但正文没引用 | `raw_draft_paper.tex` 中搜 `includegraphics` 与 `\ref` | 在 SectionWriting user prompt 或 system prompt 强化 “use ALL figures”；重跑 Section→Refinement |
| 引用了不存在的文件 | 对比 `info.json.name` 与 `\includegraphics{}` 路径 | 使用精确文件名；`name` 无后缀会补 `.png`，实际文件是 `.jpg` 时需写完整扩展名 |
| 图在 Conclusion 之后 | 检查章节顺序 | 按 format prompt 要求移到前面；或修改模板章节顺序 |
| caption 出现 “Figure 1:” 前缀 | `generate_figure_caption()` 正常会剥离；人工写入则可能残留 | 去掉前缀，让 LaTeX 编号 |
| 图片是概念图但不清晰 | `task_name` 是否被误判为 `plot` | 检查 `outline.json.plot_type`；`plot_type` 含 `chart/plot` 会走 matplotlib 代码路径 |
| 图数量少于计划 | 单图异常被 PlottingAgent 吞掉 | 看终端 `[ERROR] Failed processing <figure_id>` 与 critic 日志 |
| 图名重复/覆盖 | 文件名为 `figure_id` 小写化 | 给每个图唯一且语义化的 `figure_id` |

### 4.3 手工修复后的注意事项

如果已经在 `latex_writeup/raw_draft_paper.tex` 中手工修好引用，不要只重跑 `ContentRefinementAgent`：它会重新读取 `texfile_path` 指定的文件，`paper_writer.py` 传的就是这个文件；但更上游的 Literature/Section 重跑会覆盖它。 simplest 的恢复点是：

```text
修 outline/figures/info.json → 重跑 Plotting（如需新图）→ 重跑 SectionWriting → 重跑 ContentRefinement
```

---

## 5. 批处理与断点续跑

`run_batch_writeup()`：

- 用 `ProcessPoolExecutor(max_workers=4)` 并发处理多篇论文。
- `paper_writing_log.json` 中 `status=success` 且 `paper_path` 存在的论文会被跳过。
- 失败条目会在下一次以相同 `output_dir` 运行时重跑。
- `_load_papers_from_config()` 支持目录模式（每篇一个子目录，含 `raw_materials/` 与 `<paper_id>.pdf`）或 JSON 配置模式。
- 相对 `content_folder/pdf_path` 会基于 `paper_info_path` 所在目录解析。

常见问题：

- 并发下 API 429 增多：降低 `max_workers`，或使用多 key/Vertex 配额。
- 某篇一直失败但日志不更新：看 `future.result()` 异常是否被 `Exception handling result for paper ...` 捕获；该分支只打印，不写 log。
- 续跑误跳过失败论文：确认 log 中的 `status` 不是旧的成功记录，且 `paper_path` 文件仍存在。

---

## 6. 快速命令清单

```bash
# 环境与编译
python -c "import google.genai, openai, matplotlib, fitz, cv2; print('deps ok')"
pdflatex --version && bibtex --version

# 单篇调试（把日志留在终端）
python paper_writing_cli.py \
  --raw_materials_dir ./my_materials \
  --latex_template_dir ./templates/cvpr2025 \
  --output_dir ./debug_run \
  --research_cutoff 2025-06 \
  --use_plotting false

# 只看最终产物
ls -l debug_run/final_paper.pdf
ls -l debug_run/content_refinement_workdir/final_refined_paper.pdf
```
