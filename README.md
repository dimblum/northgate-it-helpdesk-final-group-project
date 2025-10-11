# 🧠 Northgate IT Helpdesk Assistant

Retrieval-Augmented **Llama 3.1 8B Instruct** chatbot hosted on a **DigitalOcean GPU Droplet** using **Ollama** and **Open WebUI**.  
This project simulates an internal IT helpdesk for the fictional *Northgate Institute of Technology*.

---

## Purpose

This project demonstrates how to deploy a **self-hosted LLM** that can answer IT helpdesk questions using internal documentation.  
We selected **Llama 3.1 8B Instruct** because it offers strong reasoning, a long context window for document retrieval, and smooth compatibility with **Ollama** and **Open WebUI**.  
The model runs efficiently on a single GPU droplet and supports multiple concurrent users.

---

## Architecture

| Component | Description |
|------------|-------------|
| **Backend LLM** | Llama 3.1 8B Instruct (via Ollama) |
| **Frontend** | Open WebUI |
| **Proxy** | Caddy (HTTP/HTTPS reverse proxy) |
| **Platform** | DigitalOcean GPU Droplet (L40S / RTX 6000 Ada, 48 GB VRAM, 64 GB RAM, 8 vCPUs) |
| **RAG** | Local document embedding and retrieval (PDF/KB) |

---

## ⚙️ Deployment Guide (Full Reproduction)

### 1️⃣ Create the GPU Droplet (TOR1)

In the DigitalOcean console:  
1. Go to **AI / Machine Learning → GPU Droplets → Create GPU Droplet**  
2. Image: **Ubuntu 22.04 LTS**  
3. Plan: **L40S** or **RTX 6000 Ada** (48 GB VRAM, 64 GB RAM, 8 vCPUs)  
4. Add your **SSH keys**   
5. Name: `northgate-gpu-portal`  

# --- STEP 1: SSH into droplet:
ssh -i ~/.ssh/id_ed25519 root@167.99.190.27
apt update && apt upgrade -y

# --- STEP 2: Install Docker Engine and Docker Compose ---
apt -y install curl ca-certificates gnupg

# Get the official Docker install script
curl -fsSL https://get.docker.com | sh

# Enable and start Docker
systemctl enable docker --now

# Verify Docker and Compose versions
docker --version
docker compose version || true


# --- STEP 3: Verify GPU presence ---
lspci | grep -i nvidia

# --- STEP 3a: Install tested NVIDIA drivers (L40S / RTX 6000 Ada) ---
apt -y install ubuntu-drivers-common
ubuntu-drivers autoinstall

# Add CUDA repo for up-to-date drivers
curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb -o cuda-keyring.deb
dpkg -i cuda-keyring.deb
apt update && apt -y install cuda-drivers-550

# Reboot to activate drivers
reboot

# --- Confirm GPU and driver ---
nvidia-smi
# You should see the L40S GPU with driver 550.xx

# --- STEP 4: Install NVIDIA Container Toolkit for Docker GPU access ---
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)

# Add repo and keyring
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit.gpg
curl -fsSL https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list \
 | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit.gpg] https://#' \
 | tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Install and configure runtime
apt update && apt -y install nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=docker
systemctl restart docker

# --- Test GPU inside Docker container ---
docker run --rm --gpus all nvidia/cuda:12.3.2-base-ubuntu22.04 nvidia-smi

# --- STEP 5: Create working directory for deployment ---
mkdir -p /opt/llm && cd /opt/llm

# --- STEP 5a: Create docker-compose.yml ---
cat > docker-compose.yml <<'YAML'
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - OLLAMA_NUM_PARALLEL=4
      - OLLAMA_MAX_LOADED_MODELS=1
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama:/root/.ollama

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    depends_on:
      - ollama
    ports:
      - "8080:8080"
    volumes:
      - open-webui:/app/backend/data

  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - open-webui

volumes:
  ollama:
  open-webui:
  caddy_data:
  caddy_config:
YAML

# --- STEP 5b: Create Caddyfile ---
cat > Caddyfile <<'CADDY'
:80 {
  encode zstd gzip
  reverse_proxy open-webui:8080
}
CADDY

# --- STEP 5c: Launch all containers ---
docker compose up -d
docker compose ps

# --- STEP 6: Pull Llama 3.1 8B model via Ollama ---
docker exec -it ollama ollama pull llama3.1:8b

# --- STEP 6b: Test model inference ---
docker exec -it ollama ollama run llama3.1:8b "Say 'Northgate is online and running.'"

# --- STEP 7: Access the web interface in a browser ---
# URL:
#   http://167.99.190.27
# Setup steps:
#   1. Create an admin account.
#   2. Go to Settings → Providers → Ollama.
#   3. Set OLLAMA_BASE_URL=http://ollama:11434
#   4. Choose model: llama3.1:8b
#   5. Disable self-signup for internal use.
