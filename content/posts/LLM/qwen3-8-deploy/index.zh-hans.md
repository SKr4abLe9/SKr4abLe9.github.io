---
title: '双 3090 部署 Qwen3.8-27B-FP8 并接入 Codex'
date: 2026-09-03T13:27:50+08:00
slug: qwen3-8-deploy
draft: false
description: 在双 RTX 3090 环境下使用 vLLM 部署 Qwen3.8-27B-FP8，并接入 Codex，记录 cu129、FlashInfer、128K 上下文与显存调优过程。
categories: ["LLM"]
tags: ["linux", "vLLM"]
cover:
---

## 基本环境

位于docker容器内，16G shm

Ubuntu 22.04 LTS

cpu 16核 32G内存

3090 24G x2
```text
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.211.01             Driver Version: 570.211.01     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3090        On  |   00000000:37:00.0 Off |                  N/A |
| 41%   29C    P8             28W /  350W |      11MiB /  24576MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA GeForce RTX 3090        On  |   00000000:86:00.0 Off |                  N/A |
| 42%   29C    P8             24W /  350W |      11MiB /  24576MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
+-----------------------------------------------------------------------------------------+
```

两张卡之间是 `SYS`，没有 NVLink
``` text
        GPU0    GPU1    CPU Affinity    NUMA Affinity   GPU NUMA ID
GPU0     X      SYS     0-3,8-11        0               N/A
GPU1    SYS      X      4-7,12-15       1               N/A

Legend:

  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
  NODE = Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
  PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB  = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
  PIX  = Connection traversing at most a single PCIe bridge
  NV#  = Connection traversing a bonded set of # NVLinks
```

## 环境配置

### 创建虚拟环境

安装 uv，并用 uv 管理独立 Python：
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
```

安装 Python：
```bash
uv python install 3.12
```

创建环境：
```bash
mkdir -p /root/venvs
uv venv /root/venvs/qwen-vllm --python 3.12 --seed
```

进入：
```bash
source /root/venvs/qwen-vllm/bin/activate
```

### 安装 PyTorch + vLLM

这里安装 cuda 12.9 对应的 PyTorch 和 vLLM 0.27.1

下载官方 release 的 cu129 wheel：
```bash
curl -fL 'https://github.com/vllm-project/vllm/releases/download/v0.27.1/vllm-0.27.1%2Bcu129-cp38-abi3-manylinux_2_28_x86_64.whl' -o /tmp/vllm-0.27.1+cu129-cp38-abi3-manylinux_2_28_x86_64.whl
```
安装：
```bash
uv pip install '/tmp/vllm-0.27.1+cu129-cp38-abi3-manylinux_2_28_x86_64.whl' --extra-index-url https://download.pytorch.org/whl/cu129
```

检查 PyTorch 和 vLLM 版本：
{{< tabs >}}
{{< tab label="代码" >}}
```bash
python -c "import torch,vllm; print('Torch:',torch.__version__); print('CUDA:',torch.version.cuda); print('CUDA available:',torch.cuda.is_available()); print('vLLM:',vllm.__version__)"
```
{{< /tab >}}
{{< tab label="结果" >}}
```text
Python CUDA: 12.9
Torch: 2.13.0+cu129
vLLM: 0.27.1
CUDA available: True
```
{{< /tab >}}
{{< /tabs >}}

### 下载模型

安装 Hugging Face CLI：
```bash
uv pip install -U huggingface_hub
```

下载：
```bash
hf download Qwen/Qwen3.8-27B-FP8 --local-dir {你的下载路径}/Qwen3.8-27B-FP8
```

## 部署模型

### 测试配置
```bash
VLLM_USE_FLASHINFER_SAMPLER=0 CUDA_VISIBLE_DEVICES=0,1 vllm serve \
    {你的下载路径}/Qwen3.8-27B-FP8 \
    --served-model-name Qwen3.8-27B-FP8 \
    --host 0.0.0.0 --port 8000 \
    --tensor-parallel-size 1 \
    --pipeline-parallel-size 2 \
    --max-model-len 32768 \
    --max-num-seqs 1 \
    --gpu-memory-utilization 0.90 \
    --language-model-only \
    --gdn-prefill-backend triton
