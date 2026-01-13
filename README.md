# MDriveBench Evaluation
This section serves as the internal manual evaluation pipeline for MDriveBench Challenge. It provides the our lab with a standardized workflow to manually evaluate participant submissions and ensures that all results are benchmarked against the same hardware and software constraints.

## Evaluation Metrics
MDriveBench Leadboard evaluates on two metrics: 
1. **Driving Score (DS)**: Score of route completion adjusted by infraction penalties
2. **Sucess Rate (SR)**: The precentage of routes completed without failure.

### Evaluation Scenarios
A full evaluation consists of three distinct benchmarks:
***OpenCDA (12 Scenarios):*** Uses ZIP-based scenario loading. Ensure all 12 ZIPs (including Scenes A, D, G, J) are in the opencdascenarios/ folder.
***InterDrive (Full Suite):*** Cooperative driving evaluated via the Interdrive_all set.
***Safety-Critical:*** Pre-crash scenarios.

## Evaluation Workflow
Evaluation consists of 3 main phases: Submission Retrieval, Environment Setup, and Checkpoint Evaluation.

Before evaluating any submission, ensure to follow the CoLMDriver Global Setup (https://github.com/marco-cos/CoLMDriver)

1. Verify CARLA 0.9.15 is installed and the egg is linked.
2. Ensure the vllm (for inference) and colmdriver (for simulation) environments are functional.
3. Confirm spconv (1.2.1) and pypcd are installed in the base environment, as many baselines and submissions rely on these for voxel feature generation.

### 1. Submission Retrieval
To transfer participant submissions from  HuggingFace to the lab's local evaluation server:

***Step A:*** Download & Unzip Download the participant's .zip file from the submission portal into the submissions/ directory.
```
unzip Team-A_submission.zip -d submissions/Team-A
```

***Step B:*** Verify Structure Ensure the unzipped folder contains the following files:
```
agents.py
config/
src/
weights/
model_env.yaml
```

***Step C:*** Symbolic Linking Point the evaluation suite to the new submission.
```
# Remove previous link and point to the current team
rm -rf leaderboard/team_code
ln -s ${PWD}/submissions/Team-A leaderboard/team_code
```


### 2. Enviroment Setup
To prevent discrepancies caused by library version mismatches, we build a fresh environment for every team.
```
# Build the team's specific environment
conda env create -f submissions/Test-Team/model_env.yaml -n mdrive_eval_test
conda activate mdrive_eval_test
```
### 3. Checkpoint Evaluation
***Step A:*** Inject the standardized CARLA paths into the active team environment.
```
export CARLA_ROOT=/path/to/lab/CARLA_0.9.15
export PYTHONPATH=$PYTHONPATH:$CARLA_ROOT/PythonAPI/carla/dist/carla-0.9.15-py3.7-linux-x86_64.egg
```
***Step B:*** Running VLM, LLM (from repository root)
```
#Enter conda ENV
conda activate vllm
# VLM on call
CUDA_VISIBLE_DEVICES=6 vllm serve ckpt/colmdriver/VLM --port 1111 --max-model-len 8192 --trust-remote-code --enable-prefix-caching

# LLM on call (in new terminal, with vllm env activated)
CUDA_VISIBLE_DEVICES=7 vllm serve ckpt/colmdriver/LLM --port 8888 --max-model-len 4096 --trust-remote-code --enable-prefix-caching
Make sure that the CUDA_VISIBLE_DEVICES variable is set to a GPU available, which can be checked using the nvidia-smi command

Note: make sure that the selected ports (1111,8888) are not occupied by other services. If you use other ports, please modify values of key 'comm_client' and 'vlm_client' in simulation/leaderboard/team_code/agent_config/colmdriver_config.yaml accordingly.
```

***Step C:*** Run Evaluation
```
# ==============================================================================
# BATCH 1: OpenCDA Scenarios (12 ZIPs)
# ==============================================================================
echo ">>> [BATCH 1/3] Running OpenCDA Scenarios..."
SCENARIO_DIR="opencdascenarios"
for zipfile in "$SCENARIO_DIR"/*.zip; do
    name=$(basename "$zipfile" .zip)
    $RUN_CMD tools/run_custom_eval.py \
      --zip "$zipfile" \
      --scenario-name "$name" \
      --results-tag "${name}_${TEAM_NAME}" \
      --agent "$SUB_DIR/agents.py" \
      --agent-config "$SUB_DIR/config/submission_config.yaml" \
      --port $PORT
done

# ==============================================================================
# BATCH 2: InterDrive Benchmark (Full Suite)
# ==============================================================================
echo ">>> [BATCH 2/3] Running InterDrive All..."
# Note: eval_mode.sh must be present in your scripts/eval directory
bash scripts/eval/eval_mode.sh $GPU $PORT $TEAM_NAME ideal Interdrive_all

# ==============================================================================
# BATCH 3: Safety-Critical Scenarios (4 Routes)
# ==============================================================================
echo ">>> [BATCH 3/3] Running Safety-Critical Scenarios..."
$RUN_CMD tools/run_custom_eval.py \
    --agent "$SUB_DIR/agents.py" \
    --routes "data/warmup/safety_critical.xml" \
    --scenarios "data/warmup/safety_critical.json" \
    --port $PORT \
    --results-tag "safety_${TEAM_NAME}"

echo "Evaluation Complete for $TEAM_NAME."
```

***Step D:*** Record DS & SR Collect DS & SR Extract the Driving Score (DS) and Success Rate (SR) from the generated summary.json. Verify logs manually if the score is unexpectedly low to ensure no simulator glitches occurred.

## Proposed Automated Evaluation Script (Docker approach with Conda .yaml fallback)
The purpose of this evaluation script is that it automates the environment building and the simulation loop as described in the steps 2 and 3.
### Evaluation Protocol
The evaluation script evaluate_all.sh follows a hybrid infrastructure:
Docker Execution: If the submission contains a Dockerfile, the evaluator builds a container to isolate the environment. This is the recommended method for models with complex dependencies (like VAD).
Conda Fallback: If no Dockerfile is detected, the script creates a Conda environment from model_env.yaml.
Hardware Mapping: Host GPUs are mapped directly to the evaluation process via CUDA_VISIBLE_DEVICES or the --gpus all Docker flag.

Save the script as ``` evaluate_all.sh ```
Make it executable:
```
chmod +x evaluate_all.sh
```
Run it:
```
./evaluate_all.sh
```

```
#!/bin/bash

# ==============================================================================
# MDriveBench Final Evaluation Script (Docker + Pre-setup Conda Fallback)
# Target Path: /data2/angela_test_envs/CoLMDriver
# ==============================================================================

TEAM_NAME=$1      # e.g., tcp, vad, colmdriver, or a new team name
SUBMISSION_ZIP=$2 
GPU=${3:-0}
PORT=2002 

# --- 1. Workspace Initialization ---
SUB_DIR="${PWD}/submissions/$TEAM_NAME"
mkdir -p "$SUB_DIR"
unzip -qo "$SUBMISSION_ZIP" -d "$SUB_DIR"

# --- 2. Environment Provisioning Logic ---
source "$(conda info --base)/etc/profile.d/conda.sh"

# PATH A: Docker Execution (If Dockerfile exists)
if [[ -f "$SUB_DIR/Dockerfile" ]]; then
    echo ">>> [ENV] Dockerfile detected. Building Image: mdrive_$TEAM_NAME"
    docker build -t "mdrive_$TEAM_NAME" "$SUB_DIR"
    RUN_CMD="docker run --rm --gpus all --net=host -v /path/to/lab/CARLA_0.9.10.1:/workspace/carla_root mdrive_$TEAM_NAME"

# PATH B: Pre-setup Lab Environments (Fallback 1)
else
    echo ">>> [ENV] No Dockerfile. Checking for pre-setup baseline in /data2..."
    case $TEAM_NAME in
      "tcp")        ENV_PATH="/data2/angela_test_envs/CoLMDriver/envs/tcp_codriving" ;;
      "vad")        ENV_PATH="/data2/angela_test_envs/CoLMDriver/envs/vad_env" ;;
      "colmdriver") ENV_PATH="/data2/angela_test_envs/CoLMDriver/envs/colmdriver" ;;
      "uniad")      ENV_PATH="/data2/angela_test_envs/CoLMDriver/envs/uniad_env" ;;
      "lmdrive")    ENV_PATH="/data2/angela_test_envs/CoLMDriver/envs/lmdrive" ;;
      *)            ENV_PATH="" ;; 
    esac

    if [[ -n "$ENV_PATH" && -d "$ENV_PATH" ]]; then
        echo ">>> [ENV] Activating pre-setup environment: $ENV_PATH"
        conda activate "$ENV_PATH"
        RUN_CMD="python"
    else
        # PATH C: Fresh Conda Build (Fallback 2)
        echo ">>> [ENV] New team detected. Building fresh Conda environment..."
        if ! conda info --envs | grep -q "mdrive_$TEAM_NAME"; then
            conda env create -f "$SUB_DIR/model_env.yaml" -n "mdrive_$TEAM_NAME"
        fi
        conda activate "mdrive_$TEAM_NAME"
        RUN_CMD="python"
    fi
fi

# --- 3. Global Paths & GPU ---
export CARLA_ROOT=/path/to/lab/CARLA_0.9.10.1
export PYTHONPATH=$PYTHONPATH:$CARLA_ROOT/PythonAPI/carla/dist/carla-0.9.10-py3.7-linux-x86_64.egg
export CUDA_VISIBLE_DEVICES=$GPU

# ==============================================================================
# BATCH 1: OpenCDA Scenarios (12 ZIPs)
# ==============================================================================

echo ">>> [BATCH 1/3] Running OpenCDA Scenarios from opencdascenarios/..."
for zipfile in opencdascenarios/*.zip; do
    name=$(basename "$zipfile" .zip)
    $RUN_CMD tools/run_custom_eval.py \
      --zip "$zipfile" \
      --scenario-name "$name" \
      --results-tag "${name}_${TEAM_NAME}" \
      --agent "$SUB_DIR/agents.py" \
      --agent-config "$SUB_DIR/config/submission_config.yaml" \
      --port $PORT
done

# ==============================================================================
# BATCH 2: InterDrive Benchmark (Full Suite)
# ==============================================================================
echo ">>> [BATCH 2/3] Running InterDrive All..."
bash scripts/eval/eval_mode.sh $GPU $PORT $TEAM_NAME ideal Interdrive_all

# ==============================================================================
# BATCH 3: Safety-Critical Scenarios (4 Routes)
# ==============================================================================
echo ">>> [BATCH 3/3] Running Safety-Critical..."
$RUN_CMD tools/run_custom_eval.py \
    --agent "$SUB_DIR/agents.py" \
    --routes "data/warmup/safety_critical.xml" \
    --scenarios "data/warmup/safety_critical.json" \
    --port $PORT \
    --results-tag "safety_${TEAM_NAME}"

echo "Evaluation Complete for $TEAM_NAME."
```

# MDriveBench Submission Instructions
To ensure your model is evaluated accurately, you must submit a single .zip file containing your model and code.

## Required ZIP File Structure
Your ZIP file must be organized as follows:
```
team_name.zip
├── agents.py           # Main agent class (must inherit from BaseAgent)
├── config/             # Folder containing all .yaml or .py configs
├── src/                # Folder containing model architecture & utilities
├── weights/            # Folder containing all trained checkpoints (.pth/.ckpt)
└── model_env.yaml      # Conda environment specification
```

## Enviroment Specification (model_env.yaml)
Due to the varying requirements of driving foundation models, we will build a dedicated Conda environment for your submission.

* Your model_env.yaml should include all specific dependencies (e.g., transformers, timm, spconv).
* We recommend basing your environment on Python 3.8+ and PyTorch 2.1+.
* Do not include CARLA in your YAML; the evaluation server will automatically link a standardized CARLA 0.9.15 build to your environment to ensure physics consistency.

## Evaluation Protocol 
Our team will manually verify your submission using the following pipeline:

1. Env Build: conda env create -f model_env.yaml
2. Path Injection: Standardized CARLA 0.9.15 PythonAPI will be appended to your PYTHONPATH.
3. Execution: Your agent will be run through a batch of closed-loop scenarios (Standard, Pre-crash, and Safety-critical).
4. Scoring: We will record the Driving Score (DS) and Success Rate (SR) as the official leaderboard metrics.
