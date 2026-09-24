# kev-docker

一键容器化部署 **kev** —— 基于 `Qwen2.5-0.5B` 的 LoRA 决策模型（CPU 推理）—— 及其中英双语演示页。`docker compose up` 即可运行，无需接触宿主 Python 环境。

## 特性

- **单 compose 项目**：模型服务（FastAPI）+ 演示页（nginx）在一个 `docker-compose.yml` 内，服务间走容器网络直连。
- **数据不跟随容器**：代码、微调权重、HF 缓存全部 bind mount 到宿主机目录，容器无状态；页面/代码改动实时生效、无需重建镜像。
- **多平台可构建**：`python:3.12-slim` 基础镜像 + `torch 2.2.2`，支持 Linux x86_64 / arm64。
- **国内可用**：默认使用阿里云 PyPI 镜像；支持离线 wheelhouse 快速构建。

## 目录结构

```
kev-docker/
├── docker-compose.yml           # 主编排（kev + web）
├── docker-compose.offline.yml   # (可选) 离线 wheelhouse 构建覆盖层
├── .env.example                 # 端口 / 缓存路径 / CPU 线程配置
├── server/
│   ├── Dockerfile               # 模型服务镜像（依赖镜像内安装，代码运行时挂载）
│   ├── requirements.txt         # Python 依赖锁
│   └── src/kev/                 # kev 模型服务源码（来自 kev 项目，保留其 LICENSE）
├── web/
│   ├── www/index.html           # 演示页面（实时可改）
│   └── nginx.conf               # 反向代理 /v1 /api -> kev:8009
└── runs/kev/                    # (本地) 微调权重，git 忽略，不进入仓库
```

## 快速开始

### 1. 准备模型权重

把 kev 的 run 目录放入 `runs/kev/`，需包含：

```
adapter_config.json  adapter_model.safetensors  head.pt
tokenizer.json  tokenizer_config.json  vocab.json  merges.txt  added_tokens.json  special_tokens_map.json
```

> 本仓库刻意不携带权重，`runs/` 已在 `.gitignore` 中忽略。

### 2. 准备基础模型缓存

挂载的 HF 缓存目录默认为 `$HOME/.cache/huggingface`（可用 `.env` 的 `HF_CACHE_DIR` 覆盖）。容器内设 `HF_HUB_OFFLINE=1`，因此**首次运行前**需先在本机缓存 `Qwen/Qwen2.5-0.5B`：

```bash
huggingface-cli download Qwen/Qwen2.5-0.5B     # 或任选国内 HF 镜像: export HF_ENDPOINT=https://hf-mirror.com
```

若你的机器已有模板工程下载好的缓存，直接复用即可。

### 3. 启动

```bash
cp .env.example .env    # 可选，按需修改
docker compose up -d --build
```

- 演示页：<http://localhost:8083/>
- 模型 API：<http://localhost:8083/v1/models>
- 直连：<http://localhost:8009/v1/models>

## 配置（.env）

| 变量 | 默认 | 说明 |
|------|------|------|
| `KEV_PORT` | `127.0.0.1:8009` | 模型服务宿主机监听 |
| `WEB_PORT` | `127.0.0.1:8083` | 演示页宿主机监听 |
| `HF_CACHE_DIR` | `$HOME/.cache/huggingface` | 基础模型 HF 缓存 |
| `OMP_NUM_THREADS` | `1` | CPU 推理线程数。0.5B 小模型在共享 VM 上线程越多越慢（实测 1t=3s / 4t=31s / 8t=68s），保持 1 |

## 国内加速

- 镜像默认使用阿里云 PyPI（`PIP_INDEX` build arg 可换 `https://pypi.org/simple`）。
- Docker Hub 拉取慢时，在 Docker 配置 `registry-mirrors`（奥克斯特 / Daocloud / 1ms 等）。
- 若容器内 pip 在线安装慢，可预先在**宿主机**下载全部 linux(x86_64) wheel 再离线构建（torch 用 CPU 专用 wheel，镜像可瘦身 5GB+）：

```bash
mkdir -p server/wheelhouse

# 1) torch CPU wheel（无 CUDA 依赖）
curl -fL -o server/wheelhouse/torch-2.2.2+cpu-cp312-cp312-linux_x86_64.whl \
  "https://download.pytorch.org/whl/cpu/torch-2.2.2%2Bcpu-cp312-cp312-linux_x86_64.whl"

# 2) 其余依赖（不含 torch，避免引入 nvidia/CUDA 组件）
grep -v '^torch==' server/requirements.txt > /tmp/rest_reqs.txt
pip download -r /tmp/rest_reqs.txt -d server/wheelhouse \
  -i https://mirrors.aliyun.com/pypi/simple/ \
  --only-binary=:all: \
  --platform manylinux2014_x86_64 --platform manylinux_2_28_x86_64 \
  --python-version 312 --implementation cp --abi cp312

# 3) 离线构建并启动（需本机为 linux/x86_64，arm64 直接用在线构建即可）
docker compose -f docker-compose.yml -f docker-compose.offline.yml up -d --build
```

`server/wheelhouse/` 已被 git 忽略，不会进入仓库。

## 许可

- `server/src/kev/` 为 kev 项目代码，保留其原始 `LICENSE`。
- 本编排工程按你选择的许可证开源（见 `LICENSE`，可选用 MIT / Apache-2.0）。