# HPC Agent

HPC Agent is a conversational assistant for HPC environments running Slurm. Its primary interface is a **Textual TUI** (a full-screen terminal application), with a general-purpose **MCP server** for external AI and MCP clients.

It supports Slurm knowledge search, sbatch script generation, job submission and management, VASP input generation and fixed-directory workflows, output synchronization, and calculation reports generated through Claude Code.

See the [documentation index](docs/README.md) and [user guide](docs/USER_GUIDE.md) for detailed instructions. For integrations and deployment, see the [MCP guide](docs/mcp_docs/MCP.md) and [Docker guide](docs/DOCKER.md). These supporting documents are currently primarily in Chinese.

## Features

### Slurm and HPC knowledge search

- Retrieval-augmented answers using TF-IDF, BM25, query expansion, metadata boosts, and the model configured by `LLM_MODEL`. The retrieval layer includes extension points for hybrid and semantic search.
- Knowledge documents cover cluster information, common errors, GPU usage, CUDA troubleshooting, sbatch submission, job states, pending jobs, logs, software environments, storage quotas, cancellation, and VASP workflows.
- `tools/rag_debug.py` shows retrieval matches, scores, metadata boosts, and chunk previews. `tools/rag_eval.py` evaluates retrieval against fixtures.

### General Slurm jobs

- Generate sbatch scripts from natural-language requests.
- Recommend CPU, memory, time, and GPU settings based on local job files.
- Preview `job.sh`, request confirmation, then upload and submit over SSH.
- Upload files into a separate remote directory for each job.
- Support local `.py`, `.sh`, `.slurm`, and `.sbatch` files.
- Detect dangerous commands and reject scripts containing commands such as `rm -rf`, `shutdown`, `reboot`, or `mkfs`.
- Inspect the active model and configuration, check HPC setup, and prepare a minimal `hostname` submission test.

### Job queries and monitoring

- Query job status through `squeue` and `sacct`, and read stdout and stderr.
- Diagnose a job by combining its state, logs, and suggested next steps.
- View recent jobs, job details, and local VASP jobs.
- Monitor multiple running jobs in the TUI's right-hand panel, with automatic refresh every five seconds and `Tab` to switch jobs.
- Display job ID, state, elapsed time, remote directory, and recent stdout/stderr lines.
- Suggest follow-up commands when jobs fail. VASP monitoring also includes live error and warning detection.

### Remote job management

- List remote Slurm job IDs and VASP jobs.
- Preview and confirm cleanup of individual Slurm jobs by ID or VASP job directories.
- Clean up all remote Slurm job files while preserving the root directory, or clean up all remote VASP jobs.

### VASP jobs

- Generate sbatch scripts for relaxation, static calculations, and other VASP runs.
- Generate `INCAR`, `KPOINTS`, and `POSCAR` using metadata from an existing local `POTCAR`.
- Override input parameters such as `ENCUT`, the k-point mesh, and calculation type through natural language or `/vasp gen`.
- Require confirmation before overwriting existing input files.
- Submit through a fixed directory layout: local input → remote input → remote output.
- Handle repeated job names by offering to overwrite previous results, create a new run name, or cancel. Overwriting clears the matching remote input/output and local output directories before resubmission.
- Register existing VASP jobs and synchronize output using include/exclude rules.
- Probe remote files and diagnose POTCAR errors, Fortran severe errors, out-of-memory conditions, segmentation faults, full disks, and walltime limits.
- Parse OUTCAR and OSZICAR deterministically for energies, forces, stress, band information, and convergence.
- Generate `report_context.md` as structured input for Claude Code.
- Produce `report.md` (a Chinese user report), `paper_methods.md` (English), and `paper_results.md` (English).
- Run synchronization, context generation, and report generation as one analysis workflow.

### Error diagnosis

