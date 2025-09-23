# Use your CUDA 12.6 + Python 3.11 base
ARG IMAGE_BASE="ghcr.io/ai-dock/python:3.10-v2-cuda-12.1.1-base-22.04"
FROM ${IMAGE_BASE}

LABEL org.opencontainers.image.source="https://github.com/ai-dock/comfyui"
LABEL org.opencontainers.image.description="ComfyUI Stable Diffusion backend and GUI"
LABEL maintainer="Rob Ballantyne <rob@dynamedia.uk>"

# ----- venv locations (ComfyUI isolated) -----
ENV COMFYUI_VENV=$VENV_DIR/comfyui
ENV COMFYUI_VENV_PYTHON=$COMFYUI_VENV/bin/python
ENV COMFYUI_VENV_PIP=$COMFYUI_VENV/bin/pip
ENV API_VENV=$VENV_DIR/api
ENV API_VENV_PYTHON=$API_VENV/bin/python
ENV API_VENV_PIP=$API_VENV/bin/pip
ENV IMAGE_SLUG="comfyui" OPT_SYNC=ComfyUI
ENV XPU_TARGET=NVIDIA_GPU

# ----- versions & knobs (MUST be set BEFORE layer0) -----
# layer0 will apt-install python3.11 and create the named venv
ARG PYTHON_VERSION="3.11"
ENV PYTHON_VERSION=${PYTHON_VERSION}
ARG PYTHON_VENV_NAME="comfyui"
ENV PYTHON_VENV_NAME=${PYTHON_VENV_NAME}

# Make the layer0 NVIDIA step pick the cu126 wheel channel
ENV CUDA_VERSION=12.6.0
ENV CUDA_STRING=12.6.0-base

# Ensure torch family version isn’t empty for layer0’s pip line
ARG PYTORCH_VERSION="2.8.0"
ENV PYTORCH_VERSION=${PYTORCH_VERSION}
ARG TORCHVISION_VERSION="0.23.0"
ENV TORCHVISION_VERSION=${TORCHVISION_VERSION}
ARG TORAUDIO_VERSION="2.8.0"
ENV TORAUDIO_VERSION=${TORAUDIO_VERSION}

# IMPORTANT: because layer0 uses --index-url (exclusive), add PyPI back as an extra index
# so packages like xformers can still resolve.
ENV PIP_EXTRA_INDEX_URL="https://pypi.org/simple"

# (optional) we’ll use this for an explicit post-layer0 install; safe to keep
ENV TORCH_INDEX_URL="https://download.pytorch.org/whl/cu126"

# ----- LAYER 0 -----
COPY --chown=0:1111 ./COPY_ROOT_0/ /
ARG IMAGE_BASE
RUN set -eo pipefail && /opt/ai-dock/bin/build/layer0/init.sh | tee /var/log/build.log

# ----- (optional but recommended) Explicitly pin the final torch stack in comfyui venv -----
# If you’re confident layer0 is now correct, you can delete this block.
RUN bash -lc 'set -e; \
  "$COMFYUI_VENV_PYTHON" -m pip install -U pip wheel setuptools && \
  "$COMFYUI_VENV_PIP" install --index-url "$TORCH_INDEX_URL" \
    "torch==${PYTORCH_VERSION}+cu126" \
    "torchvision==${TORCHVISION_VERSION}+cu126" \
    "torchaudio==${TORAUDIO_VERSION}+cu126"'

# sanity (fails fast if 3.11 or cu126 torch didn’t land)
RUN bash -lc 'set -e; \
  "$COMFYUI_VENV/bin/python" -c "import sys; assert sys.version_info[:2]==(3,11), sys.version"; \
  "$COMFYUI_VENV/bin/python" -c "import torch; \
     assert torch.__version__.startswith(\"${PYTORCH_VERSION}\"), torch.__version__; \
     assert torch.version.cuda.startswith(\"12.6\"), torch.version.cuda"; \
  echo \"OK: Python 3.11 and torch ${PYTORCH_VERSION}+cu126 in $COMFYUI_VENV\"'

RUN echo "bust cache"

# ----- LAYER 1 -----
ARG COMFYUI_BUILD_REF
ENV COMFYUI_BUILD_REF=${COMFYUI_BUILD_REF}
COPY --chown=0:1111 ./COPY_ROOT_1/ /
RUN set -eo pipefail && /opt/ai-dock/bin/build/layer1/init.sh | tee -a /var/log/build.log

# ----- LAYER 99 -----
COPY --chown=0:1111 ./COPY_ROOT_99/ /
RUN set -eo pipefail && /opt/ai-dock/bin/build/layer99/init.sh | tee -a /var/log/build.log

ENV PYTHON_DEFAULT_VENV=comfyui
CMD ["init.sh"]
