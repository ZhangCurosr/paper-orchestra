# PaperOrchestra 模块、Prompt 与接口参考

> 基于源码快照 `d9bae99`。本文面向接手开发的 AI：直接给出函数签名、调用关系和容易踩坑的实现细节。

## 1. CLI 接口

`paper_writing_cli.py:main()` 参数：

| 参数 | 默认值 | 说明 |
|---|---|---|
| `--raw_materials_dir` | 必填 | 输入目录；不存在则 exit 1 |
| `--latex_template_dir` | 必填 | 任意模板目录；需含 `guidelines.md` 与 `template.tex` |
| `--output_dir` | `./paper_output_<timestamp>` | 输出根目录 |
| `--idea_filename` | `idea_sparse.md` | idea 文件名 |
| `--experimental_log_filename` | `experimental_log.md` | 日志文件名 |
| `--writer_model_name` | `gemini-3.1-pro-preview` | outline/literature/section 模型 |
| `--reflection_model_name` | `gemini-3.1-pro-preview` | 精炼模型 |
| `--research_cutoff` | 当前 `YYYY-MM` | 文献时间上界 |
| `--use_plotting` | `False` | 接受 `true/1/yes` 字符串 |
| `--plotting_model_name` | `gemini-3.1-pro-preview` | 绘图文本/代码模型 |
| `--image_model_name` | `gemini-3-pro-image-preview` | 概念图图像模型 |
| `--plotting_max_critic_rounds` | `3` | 单图批评迭代上限 |

CLI 根据 `use_plotting` 动态 import：

```python
from methods.paper_writer import write_single_paper          # False
from methods.paper_writer_with_plotting import write_single_paper  # True
```

两个 `write_single_paper` 名字相同但实现不同，调试时要看清 import 路径。

## 2. 编排接口

```python
# methods/paper_writer.py
write_single_paper(base_dir, latex_template_dir,
                   idea_filename="idea_dense.md",
                   experimental_log_filename="experimental_log.md",
                   writer_model_name="gemini-3.1-pro-preview",
                   reflection_model_name="gemini-3.1-pro-preview",
                   research_cutoff="2024-11") -> bool

# methods/paper_writer_with_plotting.py
write_single_paper(base_dir, latex_template_dir,
                   idea_filename="idea_dense.md",
                   experimental_log_filename="experimental_log.md",
                   writer_model_name="gemini-3.1-pro-preview",
                   reflection_model_name="gemini-3.1-pro-preview",
                   plotting_model_name="gemini-3.1-pro-preview",
                   image_model_name="gemini-3-pro-image-preview",
                   research_cutoff="2024-11",
                   plotting_max_critic_rounds=3) -> bool
```

注意函数默认 `idea_filename="idea_dense.md"`，但 CLI 显式传入 `idea_sparse.md`。直接调用 Python API 时不传该参数会找错文件名。

批处理：

```python
run_batch_writeup(latex_template_dir, process_root_folder=True,
                  root_input_folder="", paper_info_path="", output_dir="",
                  output_dir_prefix="", date_str="",
                  idea_filename="idea_sparse.md",
                  experimental_log_filename="experimental_log.md",
                  writer_model_name="gemini-3.1-pro-preview",
                  research_cutoff="2024-11", n_papers=-1, max_workers=4)
```

## 3. Agent 类接口

### OutlineAgent

```python
OutlineAgent(model_name="gemini-3.1-pro-preview", cutoff_date="2024-10")
  .run(idea_file, experimental_log_file, latex_template_file,
       conference_guidelines_file, output_filepath=None) -> dict
```

内部用 `call_gemini_with_contents()`，把 system prompt 作为第一个 `types.Part`；`response_dict["parsed_response"]` 是 JSON 对象。输出文件为 `outline.json`。

### HybridLiteratureAgent

```python
HybridLiteratureAgent(idea_path, experimental_log_path,
                      latex_template_path, conference_guidelines_path,
                      output_dir, model_name="gemini-3.1-pro-preview",
                      max_workers=5)
  .run(outline_path, cutoff_date=None) -> None
```

重要内部方法：

| 方法 | 作用 |
|---|---|
| `_collect_search_tasks(outline)` | 从 intro strategy、related work subsections、section citation_hints 构造任务 |
| `_execute_tasks_in_parallel(...)` | Google discovery 10 workers；S2 enrichment 串行 1s/条 |
| `_discover_candidates(task, outline, cutoff_date)` | 固定用 `gemini-3-flash-preview` + Google Search tool |
| `_enrich_and_register_paper(...)` | `s2_title_search()`；标题规范化去重；裁剪 abstract 到 1500 字符 |
| `_is_paper_allowed(s2_data, cutoff_date)` | 按 publicationDate/year 过滤 cutoff |
| `_generate_key(authors_raw, year, title)` | 生成 `AuthorYearFirstTwoTitleWords` 风格 key |
| `_generate_bibtex(papers)` | `@article` 优先 journal，否则 `@inproceedings`；key 冲突加 a/b/c 后缀 |
| `_generate_citation_map(papers)` | 只有非空 abstract 的论文进入 citation_map |
| `_inject_citations_into_outline(...)` | 写 `citation_candidates` 并删除原始 `citation_hints` |
| `_synthesize_content(...)` | 只写 Introduction/Related Work，要求至少引用 90% 收集论文 |