- Diagnose logs using a case library and generic error rules.
- Check `.env`, SSH keys, local and remote paths, VASP commands, partitions, and Claude Code/API configuration, with suggested fixes.
- Draft new error cases and save them to `data/errors/real_cases.json` after confirmation.
- Adjust generated sbatch scripts for issues such as insufficient memory, time limits, or invalid partitions.
- Help investigate pending jobs and jobs that do not start.

### Terminal interface

- Full-screen Textual UI with a chat panel and a job monitor.
- Confirmation and cancellation states for submissions, cleanup, and overwrites.
- Mouse text selection in chat results. `Ctrl+Y` copies the selection, or the last assistant response when nothing is selected.
- Shortcuts including `/help`, `/help job`, `/help vasp`, `/clear`, `/clear all`, and `/exit`.

### MCP integration

- `hpc-agent-mcp` runs HPC Agent as an MCP server.
- `hpc-agent-mcp-client` connects to external MCP servers and exposes allowlisted tools as `external_<server>_<tool>`.
- Support for STDIO and Streamable HTTP transports.
- Browser-client access through Cloudflare Tunnel, ngrok, or a fixed HTTPS domain.
- A `/health` endpoint for health checks.
- `hpc_agent_chat` provides a conversational entry point similar to the TUI.
- Structured tools cover Slurm script previews, VASP job previews, input generation, local result analysis, job queries, output synchronization, and cleanup previews.
- MCP resources expose capabilities, tool schemas, safety policies, examples, deployment status, recent jobs, configuration status, skills, and cluster knowledge.
- Actual submission, synchronization, and remote cleanup require both `confirm=true` and the corresponding server-side environment switch.

See the [MCP guide](docs/mcp_docs/MCP.md) and [external MCP tool injection guide](docs/mcp_docs/EXTERNAL_MCP_INJECTION.md).

### Claude Code reports

- The analysis skill is defined in `skills/vasp_report/SKILL.md`.
- The Claude Code CLI generates reports using only `report_context.md`, with instructions against inventing unverified results.
- The model, timeout, and CLI command are configurable.

### Skills registry

- Discover `skills/*/SKILL.md` and parse skill names, types, intents, handlers, runtime adapters, and metadata.
- Support `tool`, `prompt`, and `rule` skills, with handler import validation.
- Resolve skills by intent through `SkillRegistry`, then invoke handlers through `SkillExecutor` and `runtime.adapter`.
- Supported execution protocols: `question_to_text`, `injected_diagnoser`, `tool_dispatch`, and `structured_result`.
- Built-in skills cover sbatch generation, VASP scripts and inputs, diagnosis, reports, parameter suggestions, read-only job queries, and local resource detection.
- The `get-available-resources` skill was adapted from `K-Dense-AI/scientific-agent-skills` to inspect local CPU, memory, disk, and GPU resources.
- If a skill handler fails, the assistant prefers a RAG fallback and explains the failure.
- `tools/skill_debug.py` inspects intent bindings; `tools/skill_eval.py` runs regression evaluations.
- `HPC_AGENT_CUSTOM_SKILLS_DIR` enables external read-only skills. See the [architecture guide](docs/skills_architecture.md) and [external skills guide](docs/EXTERNAL_SKILLS.md) for `SKILL.md` and `handler.py` examples.

### Project diagnostics

`/doctor` checks `.env`, SSH/HPC configuration, required directories, RAG documents, the skills registry, and local CPU, memory, disk, and GPU resources. It is read-only: it does not submit jobs, remove files, or change configuration. Warnings include prioritized suggestions.

### Job registry

- Store local job records in JSON, including job ID, type (`slurm` or `vasp`), remote/local paths, and uploaded files.
- View registry status, recent jobs, job details, and local VASP jobs.
- Preview and confirm archiving while keeping the most recent N records.
- Inspect archives, then preview and confirm restoration.

### Validation

