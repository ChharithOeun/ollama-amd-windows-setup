# Ollama AMD GPU on Windows

> **The guide that should exist but doesn't.**

Run Ollama with your AMD GPU on Windows — Native Windows (ROCm + Vulkan), WSL2+ROCm, and Docker. Covers RX 5000/6000/7000/9000 series and integrated GPUs.

---

## The Problem

Ollama's AMD GPU support on Windows is real but limited and poorly documented:

- **Official ROCm on Windows** requires ROCm 6.1, covering only RX 6800+/7000 series
- **RX 6700 XT, 6600 series, 5700 XT, iGPUs, RDNA 4** -- unsupported by official build
- **Vulkan backend** (v0.12.11+) extends coverage to most AMD cards but is experimental
- **WSL2 ROCm** -- technically possible but unreliable; AMD lacks NVIDIA's dedicated WSL2 driver
- **Docker on Windows** -- `/dev/kfd` unavailable in WSL2-backed Docker, breaking `ollama/ollama:rocm`

---

## Quick Start Decision Tree

```
What AMD GPU do you have?
|
|-- RX 6800 / 6800 XT / 6900 XT / 6950 XT
|   RX 7600 / 7700 XT / 7800 XT / 7900 series
|   --> Method 1: Official Ollama (just works)
|
|-- RX 6700 XT / 6600 series / RX 5700 XT / iGPU / RX 9070 (RDNA 4)
|   --> Method 1b: ollama-for-amd community fork (ROCm 6.4, wider support)
|       OR Method 2: OLLAMA_VULKAN=1 (no extra install, experimental)
|
|-- Any AMD GPU + want browser UI
|   --> Method 3: Docker + Vulkan
|
`-- Can dual-boot or use Linux
    --> Just install Ollama normally -- AMD ROCm is mature on Linux
```

---

## Method 1: Native Windows -- Official ROCm

**Best for:** RX 6800 / 6800 XT / 6900 XT / 6950 XT / RX 7600-7900 series

**Requirements:** Windows 10/11 64-bit, AMD Adrenalin drivers 24.x+, GPU on [official list](https://docs.ollama.com/gpu)

```powershell
# 1. Download OllamaSetup.exe from https://ollama.com/download/windows

# 2. Verify GPU is detected
$env:OLLAMA_DEBUG = "1"
ollama serve

# 3. Pull and run a model
ollama pull llama3.1:8b
ollama run llama3.1:8b

# 4. Check GPU is active (Task Manager -> GPU should show non-zero %)
ollama ps
```

**Reserve VRAM headroom** (prevents silent OOM fallback to CPU):

```powershell
$env:OLLAMA_GPU_OVERHEAD = "1073741824"   # 1 GB for 8-12 GB cards
$env:OLLAMA_GPU_OVERHEAD = "1610612736"   # 1.5 GB for 16 GB cards
$env:OLLAMA_GPU_OVERHEAD = "2147483648"   # 2 GB for 24 GB cards
ollama stop; ollama serve
```

---

## Method 1b: Community Fork -- ollama-for-amd

**Best for:** RX 6700 XT, 6600 series, 5700 XT, all iGPUs, RX 9070 series

The [`ollama-for-amd`](https://github.com/likelovewant/ollama-for-amd) fork builds against ROCm 6.4, adding support for:

| Architecture | Cards |
|---|---|
| gfx1010/1012 (RDNA 1) | RX 5700 XT, 5500 XT |
| gfx1031/1032/1034 (RDNA 2) | RX 6700 XT, 6600 XT, 6600, 6500 XT |
| gfx1103 (RDNA 3 iGPU) | Radeon 780M (Ryzen 7040) |
| gfx1150/1151 (RDNA 3.5) | Radeon 890M, 880M (Ryzen AI 300) |
| gfx1200/1201 (RDNA 4) | RX 9070 XT, RX 9070 |

```powershell
# 1. Download from https://github.com/likelovewant/ollama-for-amd/releases
# 2. Uninstall official Ollama first (Settings > Apps)
# 3. Install community fork

# For GPUs not on the ROCm list, set GFX override:
# (NOTE: HSA_OVERRIDE_GFX_VERSION only works in community fork, NOT official Ollama)
$env:HSA_OVERRIDE_GFX_VERSION = "10.3.0"   # RX 6700 XT
$env:HSA_OVERRIDE_GFX_VERSION = "10.1.0"   # RX 5700 XT
$env:HSA_OVERRIDE_GFX_VERSION = "10.3.2"   # RX 6600 XT
ollama serve

