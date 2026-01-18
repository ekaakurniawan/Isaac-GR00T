<div align="center">

  <img src="media/header_compress.png" width="800" alt="NVIDIA Isaac GR00T N1.6 Header">

  <!-- --- -->
  
  <p style="font-size: 1.2em;">
    <a href="https://developer.nvidia.com/isaac/gr00t"><strong>Website</strong></a> | 
    <a href="https://huggingface.co/nvidia/GR00T-N1.6-3B"><strong>Model</strong></a> |
    <a href="https://huggingface.co/datasets/nvidia/PhysicalAI-Robotics-GR00T-X-Embodiment-Sim"><strong>Dataset</strong></a> |
    <a href="https://arxiv.org/abs/2503.14734"><strong>Paper</strong></a> |
    <a href="https://research.nvidia.com/labs/gear/gr00t-n1_6/"><strong>Research Blog</strong></a>
  </p>
</div>

## NVIDIA Isaac GR00T

<div align="center">
  <img src="media/stacked_demo.gif" width="800" alt="GR00T Demo">
</div>

> We just released GR00T N1.6, an updated version of GR00T N1 with improved performance and new features. Check out the [release blog post](https://research.nvidia.com/labs/gear/gr00t-n1_6/) for more details.

> To use the older version, N1.5, please checkout the [n1.5-release](https://github.com/NVIDIA/Isaac-GR00T/tree/n1.5-release) branch.

NVIDIA Isaac GR00T N1.6 is an open vision-language-action (VLA) model for generalized humanoid robot skills. This cross-embodiment model takes multimodal input, including language and images, to perform manipulation tasks in diverse environments.

GR00T N1.6 is trained on a diverse mixture of robot data including bimanual, semi-humanoid and an expansive humanoid dataset. It is adaptable through post-training for specific embodiments, tasks and environments.

The neural network architecture of GR00T N1.6 is a combination of vision-language foundation model and diffusion transformer head that denoises continuous actions. Here is a schematic diagram of the architecture:

<div align="center">
<img src="media/model-architecture.png" width="800" alt="model-architecture">
</div>

Here is the general procedure to use GR00T N1.6:

1. We assume the user has already collected a dataset of robot demonstrations in the form of (video, state, action) triplets for a specific task. 
2. The user will first convert the demonstration data into the LeRobot compatible data schema (more info in [`getting_started/data_preparation.md`](getting_started/data_preparation.md)), which is compatible with the upstream [Huggingface LeRobot Dataset V2](https://github.com/huggingface/lerobot).
3. Our repo provides convenient scripts to validate zero-shot performance of the pretrained model (see [Policy API Guide](getting_started/policy.md) and [RoboCasa Zero-Shot](examples/robocasa-gr1-tabletop-tasks/README.md)).
4. Our repo provides examples of different configurations for training with different robot embodiments (see [`examples/`](examples/) and [Fine-tuning Guide](getting_started/finetune_new_embodiment.md)).
5. Our repo provides convenient scripts for finetuning the pre-trained GR00T N1.6 model on user's data, and running inference, see [`examples`](examples).
6. Our repo provides convenient scripts to run academic simulation benchmarks with finetuned checkpoints (see [LIBERO](examples/LIBERO/README.md), [SimplerEnv](examples/SimplerEnv/README.md), [RoboCasa](examples/robocasa/README.md)).
7. The user will need to connect the `Gr00tPolicy` to the robot controller to execute actions on their target hardware.

## What's New in GR00T N1.6

GR00T N1.6 represents a significant upgrade over GR00T N1.5, with improvements in both model architecture and data leading to better performance in many aspects.

### Model and Data Improvements

Architectural changes:
- Base VLM: We use an internal NVIDIA Cosmos-Reason-2B VLM variant. The VLM supports flexible resolution and can encode images in their native aspect ratio without padding. The VLM is trained both general vision-language tasks and embodied reasoning tasks like next action prediction.
- Uses 2x larger DiT (32 layers vs 16 layers in N1.5).
- Removes N1.5's post-VLM 4-layer transformer adapter. Instead, unfreezes top 4 layers of the VLM during pretraining.
- Predicts state-relative action chunks for most embodiments, rather than absolute joint angles or EEF positions.

Beyond the N1.5 data mixture, the N1.6 pretraining data additionally includes several thousand hours of teleoperated data from:
- Bimanual YAM arms
- AGIBot Genie1
- Simulated Galaxea R1 Pro on the BEHAVIOR suite
- Whole-Body Locomanipulation with Unitree G1

Other code-level improvements:
- Faster dataloader with sharded dataloader support.
- RTC and Async Policy Wrapper for inference (soon to release)
- Simplified data processing pipeline with `processing_gr00t_n1d6.py`
- Flexible Training configuration

## Target Audience

GR00T N1.6 is intended for researchers and professionals in robotics. This repository provides tools to:

- Leverage a pre-trained foundation model for robot control
- Fine-tune on small, custom datasets
- Adapt the model to specific robotics tasks with minimal data
- Deploy the model for inference

The focus is on enabling customization of robot behaviors through finetuning.

## Installation Guide and Quick Start on OpenVINO™

Tested on:
 - CPU: Intel® Core™ Ultra 9 Processor 285K
 - CPU Cores: 24 (8 Performance-cores and 16 Efficient-cores)
 - CPU Threads: 24
 - Memory: 64 GB
 - GPU: Intel® Arc™ B580 Graphics
 - GPU Memory: 12 GB
 - Storage minimum 250 GB

Install packages.

```sh
sudo apt install git-lfs
sudo git lfs install
sudo snap install astral-uv
```

Setup environment.

```sh
uv sync --python 3.10
uv pip install -e .
```

Download dataset.

```sh
git clone \
  --filter=blob:none \
  --no-checkout \
  https://huggingface.co/datasets/nvidia/PhysicalAI-Robotics-GR00T-X-Embodiment-Sim
cd PhysicalAI-Robotics-GR00T-X-Embodiment-Sim
git sparse-checkout init --cone
git sparse-checkout set \
  "gr1_unified.PnPBottleToCabinetClose_GR1ArmsAndWaistFourierHands_1000"
git checkout
```

Change modality from `ego_view` to `ego_view_bg_crop_pad_res256_freq20`
in `modality.json` the dataset, from;
```json
    "video": {
        "ego_view": {
            "original_key": "observation.images.ego_view"
        }
    },
```
to;
```json
    "video": {
        "ego_view_bg_crop_pad_res256_freq20": {
            "original_key": "observation.images.ego_view"
        }
    },
```

### Convert Model to ONNX™

```sh
uv run python scripts/deployment/export_onnx_n1d6.py \
  --model_path nvidia/GR00T-N1.6-3B \
  --dataset_path $HOME/Workspace/datasets/PhysicalAI-Robotics-GR00T-X-Embodiment-Sim/gr1_unified.PnPBottleToCabinetClose_GR1ArmsAndWaistFourierHands_1000 \
  --output_dir ./groot_n1d6_onnx
```

Error message.

```
Traceback (most recent call last):
  File "/home/eka/Workspace/Isaac-GR00T/scripts/deployment/export_onnx_n1d6.py", line 396, in <module>
    main(args)
  File "/home/eka/Workspace/Isaac-GR00T/scripts/deployment/export_onnx_n1d6.py", line 339, in main
    observation = prepare_observation(policy, dataset, traj_idx=0)
  File "/home/eka/Workspace/Isaac-GR00T/scripts/deployment/export_onnx_n1d6.py", line 110, in prepare_observation
    traj = dataset[traj_idx]
  File "/home/eka/Workspace/Isaac-GR00T/gr00t/data/dataset/lerobot_episode_loader.py", line 486, in __getitem__
    video_data = self._load_video_data(episode_id, np.arange(actual_length))
  File "/home/eka/Workspace/Isaac-GR00T/gr00t/data/dataset/lerobot_episode_loader.py", line 377, in _load_video_data
    video_data[image_key] = get_frames_by_indices(
  File "/home/eka/Workspace/Isaac-GR00T/gr00t/utils/video_utils.py", line 274, in get_frames_by_indices
    decoder = torchcodec.decoders.VideoDecoder(
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/torchcodec/decoders/_video_decoder.py", line 89, in __init__
    self._decoder = create_decoder(source=source, seek_mode=seek_mode)
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/torchcodec/decoders/_decoder_utils.py", line 27, in create_decoder
    return core.create_from_file(source, seek_mode)
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/torch/_ops.py", line 829, in __call__
    return self._op(*args, **kwargs)
NotImplementedError: There were no tensor arguments to this function (e.g., you passed an empty list of Tensors), but no fallback function is registered for schema torchcodec_ns::create_from_file.  This usually means that this function requires a non-empty list of Tensors, or that you (the operator writer) forgot to register a fallback function.  Available functions are [XPU, Meta, BackendSelect, Python, FuncTorchDynamicLayerBackMode, Functionalize, Named, Conjugate, Negative, ZeroTensor, ADInplaceOrView, AutogradOther, AutogradCPU, AutogradCUDA, AutogradXLA, AutogradMPS, AutogradXPU, AutogradHPU, AutogradLazy, AutogradMTIA, AutogradMAIA, AutogradMeta, Tracer, AutocastCPU, AutocastMTIA, AutocastMAIA, AutocastXPU, AutocastMPS, AutocastCUDA, FuncTorchBatched, BatchedNestedTensor, FuncTorchVmapMode, Batched, VmapMode, FuncTorchGradWrapper, PythonTLSSnapshot, FuncTorchDynamicLayerFrontMode, PreDispatch, PythonDispatcher].

XPU: registered at /pytorch/build/xpu/ATen/RegisterXPU_0.cpp:54657 [backend fallback]
Meta: registered at /dev/null:160 [kernel]
BackendSelect: fallthrough registered at /pytorch/aten/src/ATen/core/BackendSelectFallbackKernel.cpp:3 [backend fallback]
Python: registered at /__w/torchcodec/torchcodec/pytorch/torchcodec/src/torchcodec/_core/custom_ops.cpp:645 [kernel]
FuncTorchDynamicLayerBackMode: registered at /pytorch/aten/src/ATen/functorch/DynamicLayer.cpp:479 [backend fallback]
Functionalize: registered at /pytorch/aten/src/ATen/FunctionalizeFallbackKernel.cpp:375 [backend fallback]
Named: registered at /pytorch/aten/src/ATen/core/NamedRegistrations.cpp:7 [backend fallback]
Conjugate: registered at /pytorch/aten/src/ATen/ConjugateFallback.cpp:17 [backend fallback]
Negative: registered at /pytorch/aten/src/ATen/native/NegateFallback.cpp:18 [backend fallback]
ZeroTensor: registered at /pytorch/aten/src/ATen/ZeroTensorFallback.cpp:86 [backend fallback]
ADInplaceOrView: fallthrough registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:104 [backend fallback]
AutogradOther: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:63 [backend fallback]
AutogradCPU: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:67 [backend fallback]
AutogradCUDA: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:75 [backend fallback]
AutogradXLA: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:87 [backend fallback]
AutogradMPS: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:95 [backend fallback]
AutogradXPU: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:71 [backend fallback]
AutogradHPU: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:108 [backend fallback]
AutogradLazy: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:91 [backend fallback]
AutogradMTIA: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:79 [backend fallback]
AutogradMAIA: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:83 [backend fallback]
AutogradMeta: registered at /pytorch/aten/src/ATen/core/VariableFallbackKernel.cpp:99 [backend fallback]
Tracer: registered at /pytorch/torch/csrc/autograd/TraceTypeManual.cpp:294 [backend fallback]
AutocastCPU: fallthrough registered at /pytorch/aten/src/ATen/autocast_mode.cpp:322 [backend fallback]
AutocastMTIA: fallthrough registered at /pytorch/aten/src/ATen/autocast_mode.cpp:466 [backend fallback]
AutocastMAIA: fallthrough registered at /pytorch/aten/src/ATen/autocast_mode.cpp:504 [backend fallback]
AutocastXPU: fallthrough registered at /pytorch/aten/src/ATen/autocast_mode.cpp:542 [backend fallback]
AutocastMPS: fallthrough registered at /pytorch/aten/src/ATen/autocast_mode.cpp:209 [backend fallback]
AutocastCUDA: fallthrough registered at /pytorch/aten/src/ATen/autocast_mode.cpp:165 [backend fallback]
FuncTorchBatched: registered at /pytorch/aten/src/ATen/functorch/LegacyBatchingRegistrations.cpp:731 [backend fallback]
BatchedNestedTensor: registered at /pytorch/aten/src/ATen/functorch/LegacyBatchingRegistrations.cpp:758 [backend fallback]
FuncTorchVmapMode: fallthrough registered at /pytorch/aten/src/ATen/functorch/VmapModeRegistrations.cpp:27 [backend fallback]
Batched: registered at /pytorch/aten/src/ATen/LegacyBatchingRegistrations.cpp:1075 [backend fallback]
VmapMode: fallthrough registered at /pytorch/aten/src/ATen/VmapModeRegistrations.cpp:33 [backend fallback]
FuncTorchGradWrapper: registered at /pytorch/aten/src/ATen/functorch/TensorWrapper.cpp:210 [backend fallback]
PythonTLSSnapshot: registered at /pytorch/aten/src/ATen/core/PythonFallbackKernel.cpp:202 [backend fallback]
FuncTorchDynamicLayerFrontMode: registered at /pytorch/aten/src/ATen/functorch/DynamicLayer.cpp:475 [backend fallback]
PreDispatch: registered at /pytorch/aten/src/ATen/core/PythonFallbackKernel.cpp:206 [backend fallback]
PythonDispatcher: registered at /pytorch/aten/src/ATen/core/PythonFallbackKernel.cpp:198 [backend fallback]
```

### Convert Model to OpenVINO™

```sh
uv pip freeze | grep optimum
```
```
optimum==2.1.0
optimum-intel==1.27.0
optimum-onnx==0.1.0

```

```sh
uv run optimum-cli export openvino \
  --model nvidia/GR00T-N1.6-3B \
  --weight-format fp16 \
  /home/eka/Workspace/models/public/gr00t-n1.6-3b/FP16
```

Error message.

```
warning: The `extra-build-dependencies` option is experimental and may change without warning. Pass `--preview-features extra-build-dependencies` to disable this warning.
[2026-01-18 20:36:15,593] [WARNING] [real_accelerator.py:209:get_accelerator] Setting accelerator to CPU. If you have GPU or other accelerator, we were unable to detect it.
Multiple distributions found for package optimum. Picked distribution: optimum-onnx
Traceback (most recent call last):
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/transformers/models/auto/configuration_auto.py", line 1131, in from_pretrained
    config_class = CONFIG_MAPPING[config_dict["model_type"]]
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/transformers/models/auto/configuration_auto.py", line 833, in __getitem__
    raise KeyError(key)
KeyError: 'Gr00tN1d6'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/home/eka/Workspace/Isaac-GR00T/.venv/bin/optimum-cli", line 10, in <module>
    sys.exit(main())
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/optimum/commands/optimum_cli.py", line 219, in main
    service.run()
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/optimum/commands/export/openvino.py", line 469, in run
    main_export(
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/optimum/exporters/openvino/__main__.py", line 282, in main_export
    task = infer_task(
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/optimum/exporters/openvino/__main__.py", line 114, in infer_task
    config = AutoConfig.from_pretrained(
  File "/home/eka/Workspace/Isaac-GR00T/.venv/lib/python3.10/site-packages/transformers/models/auto/configuration_auto.py", line 1133, in from_pretrained
    raise ValueError(
ValueError: The checkpoint you are trying to load has model type `Gr00tN1d6` but Transformers does not recognize this architecture. This could be because of an issue with the checkpoint, or because your version of Transformers is out of date.

You can update Transformers with the command `pip install --upgrade transformers`. If this does not work, and the checkpoint is very new, then there may not be a release version that supports this model yet. In this case, you can get the most up-to-date code by installing Transformers from source with the command `pip install git+https://github.com/huggingface/transformers.git`
```

## Installation Guide and Quick Start on XPU

Tested on:
 - CPU: Intel® Core™ Ultra 9 Processor 285K
 - CPU Cores: 24 (8 Performance-cores and 16 Efficient-cores)
 - CPU Threads: 24
 - Memory: 64 GB
 - GPU: Intel® Arc™ B580 Graphics
 - GPU Memory: 12 GB
 - Storage minimum 250 GB

```sh
sudo snap install astral-uv
```

### Server

Setup server.

```sh
git clone https://github.com/ekaakurniawan/Isaac-GR00T.git
cd Isaac-GR00T/
git submodule update --init --recursive
uv sync --python 3.10
uv pip install -e .
```

Run server.

```sh
uv run python gr00t/eval/run_gr00t_server.py \
  --model-path nvidia/GR00T-N1.6-3B \
  --embodiment-tag GR1 \
  --use-sim-policy-wrapper \
  --device xpu
```

### Client

Setup client.

```sh
sudo apt update
sudo apt install libegl1-mesa-dev libglu1-mesa
bash gr00t/eval/sim/robocasa-gr1-tabletop-tasks/setup_RoboCasaGR1TabletopTasks.sh
```

Run client.

```sh
gr00t/eval/sim/robocasa-gr1-tabletop-tasks/robocasa_uv/.venv/bin/python gr00t/eval/rollout_policy.py \
  --n_episodes 10 \
  --policy_client_host 127.0.0.1 \
  --policy_client_port 5555 \
  --max_episode_steps=720 \
  --env_name gr1_unified/PnPBottleToCabinetClose_GR1ArmsAndWaistFourierHands_Env \
  --n_action_steps 8 \
  --n_envs 5
```

## Installation Guide on NVIDIA GPU

### Clone the Repository

GR00T relies on submodules for certain dependencies. Include them when cloning:

```sh
git clone --recurse-submodules https://github.com/NVIDIA/Isaac-GR00T
cd Isaac-GR00T
```

If you've already cloned without submodules, initialize them separately:

```sh
git submodule update --init --recursive
```

### Set Up the Environment

GR00T uses [uv](https://github.com/astral-sh/uv) for fast, reproducible dependency management.

> **Requirement:** uv **v0.8.4+** is needed to parse `[tool.uv.extra-build-dependencies]` in `pyproject.toml` (required for building `flash-attn`).

After installing uv, create the environment and install GR00T:

```sh
uv sync --python 3.10
uv pip install -e .
```

> Note: CUDA 12.4 is recommended and officially tested. However, CUDA 11.8 has also been verified to work.
> In such cases, make sure to install a compatible version of `flash-attn` manually (e.g., `flash-attn==2.8.2` was confirmed working with CUDA 11.8).

For a containerized setup that avoids system-level dependency conflicts, see our [Docker Setup Guide](docker/README.md).

For training and inference hardware recommendations (RTX PRO Servers, DGX, Jetson AGX Thor), see the [Hardware Recommendation Guide](getting_started/hardware_recommendation.md).

## Model Checkpoints

### Base Models
We provide pre-trained base VLA model checkpoints. These checkpoints have been pre-trained on 10k+ hours of robot data and can be used for finetuning on downstream tasks.

| Model | Use Case | Description | Checkpoint Path | Branch |
| ----- | -------- | ----------- | --------------- | ------ |
| GR00T N1.5 | Finetuning | Base [GR00T N1.5 model](https://research.nvidia.com/labs/gear/gr00t-n1_5/) (3B parameters) | [nvidia/GR00T-N1.5-3B](https://huggingface.co/nvidia/GR00T-N1.5-3B) | [n1.5-release](https://github.com/NVIDIA/Isaac-GR00T/tree/n1.5-release) |
| GR00T N1.6 | Finetuning | Base [GR00T N1.6 model](https://research.nvidia.com/labs/gear/gr00t-n1_6/) (3B parameters) | [nvidia/GR00T-N1.6-3B](https://huggingface.co/nvidia/GR00T-N1.6-3B) | [main](https://github.com/NVIDIA/Isaac-GR00T) |

### Finetuned Models
We also provide finetuned checkpoints for various robot platforms and benchmarks. These models are finetuned from the base models above and can be used directly for evaluation or as starting points for further finetuning.

| Model | Base Model | Description | Checkpoint Path | Example |
| ----- | ---------- | ----------- | --------------- | ------- |
| GR00T-N1.6-bridge | [nvidia/GR00T-N1.6-3B](https://huggingface.co/nvidia/GR00T-N1.6-3B) | Fine-tuned on [Bridge dataset](https://rail-berkeley.github.io/bridgedata/) for WidowX robot on manipulation tasks | [nvidia/GR00T-N1.6-bridge](https://huggingface.co/nvidia/GR00T-N1.6-bridge) | [SimplerEnv](examples/SimplerEnv/README.md) |
| GR00T-N1.6-fractal | [nvidia/GR00T-N1.6-3B](https://huggingface.co/nvidia/GR00T-N1.6-3B) | Fine-tuned on [Fractal dataset](https://www.tensorflow.org/datasets/catalog/fractal20220817_data) for Google robot on manipulation tasks | [nvidia/GR00T-N1.6-fractal](https://huggingface.co/nvidia/GR00T-N1.6-fractal) | [SimplerEnv](examples/SimplerEnv/README.md) |
| GR00T-N1.6-BEHAVIOR1k | [nvidia/GR00T-N1.6-3B](https://huggingface.co/nvidia/GR00T-N1.6-3B) | Fine-tuned on [BEHAVIOR-1K](https://behavior.stanford.edu/) for Galaxea R1 Pro robot on loco-manipulation tasks | [nvidia/GR00T-N1.6-BEHAVIOR1k](https://huggingface.co/nvidia/GR00T-N1.6-BEHAVIOR1k) | [BEHAVIOR](examples/BEHAVIOR/README.md) |
| GR00T-N1.6-G1-PnPAppleToPlate | [nvidia/GR00T-N1.6-3B](https://huggingface.co/nvidia/GR00T-N1.6-3B) | Fine-tuned for Unitree G1 loco-manipulation pick-and-place tasks | [nvidia/GR00T-N1.6-G1-PnPAppleToPlate](https://huggingface.co/nvidia/GR00T-N1.6-G1-PnPAppleToPlate) | [G1 LocoManipulation](examples/GR00T-WholeBodyControl/README.md) |

## Quick Start

We can quickly start by downloading a pre-trained checkpoint and starting the policy server for any pretrained embodiement, e.g. GR1 embodiment.
```bash
# On GPU server: Start the policy server
uv run python gr00t/eval/run_gr00t_server.py --embodiment-tag GR1 --model-path nvidia/GR00T-N1.6-3B
```

Then, refer to the [robocasa-gr1-tabletop-tasks](examples/robocasa-gr1-tabletop-tasks/README.md) for more details on how to rollout the policy with `GR1` embodiment.

## Getting started with this repo

We provide accessible Jupyter notebooks and detailed documentation in the [`./getting_started`](getting_started) folder.

## 1. Data Preparation

Please refer to the [data preparation guide](getting_started/data_preparation.md) for more details.

## 2. Inference

After data is prepared, the GR00T model can be used to generate output actions with the below simple inference script:

```bash
uv run python scripts/deployment/standalone_inference_script.py \
  --model-path nvidia/GR00T-N1.6-3B \
  --dataset-path demo_data/gr1.PickNPlace \
  --embodiment-tag GR1 \
  --traj-ids 0 1 2 \
  --inference-mode pytorch \
  --action-horizon 8
```

GR00T-N1.6-3B inference timing (4 denoising steps, single view):

| Device | Mode | Data Processing | Backbone | Action Head | E2E | Frequency |
|--------|------|-----------------|----------|-------------|-----|-----------|
| RTX 5090 | torch.compile | 2 ms | 18 ms | 16 ms | 37 ms | 27.3 Hz |
| H100 | torch.compile | 4 ms | 23 ms | 11 ms | 38 ms | 26.3 Hz |
| RTX 4090 | torch.compile | 2 ms | 25 ms | 17 ms | 44 ms | 22.8 Hz |
| Thor | torch.compile | 5 ms | 39 ms | 61 ms | 105 ms | 9.5 Hz |

For more details, please check our full [inference guide](scripts/deployment/README.md) for more details including faster inference with `TensorRT`

## 3. Finetuning

### Fine-tune on Pre-registered Post-train Embodiment Tags

GR00T provides several pre-registered embodiment tags with ready-to-use configurations:

- `LIBERO_PANDA`
- `OXE_GOOGLE`
- `OXE_WIDOWX`
- `UNITREE_G1`
- `BEHAVIOR_R1_PRO`

**Example:** To finetune Libero-Spatial on GR00T N1.6, follow the instructions in the [Libero finetuning guide](examples/LIBERO/README.md#finetune-libero-spatial-dataset). We also provide simulation environment setup for evaluation linked with post-train checkpoints and benchmark numbers.

### Fine-tune on Custom Embodiments ("NEW_EMBODIMENT")

To finetune GR00T on your own robot data and configuration, follow the detailed tutorial available at [`getting_started/finetune_new_embodiment.md`](getting_started/finetune_new_embodiment.md).

#### Prerequisites

Ensure your input data follows the **GR00T-flavored LeRobot v2 format**, and specify your modality configuration at `modality_config_path`.

#### Run Fine-tuning Script
```bash
# Set number of GPUs
export NUM_GPUS=1

CUDA_VISIBLE_DEVICES=0 uv run python \
    gr00t/experiment/launch_finetune.py \
    --base-model-path nvidia/GR00T-N1.6-3B \
    --dataset-path <DATASET_PATH> \
    --embodiment-tag NEW_EMBODIMENT \
    --modality-config-path <MODALITY_CONFIG_PATH> \
    --num-gpus $NUM_GPUS \
    --output-dir <OUTPUT_PATH> \
    --save-total-limit 5 \
    --save-steps 2000 \
    --max-steps 2000 \
    --use-wandb \
    --global-batch-size 32 \
    --color-jitter-params brightness 0.3 contrast 0.4 saturation 0.5 hue 0.08 \
    --dataloader-num-workers 4
```

> For more extensive finetuning configuration, use `gr00t/experiment/launch_train.py` instead to launch the training process.

### Recommended Fine-tuning Configuration

For optimal results, maximize your batch size based on available hardware and train for a few thousand steps.

#### Hardware Performance Considerations

**Fine-tuning Performance**
- We recommend using 1 H100 node or L40 node for optimal finetuning performance
- Other hardware configurations (e.g., A6000) will also work but may require longer training time
- Optimal batch size depends on your hardware and which model components are being tuned

#### Training Variance

Users may observe some variance in post-training results across runs, even when using the same configuration, seed, and dropout settings. In our experiments, we have observed performance differences as large as 5-6% between runs. This variance may be attributed to non-deterministic operations in image augmentations or other stochastic components. When comparing results to reported benchmarks, please keep this inherent variance in mind.

## 4. Evaluation

We recommend a two-stage evaluation approach: open-loop evaluation followed by simulation evaluation to comprehensively assess model quality.

### 4.1 Open-Loop Evaluation

Open-loop evaluation provides an offline assessment by comparing the model's predicted actions against ground truth data from your dataset.

#### Running the Evaluation

Execute the evaluation script with your newly trained model:
```bash
uv run python gr00t/eval/open_loop_eval.py \
    --dataset-path <DATASET_PATH> \
    --embodiment-tag NEW_EMBODIMENT \
    --model-path <CHECKPOINT_PATH> \
    --traj-ids 0 \
    --action-horizon 16  # ensure this is within the delta_indices of action's modality config.
```

#### Interpreting Results

The evaluation generates a visualization saved at `/tmp/open_loop_eval/traj_{traj_id}.jpeg`, which includes:
- Ground truth actions vs. predicted actions
- Unnormalized mean squared error (MSE) metrics

These plots provide a quick indicator of the policy's accuracy on the training dataset distribution.

### 4.2 Closed-Loop Evaluation

After validating performance through open-loop evaluation, test your model in closed-loop environments.

#### Understanding the Policy API

After training your model, you'll use the `Gr00tPolicy` class to load and run inference. The policy expects observations in a specific format (nested dictionaries with video, state, and language modalities) and returns actions ready for execution.

**Quick Start with Server-Client Architecture:**

```bash
# On GPU server: Start the policy server
uv run python gr00t/eval/run_gr00t_server.py \
    --embodiment-tag NEW_EMBODIMENT \
    --model-path <CHECKPOINT_PATH> \
    --device cuda:0 \
    --host 0.0.0.0 \
    --port 5555
```

```python
from gr00t.policy.server_client import PolicyClient

policy = PolicyClient(host="localhost", port=5555) # Connect to the policy server
env = YourEnvironment() # Create an environment
obs, info = env.reset() # Reset the environment
if not policy.ping(): # Verify connection
    raise RuntimeError("Cannot connect to policy server!")
action, info = policy.get_action(obs) # Run inference
obs, reward, done, truncated, info = env.step(action) # Execute the action
```

**Debugging with ReplayPolicy:**

When developing a new environment integration or debugging your inference loop, you can use `ReplayPolicy` to replay recorded actions from an existing dataset. This helps verify that your environment setup, observation formatting, and action execution work correctly—without needing a trained model.

```bash
# Start server with ReplayPolicy (replays actions from dataset)
uv run python gr00t/eval/run_gr00t_server.py \
    --dataset-path <DATASET_PATH> \
    --embodiment-tag NEW_EMBODIMENT \
    --execution-horizon 8  # should match the executed action horizon in the environment
```

The server will replay actions from the first episode of the dataset. Use `policy.reset(options={"episode_index": N})` on the client to switch to a different episode.

**For detailed documentation on:**
- How to adapt the policy to your own environment
- Server-client architecture for remote inference
- Observation and action formats
- Querying modality configurations
- Batched inference
- Troubleshooting common errors

See the complete [Policy API Guide](getting_started/policy.md).

#### Evaluation Examples

We support evaluation on available public benchmarks and our internal benchmarks. Our evaluation framework uses a server-client architecture that communicates via RESTful API. Both the policy server and simulation environment client use the same IP (usually localhost) and port to run simulation evaluation.

For the policy server, we reuse the project root's uv environment (same as finetuning) to run `run_gr00t_server`. For simulation environment clients, we provide individual setup scripts to configure uv environments, as they typically conflict with each other when using a single shared environment.

You can use [the verification script](scripts/eval/check_sim_eval_ready.py) to verify that all dependencies and environments for simulation evaluation are properly configured.

Please refer to each benchmark link below for more details.

**Zero-shot Evaluation** (evaluate without finetuning):
- **RoboCasa**: [Instructions](examples/robocasa/README.md)
- **RoboCasa GR1 Tabletop Tasks**: [Instructions](examples/robocasa-gr1-tabletop-tasks/README.md)

**Finetuned Evaluation** (test after task-specific finetuning):
- **G1 LocoManipulation**: [Instructions](examples/GR00T-WholeBodyControl/README.md)
- **LIBERO**: [Instructions](examples/LIBERO/README.md)
- **SimplerEnv**: [Instructions](examples/SimplerEnv/README.md)
- **BEHAVIOR**: [Instructions](examples/BEHAVIOR/README.md)
- **PointNav**: [Instructions](examples/PointNav/README.md)
- **SO-100**: [Instructions](examples/SO100/README.md)


# Contributing

For more details, see [CONTRIBUTING.md](CONTRIBUTING.md)


## License 

```
# SPDX-FileCopyrightText: Copyright (c) 2025 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
```


## Citation
<!-- TODO: Update -->
[Paper Site](https://research.nvidia.com/labs/lpr/publication/gr00tn1_2025/)
```bibtex
@inproceedings{gr00tn1_2025,
  archivePrefix = {arxiv},
  eprint     = {2503.14734},
  title      = {{GR00T} {N1}: An Open Foundation Model for Generalist Humanoid Robots},
  author     = {NVIDIA and Johan Bjorck and Fernando Castañeda, Nikita Cherniadev and Xingye Da and Runyu Ding and Linxi "Jim" Fan and Yu Fang and Dieter Fox and Fengyuan Hu and Spencer Huang and Joel Jang and Zhenyu Jiang and Jan Kautz and Kaushil Kundalia and Lawrence Lao and Zhiqi Li and Zongyu Lin and Kevin Lin and Guilin Liu and Edith Llontop and Loic Magne and Ajay Mandlekar and Avnish Narayan and Soroush Nasiriany and Scott Reed and You Liang Tan and Guanzhi Wang and Zu Wang and Jing Wang and Qi Wang and Jiannan Xiang and Yuqi Xie and Yinzhen Xu and Zhenjia Xu and Seonghyeon Ye and Zhiding Yu and Ao Zhang and Hao Zhang and Yizhou Zhao and Ruijie Zheng and Yuke Zhu},
  month      = {March},
  year       = {2025},
  booktitle  = {ArXiv Preprint},
}
```