Local checks cover Python syntax, Slurm and VASP assistants, error diagnosis, intent routing, job queries, and submission previews. Live HPC checks are available through `--live-hpc` and include configuration validation, SSH connectivity, and a real Slurm test submission.

## Quick start

### Run from source

Install dependencies and create a local configuration:

```bash
uv sync
cp .env.example .env
```

Edit `.env` for your model API and HPC environment, then start the TUI:

```bash
uv run python app.py
```

An installed distribution also provides:

```bash
hpc-agent
```

### Release installation (0.2.6)

The following URL is a **placeholder**. Replace it with your own release service before running this example, or use the source installation above.

```bash
curl -fL \
  "https://packages.example.com/repository/agents/models/ai-llm/hpc-agent/stable/install.sh" \
  -o /tmp/hpc-agent-install.sh

sh /tmp/hpc-agent-install.sh
```

Check the installed version:

```bash
"$HOME/.local/share/hpc-agent/.venv/bin/python" -c "import importlib.metadata as m; print(m.version('hpc-agent'))"
```

For release 0.2.6, the expected output is `0.2.6`.

### Start the MCP HTTP server

```bash
hpc-agent-mcp --transport streamable-http --host 127.0.0.1 --port 8000 --path /mcp
```

Check its health from another terminal:

```bash
curl http://127.0.0.1:8000/health
```

## Environment variables

For a source checkout, create `.env` in the project root. Example:

```env
LLM_BASE_URL=https://your-api-base-url
LLM_API_KEY=your-api-key
LLM_MODEL=DeepSeek-V4-Pro

HPC_HOST=your-hpc-host
HPC_USERNAME=your-hpc-username
HPC_KEY_PATH=/path/to/your/private/key
HPC_LOCAL_WORKDIR=/path/to/local/hpc-agent-jobs
HPC_REMOTE_WORKDIR=/path/to/remote/hpc-agent-jobs
HPC_DEFAULT_PARTITION=

HPC_LOCAL_VASP_JOBS_INPUT_DIR=/path/to/local/vasp-jobs-input
HPC_LOCAL_VASP_JOBS_OUTPUT_DIR=/path/to/local/vasp-jobs-output
HPC_VASP_REMOTE_INPUT_DIR=/path/to/remote/vasp-hpc-jobs-input
HPC_VASP_REMOTE_OUTPUT_DIR=/path/to/remote/vasp-hpc-jobs-output
HPC_VASP_PARTITION=
HPC_VASP_SETUP_COMMAND=source /path/to/vasp/env/setup.sh
HPC_VASP_COMMAND=mpirun /path/to/vasp/bin/vasp_std
HPC_VASP_MODULE=

HPC_CLAUDE_CODE_COMMAND=claude
HPC_VASP_REPORT_MODEL=DeepSeek-V4-Pro
HPC_CLAUDE_CODE_TIMEOUT_SECONDS=1800

HPC_AGENT_CUSTOM_SKILLS_DIR=/home/example/customize-skills
HPC_AGENT_TRUST_EXTERNAL_PYTHON=false
HPC_AGENT_EXTERNAL_PYTHON_TIMEOUT_SECONDS=10

# MCP server
HPC_AGENT_MCP_HOST=127.0.0.1
HPC_AGENT_MCP_PORT=8000
HPC_AGENT_MCP_PATH=/mcp
HPC_AGENT_MCP_ALLOWED_HOST=
HPC_AGENT_MCP_ALLOWED_ORIGIN=
HPC_AGENT_LOG_DIR=/home/example/.local/share/hpc-agent/logs
HPC_AGENT_MCP_AUDIT_LOG=/home/example/.local/share/hpc-agent/mcp_audit.jsonl
HPC_AGENT_MCP_ENABLE_WRITE=0
HPC_AGENT_MCP_ENABLE_DESTRUCTIVE=0
```

