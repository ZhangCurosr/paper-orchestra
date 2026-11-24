# PaperOrchestra 运行指南

> 基于源码快照 `d9bae99`。命令、依赖和环境变量来自 `README.md`、`requirements.txt`、`paper_writing_cli.py`、`frontend/` 与 `utils/*`。

## 1. 为什么不需要 GPU

PaperOrchestra 不在本地训练或运行大语言模型/图像模型：

- 文本、视觉理解、图像生成都通过远端 API：`utils/gemini_utils.py`（Vertex AI 或 Gemini API）与 `utils/openai_utils.py`（OpenAI API）。
- `PlottingAgent` 的统计图路径只用 **本地 matplotlib** 执行 LLM 生成的 Python 代码；`opencv-python` 用于图像字节处理，均为 CPU 工作负载。
- PDF 编译、截图与文本抽取只用 `pdflatex`/`bibtex`、PyMuPDF/pypdf，也不依赖 GPU。
- 图像生成模型（默认 `gemini-3-pro-image-preview`）在远端执行，本地只保存 base64 解码后的 JPEG。

因此普通笔记本电脑即可运行；瓶颈通常是 API 配额/费用、`pdflatex` 编译与文献 API 限流，而不是显存。

## 2. 安装

README 推荐 Python 3.11 与 Conda：

```bash
cd paper-orchestra
conda create -n paper_orchestra python=3.11
conda activate paper_orchestra
pip install -r requirements.txt
```

`requirements.txt` 主要依赖：`google-genai`, `openai`, `pydantic`, `json_repair`, `matplotlib`, `numpy`, `opencv-python`, `PyMuPDF`, `pymupdf4llm`, `pypdf`, `streamlit`, `python-dotenv`, `requests`, `thefuzz`, `tqdm`, `rich` 等。

系统依赖：**本机必须有 `pdflatex` 与 `bibtex`**（通常由 TeX Live 提供）。`utils/pdf_utils.py:compile_latex()` 直接调用它们；缺失时 PDF 精炼无法进行。Git Bash/WSL/Linux/macOS 均可，Windows 需确保命令在 PATH 中。

## 3. API 与环境变量

在仓库根目录创建 `.env`：

```env
# Gemini（满足其一）
VERTEX_AI_PROJECT=your_vertex_ai_project
VERTEX_AI_LOCATION=your_vertex_ai_location
# 或
GEMINI_API_KEY=your_gemini_api_key

# OpenAI（仅当 model name 含 gpt/o1/o3 时需要）
OPENAI_API_KEY=your_openai_api_key

# 文献元数据补全（可选但建议）
SEMANTIC_SCHOLAR_API_KEY=your_semantic_scholar_api_key

# Streamlit Demo 完成通知（可选）
SMTP_EMAIL=your_email
SMTP_PASSWORD=your_password
```

要点：

- Vertex AI 配置优先于 `GEMINI_API_KEY`。
- 客户端是惰性创建的：如果全流程只用 OpenAI 模型，不设置 Gemini key 也可以；但源码中有固定走 Gemini 的调用点（Outline、文献候选发现、AgentReview、绘图风格/批评），实际默认路径仍需要 Gemini 凭据。
- `SEMANTIC_SCHOLAR_API_KEY` 未设置时请求不带 header，可能触发限流；`s2_title_search()` 每次调用超时 5 秒。
- `.env` 由多个 utils 各自加载；不要在运行时只依赖 shell export 而忘记文件路径。

## 4. 单篇 CLI 运行

### 4.1 准备材料

```text
my_materials/
├── idea_sparse.md
├── experimental_log.md
└── figures/                  # 不绘图时可选；绘图时由 agent 写入
    └── info.json             # 若要正文自动引用用户图片，提供 [{"name":"x.png","caption":"..."}]
```

### 4.2 不绘图（使用已有图片）

```bash
python paper_writing_cli.py \
  --raw_materials_dir ./my_materials \
  --latex_template_dir ./templates/cvpr2025 \
  --output_dir ./paper_output_cvpr \
  --use_plotting false \
  --writer_model_name gemini-3.1-pro-preview \
  --reflection_model_name gemini-3.1-pro-preview
```