# iGPU + dGPU: select discrete GPU explicitly
$env:ROCR_VISIBLE_DEVICES = "1"
$env:HIP_VISIBLE_DEVICES = "1"
ollama serve
```

---

## Method 2: Vulkan Backend

**Best for:** Any AMD GPU when ROCm path fails. No extra drivers needed -- Adrenalin includes Vulkan.

**Requires:** Ollama v0.12.11+

```powershell
# Enable Vulkan
$env:OLLAMA_VULKAN = "1"
ollama serve

# Verify (look for "vulkan" in output)
$env:OLLAMA_DEBUG = "1"; $env:OLLAMA_VULKAN = "1"; ollama serve
```

**Persistent setup:** Add to System Properties -> Environment Variables: `OLLAMA_VULKAN` = `1`

**Limitations:**
- 15-30% slower than ROCm/HIP for same GPU
- May select iGPU instead of dGPU (disable iGPU in BIOS as workaround)
- Flash Attention not reliably supported via Vulkan on Windows
- Still experimental internally

---

## Method 3: WSL2 + ROCm

**Reality check:** AMD has no dedicated WSL2 GPU driver (unlike NVIDIA). ROCm requires `/dev/kfd`,
which many WSL2 configs don't expose. Use this path only if comfortable with Linux.

**Automated setup:**

```bash
# Inside WSL2 Ubuntu 22.04 terminal:
chmod +x wsl2_setup.sh && ./wsl2_setup.sh
```

**Manual setup:**

```powershell
# PowerShell (Admin): Install WSL2 + Ubuntu 22.04
wsl --install; wsl --set-default-version 2
wsl --install -d Ubuntu-22.04; wsl --update
```

```bash
# Inside Ubuntu 22.04:
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y wget gnupg2 lsb-release

wget -q -O - https://repo.radeon.com/rocm/rocm.gpg.key | \
    sudo gpg --dearmor -o /usr/share/keyrings/rocm.gpg

# Add ROCm 6.2 repo
CODENAME=$(lsb_release -cs)
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/rocm.gpg] \
    https://repo.radeon.com/rocm/apt/6.2 $CODENAME main" | \
    sudo tee /etc/apt/sources.list.d/rocm.list

echo -e "Package: *\nPin: release o=repo.radeon.com\nPin-Priority: 600" | \
    sudo tee /etc/apt/preferences.d/rocm-pin-600

sudo apt-get update
sudo apt-get install -y rocm-hip-runtime hip-dev rocm-smi-lib rocinfo
sudo usermod -aG render,video "$USER"

# Configure environment
echo 'export PATH="$PATH:/opt/rocm/bin"' >> ~/.bashrc
echo 'export OLLAMA_FLASH_ATTENTION=1' >> ~/.bashrc
source ~/.bashrc

# Verify GPU
rocinfo | grep gfx    # If empty: /dev/kfd not available, use Method 1 instead

# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh
export OLLAMA_MODELS="$HOME/.ollama/models"   # NEVER store at /mnt/c/ -- very slow
ollama serve &
ollama run llama3.1:8b
```

**Access from Windows:** Get WSL2 IP (`ip addr show eth0 | grep inet`), then in PowerShell (Admin):
```powershell
netsh interface portproxy add v4tov4 listenport=11434 listenaddress=0.0.0.0 connectport=11434 connectaddress=<wsl-ip>
```

---

## Method 4: Docker Desktop

**Reality check:** Docker Desktop uses WSL2, so `/dev/kfd` is unavailable and `ollama/ollama:rocm` cannot access your AMD GPU. Use the Vulkan backend instead.

```powershell
docker compose up -d
# See docker-compose.yml for full configuration
```

---

## Performance Tuning

### Context window vs VRAM

| Model | 4K context | 16K context | 32K context |
|---|---|---|---|
| 7B/8B | ~0.5 GB KV | ~2 GB KV | ~4.5 GB KV |
| 13B | ~0.8 GB KV | ~3.2 GB KV | ~6.5 GB KV |

Reduce context: `ollama run model --option num_ctx 4096`

### Quantization by VRAM

| VRAM | Best quant | Notes |
|---|---|---|
| 4 GB | Q3_K_S or Q2_K | Very limited |
| 6-8 GB | Q4_K_M | Best balance for 7B/8B |
| 10-12 GB | Q4_K_M | Room for 13B |
| 16 GB | Q5_K_M | High quality 13-14B; 32B tight |
| 24 GB | Q4_K_M for 32B | 70B needs Q2_K to fit |

### Flash Attention (Linux/ROCm only)

```bash
export OLLAMA_FLASH_ATTENTION=1   # Significantly reduces KV cache VRAM
ollama serve
```

Not reliably supported on Windows as of early 2026.

---

## GPU Compatibility Summary

| GPU | VRAM | Method | Max Model (Q4_K_M) |
|---|---|---|---|
| RX 9070 XT | 16 GB | Community / Vulkan | 13B clean, 32B tight |
| RX 7900 XTX | 24 GB | Official | 32B clean, 70B Q2 |
| RX 7900 XT | 20 GB | Official | 32B clean |
| RX 7900 GRE | 16 GB | Official | 13B clean, 32B tight |
| RX 7800 XT | 16 GB | Official | 13B clean, 32B tight |
| RX 7700 XT | 12 GB | Official | 13B clean |
| RX 7600 XT | 16 GB | Official | 13B clean, 32B tight |
| RX 7600 | 8 GB | Official | 7B/8B clean |
| RX 6950/6900/6800 XT/6800 | 16 GB | Official | 13B clean, 32B tight |
| RX 6700 XT | 12 GB | Community / Vulkan | 13B clean |
| RX 6700 | 10 GB | Community / Vulkan | 7B/8B clean |
| RX 6600 XT / 6600 | 8 GB | Community / Vulkan | 7B/8B clean |
| RX 5700 XT | 8 GB | Community / Vulkan | 7B/8B clean |
| Radeon 780M (iGPU) | up to 8 GB | Community / Vulkan | 7B tight |
| Radeon 890M (iGPU) | up to 16 GB | Community v0.18.2+ | 13B if VRAM allocated |

Full table with GFX codes: [gpu_compatibility.md](gpu_compatibility.md)

---

## Common Failures Quick Reference

| Symptom | Fix |
|---|---|
| "Unable to load amdhip64_6.dll" | Update AMD Adrenalin drivers |
| GPU detected but 0% GPU utilization | Model too large for VRAM, or iGPU selected |
| 0xc0000005 crash after Ollama update | Delete `%LOCALAPPDATA%\Programs\Ollama\lib\ollama`, reinstall |
| HSA_OVERRIDE_GFX_VERSION ignored | Requires community fork, not official Windows binary |
| OLLAMA_VULKAN=1 does nothing | Update to Ollama v0.12.11+ |
| RX 6800/6900 not working | Update from v0.12.1-0.12.10 to v0.12.11+ (gfx1030 regression) |
| AMD GPU not detected in Docker | Use Vulkan; `/dev/kfd` unavailable in WSL2-backed Docker |

Full troubleshooting with root causes and fixes: [troubleshooting.md](troubleshooting.md)

---

## Environment Variables Reference

| Variable | Value | Purpose |
|---|---|---|
| `OLLAMA_VULKAN` | `1` | Vulkan backend (v0.12.11+, all AMD with Adrenalin drivers) |
| `OLLAMA_GPU_OVERHEAD` | bytes | Reserve VRAM before model load |
| `OLLAMA_FLASH_ATTENTION` | `1` | Reduce KV cache VRAM (ROCm Linux only) |
| `OLLAMA_DEBUG` | `1` | Verbose GPU detection logs |
| `OLLAMA_KEEP_ALIVE` | seconds or `-1` | Keep model in VRAM after last request |
| `ROCR_VISIBLE_DEVICES` | `0`, `1` | Select AMD GPU by index (ROCm) |
| `HIP_VISIBLE_DEVICES` | `0`, `1` | Alternate device selection |
| `HSA_OVERRIDE_GFX_VERSION` | e.g. `10.3.0` | Override GPU arch (community fork only on Windows) |
| `OLLAMA_HOST` | `0.0.0.0:11434` | Bind address for network access |
| `OLLAMA_MODELS` | path | Model storage directory |

---

## Verification Script

```powershell
pip install requests
python verify_ollama_gpu.py

# Options:
#   --model llama3.2:3b     Test with specific model
#   --skip-inference         Skip inference test
#   --json                   JSON output for scripting
```

---

## Useful Links

- [Ollama GPU docs](https://docs.ollama.com/gpu)
- [ollama-for-amd community fork](https://github.com/likelovewant/ollama-for-amd/releases)
- [AMD Adrenalin drivers](https://www.amd.com/en/support/download/drivers.html)
- [Ollama GitHub issues (AMD)](https://github.com/ollama/ollama/issues?q=label%3Aamd)
- [Phoronix: Ollama Vulkan support](https://www.phoronix.com/news/ollama-0.12.11-Vulkan)
- [Ollama AMD blog post (March 2024)](https://ollama.com/blog/amd-preview)

---

## Contributing

Open a PR or issue with: GPU model, VRAM, Ollama version, method used, tokens/sec from `python verify_ollama_gpu.py`, env variables set.

---

## License

MIT -- see [LICENSE](LICENSE)
