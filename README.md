# Refine

**A general, context-aware agent for turning draft program repairs into stronger patches.**

> 🎉 **Refine has been accepted to EMNLP 2026.** The accepted manuscript is
> included in this repository as [`EMNLP2026_REFINE.pdf`](EMNLP2026_REFINE.pdf).

The accepted paper is titled *Refine: A General Context-Aware Agent for
Evolving Program Repair Agents*.

Refine is a plug-and-play refinement layer for automated program repair (APR)
agents. It starts from an issue, its repository context, and one or more draft
patches. Refine then regularizes the context, explores diverse patch deltas at
test time, and uses an LLM-driven aggregation stage to produce a more complete
and robust repair.

```text
Issue + repository + draft patch
               │
               ▼
  Context regularization
               │
               ▼
  Diverse delta generation
               │
               ▼
    Aggregation and review
               │
               ▼
        Refined patch
```

This repository contains the implementation, the accepted
[EMNLP 2026 paper](EMNLP2026_REFINE.pdf), the earlier
[preprint](preprint.pdf), and the complete published [`results/`](results/)
artifact. The current arXiv record is
[`arXiv:2510.03588`](https://arxiv.org/abs/2510.03588).

## How Refine relates to SemAgent and AutoCodeRover

This codebase was developed under the working name **SemAgent**. In this
repository, SemAgent is the implementation of **Refine on top of
[AutoCodeRover](https://github.com/AutoCodeRoverSG/auto-code-rover)**:
AutoCodeRover supplies the underlying program-repair workflow and draft
patches, while Refine adds context-aware generation, refinement, and review.

Refine is not limited to AutoCodeRover. Its black-box design can refine patches
from other APR agents without requiring changes to their internals; the paper
evaluates this behavior across five repair systems.

## Results

The paper reports that Refine:

- raises AutoCodeRover's SWE-bench Lite resolution rate by **14.67 percentage
  points**, reaching **51.67%**;
- raises its SWE-bench Verified resolution rate by **12.2 percentage points**,
  reaching **63.8%**; and
- consistently improves multiple APR systems, demonstrating that refinement is
  useful beyond a single base agent.

See the [accepted paper](EMNLP2026_REFINE.pdf) for the method and evaluation,
and [`results/`](results/) for the released trajectories, logs, predictions,
and evaluation artifacts.

## Prerequisites

The released environment targets **64-bit Linux**. Before starting, install:

- Git;
- Conda or Miniconda;
- Docker with the daemon running;
- credentials for at least one supported model provider; and
- enough storage for this repository, SWE-bench task repositories, containers,
  and generated outputs.

Confirm Docker is available before creating benchmark tasks:

```bash
docker info
```

## Install Refine

Clone the public repository and create the pinned `auto-code-rover` Conda
environment:

```bash
git clone https://github.com/AnvithPabba/Refine-Program-Repair.git
cd Refine-Program-Repair
conda env create -f environment.yml
conda activate auto-code-rover
```

The source contains `.gitmodules` metadata for SWE-bench-docker but not a
populated submodule. Clone and install the exact tested revision explicitly:

```bash
git clone https://github.com/Marti2203/SWe-bench-docker.git SWE-bench-docker
git -C SWE-bench-docker checkout --detach f7db5326d5175398bd6fc04c523e133430eb465a
python -m pip install ./SWE-bench-docker
```

Verify the Refine command-line interface and Docker integration package:

```bash
PYTHONPATH=. python app/main.py --help
PYTHONPATH=. python app/main.py swe-bench --help
python -c "import swebench_docker; print('SWE-bench-docker import OK')"
```

If the Conda environment already exists, update it from the repository instead:

```bash
conda env update --name auto-code-rover --file environment.yml --prune
conda activate auto-code-rover
```

## Model credentials

Set credentials only in your shell or an external secret manager. For example,
choose the provider you intend to use:

```bash
# OpenAI
export OPENAI_KEY="<your-openai-api-key>"

# Anthropic
export ANTHROPIC_API_KEY="<your-anthropic-api-key>"

# Groq
export GROQ_API_KEY="<your-groq-api-key>"
```

Other implemented providers use these variables:

| Provider | Environment variables |
| --- | --- |
| Azure OpenAI | `AZURE_OPENAI_API_KEY`, `ENDPOINT_URL` |
| AWS Bedrock | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME` |
| Gemini API | `GEMINI_API_KEY` |
| Google Vertex AI | `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_GENAI_USE_VERTEXAI=true` |

**Never commit credentials.** Do not add real keys to `README.md`,
`app/config.py`, tracked JSON files, or `.env` files. Keep Google service-account
JSON outside the repository and point `GOOGLE_APPLICATION_CREDENTIALS` to that
external path.

## Configure draft patches

Workflow defaults live in [`app/config.py`](app/config.py). Refine's primary
mode expects an initial patch for each task:

```python
is_initial_patch_given: bool = True
path_to_patches: str = "/absolute/path/to/draft_patches.json"
```

The draft-patch file maps SWE-bench instance IDs to unified diffs:

```json
{
  "django__django-11133": "<unified diff>",
  "another-instance-id": "<unified diff>"
}
```

Also review these settings before a run:

- Set `use_cached_results = False` unless `cached_results_path` points to a
  valid cache JSON file.
- Set `path_to_unit_test_patch_diffs` only when
  `use_unit_test_patch_diffs = True`.
- Keep Vertex project and credential-path settings empty unless using Vertex
  AI.
- Set `is_initial_patch_given = False` only when running the inherited
  AutoCodeRover workflow without supplied draft patches.

## Set up SWE-bench tasks

Refine uses the
[`yuntongzhang/SWE-bench`](https://github.com/yuntongzhang/SWE-bench) fork to
materialize task environments. Clone the verified revision next to Refine:

```bash
cd ..
git clone https://github.com/yuntongzhang/SWE-bench.git SWE-bench
git -C SWE-bench checkout --detach 54fd416dc4997e78f1b518bba2d934585d90ff59
cd SWE-bench
conda env create -f environment.yml
conda activate swe-bench
```

Install the Linux system packages listed by that fork, then create a task list
with one instance ID per line and materialize it:

```bash
printf '%s\n' django__django-11133 > tasks.txt
python harness/run_setup.py \
  --log_dir logs \
  --testbed testbed \
  --result_dir setup_result \
  --subset_file tasks.txt
```

The command creates `setup_result/setup_map.json` and
`setup_result/tasks_map.json`, which Refine consumes. Materialize every task in
a multi-task list before starting a run. Avoid high setup parallelism: Conda
environment creation is not thread-safe.

## Run Refine

The following commands assume `Refine-Program-Repair/` and `SWE-bench/` are
sibling directories and are run from the Refine repository root.

Run one task:

```bash
conda activate auto-code-rover
PYTHONPATH=. python app/main.py swe-bench \
  --model gpt-4o-2024-08-06 \
  --setup-map ../SWE-bench/setup_result/setup_map.json \
  --tasks-map ../SWE-bench/setup_result/tasks_map.json \
  --output-dir output \
  --task django__django-11133 \
  --enable-validation \
  --reproduce-and-review
```

Run a prepared task list:

```bash
PYTHONPATH=. python app/main.py swe-bench \
  --model gpt-4o-2024-08-06 \
  --setup-map ../SWE-bench/setup_result/setup_map.json \
  --tasks-map ../SWE-bench/setup_result/tasks_map.json \
  --output-dir output \
  --task-list-file conf/swe_lite_tasks_30_subset.txt \
  --model-temperature 0 \
  --enable-validation \
  --reproduce-and-review \
  --num-processes 4
```

The selected repair is written to `selected_patch.json` inside each task's
output directory. Choose `--num-processes` based on host memory, Docker
capacity, and provider rate limits.

## Reproduce the paper experiments

1. Obtain the source agents' draft patches. Public SWE-bench submissions are
   available from the
   [`SWE-bench/experiments`](https://github.com/SWE-bench/experiments)
   repository.
2. Convert the patches to the JSON format in
   [Configure draft patches](#configure-draft-patches), then set
   `path_to_patches` in `app/config.py`.
3. Choose and fully materialize one of the released task lists:
   [`conf/swe_lite_tasks_30_subset.txt`](conf/swe_lite_tasks_30_subset.txt),
   [`conf/swe_lite_tasks.txt`](conf/swe_lite_tasks.txt), or
   [`conf/swe_verified_tasks.txt`](conf/swe_verified_tasks.txt).
4. Run the multi-task command above with the model and ablation settings for
   the desired research question.

The released study uses:

- **RQ1:** the 30-issue subset with draft patches from AutoCodeRover, Agentless,
  CodeV, BlackBoxAI, and ExpeRepair;
- **RQ2:** AutoCodeRover patches on SWE-bench Lite and SWE-bench Verified; and
- **RQ3:** AutoCodeRover SWE-bench Lite patches with Refine's reviewer and
  context-retrieval components ablated through `app/config.py`.

The checked-in [`results/`](results/) tree contains the published outputs used
for inspection and analysis; reruns require model access and can incur API and
compute costs.

## Repository contents

| Path | Purpose |
| --- | --- |
| [`app/`](app/) | Refine and inherited AutoCodeRover implementation |
| [`conf/`](conf/) | Example configurations and SWE-bench task lists |
| [`scripts/`](scripts/) | Experiment and result-processing utilities |
| [`results/`](results/) | Released trajectories, logs, predictions, and evaluation artifacts |
| [`demo_vis/`](demo_vis/) | Demonstration visualization utilities |
| [`EMNLP2026_REFINE.pdf`](EMNLP2026_REFINE.pdf) | Accepted EMNLP 2026 manuscript |
| [`preprint.pdf`](preprint.pdf) | Earlier paper preprint included with the artifact |

The `results/` directory is intentionally large because this release preserves
the complete research artifact rather than only the runtime source.

## Citation

If Refine supports your research, please cite the current arXiv record:

```bibtex
@article{pabba2025refine,
  title={Refine: Enhancing program repair agents through context-aware patch refinement},
  author={Pabba, Anvith and Chen, Simin and Mathai, Alex and Chakraborty, Anindya and Ray, Baishakhi},
  journal={arXiv preprint arXiv:2510.03588},
  year={2025}
}
```

## Security

- `.env`, virtual environments, outputs, and known local credential paths are
  ignored by Git.
- Use environment variables or a provider-managed external credential store.
- Before publishing changes, scan both the working tree and Git history for
  accidentally committed credentials.
- If a real credential is ever committed, revoke or rotate it immediately;
  removing it from the latest tree does not remove it from Git history.

Some files under `results/` contain intentionally fake, sensitive-looking
values from public SWE-bench fixtures. They are preserved as benchmark evidence
and are not live credentials.

## License and acknowledgements

This source is distributed under the
[SONAR Source-Available License v1.0](LICENSE).

Refine builds on
[AutoCodeRover](https://github.com/AutoCodeRoverSG/auto-code-rover). We thank
the AutoCodeRover team; parts of this implementation and setup workflow derive
from their work.