若 `figures/` 中没有任何 `.pdf/.png/.jpg/.jpeg`，CLI 会警告“编译可能失败”。若没有 `info.json`，图片会被复制到 LaTeX 目录，但 `SectionWritingAgent` 不会知道它们；需要手工补 caption/引用或修改 writer。

### 4.3 绘图

```bash
python paper_writing_cli.py \
  --raw_materials_dir ./my_materials \
  --latex_template_dir ./templates/iclr2025 \
  --output_dir ./paper_output_iclr \
  --use_plotting true \
  --plotting_model_name gemini-3.1-pro-preview \
  --image_model_name gemini-3-pro-image-preview \
  --plotting_max_critic_rounds 3
```

绘图路径会并行执行文献与图表 agent，并额外生成 `plotting_results.json`、`figures/*.jpg`、`figures/info.json`、`time_breakdown.json`。

### 4.4 关于 NeurIPS / ICML 模板

当前仓库只内置：

- `templates/cvpr2025/`
- `templates/iclr2025/`

Prompt 中出现 “NeurIPS/ICML” 只是会议风格示例，并不存在对应模板目录。要生成 NeurIPS/ICML 格式：

1. 准备自定义目录，例如 `templates/neurips2025_custom/`，至少包含 `template.tex` 与 `guidelines.md`，以及官方 `.sty/.bst/.cls`；
2. CLI 用 `--latex_template_dir templates/neurips2025_custom`；
3. 若通过 Streamlit 使用，需要修改 `frontend/app.py:render_sidebar()` 中 `templates = ["cvpr2025", "iclr2025"]`；
4. 核对 `template.tex` 的章节命令与 outline prompt 的期望一致（Introduction / Related Work / Methodology / Experiments / Conclusion）。

不要直接把 ICLR 模板改名为 NeurIPS 后投稿；页边距、匿名规则和样式文件必须由官方模板提供。

### 4.5 可选外部资源 PaperBanana

`utils/paper_banana_utils.py` 中 `PB_DIR = <repo>/../PaperBanana`，用于读取：

- `data/PaperBananaBench/<task>/ref.json`：few-shot 图表示例；
- `data/PaperBananaBench/<task>/<gt image>`：参考图；
- `style_guides/neurips2025_<task>_style_guide.md`：风格指南。

本工作区没有 `PaperBanana/` 目录。代码在 ref 缺失时打印 `Warn: ... not found. Proceeding without few-shot.`，风格指南缺失时以空字符串继续。绘图功能仍可运行，但风格与示例质量会下降；需要完整效果时应单独准备该外部资源。

## 5. Streamlit Demo

```bash
cd paper-orchestra
conda activate paper_orchestra
streamlit run frontend/app.py
```

浏览器打开终端给出的地址（通常 `http://localhost:8501`）。UI 侧：

- 模型下拉：`gemini-3.1-pro-preview` / `gemini-3-flash-preview`；
- 模板下拉：`cvpr2025` / `iclr2025`（硬编码）；
- “Enable Figure Generation” 默认 `Yes`；
- Email 必填，用于结束通知；SMTP 未配置时通知失败不影响 PDF 生成；
- “Load CVPR Example” 会加载 `frontend/examples/cvpr_example.json` 与示例图。

后台进程由 `subprocess.Popen` 启动，输出逐行映射到进度条。Linux 下使用 `prctl(PR_SET_PDEATHSIG, SIGKILL)` 随 Streamlit 退出；异常断开时手动清理：

```bash
pkill -f paper_writing_cli.py
# 或 pgrep -f paper_writing_cli.py 后 kill <PID>
```

Demo 输出位于仓库上级目录 `demo_logs/paper_orchestra_run_<timestamp>/`。

## 6. 批处理

仓库提供两种批量入口：

```bash
# 示例脚本：写死 dataset 路径与 paper_id，按需修改
./paper_writing_cli.sh
```

