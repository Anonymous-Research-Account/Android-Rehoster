# Android Emulator Connector Service and Rehoster

This folder contains a comprehensive toolkit for building, deploying, and managing Android emulators in Docker containers. This repository provides the infrastructure to run multiple Android emulators with gRPC API exposure via an Envoy reverse proxy, WebRTC support through a Coturn server, and advanced AOSP (Android Open Source Project) build injection capabilities.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Re-Hosting - AOSP Build Injection](#rehosting)
- [Streaming Emulators to a WebView](#streaming)

## Overview
This repo contains two parts. One part of the tooling is designed to facilitate Android firmware analysis and emulation at scale. The other part is to re-host Android vendor firmware to the Android emulator.

```
FMD-AECS/
├── aosp_apex_injector.py        # APEX file repackaging and injection
├── aosp_build_injector.py       # Main AOSP build injection script
├── aosp_module_type.py          # AOSP module type definitions
├── aosp_post_build_injector.py  # Post-build file injection
├── aosp_post_build_app_injector.py  # App-specific post-build injection
├── common.py                    # Shared utility functions
├── compare_folders.py           # Folder comparison utilities
├── config.py                    # Main configuration file
├── ConfigManager.py             # Configuration management
├── create_docker_emulator_images.py  # Build emulator Docker images
├── create_docker_startup_scripts.py  # Generate Docker Compose configs
├── fmd_backend_requests.py      # FirmwareDroid API client
├── parse_lddtree_to_json.py     # Dependency tree parser
├── setup_logger.py              # Logging configuration
├── shell_command.py             # Shell command utilities
├── requirements.txt             # Python dependencies
│
├── device_configs/              # Device-specific AOSP configurations
│   ├── development/             # Development device configs
│   └── native_injection/        # Native library injection configs
│
├── emulator/                    # Emulator Docker configurations
│   ├── Dockerfile_arm64         # ARM64 emulator Dockerfile
│   ├── Dockerfile_x86_64        # x86_64 emulator Dockerfile
│   ├── Dockerfile_base_emulator_* # Base emulator Dockerfiles
│   ├── emulator_start.sh        # Emulator startup script
│   ├── avd/                     # Android Virtual Device configs
│   └── prebuilts/               # Prebuilt binaries (ignored)
│
├── env/                         # Environment configurations
│   ├── .env                     # Environment variables
│   ├── envoy/                   # Envoy proxy configuration
│   ├── coturn/                  # Coturn server configuration
│   └── nginx/                   # Nginx web server (for ACME/Let's Encrypt)
│
├── templates/                   # Configuration templates
│   ├── docker-compose.yaml      # Docker Compose template
│   ├── envoy.yaml               # Envoy template
│   ├── docker_emulator.txt      # Emulator service template
│   ├── envoy_match.txt          # Envoy route matching template
│   ├── envoy_cluster.txt        # Envoy cluster template
│   ├── build_image.py           # AOSP image building script
│   ├── file_contexts            # SELinux file contexts
│   └── apex/                    # APEX configuration templates
│
├── image_artefacts/             # Built image artifacts (ignored, created during build)
│   ├── arm64-v8a/               # ARM64 emulator image files (auto-generated)
│   └── x86_64/                  # x86_64 emulator image files (auto-generated)
├── out/                         # Build output directory (ignored)
├── nexus/                       # Nexus repository integration
└── testing_service/             # Testing utilities and services
```

## Architecture

Re-Hosting Overview:

```
+------------------------------------------------+
| Download Firmware + Metadata |
| from FirmwareDroid |
+----------------------------+-------------------+
|
v
+------------------------------------------------+
| Inject Build Modules into AOSP Source |
+----------------------------+-------------------+
|
v
+------------------------------------------------+
| Run AOSP Build |
+----------------------------+-------------------+
|
v
+------------------------------------------------+
| Inject Post-Build Artefacts |
+----------------------------+-------------------+
|
v
+------------------------------------------------+
| Build Emulator Image |
+----------------------------+-------------------+
|
v
+------------------------------------------------+
| Upload Emulator Image to Nexus Repository |
+------------------------------------------------+
```


Streaming Emulator to a WebInterface: The system consists of several key components:
```
┌─────────────────────────────────────────────────────────┐
│                    Client Applications                   │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Envoy Reverse Proxy                         │
│         (gRPC API Gateway + Load Balancer)              │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
┌───────▼──────┐ ┌──▼────────┐ ┌─▼───────────┐
│  Emulator 1  │ │Emulator 2 │ │ Emulator N  │
│  (Docker)    │ │(Docker)   │ │  (Docker)   │
└──────────────┘ └───────────┘ └─────────────┘
        │
┌───────▼──────────────────────────────────────┐
│         Coturn Server (WebRTC)               │
└──────────────────────────────────────────────┘
```

# ReHosting
The `aosp_build_injector.py` script downloads firmware build files from FirmwareDroid and injects them into AOSP builds.

## Pre-Requisits

1. Install FirmwareDroid and import firmware ([see FirmwareDroid's Guide](https://firmwaredroid.github.io/))
2. Download and prepare AOSP source code (see [PRE-GUIDE](https://github.com/Anonymous-Research-Account/Android-Rehoster/blob/main/ReHosterCode/PRE_GUIDE.md))
3. Install python dependencies
```bash
# Create virtual environment and install dependencies
python -m venv venv
source venv/bin/activate 
pip install -r requirements.txt
```

**Basic Usage:**
- The main script to start the re-hosting is called `aosp_build_injector.py`. It has a help command:
```bash
python aosp_build_injector.py --help
```
- Following we provide an easy wrapper script to start the re-hoster. Please adjust the parameters "-f", "-r" and "-p" to match your environment.
```
#!/bin/bash
export PYTHONWARNINGS="ignore:Unverified HTTPS request"
export FMD_PASSWORD=XXXXXX
export DOCKER_REPO_PASSWORD=XXXXXXXX
export FMD_PHONE64_TEST_BUILD="True"

python3 ./aosp_build_injector.py -f "https://firmwaredroid.XXXX.com" -u "admin" -s "/home/ubuntu/aosp/aosp12" -e "12" -r "https://nexus-repo.XXX:8443/repository/emulator-images/" -d "insecure_uploader" -a "arm64" -p "68ece65ee1fb94b42d457670" --pre_injector_config "./device_configs/development/12/pre_injector_config_v1.json" --post_injector_config "./device_configs/development/12/post_injector_config_v1.json"
```

**What it does:**
1. Authenticates with FirmwareDroid backend
2. Downloads firmware build files and packages
3. Injects files into AOSP build structure
4. Handles APEX repackaging and signing
5. Builds custom Android system images
6. Uploads resulting images to nexus registry

**Supported AOSP Versions:**
- Android 11 (untested)
- Android 12
- Android 13
- Android 14 (testing)

## Configuration

### Configuration Files
- **`config.py`**: Main configuration file with build settings, paths, and constants
- **``./device_configs/``**: Injection Strategy Configuration files for each Android version

### Starting the emulator image

After the build the images can be downloaded from the nexus repository and can be started on an ARM machine with the official
Android emulator. Create a new empty AVD (e.g., `Arm64_test`) and extract the image zip file to your sdk folder where the firmware images reside (e.g., `Android/sdk/system-images/android-12-test/IMAGE_HERE`

Start the emulator with your avd.
```bash
emulator -avd Arm64_test -show-kernel -verbose
```
Check if there is an error logs during startup:
```
adb logcat "*:E"
```

# Streaming
# Key Components:
- **Envoy Proxy**: Routes and load-balances gRPC requests to emulator instances
- **Android Emulators**: Run in Docker containers with full Android system images
- **Coturn Server**: Provides STUN/TURN services for WebRTC connections
- **AOSP Build Tools**: Scripts for building and customizing Android system images
- **FirmwareDroid Integration**: Backend connection for firmware download and analysis

## Features

- **Multi-Architecture Support**: Both x86_64 and ARM64 emulator support
- **Scalable Deployment**: Run multiple emulator instances in parallel
- **gRPC API Exposure**: Access emulator APIs through standardized gRPC interface
- **WebRTC Streaming**: Real-time audio/video streaming from emulators
- **AOSP Build Pipeline**: Complete toolchain for building custom Android images
- **Firmware Injection**: Inject firmware packages and APEXs into AOSP builds
- **Docker-based**: Easy deployment and isolation using containers
- **Dynamic Configuration**: Template-based configuration for flexible setups

## Prerequisites

### System Requirements

- **Operating System**: Linux (Ubuntu 20.04+ recommended)
- **CPU**: x86_64 or ARM64 architecture
- **RAM**: Minimum 16GB (32GB+ recommended for multiple emulators)
- **Disk Space**: 100GB+ available (AOSP builds require significant storage)
- **Docker**: Version 20.10+
- **Docker Compose**: Version 2.0+

### Software Dependencies
- **Python**: 3.8 or higher
- **Git**: For repository management
- **AOSP Build Tools** (optional, for building custom images):
  - Java Development Kit (JDK) 11
  - Android SDK Platform Tools
  - Required build dependencies (see AOSP documentation)

## Installation

### 1. Clone the Repository

```bash
git clone ANON_SUBMISSION
cd ANON_SUBMISSION
```

### 2. Install Python Dependencies

```bash
pip install -r requirements.txt
```

The required packages include:
- Jinja2 (template engine)
- requests (HTTP library)
- docker (Docker Python SDK)
- werkzeug (utilities)
- tqdm (progress bars)
- filelock (file locking)
- protobuf (Protocol Buffers)

### 3. Set Up Docker

Ensure Docker and Docker Compose are installed and running:

```bash
docker --version
docker-compose --version
```

### 4. Configure Environment

Review and update the environment configuration file with your settings:

```bash
# Review the existing configuration
cat env/.env

# The file contains settings for WebRTC/TURN server
# Edit if you need to customize TURN credentials and server URL
nano env/.env
```

**Note**: The `docker-compose.yaml` and `env/envoy/envoy.yaml` files are auto-generated from templates by the startup scripts. The `env/.env` file is manually configured and primarily contains TURN server credentials for WebRTC functionality.

## Quick Start

### Running Pre-built Emulators

1. **Create startup scripts** for your desired architecture:

```bash
# For ARM64 architecture
python create_docker_startup_scripts.py -c linux/arm64

# For x86_64 architecture
python create_docker_startup_scripts.py -c linux/amd64
```

2. **Start the services**:

```bash
docker-compose up -d
```

3. **Access emulators**:
   - gRPC API: `localhost:8554` (default)
   - ADB: `localhost:5555` (default)
   - SSH: `localhost:2222` (default)

### Building Docker Emulator Images

1. **Download or prepare emulator images**:

```bash
# Build from local images
python create_docker_emulator_images.py -l -i ./emulator_images

# Or download from repository
python create_docker_emulator_images.py \
  -r https://your-repo-url/emulator-images \
  -u username \
  -d docker-registry-url
```

2. **Build Docker images**:

The script will automatically build Docker images for your architecture.

## Usage

### Building Docker Emulator Images

The `create_docker_emulator_images.py` script handles downloading and building emulator Docker images.

**Basic Usage:**

```bash
# Build from local files
python create_docker_emulator_images.py -l -i ./emulator_images

# Download and build from repository
python create_docker_emulator_images.py \
  -r https://repository-url/service/rest/v1/assets?repository=emulator-images \
  -u repository-username \
  -d docker-registry-url
```

**Options:**
- `-l, --create_local`: Build from local files (skip download)
- `-r, --repository-url`: URL to the repository for downloading images
- `-d, --docker-repo-url`: Docker registry URL for pushing images
- `-u, --repository-username`: Authentication username
- `-i, --input-dir`: Directory containing emulator images (default: `./emulator_images`)
- `--file-list`: Comma-separated list of specific files to download

### Creating Docker Startup Scripts

The `create_docker_startup_scripts.py` script generates `docker-compose.yaml` and Envoy configuration files.

**Basic Usage:**

```bash
# Default configuration (ARM64)
python create_docker_startup_scripts.py

# Custom port configuration
python create_docker_startup_scripts.py \
  -g 8554 \
  -a 5555 \
  -s 2222 \
  -c linux/arm64
```

**Options:**
- `-g, --grpc-start-port`: Starting port for gRPC service (default: 8554)
- `-a, --adb-start-port`: Starting port for ADB service (default: 5555)
- `-s, --ssh-start-port`: Starting port for SSH service (default: 2222)
- `-c, --cpu-arch`: CPU architecture - `linux/amd64` or `linux/arm64` (default: linux/arm64)
- `-d, --debug`: Enable debug mode

**Output Files:**
- `docker-compose.yaml`: Docker Compose configuration
- `env/envoy/envoy.yaml`: Envoy proxy configuration

**Configuration Files:**
- **`env/envoy/envoy.yaml`**: Envoy proxy configuration
- **`env/coturn/turnserver.conf`**: Coturn server configuration
- **`templates/docker-compose.yaml`**: Docker Compose template
- **`device_configs/`**: Device-specific AOSP configurations