Pydantic schema：`CandidatePaper(title, year, reason)`, `DiscoveryResult(section_name, candidates)`, `PaperData(...)` 等。

### PlottingAgent

```python
PlottingAgent(model_name="gemini-3.1-pro-preview",
              image_model_name="gemini-3-pro-image-preview",
              max_critic_rounds=3)
  .run(outline_json_path, raw_materials_dir, output_filepath=None) -> list[dict]

process_single_figure(fig_plan, raw_materials_dir, model_name,
                      image_model_name, max_critic_rounds) -> dict
```

返回项含 `figure_id,title,task_name,description,caption,aspect_ratio,critic_history,image_path`（`base64_img` 写盘后删除）。`utils/paper_banana_utils.py` 的关键函数：

- `retrieve_few_shot_examples(task_name, raw_content, description, model_name)`
- `predict_figure_content(...)`
- `style_figure_content(...)`
- `generate_figure_visuals(...)`：`plot` → LLM 生成 matplotlib 代码 → `execute_plot_code_worker()`；`diagram` → `generate_image_with_gemini()`
- `critique_and_revise_figure(...)`
- `generate_figure_caption(...)`

`execute_plot_code_worker()` 直接 `exec(code, {})`，无沙盒；从安全角度看这是受信任环境假设。

### SectionWritingAgent

```python
SectionWritingAgent(outline_path, template_path, idea_path,
                    experimental_log_path, citation_map_path,
                    figures_info_path, guidelines_path,
                    model_name="gemini-3.1-pro-preview")
  .run(output_path) -> str
```

辅助方法：`_read_file`, `_read_json`, `_format_figures_context`, `_format_citation_library`。图片路径从 `figures_info_path` 所在目录解析；只有 png/jpg 会加入多模态输入，pdf 不传。缺失文件返回 `"[Error: File ... not found]"`，但 `_read_json` 对缺失返回 `{}`。

### ContentRefinementAgent

```python
ContentRefinementAgent(experimental_log_path, citation_map_path,
                       guidelines_path, model_name="gemini-3.1-pro-preview",
                       max_reflections=3, work_dir="./paper_workspace")
  .run(texfile_path) -> str   # 返回 final_refined_paper.pdf 路径
```

内部状态：`current_tex`, `initial_score`, `current_score`, `worklogs`, `format_worklogs`。关键方法：

- `_get_peer_review(pdf_path, iteration_label)` → AgentReview JSON + Overall；
- `_get_formatting_review(pdf_path, iteration_label)` → VLM 版面 JSON；
- `_extract_scores(review)` → 9 个 axis 的 float。

接受规则见 `pipeline_and_dataflow.md`；不要移除编译成功检查，否则会把未编译的 LaTeX 当成最终版本。

## 4. Prompt 设计

| Prompt 文件 | 消费者 | 设计要点 |
|---|---|---|
| `methods/prompts/outline_agent.py:outline_agent_system_prompt` | OutlineAgent | 单 JSON 输出；plotting_plan/intro_related_work_plan/section_plan；严格枚举和 citation hints |
| `methods/prompts/literature_review_agent.py:literature_review_agent_writter_prompt` | HybridLiteratureAgent | 只填 Intro/Related Work；只能引用 collected_papers；cutoff 和 eval claim 规则；返回完整 template.tex |
| `methods/prompts/section_writing_agent.py:section_writing_agent_prompt` | SectionWritingAgent | 保留已有内容；booktabs 表格；cite keys；使用所有图片；full LaTeX |
| `methods/prompts/content_refinement_agent.py:content_refinement_agent_system_prompt` | ContentRefinementAgent | JSON worklog + full LaTeX；数字以实验日志为准；忽略要求新实验的评审意见 |
| `methods/prompts/format_agent.py:format_agent_system_prompt` | 当前未被 writer 主路径导入；定义格式 agent 风格 | 版面、页数、图表位置和 typo 规则 |
| `utils/prompt_utils.py:UNIVERSAL_NO_LEAKAGE_PROMPT` | 追加到多个 prompt | 只用会话材料、禁止训练记忆与作者身份泄露 |