或在 Python 中调用：

```python
from methods.paper_writer import run_batch_writeup

run_batch_writeup(
    latex_template_dir="templates/cvpr2025",
    process_root_folder=True,
    root_input_folder="datasets/cvpr2025/papers",
    output_dir="./batch_output",
    n_papers=-1,
    max_workers=4,
)
```

输入布局：

```text
<root_input_folder>/
└── <paper_id>/
    ├── raw_materials/
    └── <paper_id>.pdf
```

断点续跑依赖 `output_dir/paper_writing_log.json`：状态为 `success` 且 PDF 路径存在的 paper 会被跳过。`n_papers=-1` 处理全部。`max_workers` 是进程数，注意乘以每个进程内部 LLM 并发（文献 10、绘图 5）后的 API 压力。仓库中 dataset 未包含，需自行准备。

## 7. 输出验收

```text
<output_dir>/
├── final_paper.pdf                              # 最终 PDF
├── outline.json
├── literature_agent_output/{outline_v1.json,updated_template.tex,references.bib,citation_map.json}
├── latex_writeup/{template.tex,raw_draft_paper.tex,references.bib,figures/}
├── content_refinement_workdir/
│   ├── content_refinement_worklog.json
│   ├── format_refinement_worklog.json
│   ├── peer_reviews/
│   └── final_refined_paper.{tex,pdf}
└── plotting_results.json / time_breakdown.json  # 绘图路径
```

检查清单：

1. `final_paper.pdf` 存在且页数符合 `guidelines.md`；
2. LaTeX 表格数字可在 `experimental_log.md` 找到；
3. `\cite{}` key 都在 `citation_map.json`/`references.bib` 中；
4. 所有 figures 在正文中被引用，且没有出现在 Conclusion 之后；
5. `content_refinement_worklog.json` 的最终 outcome 是接受类（`ACCEPTED_*`），而不是 `REJECTED_*`；
6. 双盲检查：无作者姓名、机构、致谢中的身份线索。

## 8. 排障

| 症状 | 源码原因 | 处理 |
|---|---|---|
| `No idea file detected at ...` | 文件名与 `--idea_filename` 不一致 | 改参数或重命名文件 |
| `No guidelines file detected` / `No template file detected` | 模板目录缺必需文件 | 换官方模板目录 |
| Gemini 凭据错误 | `get_genai_client()` 无 Vertex 也无 API key | 设置 `.env` |
| OpenAI key 错误 | model name 含 gpt/o1/o3 但无 key | 设置 `OPENAI_API_KEY` 或换模型 |
| S2 限流/候选为空 | 无 key 或每次 1s 仍不够 | 设置 `SEMANTIC_SCHOLAR_API_KEY`；检查 cutoff 与标题 |
| PDF 编译失败 | 无 pdflatex/bibtex 或 LaTeX 语法错误 | 看 `compile_latex()` 打印的日志；检查 `content_refinement_workdir/*.tex` |
| 图未出现在正文 | 无绘图模式缺 `figures/info.json`；或编译失败被跳过 | 补 info.json；检查 `plotting_results.json` 的 `image_path` |
| 精炼没有提升分数 | Overall 下降或子项退化，代码主动回退 | 查看 worklog；提高初稿质量或调整 Prompt |
| Streamlit 后台进程残留 | 非正常断开 | `pkill -f paper_writing_cli.py` |
| 生成内容不是 NeurIPS/ICML 版式 | 仓库无对应模板 | 自备模板目录并（UI 场景）修改下拉列表 |

## 9. 安全与成本提醒

- 单篇生成包含大量 LLM 调用：outline + 多检索任务 + S2 + 写作 + 最多 3 轮审稿/修订 + 1 轮格式；绘图还按 figure 并行。运行前应估算 API 配额和费用。
- 统计图路径使用 `exec()` 执行 LLM 生成代码，没有沙盒；不要处理不可信输入。
- Prompt 明确要求匿名化；不要在 idea/log 或模板作者区写入真实身份。
