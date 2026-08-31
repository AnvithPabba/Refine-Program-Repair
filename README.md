# Refine: A General Context-Aware Agent for Evolving Program Repair Agents

Refine is a general context-aware refinement agent that improves **Draft Patches** produced by LLM-based automated program repair (APR) systems. Operating in a black-box, plug-and-play manner, Refine augments existing APR agents without model retraining or architectural changes. It resolves ambiguous issue and code context through structured contextualization, improves patch quality through test-time scaling and diversified generation, and uses LLM-driven refinement to consolidate partial fixes into more complete and robust patches.

Refine generalizes across diverse APR systems, yielding an average **14% improvement in resolution rate**. On **SWE-Bench Lite**, it improves AutoCodeRover by **14.67 percentage points**, reaching **51.67% resolution** and achieving state-of-the-art performance.


## Accepted to EMNLP findings '26!

published experiment artifacts are available under [`results/`](results/).

## Table of contents

- [Results](#results)
- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Model credentials](#model-credentials)
- [Configuration](#configuration)
- [Set up SWE-bench](#set-up-swe-bench)
- [Run Refine](#run-refine)
- [Reproduce the paper results](#reproduce-the-paper-results)
- [Repository contents](#repository-contents)
- [Security](#security)
- [License and acknowledgements](#license-and-acknowledgements)

## Results

Refine consistently improves the performance of existing automated program repair systems across both workflow-based and agent-based approaches.

- **SWE-Bench Lite:** Refine improves AutoCodeRover from **37.00% to 51.67%**, a **+14.67 percentage-point improvement**, resolving **44 additional issues**.
- **SWE-Bench Verified:** Refine improves AutoCodeRover from **51.60% to 63.80%**, a **+12.2 percentage-point improvement**, resolving **61 additional issues**.
- **Across APR systems:** Refine provides an average **14% improvement in resolution rate**, demonstrating its generality across diverse repair systems.

On **SWE-Bench Lite**, Refine achieves **state-of-the-art performance** among the evaluated baselines.

See the [paper](preprint.pdf) for the full experimental setup, evaluation, and ablation studies. Reproduction artifacts and detailed outputs are available in [`results/`](results/).

## Prerequisites

The primary setup targets Linux. The experiments were run with a Conda
environment and Docker-based SWE-bench tooling.

Install the following before continuing:

- Git
- Conda or Miniconda
- Docker
- Credentials for at least one supported model provider
- Sufficient disk space for SWE-bench task repositories, containers, and run
  outputs

## Quick start

Clone this repository and create the pinned Conda environment:

```bash
git clone https://github.com/AnvithPabba/Refine-Program-Repair.git
cd Refine-Program-Repair
conda env create -f environment.yml
conda activate auto-code-rover
```

Refine expects the AutoCodeRover-compatible SWE-bench Docker package at
`SWE-bench-docker/`. The source snapshot includes the repository metadata but
not a populated Git submodule, so install the tested revision explicitly:

```bash
git clone https://github.com/Marti2203/SWe-bench-docker.git SWE-bench-docker
git -C SWE-bench-docker checkout f7db5326d5175398bd6fc04c523e133430eb465a
python -m pip install ./SWE-bench-docker
```

## Model credentials

Provide credentials through environment variables. For example, choose one of:

```bash
export OPENAI_KEY="<your-openai-api-key>"
export ANTHROPIC_API_KEY="<your-anthropic-api-key>"
```

`OPENAI_KEY` is the variable name expected by this codebase. Other implemented
providers use the following variables:

| Provider | Environment variables |
| --- | --- |
| Azure OpenAI | `AZURE_OPENAI_API_KEY`, `ENDPOINT_URL` |
| AWS Bedrock | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME` |
| Gemini | `GEMINI_API_KEY` or `GOOGLE_APPLICATION_CREDENTIALS` |
| Groq | `GROQ_API_KEY` |

Never commit real credentials, `.env` files, cloud service-account files, or
provider configuration containing secrets. See [Security](#security) for more
details.

## Configuration

Workflow behavior is controlled by [`app/config.py`](app/config.py), while the
command line supplies task, model, output, and concurrency settings.

Before a run, review the path-valued options in `app/config.py`. In particular:

- Set `path_to_patches` to the JSON file containing draft patches when
  `is_initial_patch_given` is enabled.
- Set `cached_results_path` if cached results are enabled for your run.
- Set Vertex AI project and credential-path options only when using Vertex AI.
- Keep credentials outside the repository; configuration files should contain
  paths or non-secret settings only.

The draft-patch JSON file uses the following shape:

```json
{
  "django__django-11133": "<patch text>",
  "another-instance-id": "<patch text>"
}
```

## Set up SWE-bench

Refine uses the
[yuntongzhang/SWE-bench fork](https://github.com/yuntongzhang/SWE-bench) to set
up task instances. Follow that repository's installation instructions, then
create a task list with one instance ID per line:

```bash
cd <SWE-bench-path>
printf '%s\n' django__django-11133 > tasks.txt
```

For Apple silicon, `django__django-16041` is an alternative example that does
not depend on the Python 3.6 environment used by some older instances.

Set up the selected tasks:

```bash
cd <SWE-bench-path>
conda activate swe-bench
python harness/run_setup.py \
  --log_dir logs \
  --testbed testbed \
  --result_dir setup_result \
  --subset_file tasks.txt
```

After setup completes, `setup_result/setup_map.json` and
`setup_result/tasks_map.json` describe the local task environments.

## Run Refine

Run a single SWE-bench task from the Refine repository:

```bash
conda activate auto-code-rover
PYTHONPATH=. python app/main.py swe-bench \
  --model <model-name> \
  --setup-map <SWE-bench-path>/setup_result/setup_map.json \
  --tasks-map <SWE-bench-path>/setup_result/tasks_map.json \
  --output-dir output \
  --task django__django-11133 \
  --enable-validation \
  --reproduce-and-review
```

The selected patch is written to `selected_patch.json` inside the task's output
directory.

To run every task listed in a file:

```bash
PYTHONPATH=. python app/main.py swe-bench \
  --model <model-name> \
  --setup-map <SWE-bench-path>/setup_result/setup_map.json \
  --tasks-map <SWE-bench-path>/setup_result/tasks_map.json \
  --output-dir output \
  --task-list-file <SWE-bench-path>/tasks.txt \
  --model-temperature 0 \
  --enable-validation \
  --reproduce-and-review \
  --num-processes <number-of-processes>
```

All tasks in the list must be set up before the run starts. Parallel processes
share local task environments, so choose a process count appropriate for the
host's memory, Docker capacity, and model-provider rate limits.

## Reproduce the paper results

1. Download the patches or predictions for the APR approaches being evaluated.
   Public SWE-bench submissions are available from the
   [SWE-bench experiments repository](https://github.com/SWE-bench/experiments).
2. Convert the collected patches to the JSON shape shown in
   [Configuration](#configuration) and set `path_to_patches` in `app/config.py`.
3. Select a task list from:
   - [`conf/swe_lite_tasks_30_subset.txt`](conf/swe_lite_tasks_30_subset.txt)
   - [`conf/swe_lite_tasks.txt`](conf/swe_lite_tasks.txt)
   - [`conf/swe_verified_tasks.txt`](conf/swe_verified_tasks.txt)
4. Set up those task instances and run the multi-task command above.

The paper's research questions use these configurations:

- **RQ1:** Run Refine on the 30-issue subset using patches from AutoCodeRover,
  Agentless, CodeV, BlackBoxAI, and ExpeRepair.
- **RQ2:** Run Refine on AutoCodeRover patches for SWE-bench Lite and
  SWE-bench Verified.
- **RQ3:** Run Refine on AutoCodeRover SWE-bench Lite patches while toggling
  the reviewer and context-retrieval settings in `app/config.py`.

Additional inherited AutoCodeRover experiment notes are retained in
[`EXPERIMENT.md`](EXPERIMENT.md).

## Repository contents

| Path | Purpose |
| --- | --- |
| `app/` | Refine and inherited AutoCodeRover implementation |
| `conf/` | Example configuration and SWE-bench task lists |
| `scripts/` | Experiment and result-processing utilities |
| `results/` | Published trajectories, logs, predictions, and evaluation artifacts |
| `demo_vis/` | Demonstration visualization utilities |
| `preprint.pdf` | Paper preprint included with this artifact |

The `results/` directory is intentionally large because the repository includes
the complete published artifact rather than only the runtime source code.

## Security

- `.env`, virtual environments, outputs, and known local credential-file paths
  are ignored by Git.
- Use environment variables or your provider's external credential store.
- Use placeholders in documentation and configuration examples.
- Before publishing changes, scan both the working tree and Git history for
  accidentally committed credentials.
- If a real secret is ever committed, revoke or rotate it immediately; deleting
  it from the latest file is not sufficient because Git retains history.

Some files under `results/` contain intentionally fake, sensitive-looking
values from public SWE-bench test fixtures. They are preserved because they are
part of the benchmark artifacts and are not live credentials.

## License and acknowledgements

This source is distributed under the
[SONAR Source-Available License v1.0](LICENSE).

Refine builds on
[AutoCodeRover](https://github.com/AutoCodeRoverSG/auto-code-rover). We thank
the AutoCodeRover team; portions of the implementation and setup instructions
originate from that project.
