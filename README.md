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
To transfer participant submissions from the HuggingFace Hub to the lab's local evaluation server while maintaining version control:

***Step A: Locate & Clone***
Each team will provide a link to their HuggingFace repository. Use Git LFS to ensure large model weights are downloaded correctly.
```
# Ensure git-lfs is installed
git lfs install

# Clone the team's repo into the submissions directory
cd submissions/
git clone [https://huggingface.co/teams/Test-Team](https://huggingface.co/teams/Test-Team)
```

***Step B: Verify the files*** Before continuing, verify the following:
1. Weights: Ensure the .pth file in the ckpt/ folder is the correct size (not a small LFS pointer file).
2. Env: Ensure model_env.yaml exists.
3. Entry Point: Verify that team_code/planner.py exists

***Step C: Symbolic Linking*** We use a symbolic link (symlink) to "point" our evaluation script to the team's code without moving files. This ensures a clean workspace.
```
# Remove the link from the previous evaluation
rm -rf leaderboard/team_code

# Create a new shortcut pointing to the current team
ln -s ${PWD}/submissions/Team-Alpha/team_code leaderboard/team_code
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
TEAM_NAME=$1
REPO_URL=$2

echo "Starting Evaluation for $TEAM_NAME..."

# 1. Clone
git clone $REPO_URL submissions/$TEAM_NAME

# 2. Symlink
rm -rf leaderboard/team_code
ln -s ${PWD}/submissions/$TEAM_NAME/team_code leaderboard/team_code

# 3. Env Build (if not exists)
conda env create -f submissions/$TEAM_NAME/model_env.yaml -n mdrive_$TEAM_NAME

# 4. Run
conda activate mdrive_$TEAM_NAME
export CARLA_ROOT=/path/to/CARLA_0.9.15
export PYTHONPATH=$PYTHONPATH:$CARLA_ROOT/PythonAPI/carla/dist/carla-0.9.15-py3.7-linux-x86_64.egg

python tools/run_custom_eval.py --agent leaderboard/team_code/planner.py --port 2000
```

# MDriveBench Submission Instructions
To ensure your model is compatible and evaluated accurately by our lab, all submissions must adhere to the following directory structure and environment requirements.

## Required Repository Structure
Your HuggingFace repository must be organized as follows:
```
/ (root)
├── team_code/
│   ├── planner.py          # Main agent class (must inherit from BaseAgent)
│   └── [others].py         # Your model architecture and utility files
├── ckpt/
│   └── model.pth           # Trained weights (or multiple files if required)
├── configs/
│   └── submission_config.yaml  # Model hyperparams, sensor FOV, and V2X settings
└── model_env.yaml          # Standard Conda environment specification
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
