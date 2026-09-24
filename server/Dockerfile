FROM python:3.12-slim

# USE_WHEELHOUSE=1 时离线安装（需 server/wheelhouse 内有 linux wheel 包，构建前自行预下载）
ARG USE_WHEELHOUSE=0
# 在线安装的 PyPI 镜像，国内默认阿里云；海外可改用 https://pypi.org/simple
ARG PIP_INDEX=https://mirrors.aliyun.com/pypi/simple/

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

COPY requirements.txt /tmp/requirements.txt
COPY wheelhouse /tmp/wheelhouse

# torch 使用 CPU 专用 wheel（无 nvidia/CUDA 依赖，体积小 5GB+）离线安装
RUN grep -v -E '^torch==' /tmp/requirements.txt > /tmp/requirements-other.txt \
    && if [ "$USE_WHEELHOUSE" = "1" ]; then \
           pip install --no-cache-dir --no-index --no-deps \
               /tmp/wheelhouse/torch-2.2.2+cpu-cp312-cp312-linux_x86_64.whl \
           && pip install --no-cache-dir --no-index --find-links /tmp/wheelhouse \
               -r /tmp/requirements-other.txt; \
       else \
           pip install --no-cache-dir -i "$PIP_INDEX" --retries 12 --timeout 180 \
               -r /tmp/requirements.txt; \
       fi

# 代码与权重通过 bind mount 挂载，镜像不含业务文件
CMD ["python", "-m", "kev.serve", "--run", "/app/runs/kev", "--port", "8009"]