| Variable | Purpose |
| --- | --- |
| `LLM_BASE_URL` | Model API base URL. |
| `LLM_API_KEY` | Model API key. |
| `LLM_MODEL` | Main assistant model for general/RAG answers, intent-classification fallback, and script assistance. VASP reports have a separate setting. |
| `HPC_HOST` | HPC SSH login host. |
| `HPC_USERNAME` | HPC username. |
| `HPC_KEY_PATH` | Absolute path to the local SSH private key (Ed25519). |
| `HPC_LOCAL_WORKDIR` | Local directory for general jobs and test files. Bare filenames are searched for here and in the launch directory. |
| `HPC_REMOTE_WORKDIR` | Remote root directory for general Slurm jobs. |
| `HPC_DEFAULT_PARTITION` | Default partition for general jobs. Leave empty to omit `#SBATCH --partition` and use the cluster default. |
| `HPC_LOCAL_VASP_JOBS_INPUT_DIR` | Local VASP input root. |
| `HPC_LOCAL_VASP_JOBS_OUTPUT_DIR` | Local VASP output and analysis root. |
| `HPC_VASP_REMOTE_INPUT_DIR` | Remote VASP input root. |
| `HPC_VASP_REMOTE_OUTPUT_DIR` | Remote VASP output/run root. |
| `HPC_VASP_PARTITION` | Default VASP partition. Leave empty to use the cluster default. |
| `HPC_VASP_SETUP_COMMAND` | Environment setup before launching VASP. For example, an MPI installation may require sourcing its environment first. |
| `HPC_VASP_COMMAND` | VASP launch command; use the actual executable path on your cluster. General MPI tests use `srun` by default. |
| `HPC_VASP_MODULE` | Optional module to load. Leave empty to skip `module load`. |
| `HPC_CLAUDE_CODE_COMMAND` | Claude Code CLI command; default: `claude`. |
| `HPC_VASP_REPORT_MODEL` | Report model. Fallback order: legacy `HPC_CLAUDE_CODE_MODEL`, `ANTHROPIC_MODEL`, then `LLM_MODEL`; if none is set and `LLM_BASE_URL` is present, use `DeepSeek-V4-Pro`. |
| `HPC_CLAUDE_CODE_TIMEOUT_SECONDS` | Report generation timeout; default: 1800 seconds. |
| `HPC_AGENT_CUSTOM_SKILLS_DIR` | External read-only skills directory, such as `/home/example/customize-skills`. |
| `HPC_AGENT_TRUST_EXTERNAL_PYTHON` | Allow external Python handlers. Keep `false` unless the directory is trusted. |
| `HPC_AGENT_EXTERNAL_PYTHON_TIMEOUT_SECONDS` | External Python handler timeout; default: 10 seconds. |
| `HPC_AGENT_MCP_HOST` / `HPC_AGENT_MCP_PORT` / `HPC_AGENT_MCP_PATH` | MCP HTTP listen address, port, and path. |
| `HPC_AGENT_MCP_ALLOWED_HOST` | Allowed public tunnel or fixed-domain host for browser clients. |
| `HPC_AGENT_MCP_ALLOWED_ORIGIN` | Optional allowed origin. |
| `HPC_AGENT_LOG_DIR` | Log directory used by the MCP HTTP startup script. |
| `HPC_AGENT_MCP_AUDIT_LOG` | Audit log path for MCP writes, execution, and destructive operations. |
| `HPC_AGENT_MCP_ENABLE_WRITE` | Enable actual MCP job submissions and VASP output synchronization; default: `0`. |
| `HPC_AGENT_MCP_ENABLE_DESTRUCTIVE` | Enable remote cleanup through MCP; default: `0`. |

For Claude Code reports, `LLM_API_KEY` is passed as both `ANTHROPIC_API_KEY` and `ANTHROPIC_AUTH_TOKEN`. `LLM_BASE_URL` supplies `ANTHROPIC_BASE_URL` when that variable is not already set. If authentication fails, check whether the key has expired and whether it can access the report model.

Do not commit `.env` to Git.

