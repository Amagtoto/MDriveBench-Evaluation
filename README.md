# MDriveBench Evaluation
This section serves as the internal manual evaluation pipeline for MDriveBench Challenge. It provides the our lab with a standardized workflow to manually evaluate participant submissions and ensures that all results are benchmarked against the same hardware and software constraints.

## Evaluation Metrics
MDriveBench Leadboard evaluates on two metrics: 
1. **Driving Score (DS)**: Score of route completion adjusted by infraction penalties
2. **Sucess Rate (SR)**: The precentage of routes completed without failure.

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
python tools/run_custom_eval.py \
    --agent leaderboard/team_code/planner.py \
    --checkpoint submissions/[TEAM_ID]/ckpt/model.pth \
    --routes data/warmup_routes.xml \
    --port 2000
```

***Step D:*** Record DS & SR Collect DS & SR Extract the Driving Score (DS) and Success Rate (SR) from the generated summary.json. Verify logs manually if the score is unexpectedly low to ensure no simulator glitches occurred.

## Proposed Automated Evaluation Script
The purpose of this evaluation script is that it automates the environment building and the simulation loop as described in the steps 2 and 3.
```
#!/bin/bash

# Usage: bash evaluate_team.sh [TEAM_NAME] [ZIP_FILE_PATH] [GPU_ID]
TEAM_NAME=$1
ZIP_PATH=$2
GPU=${3:-0}

echo "-------------------------------------------------------"
echo "Starting MDriveBench Evaluation for: $TEAM_NAME"
echo "-------------------------------------------------------"

# 1. Prepare Workspace
SUBMISSION_DIR="${PWD}/submissions/$TEAM_NAME"
mkdir -p "$SUBMISSION_DIR"

# 2. Extract Submission
if [[ $ZIP_PATH == *.zip ]]; then
    echo "[1/6] Extracting ZIP file..."
    unzip -qo "$ZIP_PATH" -d "$SUBMISSION_DIR"
else
    echo "ERROR: Please provide a valid .zip file."
    exit 1
fi

# 3. ZIP Format Validation (NEW)
echo "[2/6] Validating file structure..."
MISSING_FILES=()
[[ ! -f "$SUBMISSION_DIR/agents.py" ]] && MISSING_FILES+=("agents.py")
[[ ! -d "$SUBMISSION_DIR/config" ]] && MISSING_FILES+=("config/")
[[ ! -d "$SUBMISSION_DIR/src" ]] && MISSING_FILES+=("src/")
[[ ! -d "$SUBMISSION_DIR/weights" ]] && MISSING_FILES+=("weights/")
[[ ! -f "$SUBMISSION_DIR/model_env.yaml" ]] && MISSING_FILES+=("model_env.yaml")

if [ ${#MISSING_FILES[@]} -ne 0 ]; then
    echo "ERROR: Invalid ZIP format. Missing: ${MISSING_FILES[*]}"
    exit 1
fi
echo "Structure validated successfully."

# 4. Update Symbolic Link
echo "[3/6] Updating Symbolic Link..."
rm -rf leaderboard/team_code
ln -s "$SUBMISSION_DIR" leaderboard/team_code

# 5. Environment Provisioning
echo "[4/6] Setting up Conda Environment..."
# Initialize conda for bash
source "$(conda info --base)/etc/profile.d/conda.sh"

ENV_NAME="mdrive_$TEAM_NAME"
if ! conda info --envs | grep -q "$ENV_NAME"; then
    echo "Building new environment: $ENV_NAME..."
    conda env create -f "$SUBMISSION_DIR/model_env.yaml" -n "$ENV_NAME" || { echo "Conda build failed"; exit 1; }
fi

# 6. Pre-Run 
echo "-------------------------------------------------------"
echo "CRITICAL CHECK: Are VLM and LLM active?"
read -p "Confirm (y/n): " -n 1 -r
echo
[[ ! $REPLY =~ ^[Yy]$ ]] && { echo "Aborting."; exit 1; }

# 7. Execute Evaluation
echo "[5/6] Launching CARLA Simulation..."
conda activate "$ENV_NAME"

# Path Injection (Standardized CARLA 0.9.15)
export CARLA_ROOT=/path/to/lab/CARLA_0.9.15
export PYTHONPATH=$PYTHONPATH:$CARLA_ROOT/PythonAPI/carla/dist/carla-0.9.15-py3.7-linux-x86_64.egg
export CUDA_VISIBLE_DEVICES=$GPU

python tools/run_custom_eval.py \
    --agent leaderboard/team_code/agents.py \
    --checkpoint "$SUBMISSION_DIR/weights/model.pth" \
    --routes data/warmup_routes.xml \
    --port 2000

# 8. Finalize
echo "[6/6] Evaluation Complete."
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
