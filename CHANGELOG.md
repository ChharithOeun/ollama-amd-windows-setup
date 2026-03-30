# Changelog

All notable changes to this guide will be documented here.

## [1.0.0] - 2026-03-29

### Added
- Initial release covering three methods: WSL2+ROCm, Native Windows (ROCm+Vulkan), Docker
- GPU compatibility table for RX 5000/6000/7000/9000 series
- `verify_ollama_gpu.py` — automated detection + inference test script
- `wsl2_setup.sh` — automated WSL2 ROCm + Ollama installer
- `docker-compose.yml` — Docker method with Vulkan fallback
- `troubleshooting.md` — 10 failure modes with root causes and fixes
- `gpu_compatibility.md` — full GFX version matrix with per-card method recommendations
- Coverage of `ollama-for-amd` community fork (v0.18.2, ROCm 6.4)
- Vulkan backend documentation (Ollama v0.12.11+, `OLLAMA_VULKAN=1`)
- Environment variable reference

### Sources
- Ollama official hardware docs: https://docs.ollama.com/gpu
- Ollama AMD preview blog post (March 2024): https://ollama.com/blog/amd-preview
- Phoronix Vulkan coverage: https://www.phoronix.com/news/ollama-0.12.11-Vulkan
- Community fork: https://github.com/likelovewant/ollama-for-amd
- GitHub issues: #12388, #12752, #10781, #11975, #3107
