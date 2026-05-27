# AgentMob: Anonymous Artifact README

This repository contains the implementation and reproduction materials for the anonymous submission:

> Towards Efficient and Evidence-grounded Mobility Prediction with LLM-Driven Agents

AgentMob is a training-free LLM-driven agent framework for individual next-location prediction. Instead of using a fixed prompt or training a task-specific sequence model, AgentMob first handles routine cases with a historical-regularity fast path, then uses an LLM controller to invoke mobility-analysis tools for ambiguous cases. The tool outputs provide timestamp-bounded evidence over recent trajectories, historical behavior, stay-move likelihood, geographical distance, and location semantics.

## Anonymous Review Notice

This artifact is prepared for double-blind review. Please do not infer author identity from repository metadata, commit history, package namespaces, or file paths. Any author-identifying information has been removed or replaced with neutral placeholders where possible.

If you discover identity-revealing metadata in the artifact, please ignore it for the purpose of review and report it as an artifact issue.

## Repository Structure

The expected repository layout is:

```text
.
|-- README.md
|-- requirements.txt / pyproject.toml
|-- configs/
|   |-- bw.yaml
|   |-- yjmob100k.yaml
|   `-- shanghai_isp.yaml
|-- data/
|   |-- raw/                 # not included when redistribution is restricted
|   |-- processed/           # processed trajectories and spatial-unit metadata
|   `-- descriptions/        # generated location descriptions
|-- agentmob/
|   |-- agents/              # LLM controller and tool-calling loop
|   |-- tools/               # mobility context, geography, stay-move, behavior tools
|   |-- data/                # preprocessing and split utilities
|   |-- baselines/           # baseline prompt wrappers and statistical variants
|   `-- evaluation/          # Acc@1, MRR@5, distance, trace analysis
|-- scripts/
|   |-- preprocess_*.py
|   |-- run_agentmob.py
|   |-- run_baselines.py
|   |-- evaluate.py
|   `-- summarize_results.py
`-- outputs/
    |-- traces/              # per-sample tool-use traces
    |-- predictions/         # ranked prediction files
    `-- tables/              # aggregated metrics
```

Some paths may differ slightly in the released code. The important entry points are the preprocessing scripts, AgentMob inference script, baseline runners, and evaluation scripts.

## Main Components

AgentMob consists of the following components.

`Fast-path predictor` resolves highly regular samples directly when the same weekday-hour history has a dominant next-location signal.

`LLM controller` decides whether more evidence is needed, invokes tools, interprets tool outputs, and produces a ranked candidate list.

`Mobility Context Retriever` summarizes recent trajectory context and same weekday-hour visitation statistics.

`Geographical Information Retriever` returns candidate distances and location descriptions.

`Stay-Move Estimator` estimates whether the user is likely to remain at the current location or move elsewhere based on stay-duration history.

`Historical Behavior Retriever` retrieves visit frequency, dwell time, transition tendencies, and comparable historical behavior.

`Trace logger` stores tool invocations, serialized evidence, final predictions, and evaluation metadata for auditability.

## Environment Setup

Create a fresh Python environment:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -r requirements.txt
```

If the repository uses `pyproject.toml`, install it in editable mode:

```bash
pip install -e .
```

The experiments require an LLM backend for the agent controller. Configure the backend with environment variables, for example:

```bash
export OPENAI_API_KEY="..."
export AGENTMOB_MODEL="gpt-5.4"
```

For open-source local backbones, configure the model endpoint in the corresponding config file under `configs/`.

## Data

The paper evaluates AgentMob on three mobility datasets:

| Dataset | Spatial unit | Evaluation subset |
| --- | --- | --- |
| BW | third-level administrative polygons | top-100 active users |
| YJMob100K | 500 m x 500 m grid cells | top-100 active users |
| Shanghai ISP | 500 m x 500 m grid cells | 200 first-test users |

Due to dataset licensing and privacy restrictions, raw mobility records may not be redistributed in this repository. Place downloaded raw datasets under:

```text
data/raw/bw/
data/raw/yjmob100k/
data/raw/shanghai_isp/
```

Then run the preprocessing scripts to generate chronological splits, spatial-unit sequences, candidate sets, and location-description files:

```bash
python scripts/preprocess_bw.py --config configs/bw.yaml
python scripts/preprocess_yjmob100k.py --config configs/yjmob100k.yaml
python scripts/preprocess_shanghai_isp.py --config configs/shanghai_isp.yaml
```

The generated files should appear under `data/processed/` and `data/descriptions/`.

## Running AgentMob

Run AgentMob on one dataset:

```bash
python scripts/run_agentmob.py \
  --config configs/bw.yaml \
  --model "$AGENTMOB_MODEL" \
  --output outputs/predictions/bw_agentmob_gpt54.jsonl \
  --trace-dir outputs/traces/bw_agentmob_gpt54/