## Common commands

The current command examples and confirmation phrases are primarily Chinese. They are preserved below as executable input examples; translating this README does not change the application's command parser or output language. Slash commands are included where available.

### General jobs

Request a run with resource settings, prepare a test script, or confirm/cancel a submission:

```text
跑 train.py，4核，15分钟
帮我提交 ./run.sh，2核，30分钟
帮我提交一个作业运行 python train.py，8核，1小时，16G内存
创建 srun -n 4 hostname 测试脚本并运行
确认提交
取消提交
```

### Job queries and monitoring

Inspect status and logs, diagnose a job, start or stop monitoring, or list jobs:

```text
查看 11814753 的状态
读取 11814753 的输出
读取 11814753 的错误日志
诊断作业 11814753
监控 11814753
取消监控 11814753
/job monitor 11814753
/job stop-monitor 11814753
查看最近作业
查看作业详情 11814753
列出 VASP 作业
列出远端作业编号
```

### Local job records

Inspect the registry, preview archiving while retaining recent records, confirm archiving, and preview or confirm restoration:

```text
查看本地作业记录状态
预览归档本地作业记录，只保留最近 100 个
确认归档本地作业记录
查看归档记录
预览恢复最近一次本地作业记录归档
确认恢复本地作业记录归档
```

### Remote Slurm cleanup

Request cleanup of one job or all general jobs, then confirm the requested operation:

```text
清理远端作业 11817627 的文件
确认清理
清理远端普通作业目录下所有作业文件
确认清理全部
```

### VASP

Generate inputs, confirm overwrites, prepare and submit jobs, register existing jobs, synchronize results, generate reports, or clean up job directories:

```text
帮我生成我的 VASP 作业 Al_test 的配置文件
帮我生成 Al_test 的 VASP 输入，ENCUT 400，KPOINTS 2x2x2，静态计算
/vasp gen Al_test --type static --encut 400 --kpoints 2x2x2
覆盖已有配置文件
确认覆盖
帮我生成一个 VASP 结构优化脚本，1 个节点，每节点 32 核，运行 24 小时
帮我提交 VASP 作业，目录名 si_static_test，1 个节点 32 核，运行 10 分钟
确认提交
登记 VASP 作业 11817144，目录名 si_static_test
同步 VASP 作业 11817144 输出到本地
生成 VASP 作业 si_static_test 报告
一键分析 VASP 作业 si_static_test
列出远端 VASP 作业
清理远端 VASP 作业 si_static_test 的文件
清理全部远端 VASP 作业文件
```

### Skill debugging

The Python commands below run in a shell. Enter `/skill` commands in the TUI:

```bash
.venv/bin/python tools/skill_debug.py --validate
.venv/bin/python tools/skill_debug.py --route "读取 11814753 的输出" --validate
.venv/bin/python tools/skill_eval.py --execute
.venv/bin/python tools/route_debug.py "帮我生成我的vasp作业Al_test的配置文件"
/skill list
/skill route 读取 11814753 的输出
```

### Error diagnosis

Paste an error message or ask why a job remains pending:

```text
CUDA out of memory
ModuleNotFoundError: No module named numpy
sbatch: error: Batch job submission failed: Invalid partition name specified
我的任务一直 pending
```

## General job workflow

The assistant generates and previews `job.sh` first. It connects to the HPC system over SSH and submits only after the user confirms with `确认提交`.

Each general job gets a separate directory under `HPC_REMOTE_WORKDIR`, containing:

```text
job.sh
<uploaded or locally selected job files>
hpc_agent_job_<jobid>.out
hpc_agent_job_<jobid>.err
```

Local `.py`, `.sh`, `.slurm`, and `.sbatch` paths are supported. If CPU, time, memory, or GPU requirements are omitted, the assistant tries to recommend them from file contents. Explicit user settings take priority.

