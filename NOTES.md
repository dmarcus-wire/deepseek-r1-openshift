# Procedure

with ArgoCD
1. pull the code from git and images from quay
1. build the image
1. pull LLM from minio
1. spin up container
1. setup networks and then you will have this project up and running.

## Create repo structure

```sh
mkdir -p deploy/templates
```

### Create files

```sh
touch deploy/templates/{buildConfig.yaml,imageStreamOutput.yaml,sitrep-prd-deployment.yaml,sitrep-prd-svc.yaml}
```

## Create a Dockerfile

pull base image

```sh
FROM nvidia/cuda:12.1.1-cudnn8-runtime-ubuntu22.04
```

### setup ENV

```sh
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
    REQUESTS_CA_BUNDLE=""  # to bypass ssl issue when api request
```

### Add tools

```sh
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
```

### Add Minio 'mc' Tools

minio enables downloading the LLM instead of hitting the huggingface api. Alternative, use quay.

```sh
RUN wget --no-check-certificate https://dl.min.io/client/mc/release/linux-amd64/mc && \
    chmod +x mc && \
    mv mc /usr/local/bin/mc
```

### Setup the project

```sh
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
```

### Pull minio objects

```sh
RUN mkdir -p /app/models/Janus-Pro-7B && \
    mc alias set myminio https://minio-api-minio.xxxxxxxx \
    --insecure \
    $MINIO_ACCESS_KEY $MINIO_SECRET_KEY&& \
    mc cp --recursive --insecure myminio/janus/Janus-Pro-7B /app/models/
```

### minio secret management

```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: janus
spec:
  template:
    spec:
      containers:
      - name: janus
        env:
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: MINIO_ACCESS_KEY
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: MINIO_SECRET_KEY
```

### Launch the app

```sh
CMD ["python", "demo/app_januspro.py"]
```

## Build Config

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: custom-image-build-janus-prd

spec:
  source:
    type: Git
    git:
      uri: https://github.xxxxxx/Janus.git
    sourceSecret: 
      name: git-secret
   
  strategy:
    type: Docker                      
    dockerStrategy:
      dockerfilePath: Dockerfile
  output:
    to:
      kind: ImageStreamTag
      name: janus-prd-stream:latest
  triggers:
  - type: ImageChange
  - type: ConfigChange
```

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    application: janus-prd
  name: janus-prd
spec:
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      deploymentconfig: janus-prd
  template:
    metadata:
      labels:
        app: janus-prd
        deploymentconfig: janus-prd
      name: janus-prd
    spec:
      containers:
      - name: janus-prd
        image: image-registry.openshift-image-registry.svc:5000/prd/janus-prd-stream:latest
        # i am using a image-register for store the image, will need to change to quay

        ports:
        - containerPort: 9860
          name: http
          protocol: TCP

        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        livenessProbe:
          httpGet:
            path: /
            port: 9860
            scheme: HTTP
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 9860
            scheme: HTTP
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
```

## Network

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: janus-prd
  name: janus  # this is the name that will appear in the url janus-{ns}-xxxxxx
spec:
  port:
    targetPort: 9860-tcp
  to:
    kind: Service
    name: janus-prd
    weight: 100
  wildcardPolicy: None
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect



apiVersion: v1
kind: Service
metadata:
  labels:
    app: janus-prd
  name: janus-prd
spec:
  ports:
  - name: 9860-tcp
    port: 9860
    protocol: TCP
    targetPort: 9860
  selector:
    app: janus-prd
    deploymentconfig: janus-prd
  sessionAffinity: None
  type: ClusterIP
```