```

Repeat for the other datasets:

```bash
python scripts/run_agentmob.py --config configs/yjmob100k.yaml --model "$AGENTMOB_MODEL"
python scripts/run_agentmob.py --config configs/shanghai_isp.yaml --model "$AGENTMOB_MODEL"
```

Each prediction trace records the user id, target timestamp, allowed history range, invoked tools, tool outputs, final ranked prediction, and optional reasoning summaries.

## Running Baselines

Run LLM-based baselines with the same chronological split and candidate sets:

```bash
python scripts/run_baselines.py --method agentmove --config configs/bw.yaml --model "$AGENTMOB_MODEL"
python scripts/run_baselines.py --method llm_mob --config configs/bw.yaml --model "$AGENTMOB_MODEL"
python scripts/run_baselines.py --method trajllm --config configs/bw.yaml --model "$AGENTMOB_MODEL"
python scripts/run_baselines.py --method llm_urban_residents --config configs/bw.yaml --model "$AGENTMOB_MODEL"
```

Run the non-LLM statistical variant used in the controller analysis:

```bash
python scripts/run_agentmob.py \
  --config configs/bw.yaml \
  --controller statistics \
  --output outputs/predictions/bw_agentmob_statistics.jsonl
```

If supervised baselines are included in this artifact, train and evaluate them with:

```bash
python scripts/run_supervised_baseline.py --method deepmove --config configs/bw.yaml
python scripts/run_supervised_baseline.py --method transformer --config configs/bw.yaml
```

## Evaluation

Evaluate a prediction file:

```bash
python scripts/evaluate.py \
  --config configs/bw.yaml \
  --predictions outputs/predictions/bw_agentmob_gpt54.jsonl \
  --metrics acc_at_1 mrr_at_5 distance
```

Aggregate all completed runs into paper-style tables:

```bash
python scripts/summarize_results.py \
  --input outputs/predictions/ \
  --output outputs/tables/main_results.csv
```

The main reported metrics are:

`Acc@1`: whether the top-ranked prediction equals the ground-truth location.

`MRR@5`: reciprocal rank of the ground-truth location among the top five predictions, or zero if absent.

`Distance`: Haversine distance in kilometers between the top-1 predicted location and ground truth.

## Expected Results

The paper reports the following AgentMob GPT-5.4 performance:

| Dataset | Acc@1 | MRR@5 | Distance |
| --- | ---: | ---: | ---: |
| BW | 71.42% | 78.84% | 2.20 km |
| YJMob100K | 33.14% | 46.55% | 4.29 km |
| Shanghai ISP | 33.50% | 47.44% | 4.23 km |

Small deviations are expected because LLM inference can vary across model versions, sampling settings, API backends, and retry behavior. For reproducibility, use deterministic decoding when supported and record the model version, timestamp, and provider response metadata.

## Reproducing Paper Analyses

Controller contribution:

```bash
python scripts/analyze_controller_effect.py \
  --agentmob outputs/predictions/bw_agentmob_gpt54.jsonl \
  --statistics outputs/predictions/bw_agentmob_statistics.jsonl \
  --output outputs/tables/controller_effect.csv
```

Tool ablations:

```bash
python scripts/run_agentmob.py --config configs/bw.yaml --ablate stay_move
python scripts/run_agentmob.py --config configs/bw.yaml --ablate historical_behavior
python scripts/run_agentmob.py --config configs/bw.yaml --ablate mobility_context
python scripts/run_agentmob.py --config configs/bw.yaml --ablate location_profiler
```

Efficiency statistics:

```bash
python scripts/analyze_efficiency.py \
  --trace-dir outputs/traces/ \
  --output outputs/tables/efficiency.csv
```

Case studies:

```bash
python scripts/export_case_studies.py \
  --trace-dir outputs/traces/ \
  --output outputs/case_studies/
```

## Reproducibility Notes

All preprocessing and inference must respect chronological validity. Tools should only access the training split and observations available before the target timestamp.

Location descriptions are generated once and shared across LLM-based methods to keep the comparison fair.

For privacy, logs should not include raw GPS coordinates unless the corresponding dataset license allows redistribution.

For double-blind review, do not include author names, institution names, personal account names, acknowledgements, or non-anonymous cloud storage links in config files, logs, notebook metadata, or commit metadata.

## License and Dataset Terms

This code is released for anonymous academic review. A full license will be provided in the camera-ready or public release.

Dataset access is governed by the original dataset providers. Users are responsible for obtaining permission to use BW, YJMob100K, and Shanghai ISP data and for complying with all applicable privacy and redistribution terms.

## Contact During Review

For double-blind review, please use the conference review system for questions and artifact issues. Do not contact the authors directly during the anonymous review period.
