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

STEP 1: SSH into the droplet and update the system
Connect to your droplet:  
ssh -i ~/.ssh/id_ed25519 root@167.99.190.27

Update and upgrade packages:  
apt update && apt upgrade -y

Install helpful utilities:  
apt -y install curl ca-certificates gnupg jq



STEP 2: Install Docker Engine and Compose plugin
Install Docker:  
curl -fsSL https://get.docker.com | sh

Enable Docker to start automatically:  
systemctl enable docker --now

Verify Docker and Compose:  
docker --version
docker compose version || true


STEP 3: Install NVIDIA driver (CUDA 550) and verify GPU
Check that the GPU is visible:  
lspci | grep -i nvidia

Add NVIDIA CUDA repository keyring:  
curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb -o cuda-keyring.deb
dpkg -i cuda-keyring.deb
apt update

Install the 550 driver (works for L40S / RTX 6000 Ada):  
apt -y install cuda-drivers-550

Reboot and reconnect:  
reboot
(After about a minute, reconnect)
ssh -i ~/.ssh/id_ed25519 root@167.99.190.27

Verify driver and GPU:  
nvidia-smi

STEP 4: Install NVIDIA Container Toolkit for Docker GPU access
Record Ubuntu version for repo setup:  
distribution=$(. /etc/os-release; echo ${ID}${VERSION_ID})

Add NVIDIA GPG key and repository:  
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit.gpg
curl -fsSL https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list \
 | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit.gpg] https://#' \
 | tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

Install the toolkit and configure Docker runtime:  
apt update && apt -y install nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=docker

Restart Docker and test GPU inside containers:  
systemctl restart docker
docker run --rm --gpus all nvidia/cuda:12.3.2-base-ubuntu22.04 nvidia-smi

STEP 5: Deploy the LLM stack (Ollama + Open WebUI + Caddy)
Move to the project directory:  
cd /opt/llm

Start all containers in the background:  
docker compose up -d

Show running containers and ports:  
docker compose ps

View recent logs if needed:  
docker logs open-webui --tail=50
docker logs ollama --tail=50
docker logs caddy --tail=50

STEP 6: Pull the Llama 3.1 model and test it
Open a shell inside the Ollama container:  
docker exec -it ollama bash

Download the 8B model weights:  
ollama pull llama3.1:8b

Quick inference test:  
ollama run llama3.1:8b "Say 'Northgate is online and running.'"

Exit the container:  
exit



STEP 7: Access the web interface in a browser
URL:http://167.99.190.27
Setup steps:
1. Create an admin account.
2. Select Workspaces, add new model, select llama3.1:8b as basemodel
3. Upload Knowledge Base articles to model
4. Disable self-signup for internal use.
5. Create user accounts for helpdesk specialists


Deployment Command References

Docker. (2025). Get Docker Engine – Installation instructions. Docker Docs. https://docs.docker.com/engine/install/

Docker. (2025). Compose overview. Docker Docs. https://docs.docker.com/compose/

NVIDIA. (2025). NVIDIA Container Toolkit documentation. NVIDIA Developer. https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/

NVIDIA. (2025). CUDA Toolkit 12.3 installation guide for Linux. NVIDIA Developer. https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html

Caddy. (2025). Getting started with Caddy. Caddy Docs. https://caddyserver.com/docs/getting-started

Ollama. (2025). Running Ollama on Linux. Ollama Docs. https://github.com/ollama/ollama

Open WebUI. (2025). Open WebUI deployment guide. GitHub. https://github.com/open-webui/open-webui

DigitalOcean. (2025). Droplets: Create, manage, and connect. DigitalOcean Docs. https://docs.digitalocean.com/products/droplets/how-to/create/

Ubuntu. (2025). UFW — Uncomplicated Firewall. Ubuntu Documentation. https://help.ubuntu.com/community/UFW

curl. (2025). curl command-line tool documentation. curl.se. https://curl.se/docs/