For a bare Python filename such as `test.py`, or an uploaded `.py` file with no run command, the assistant infers `python3 test.py`. An explicit command such as `python train.py` is preserved.

## VASP fixed-directory workflow

### Prepare inputs

Before submitting, prepare a complete local job directory:

```text
$HPC_LOCAL_VASP_JOBS_INPUT_DIR/<job-folder>/
├── INCAR
├── KPOINTS
├── POSCAR
└── POTCAR
```

### Submit a job

1. Select `$HPC_LOCAL_VASP_JOBS_INPUT_DIR/<job-folder>`.
2. Upload its files to `$HPC_VASP_REMOTE_INPUT_DIR/<job-folder>`.
3. Write and run `job.sh` in `$HPC_VASP_REMOTE_OUTPUT_DIR/<job-folder>`.

VASP stdout, stderr, and calculation results go into the remote output directory. The assistant does not generate real `POTCAR` files; supply one from a VASP pseudopotential library you are authorized to use.

If a local job already contains `POTCAR`, generate the remaining inputs from its metadata:

```text
/vasp gen Al_test --type static --encut 400 --kpoints 2x2x2
```

This writes `INCAR`, `KPOINTS`, and `POSCAR` under `$HPC_LOCAL_VASP_JOBS_INPUT_DIR/Al_test/`. Existing inputs require confirmation with `确认覆盖` or `覆盖已有配置文件`. Without structure parameters, the generated structure is a default smoke-test structure for checking VASP startup, POTCAR readability, and submission. It does not represent a real material structure.

If the same job name already exists in remote or local output, choose to overwrite results, create a new run name, or cancel. Overwriting first clears the matching remote input/output and local output directories, then uploads and submits again so that old and new results are not mixed.

For general Slurm/MPI tests, use `srun -n N ...`. For VASP, initialize the environment with `HPC_VASP_SETUP_COMMAND`, then run `HPC_VASP_COMMAND`. Configure these execution paths separately.

### Synchronize output

After completion, request `同步 VASP 作业 <job_id> 输出到本地` to download the required results:

```text
$HPC_LOCAL_VASP_JOBS_OUTPUT_DIR/<job-folder>/
├── raw_output/
│   ├── INCAR
│   ├── KPOINTS
│   ├── POSCAR
│   ├── OUTCAR
│   ├── OSZICAR
│   ├── CONTCAR
│   └── vasprun.xml
└── analysis/
    ├── file_manifest.json
    └── report_context.md
```

Synchronization rules:

- **Include:** `INCAR`, `POSCAR`, `KPOINTS`, `OUTCAR`, `OSZICAR`, `CONTCAR`, `vasprun.xml`, `*.out`, and `*.err`.
- **Exclude:** `WAVECAR`, `CHGCAR`, `AECCAR*`, and `POTCAR`, to avoid transferring large files and pseudopotentials.

### Analysis and report generation

The request `一键分析 VASP 作业 <job-folder>` runs the full workflow:

1. Synchronize results into the local output directory.
2. Generate `report_context.md` with deterministically parsed facts, input summaries, log excerpts, and diagnostics.
3. Invoke Claude Code and write the report files.

| Output | Content |
| --- | --- |
| `analysis/report.md` | Chinese user report. |
| `analysis/paper_methods.md` | English methods text. |
| `analysis/paper_results.md` | English results text. |

You can also synchronize and generate reports separately:

```text
同步 VASP 作业 11817144 输出到本地
生成 VASP 作业 si_static_test 报告
```

Facts parsed from OUTCAR and OSZICAR include:

- Convergence and energy values: TOTEN, energy without entropy, and energy sigma→0.
- E-fermi, ISMEAR, SIGMA, and NELECT.
- Cell volume, lattice vectors, and ENCUT.
- Stress tensor and external pressure.
- Maximum force magnitude and mean force norm.
- CPU time, elapsed time, and maximum memory.
- VBM, CBM, and band gap.
- Ion species, counts, and atomic masses.
- DAV/RMM iteration counts.