Prompt 包装约定：LaTeX 输出用 ```` ```latex ````，JSON 输出用 ```` ```json ````，worklog+LaTeX 用两个连续代码块。解析器对代码块缺失有裸文本 fallback，但稳定性下降。

## 5. LLM 与工具 Utils

### LLM backend

`utils/llm_backend_utils.py`：

```python
get_llm_parser(model_name, return_json=True)
call_llm_with_text_prompt(prompt, model_name, generation_configs=None,
                          check_parsed_response_not_none=True,
                          return_json=True, result_parsing_func=None)
call_llm_with_images(prompt, images, model_name, ...)
call_llm_with_pdf(pdf_path, prompt, model_name, system_instruction=None, ...)
```

模型名含 `gpt`/`o1`/`o3` → OpenAI，否则 Gemini。OpenAI 分支会剥掉 Gemini 风格的 `system_instruction`/`temperature`，转换为 `system`/`developer` role；`call_llm_with_pdf` 对 OpenAI 传抽取文本，对 Gemini 传 PDF 字节。

### Gemini / OpenAI 客户端

- `utils/gemini_utils.py:get_genai_client()`：Vertex AI 优先，GEMINI_API_KEY 次之；惰性单例 + 线程锁。
- `call_gemini_with_contents(contents, model_name, max_retries=5, base_interval_sec=5, result_parsing_func=..., generation_configs=...)`。
- `generate_image_with_gemini(model_name, prompt, aspect_ratio, generation_configs, save_path)`：设置 `response_modalities=["IMAGE"]`、`ImageConfig(aspect_ratio, image_size="1k")`，返回 base64。
- `utils/openai_utils.py:get_openai_client()`：要求 `OPENAI_API_KEY`；`call_openai_models_with_content()` 支持文本/图片 content list。

### PDF 与图像

- `utils/pdf_utils.py:compile_latex(cwd, pdf_file, texfile_name="template", timeout=30)`：清 TeX 环境变量，依次 pdflatex / bibtex / pdflatex ×2，成功则把 PDF 移到目标路径。
- `load_paper(pdf_path)`：PyMuPDF 优先，pypdf 回退。
- `pdf_to_grid_images(pdf_path, output_dir, scale=2.0)`：页面网格截图，供 VLM 格式审查。
- `utils/scholar_utils.py:s2_title_search(title_query, year_hint, cutoff_date)`：Semantic Scholar Graph API，字段固定，5 秒超时，fuzzy ratio >70。

### 环境变量

`.env` 在 `utils/gemini_utils.py`、`utils/openai_utils.py`、`utils/scholar_utils.py`、`frontend/app.py` 中加载（路径为仓库根或当前目录）：

| 变量 | 用途 |
|---|---|
| `VERTEX_AI_PROJECT`, `VERTEX_AI_LOCATION` | Gemini via Vertex AI，优先于 API key |
| `GEMINI_API_KEY` | Gemini API |
| `OPENAI_API_KEY` | OpenAI 模型 |
| `SEMANTIC_SCHOLAR_API_KEY` | S2 元数据补全，可选但强烈建议 |
| `SMTP_EMAIL`, `SMTP_PASSWORD` | Streamlit 完成后发送通知，仅 Demo 使用 |

## 6. Autoraters

| 文件 | 作用 |
|---|---|
| `autoraters/agent_review.py` | 主路径使用：ensemble review + meta review，产出 Overall 等评分 |
| `autoraters/citation_f1.py` | 引用 F1 评估，依赖未随仓库发布的 `datasets/` |
| `autoraters/lit_review_quality.py` | 单篇/批量综述质量打分 |
| `autoraters/sxs_lit_review_quality.py` | 两篇综述质量 side-by-side 对比 |
| `autoraters/sxs_paper_quality.py` | 两篇论文 side-by-side 视觉/文本质量对比 |

这些评估脚本不是 `write_single_paper()` 主路径；除 `agent_review.py` 外，不要假设它们会在生成过程中自动运行。

## 7. 扩展建议（源码允许范围内）

- 加会议模板：复制 `templates/iclr2025/` 结构，保留 `template.tex` + `guidelines.md` + 必要样式；CLI 可直接 `--latex_template_dir` 指向它，UI 需改 `render_sidebar()` 的 templates 列表。
- 换 LLM：CLI model name 决定 backend；Gemini-only 调用点（Outline、文献 discovery、AgentReview）不会自动切到 OpenAI，需改对应 agent。
- 修改 prompt：所有写作 prompt 尾部都有 no-leakage 约束；不要删除，否则可能破坏双盲要求。
- 改图表类型：Outline prompt 限定 `plot|diagram` 与 aspect_ratio 枚举；PlottingAgent 按该字段分流。
