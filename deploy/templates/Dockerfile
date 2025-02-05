# Pull the image
FROM nvidia/cuda:12.1.1-cudnn8-runtime-ubuntu22.04

# Setup ENV
ENV DEBIAN_FRONTEND=noninteractive \
    HF_HOME=/app/huggingface_cache \
    TRANSFORMERS_CACHE=/app/huggingface_cache \
    PYTHONUNBUFFERED=1 \
    PYTHON_VERSION=3.9 \
    MPLCONFIGDIR="/tmp/matplotlib" \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH="/app:${PYTHONPATH}" \
    TF_ENABLE_ONEDNN_OPTS=0 \
    MC_HOST_myminio="https://$MINIO_ACCESS_KEY $MINIO_SECRET_KEY@minio-api-minio.apps.xxxxx" \
    REQUESTS_CA_BUNDLE=""  
# to bypass ssl issue when api request

# Install tools
RUN apt-get update && apt-get install -y \
    wget \
    python3-pip \
    python3-venv \
    git \
    curl \
    unzip \
    libgl1-mesa-glx \
    make \
    && rm -rf /var/lib/apt/lists/*

# Install mc for minio
RUN wget --no-check-certificate https://dl.min.io/client/mc/release/linux-amd64/mc && \
    chmod +x mc && \
    mv mc /usr/local/bin/mc

# Set working directory inside the container
WORKDIR /app

# Copy the rest of the project files (avoids rebuilding dependencies on minor changes)
COPY . .

# Upgrade setuptools to support PEP 660
RUN pip install --upgrade setuptools

# Copy only pyproject.toml and poetry.lock first (for caching)
COPY pyproject.toml poetry.lock* ./

# Install Poetry globally
RUN pip install --no-cache-dir poetry

# Install dependencies using Poetry
RUN poetry install --no-root || echo "Poetry install failed, check logs"

# Ensure janus is installed as a package
RUN pip install . 
# RUN pip install --use-pep517 -e .  # ✅ If you need editable mode

# Copy the requirements file first for caching
COPY requirements.txt .

# Install dependencies using pip
RUN pip install --no-cache-dir -r requirements.txt

# Pull minio objects
RUN mkdir -p /app/models/Janus-Pro-7B && \
    mc alias set myminio https://minio-api-minio.xxxxxxxx \
    --insecure \
    $MINIO_ACCESS_KEY $MINIO_SECRET_KEY&& \
    mc cp --recursive --insecure myminio/janus/Janus-Pro-7B /app/models/

# Bring up the app
CMD ["python", "demo/app_januspro.py"]
