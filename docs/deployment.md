# Deployment Guide

## Scope

This guide documents the **initial, tested implementation** of the Local AI Security Lab on Kali Linux. It covers Docker Engine, Docker Compose, NVIDIA Container Toolkit, Ollama, Open WebUI and Qdrant, followed by local model and RAG setup.

The steps follow the laboratory's Obsidian installation records, adapted to the **public GitHub repository layout**. They are intended to support a fresh deployment, but a **complete rebuild on another Kali installation has not yet been validated**. In particular, NVIDIA driver installation was outside the recorded procedure, and the original container images use mutable tags.

## Reference environment

| Component | Verified environment |
| --- | --- |
| Host OS | Kali Linux 2026.3 (bare metal, external NVMe) |
| Kernel | `7.2.15+kali-amd64` |
| GPU | NVIDIA GeForce RTX 5060 Ti, 16 GB |
| NVIDIA driver | `620.70.06` |
| Compose | V2 (`2.40.3` in the original lab) |
| Original local project directory | `~/Projects/local-ai-security-lab` |
| Public deployment files | `config/compose.yaml` and `config/.env.example` |
| Browser access | `http://127.0.0.1:3000` on the host only |

**Prerequisites:** a compatible NVIDIA driver installed and operational (`nvidia-smi`), network access to obtain packages/images/models, and sufficient storage and memory. Git is required to clone the published repository. The exact driver installation is hardware- and kernel-dependent and was **not** reproduced in the initial project.

## 1. Check the host

```bash
cat /etc/os-release
uname -r
nvidia-smi
```

Confirm the expected Kali installation and that the NVIDIA GPU is detected before installing the container runtime.

## 2. Install Docker Engine and Compose

The original lab used the Kali package repositories:

```bash
sudo apt update
sudo apt install docker.io docker-compose
sudo systemctl enable --now docker
sudo systemctl status docker
```

Verify the engine and Compose:

```bash
sudo docker run --rm hello-world
docker compose version
```

The initial test returned `Hello from Docker!`. Administrative Docker commands are deliberately run with `sudo`; the user was not added to the privileged `docker` group.

**Compatibility note:** the recorded environment used the `docker compose` (V2) command. Package names and availability can vary by Kali release; if `docker compose version` fails, install a compatible Docker Compose V2 plugin before continuing. The legacy `docker-compose` executable is not a substitute for the command shown below without checking compatibility.

## 3. Configure NVIDIA Container Toolkit

Install the repository prerequisites:

```bash
sudo apt install ca-certificates curl gnupg
```

Import NVIDIA's repository signing key and configure the toolkit repository:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -fsSL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Install and configure the runtime:

```bash
sudo apt update
sudo apt install nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Verify GPU access from Docker:

```bash
sudo docker info
sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

In the original lab the temporary Ubuntu container detected the NVIDIA RTX 5060 Ti. On a different host, check driver/runtime compatibility before continuing.

## 4. Obtain the published Compose configuration

Rather than creating another `compose.yaml` manually, clone this repository. It already contains the tested service definitions and a safe environment-file template:

```bash
sudo apt install git
mkdir -p ~/Projects
cd ~/Projects
git clone https://github.com/Abel-MV/local-ai-security-lab.git
cd local-ai-security-lab/config
```

The working directory for the following Compose commands is now **`~/Projects/local-ai-security-lab/config`**.

```text
local-ai-security-lab/
├── .gitignore
├── config/
│   ├── compose.yaml
│   └── .env.example
├── docs/
└── evidence/
```

The published [`compose.yaml`](../config/compose.yaml) defines:

- **Ollama**, with GPU reservation and the `ollama_data` named volume.
- **Qdrant**, using the `qdrant_data` named volume and an internal Docker port.
- **Open WebUI**, bound to `127.0.0.1:3000` and using the `open_webui_data` named volume.
- A shared `ai_network` Docker bridge network.

**Reproducibility limitation:** the original images use `latest` and `main`, which are mutable tags. The published Compose file reproduces the *recorded configuration*, not necessarily the exact image builds originally tested. Pin validated image digests for strict repeatability. Qdrant is not configured with API-key authentication; do not expose this network to untrusted containers or external users.

## 5. Protect the environment secret

From the **`config/` directory**, generate a fresh random Open WebUI secret in a new `.env` file:

```bash
openssl rand -hex 32 | awk '{print "WEBUI_SECRET_KEY=" $0}' > .env
chmod 600 .env
stat -c '%a %n' .env
```