Report generation loads `skills/vasp_report/SKILL.md` and uses `analysis/report_context.md` as its controlled input. Numerical values come from deterministic parsers; Claude Code organizes the Chinese report and English methods/results text. The skill instructs the model to use only facts confirmed in the context, rather than loading large raw outputs or inventing results.

Claude Code runs in `--bare` mode and is asked to return JSON. Python writes the final `report.md`, `paper_methods.md`, and `paper_results.md` files.

## Live VASP diagnostics

When a VASP job is monitored in the TUI, probes inspect key files in the remote output directory.

- **Errors:** POTCAR conversion/input problems, Fortran severe errors, out-of-memory conditions, segmentation faults, full disks, walltime limits, missing files, and VASP crashes.
- **Warnings:** BRMIX charge mixing, ZBRENT convergence issues, and failed geometry optimization steps.

Diagnostics include severity (`ok`, `warning`, or `error`), identified issues, supporting evidence, and suggested fixes.

## TUI layout and keyboard shortcuts

The top bar shows connection information, the remote job root, and shortcut hints. Chat appears on the left, the job monitor on the right, and a fixed input field at the bottom.

| Key | Action |
| --- | --- |
| `Ctrl+R` | Refresh the currently monitored job. |
| `Ctrl+Y` | Copy selected text, or the previous assistant response if nothing is selected. |
| `Ctrl+S` | Confirm a pending job submission. |
| `Tab` | Switch between monitored jobs. |
| `Esc` | Cancel the operation awaiting confirmation. |
| `Ctrl+X`, `F10`, `q` | Exit. |

### Initial setup checks

```text
查看当前模型
检查我的超算配置
一键测试超算提交流程
/help
/help job
/help vasp
/help skill
/resources
/doctor
/clear
/clear all
/exit
```

The first three commands show the current model/configuration, check HPC configuration, and prepare a minimal submission test, respectively. The model summary masks the API key. Configuration checks suggest fixes without changing settings. The submission test previews a `hostname` job and still requires confirmation before submission.

## Error diagnosis

`data/errors/real_cases.json` takes priority for known application cases; `data/errors/generic_errors.json` supplies generic HPC, Slurm, Linux, and Python rules. Results include likely causes, suggested fixes, and troubleshooting commands.

To draft a new case, use `把这个错误整理成案例：` followed by the error log, or `把这个错误整理成案例` after providing the log. The assistant waits for `确认` before saving to `data/errors/real_cases.json`. Draft preparation attempts to redact local paths, usernames, hostnames, API keys, and other sensitive information.

The assistant can also adjust generated sbatch scripts:

| Issue | Script adjustment |
| --- | --- |
| Out of memory | Add or increase `--mem`. |
| Time limit | Add or extend `--time`. |
| Invalid partition | Add `--partition=general`. |
| Missing node/CPU settings | Add `--nodes=1 --cpus-per-task=4`. |
| GPU out of memory | Add `--gres=gpu:1 --mem=32G`. |
| Missing CUDA | Add `--gres=gpu:1` and `module load cuda`. |

Partition names and modules must match your target cluster.

## Testing

Run local checks:

```bash
hpc-agent-check
# Or, from a source checkout:
.venv/bin/python tests/run_all_checks.py
```

### Routing regression tests

Natural-language intent fixtures live in `tests/fixtures/route_cases.json`. When adding a feature or correcting a routing error, add a representative user request, the expected intent, and optionally a risk level:

```json
{"text": "诊断作业 11838843", "intent": "diagnose_job"}
```

Tests check `detect_intent()`, risk classification, and routing explanations.

Run live HPC workflow checks:

```bash
hpc-agent-check --live-hpc
# Or:
.venv/bin/python tests/run_all_checks.py --live-hpc
```

By default, only local regression checks run. `--live-hpc` additionally checks HPC configuration, connects to the cluster, and submits a real Slurm test job.

