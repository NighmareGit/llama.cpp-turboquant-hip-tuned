# Docker Images — Romulus Compute Grid (AMD ROCm + TurboQuant)

This directory contains the **official compartmentalized Dockerfiles** for the Romulus Compute Grid running on AMD Radeon RX 7900 XTX (gfx1100) with ROCm 6.4.

## Philosophy

We maintain **two separate, single-purpose images** instead of one combined image:

| Image | Purpose | When to use |
|-------|---------|-------------|
| `Dockerfile.llama-amd-tom-tq-pure` | Standard `llama-server` inference worker | Most nodes in the grid |
| `Dockerfile.llama-amd-tom-rpc-tq` | Dedicated `llama-rpc-server` | KV cache coordination, distributed workers, RPC frontends |

This separation improves maintainability, allows independent updates, clearer resource allocation, and follows the principle of least privilege.

## Files

- `Dockerfile.llama-amd-tom-tq-pure` — Pure TurboQuant `llama-server` image
- `Dockerfile.llama-amd-tom-rpc-tq` — Dedicated TurboQuant `llama-rpc-server` image
- `README.md` — This file

## Building the Images

All images are based on our tuned fork:  
**https://github.com/NighmareGit/llama.cpp-turboquant-hip-tuned** (branch `feature/turboquant-cuda-rpc-build-fixed`)

### Basic build (using default proxy)

```bash
# Pure inference worker
docker build \
  -t romulus-llama-tq-pure:latest \
  -f docker/Dockerfile.llama-amd-tom-tq-pure .

# Dedicated RPC server
docker build \
  -t romulus-llama-rpc-tq:latest \
  -f docker/Dockerfile.llama-amd-tom-rpc-tq .
```

### Build with custom proxy IP (e.g. your other machine)

```bash
docker build --build-arg PROXY_IP=192.168.8.116 \
  -t romulus-llama-tq-pure:latest \
  -f docker/Dockerfile.llama-amd-tom-tq-pure .
```

> The `PROXY_IP` build argument configures the local Squid + proxpi cache. It gracefully falls back if the proxy is offline.

## Running the Containers

### Pure inference worker

```bash
docker run -d --name llama-worker \
  --device=/dev/kfd --device=/dev/dri \
  --security-opt seccomp=unconfined \
  -v /mnt/models:/models:ro \
  -p 8080:8080 \
  romulus-llama-tq-pure:latest
```

### Dedicated RPC server

```bash
docker run -d --name llama-rpc \
  --device=/dev/kfd --device=/dev/dri \
  --security-opt seccomp=unconfined \
  -v /mnt/models:/models:ro \
  -p 50053:50053 \
  romulus-llama-rpc-tq:latest
```

You can override the default CMD to run `llama-server` inside the RPC image if needed for dual-use testing.

## Key Features (both images)

- ROCm 6.4 + gfx1100 optimized build
- TurboQuant KV cache support (via our tuned fork)
- Aggressive optimizations (`-Ofast`, LTO, AMD-specific LLVM flags)
- `GGML_SCHED_MAX_COPIES=2` (balanced CPU safety on high-core systems like i7-14700K)
- Non-destructive / additive-only builds (permanent Docker safety rule compliant)
- Local cache integration with graceful fallback

## Source & Maintenance

These Dockerfiles are maintained alongside the tuned llama.cpp fork.  
The actual source is cloned at build time from the internal gitea instance (`192.168.8.108:3005`).

For questions or contributions, open an issue in the main repository.

---
**Romulus Compute Grid** — Built for performance, safety, and clarity.