The original lab recorded file mode `600`. Never publish the actual `.env` file or its contents. The repository already includes [`config/.env.example`](../config/.env.example) with a placeholder and a root [`.gitignore`](../.gitignore) that excludes real `.env` files; **do not replace the placeholder in `.env.example` with a real key**.

Note that `.gitignore` reduces the risk of accidentally committing an untracked secret but does not remove anything already committed or published.

## 6. Deploy and verify the containers

Run these commands from **`~/Projects/local-ai-security-lab/config`**, where both `compose.yaml` and `.env` reside. The explicit `--env-file` and `-f` options avoid ambiguity about which configuration Docker Compose uses:

```bash
sudo docker compose --env-file .env -f compose.yaml config --quiet
sudo docker compose --env-file .env -f compose.yaml up -d
sudo docker compose --env-file .env -f compose.yaml ps
```

The original deployment showed Ollama and Qdrant running, with Open WebUI reported as healthy. The `depends_on` setting orders startup but does not guarantee application readiness; check the actual service state.

Open **`http://127.0.0.1:3000`** on the Kali host and create the local Open WebUI administrator account.

Check the published ports:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

| Service | Internal port | Host binding |
| --- | --- | --- |
| Open WebUI | `8080` | `127.0.0.1:3000` |
| Ollama | `11434` | Not published |
| Qdrant | `6333` | Not published |

The bridge network permits service-to-service communication; **not publishing a port is not the same as isolating a service from all other containers or blocking outbound connections**.

## 7. Download and test local models

The following model identifiers were used in the recorded lab:

```bash
sudo docker exec ollama ollama pull qwen3.5:9b
sudo docker exec ollama ollama pull gemma4:12b
sudo docker exec ollama ollama pull embeddinggemma-2:740m
sudo docker exec ollama ollama list
```

Test a language model through Open WebUI and inspect resource usage:

```bash
sudo docker exec ollama ollama ps
nvidia-smi
```

The initial tests confirmed GPU use for Qwen 3.5 and Gemma 4. Model availability and memory requirements may change across releases.

## 8. Configure document retrieval (RAG)

1. In **Open WebUI → Admin Panel → Settings → Connections**, use `http://ollama:11434` for Ollama and verify the connection.
2. In **Admin Panel → Settings → Documents**, select the **Ollama** embedding engine, API base URL `http://ollama:11434`, and model `embeddinggemma-2:740m`.
3. Use **Qdrant** as the vector database (the Compose configuration supplies `VECTOR_DB=qdrant` and `QDRANT_URI=http://qdrant:6333`).
4. Create a private knowledge base, import a synthetic test document, and associate it with `qwen3.5:9b`.
5. Ask source-specific questions and compare answers with the document. Include a question about information **not** in the source to check whether the model acknowledges the omission.

The original private knowledge base, **`Network Forensics - RAG Test`**, indexed a synthetic document (`rag-test.md`). The embedding vectors were 768-dimensional, and the observed Qdrant collection contained six points. These are recorded test results, **not guarantees** for every document or model run. The original `rag-test.md` is not bundled into this public guide.

## 9. Restart, persistence and maintenance

Return to **`~/Projects/local-ai-security-lab/config`** if you are working in another directory:

```bash
cd ~/Projects/local-ai-security-lab/config
sudo docker compose --env-file .env -f compose.yaml restart
sudo docker compose --env-file .env -f compose.yaml ps
```

After a restart, the original lab retained downloaded Ollama models, Open WebUI configuration/knowledge and Qdrant collections in the three named volumes.

To stop or remove project containers without deliberately deleting named volumes:

```bash
sudo docker compose --env-file .env -f compose.yaml stop
# When removal is required:
sudo docker compose --env-file .env -f compose.yaml down
```

**Do not use `docker compose down -v` unless you intentionally want to delete project volumes and their data.**

The original backup contained configuration files (`compose.yaml`, `.env.example`, `.gitignore` and `rag-test.md`) and passed an SHA-256 comparison. It **did not** include application volumes, models or a tested full restore. Back up persistent data separately before relying on this system. This historical backup is distinct from the current public repository contents.

## Known limitations

- NVIDIA driver installation from a clean OS was not documented or tested as part of this deployment.
- A full clean-install redeployment and restore on a second system remain untested.
- Image tags are unpinned; updates may change behaviour.
- Qdrant lacks application-level API authentication in this local configuration.
- Container hardening, comprehensive backups and adversarial RAG testing remain future work.

## References

- [Docker Engine](https://docs.docker.com/engine/)
- [Docker Compose](https://docs.docker.com/compose/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- [Ollama on Docker](https://docs.ollama.com/docker)
- [Open WebUI](https://docs.openwebui.com/)
- [Qdrant](https://qdrant.tech/documentation/)
