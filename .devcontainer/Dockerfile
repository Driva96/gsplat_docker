# Devcontainer CUDA + Python environment for gsplat.
# Uses a PyTorch image that already includes CUDA, cuDNN, and NCCL.
# JIT CUDA build recommended (see docs/DEV.md); set BUILD_NO_CUDA=1 for editable install.
FROM pytorch/pytorch:2.9.1-cuda12.8-cudnn9-devel

ARG USER=vscode
ARG UID=1000
ARG GID=1000

# Set architecture to speed up build (Change 8.9 to your GPU arch, e.g. 7.5, 8.0, 8.6, 9.0)
# Set MAX_JOBS to prevent OOM during compilation
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_ROOT_USER_ACTION=ignore \
    BUILD_NO_CUDA=0 \
    TORCH_CUDA_ARCH_LIST="8.6+PTX" \ 
    MAX_JOBS=4 \
    TZ=Etc/UTC

# System dependencies (Added libglm-dev just in case)
RUN apt-get update && apt-get install -y --no-install-recommends \
    git curl wget ca-certificates sudo bash build-essential \
    ninja-build cmake pkg-config \
    clang-format clang bear \
    libgl1-mesa-dev libglib2.0-0 \
    libjpeg-dev libpng-dev libtiff-dev \
    libboost-all-dev libeigen3-dev libglm-dev \
    && rm -rf /var/lib/apt/lists/*

# Keep root for installation steps; switch to non-root later
WORKDIR /gsplat

# Upgrade pip and prepare cache layer (Python is provided by base image)
RUN python -m pip install --upgrade pip

# Copy only dependency descriptors first for caching
COPY docs/requirements.txt ./docs/ 
COPY examples/requirements.txt ./examples/
COPY gsplat/version.py gsplat/__init__.py ./gsplat/ 

# Install dev dependencies (editable, deferred CUDA build)
# Fail the build if CUDA-enabled PyTorch/toolchain are not present during build
RUN python - <<'EOF'
import torch
print("Torch CUDA version:", torch.version.cuda)
print("Torch CUDA available:", torch.cuda.is_available())
EOF

RUN nvcc --version

# Install reqs for simple_trainer etc.
RUN python -m pip install -r examples/requirements.txt --no-build-isolation

# Copy the project 
COPY . .
RUN python -m pip install .[dev] --no-build-isolation

# --no-build-isolation
RUN python -m pip install ipykernel nbstripout

# (Opt --no-build-isolationional) install doc build deps
# RUN python -m pip install -r docs/requirements.txt

# Restore full source (mounted by devcontainer; this ensures fast rebuilds)
# In devcontainer, the repository will be bind-mounted; below is a safety copy step.
COPY . .

# Pre-create cache directory for JIT extensions to avoid permission issues
RUN mkdir -p /home/${USER}/.cache/torch_extensions || true

# Print environment summary
RUN python -c "import torch, os; print('Torch:', torch.__version__); print('CUDA:', torch.version.cuda); print('Build_NO_CUDA:', os.getenv('BUILD_NO_CUDA'))"

# Create non-root user (VS Code devcontainer default) after installs
RUN groupadd -g ${GID} ${USER} && useradd -m -u ${UID} -g ${GID} -s /bin/bash ${USER} \
    && echo "${USER} ALL=(ALL) NOPASSWD:ALL" >/etc/sudoers.d/${USER} \
    && chown -R ${UID}:${GID} /gsplat /home/${USER}

USER ${USER}

# Default command
CMD ["/bin/bash"]