```

- `VLLM_USE_FLASHINFER_SAMPLER=0`：禁用 FlashInfer sampler，nvcc版本低时会有编译问题因此关闭
- `CUDA_VISIBLE_DEVICES=0,1`：指定使用 GPU 0 和 GPU 1
- `--served-model-name`：指定 OpenAI 兼容 API 中暴露的模型名称
- `--host --port`：指定监听的 ip 和端口
- `--tensor-parallel-size`：设置张量并行
- `--pipeline-parallel-size`：设置流水线并行，即将模型按层切分成多少个连续的阶段，并分配到不同的 GPU 或节点上顺序执行
- `--max-model-len`：设置最大上下文长度，包括输入和输出
- `--max-num-seqs`：设置单次调度最多同时处理 sequence 的数量
- `--gpu-memory-utilization`：设置 vLLM 的 GPU 显存预算
- `--language-model-only`：只加载语言模型部分，不启用视觉/多模态模块
- `--gdn-prefill-backend`：设置 prefill 阶段使用的 backend

### 验证 vLLM 服务

查看模型列表：
```bash
curl http://127.0.0.1:8000/v1/models
```

测普通 Chat Completions：
```bash
curl http://127.0.0.1:8000/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"Qwen3.8-27B-FP8","messages":[{"role":"user","content":"你好，请用一句话介绍一下自己"}],"max_tokens":128}'
```

测 Responses API：
```bash
curl http://127.0.0.1:8000/v1/responses -H "Content-Type: application/json" -d '{"model":"Qwen3.8-27B-FP8","input":"解释一下 C++ RAII 的作用"}'
```

### 参数调整

为了让模型能在 codex 中使用，需要加上以下3个参数：
```bash
    --reasoning-parser qwen3 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder
```
- `--reasoning-parser qwen3`：解析 Qwen3 的 thinking/reasoning 输出，并按推理内容与最终回答进行处理。
- `--enable-auto-tool-choice`：允许模型自主判断是否需要调用工具，适用于 Codex 这类 Agent 场景。
- `--tool-call-parser qwen3_coder`：解析 Qwen3 Coder 的工具调用格式，并转换成 OpenAI-compatible 的结构化 tool call。

根据启动时的日志输出，可以调大 `--gpu-memory-utilization` 并将上下文设置为 `128k`。
```text
GPU KV cache size: 128,159 tokens
Maximum concurrency for 32,768 tokens per request: 3.91x
```

最终配置：
```bash
VLLM_USE_FLASHINFER_SAMPLER=0 CUDA_VISIBLE_DEVICES=0,1 vllm serve \
    {你的下载路径}/Qwen3.8-27B-FP8 \
    --served-model-name Qwen3.8-27B-FP8 \
    --host 0.0.0.0 --port 8000 \
    --tensor-parallel-size 1 \
    --pipeline-parallel-size 2 \
    --max-model-len 131072 \
    --max-num-seqs 1 \
    --gpu-memory-utilization 0.92 \
    --language-model-only \
    --gdn-prefill-backend triton \
    --reasoning-parser qwen3 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder
```

## 接入Codex

这里使用 `cc-switch` 配置并将模型接入 Codex 中使用

API Key随便填写

配置 `config.toml`：
```toml
model = "Qwen3.8-27B-FP8"
model_provider = "qwen-vllm"

model_context_window = 131072
model_auto_compact_token_limit = 110000

model_supports_reasoning_summaries = true
model_reasoning_summary = "auto"

[model_providers.qwen-vllm]
name = "Qwen3.8-27B-FP8 (Local)"
base_url = "http://127.0.0.1:8000/v1"
wire_api = "responses"
requires_openai_auth = false
```

## 相关链接
1. https://huggingface.co/Qwen/Qwen3.8-27B-FP8
2. https://github.com/vllm-project/vllm
3. https://github.com/farion1231/cc-switch