## Project structure

```text
hpc-agent/
├── app.py                    # Main entry point; starts the Textual TUI
├── textual_cli.py            # Terminal interface
├── pyproject.toml            # Project metadata and dependencies
├── .env.example              # Environment variable template
├── modules/
│   ├── core/                 # Runtime, context, confirmations, tool calling
│   ├── routing/              # Intent detection, LLM fallback, dispatch
│   ├── skills/               # Registry, SKILL.md parsing, handler validation
│   ├── slurm/                # Submission, queries, monitoring, cleanup, logs
│   ├── vasp/                 # Inputs, parsing, diagnostics, reports
│   ├── tui/                  # UI helpers, formatters, monitor, workflow state
│   ├── mcp/                  # MCP server, tools, and resources
│   ├── mcp_client/           # External MCP client and tool injection
│   └── knowledge/            # RAG and error diagnosis
├── data/
│   ├── hpc_documents/        # RAG knowledge documents
│   │   ├── cluster_info.txt
│   │   ├── cluster_sinfo_example.txt
│   │   ├── common_errors.txt
│   │   ├── environment_modules_conda.txt
│   │   ├── gpu_cuda_troubleshooting.txt
│   │   ├── gpu_usage.txt
│   │   ├── slurm_output_logs.txt
│   │   ├── slurm_pending.txt
│   │   ├── slurm_submit.txt
│   │   ├── slurm_status.txt
│   │   ├── slurm_cancel.txt
│   │   ├── storage_quota_files.txt
│   │   ├── vasp_troubleshooting.txt
│   │   └── vasp_workflow.txt
│   ├── errors/
│   │   ├── README.md         # Error library maintenance
│   │   ├── real_cases.json   # Case library
│   │   └── generic_errors.json # Generic diagnostic rules
│   └── jobs/                 # Local runtime data; excluded from public source
│       ├── job_registry.json # Created during use
│       └── archive/          # Archived local records
├── skills/
│   ├── diagnose_error/SKILL.md
│   ├── generate_vasp_inputs/SKILL.md
│   ├── generate_sbatch/SKILL.md
│   ├── generate_vasp_job/SKILL.md
│   ├── get_available_resources/SKILL.md
│   ├── inspect_job/SKILL.md
│   ├── suggest_params/SKILL.md
│   └── vasp_report/SKILL.md
├── tools/
│   ├── rag_debug.py          # Retrieval debugging
│   ├── rag_eval.py           # Retrieval evaluation
│   ├── route_debug.py        # Routing debugging
│   ├── skill_debug.py        # Registry debugging
│   └── skill_eval.py         # Skill routing and execution evaluation
├── tests/
│   ├── run_all_checks.py     # Check runner
│   ├── core/
│   ├── routing/
│   ├── slurm/
│   ├── vasp/
│   └── knowledge/
├── docs/
│   ├── USER_GUIDE.md
│   ├── LIVE_TEST_CHECKLIST.md
│   ├── skills_architecture.md
│   └── VASP_TEST_TEMPLATES.md
└── README.md
```

For build commands and release templates, see [Packaging and releases](packaging/README.md).

## Public source and local configuration

Model connections use `LLM_BASE_URL`, `LLM_API_KEY`, and `LLM_MODEL`. Keep actual connection details in a local `.env`; do not commit credentials, accounts, internal host addresses, or real cluster snapshots. `cluster_sinfo_example.txt` contains fictional teaching data. Use the target cluster's actual partitions and resource limits.

`packages.example.com` is a placeholder release address. Replace it with your own distribution service, or install from source.

Deleting sensitive content from current files does not remove it from Git history. If an existing repository's history contains internal information, create a new repository from a reviewed source snapshot. Do not copy the old `.git`, branches, tags, build artifacts, or runtime records into the public repository. Rebuild distribution packages from the cleaned source before publishing them.
