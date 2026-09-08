---
layout: post
title: Working on Caviness with containers and VS Code tunnels
date: 2026-09-07 00:30:00-0400
description: A step-by-step setup for a reproducible Python environment on the University of Delaware's Caviness cluster. Build a Docker image on your laptop, pull it as a Singularity image on the cluster, open a VS Code tunnel on a GPU node, and keep the code in GitHub.
tags: hpc docker singularity vscode caviness
categories: tooling
featured: true
related_posts: false
toc:
  sidebar: left
mermaid:
  enabled: true
  zoomable: true
---

This post is a guide to my workflow for running experiments on [Caviness](https://docs.hpc.udel.edu/abstract/caviness/caviness), the University of Delaware's community cluster. For most of my projects I build a container image on my laptop for the environment, keep the code in a GitHub repository, and work on a compute node through a VS Code tunnel, much like I would on my own machine.

The post walks through every step from an empty folder to a running tunnel, so someone starting from zero can follow along. Each step ends with the commands, so you can skip the explanations if you only need those.


## What you need

- **A Caviness account and a workgroup.** You need the workgroup name for the `workgroup` command below.
- **Docker installed locally.** [Docker Desktop](https://docs.docker.com/get-docker/) on macOS or Windows, or Docker Engine on Linux. You build the image here; the cluster only runs it.
- **A Docker Hub account.** A free account with public repositories is enough, since the image only contains libraries.
- **A GitHub account.** VS Code tunnels authenticate with a GitHub or Microsoft account, and both ends must use the same one. I recommend GitHub, because the same account can host your code, and the last section relies on that.
- **VS Code** with the [Remote - Tunnels](https://marketplace.visualstudio.com/items?itemName=ms-vscode.remote-server) extension installed.

Everything in this post uses placeholders: `<workgroup>` for your Caviness workgroup, `<user>` for your cluster username, `<dockerhub-user>` for your Docker Hub username, and `<project>` for the project name.

## Step 1: Separate the environment from the code

The container only holds the environment: Python, the libraries, a few system tools, and the VS Code CLI. The code and the data stay on the cluster's file systems and are bind-mounted into the container at run time. Keeping them apart has two advantages. You only rebuild the image when a dependency changes, which does not happen often, while the code changes every day. And the image can be public, since nothing in it belongs to you.

My projects keep the container recipe in its own folder next to the code repository:

```
<project>/
  code/            git repository, cloned on the cluster too
  container/
    Dockerfile
    pyproject.toml
    start-tunnel.sh
    verify_env.py
```

The `container/` folder is the build context. Nothing outside it is visible to `docker build`, so the build stays small and the data cannot end up in the image by accident.

## Step 2: Declare the dependencies

The dependencies live in a standard `pyproject.toml`, and the image installs them with [uv](https://docs.astral.sh/uv/). Two details are easy to get wrong.

**The CUDA version has to match the cluster driver.** PyTorch wheels bundle their own CUDA runtime, but the driver on the compute node caps what that runtime can use. On Caviness the driver supports CUDA 12.4, so I pin the `cu124` wheel index. If you pick a newer index, `torch.cuda.is_available()` returns `False` and nothing tells you why.

**Keep the PyTorch index explicit.** With `explicit = true`, uv only consults that index for the packages that name it. Everything else resolves from PyPI as usual.

A trimmed version of the file I use:

```toml
[project]
name = "<project>"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "numpy>=2.0.0",
    "pandas>=2.3.0",
    "polars>=1.30.0",
    "scikit-learn>=1.6.0",
    "torch>=2.6.0",
    "torchvision>=0.21.0",
    "lightning>=2.5.0",
    "hydra-core>=1.3.2",
    "wandb>=0.20.0",
    "matplotlib>=3.10.0",
    "ipykernel>=6.29.0",
    "debugpy>=1.8.0",
]

[tool.uv.sources]
torch = [{ index = "pytorch-cu124" }]
torchvision = [{ index = "pytorch-cu124" }]

[[tool.uv.index]]
name = "pytorch-cu124"
url = "https://download.pytorch.org/whl/cu124"
explicit = true
```

VS Code needs `ipykernel` to run notebooks inside the container and `debugpy` for the Python debugger.

## Step 3: Write the Dockerfile

The Dockerfile has two stages. The builder stage installs the dependencies with uv. The runtime stage starts from the same slim base, copies the installed packages over, adds the handful of system tools the slim image lacks, and installs the VS Code CLI. uv and the build tools stay in the first stage.

```dockerfile
# syntax=docker/dockerfile:1
#
ARG PYTHON_VERSION=3.12

# -----------------------
# Stage 1: builder
# -----------------------
FROM python:${PYTHON_VERSION}-slim AS builder

ENV DEBIAN_FRONTEND=noninteractive
WORKDIR /project

# uv, copied from its official image
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

ENV UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1 \
    UV_PYTHON_DOWNLOADS=never

# Only the dependency file enters the context, so this layer is cached
# until pyproject.toml changes.
COPY pyproject.toml ./

# Install into the image's system Python, then drop uv.
RUN uv pip install \
      --python /usr/local/bin/python3 \
      --system . \
      --no-cache-dir \
 && rm -f /usr/local/bin/uv

# -----------------------
# Stage 2: runtime
# -----------------------
FROM python:${PYTHON_VERSION}-slim AS runtime

ENV DEBIAN_FRONTEND=noninteractive
WORKDIR /project

# ca-certificates and curl: download the VS Code CLI and let it reach GitHub.
# git: pull and push from inside the tunnel.
# tini: a small init as PID 1, so signals reach `code tunnel` cleanly.
# libgomp1: OpenMP runtime that xgboost, catboost, and scikit-learn wheels link against.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git tini libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Same base image in both stages, so /usr/local can be copied as a whole.
COPY --from=builder /usr/local /usr/local

ENV PATH="/usr/local/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# VS Code CLI. The alpine build is statically linked, so it runs on any
# x86_64 Linux, including inside Singularity on the cluster.
RUN set -eux; \
    mkdir -p /tmp/vscode-cli; \
    curl -L 'https://code.visualstudio.com/sha/download?build=stable&os=cli-alpine-x64' \
      -o /tmp/vscode-cli.tar.gz; \
    tar -xzf /tmp/vscode-cli.tar.gz -C /tmp/vscode-cli; \
    install -m 0755 /tmp/vscode-cli/code /usr/local/bin/code; \
    rm -rf /tmp/vscode-cli /tmp/vscode-cli.tar.gz

COPY start-tunnel.sh /usr/local/bin/start-tunnel.sh
RUN chmod +x /usr/local/bin/start-tunnel.sh

ENTRYPOINT ["/usr/bin/tini", "-s", "--"]
CMD ["/usr/local/bin/start-tunnel.sh"]
```

A few decisions in this file are not obvious.

**No virtual environment.** Installing into the system Python of the official image means there is nothing to activate. `python` inside the container is always the right interpreter, whether you run it through the tunnel, through `singularity exec`, or in a batch job.

**Why `tini`.** Singularity runs the container's command as a normal process, and `code tunnel` expects to receive signals directly. `tini` forwards them and reaps child processes, so the tunnel shuts down cleanly when Slurm cancels the job.

**No CUDA toolkit.** The PyTorch wheels ship their own CUDA runtime, and Singularity's `--nv` flag brings in the host driver. Without the toolkit the image is around two gigabytes instead of eight.

## Step 4: The tunnel entry script

The default command of the image starts a VS Code tunnel. Two environment variables adapt it to a shared cluster.

```sh
#!/bin/sh
# Start a VS Code tunnel named after this node, with per-node CLI state.
#
# TUNNEL_NAME: tunnel name shown in VS Code. Defaults to the node hostname.
#   Rules: lowercase letters, digits, and hyphens, 4 to 20 characters.
# TUNNEL_DATA_DIR: where the CLI keeps its state, the singleton lock, and the
#   downloaded server. Defaults to a per-node directory under $HOME.
set -eu

name="${TUNNEL_NAME:-$(uname -n | cut -d. -f1 | tr 'A-Z' 'a-z' | tr -cs 'a-z0-9' '-' | sed 's/^-//;s/-$//' | cut -c1-20)}"
data_dir="${TUNNEL_DATA_DIR:-${HOME:-/tmp}/.vscode-cli-${name}}"
mkdir -p "$data_dir"

# exec so `code tunnel` replaces this shell and becomes tini's direct child.
exec code tunnel --accept-server-license-terms --name "$name" --cli-data-dir "$data_dir" "$@"
```

`TUNNEL_NAME` is the name that shows up in the VS Code Remote Explorer. The default is the node's hostname, which is convenient on a cluster because it tells you where you are running.

`TUNNEL_DATA_DIR` fixes a problem you will hit the second time you start a tunnel. The CLI keeps a lock file in its data directory. Your home directory is the same on every node, so two tunnels on two nodes fight over one lock, and the second one dies with `error access singleton`. Giving each tunnel its own directory, keyed by node name, avoids that. If your home quota is tight, point it at node-local scratch instead, as shown later.

## Step 5: Build and test the image locally

Caviness nodes are x86_64. If your laptop is an Apple Silicon Mac, Docker builds ARM images by default, and the cluster will refuse them with `exec format error`. Always pass the platform flag.

```bash
cd <project>
docker build --platform linux/amd64 -t <dockerhub-user>/<project>:v1 container/
```

The first build downloads PyTorch and takes a while. Later builds reuse the cached layers unless `pyproject.toml` changed.

Test the image before pushing it. The tunnel needs no GPU to start, and the imports need no GPU to succeed:

```bash
docker run --rm <dockerhub-user>/<project>:v1 python -c "import torch, lightning; print(torch.__version__)"
docker run --rm -it --entrypoint bash <dockerhub-user>/<project>:v1
```

The second command drops you into a shell inside the image, which is the fastest way to check that a tool you expect is there.

> On an Apple Silicon Mac the `linux/amd64` image runs under emulation, so it is slow. That is fine for checking imports, but do not time anything there.
{: .block-tip }

## Step 6: Push the image to Docker Hub

```bash
docker login
docker push <dockerhub-user>/<project>:v1
```

Use a version tag instead of `latest`. When you rebuild with new dependencies, push `v2`. The old `.sif` on the cluster keeps working until you decide to switch, and you always know which environment produced which result.

## Step 7: Pull the image on Caviness

Log in, set your workgroup, and load Singularity.

```bash
ssh <user>@caviness.hpc.udel.edu
workgroup -g <workgroup>
vpkg_require singularity
```

Before pulling, decide where things live. Caviness has [four file systems](https://docs.hpc.udel.edu/abstract/caviness/filesystems/filesystems), and each has a different purpose.

| Location | What goes there | Why |
| --- | --- | --- |
| `/home/<uid>` | Dotfiles, SSH keys | 20 GB quota. An image pull alone can fill it. |
| `/work/<workgroup>` | The `.sif`, the code repository, the Singularity cache | Shared group storage, at least 1 TB, backed up. Executables are allowed here. |
| `/lustre/scratch` | Datasets and large intermediate files | Fast parallel file system, but purged periodically and no executables allowed. |
| `$TMPDIR` | Per-job scratch, the tunnel's CLI state | Node-local disk, gone when the job ends. |

One setting to get right on the first day is the cache directory. Singularity unpacks every Docker layer into a cache before assembling the `.sif`, and by default that cache is in your home directory. Point it at your workgroup storage, and put the export in your `.bashrc` so you do not have to remember it.

```bash
mkdir -p /work/<workgroup>/<user>/containers /work/<workgroup>/<user>/.singularity
export SINGULARITY_CACHEDIR=/work/<workgroup>/<user>/.singularity

cd /work/<workgroup>/<user>/containers
singularity pull <project>.sif docker://<dockerhub-user>/<project>:v1
```

The pull runs on the login node, which has internet access, and produces a single file, `<project>.sif`. Everything after this point runs on a compute node.

## Step 8: Get a compute node

Interactive work goes through `salloc`. The `_workgroup_` partition resolves to your own workgroup's priority nodes; use `standard` if you are fine with preemption, or `devel` for quick tests. Request only what you need, since a GPU sitting idle in an interactive session is unavailable to everyone else.

```bash
salloc --partition=_workgroup_ --gres=gpu:1 --cpus-per-task=8 --mem=48G --time=04:00:00
```

To ask for a specific GPU type, replace `--gres=gpu:1` with `--gres=gpu:v100:1` (Caviness has `p100`, `v100`, `t4`, and `a100` nodes, depending on the workgroup). When the allocation starts, your prompt changes to the compute node's hostname. Everything from here on runs on that node.

> An interactive allocation ends when your SSH session ends. If your connection is unreliable, start `tmux` on the login node before running `salloc`, so a dropped connection does not kill the job.
{: .block-tip }

## Step 9: Verify the environment

Before opening an editor, confirm that the image sees the GPU. This small script checks the pieces that fail most often, in dependency order, so the first failure points at the cause:

```python
#!/usr/bin/env python
"""Sanity checks for the container. Run: singularity exec --nv <project>.sif python verify_env.py"""
import importlib
import sys

failures = 0


def check(label, fn):
    global failures
    try:
        print(f"[ OK ] {label}: {fn()}")
    except Exception as exc:  # noqa: BLE001
        failures += 1
        print(f"[FAIL] {label}: {type(exc).__name__}: {exc}")


check("python", lambda: sys.version.split()[0])
check("torch", lambda: importlib.import_module("torch").__version__)
check("torch cuda build", lambda: importlib.import_module("torch").version.cuda)
check("cuda available", lambda: importlib.import_module("torch").cuda.is_available() or (_ for _ in ()).throw(RuntimeError("no GPU")))
check("gpu", lambda: importlib.import_module("torch").cuda.get_device_name(0))
for pkg in ["numpy", "pandas", "polars", "sklearn", "lightning", "hydra", "wandb"]:
    check(pkg, lambda p=pkg: importlib.import_module(p).__version__)

sys.exit(1 if failures else 0)
```

Keep it in `container/` and run it through the image:

```bash
cd /work/<workgroup>/<user>/<project>
singularity exec --nv containers/<project>.sif python container/verify_env.py
```

The `--nv` flag mounts the host's NVIDIA driver into the container. Without it, the `cuda available` check fails and everything else passes.

## Step 10: Start the tunnel and connect

Run the image. Its default command is the tunnel script from Step 4.

```bash
singularity run --nv \
  --env TUNNEL_NAME=<project>-gpu \
  --env TUNNEL_DATA_DIR=$TMPDIR/vscode-cli \
  -B /work/<workgroup>:/work/<workgroup> \
  /work/<workgroup>/<user>/containers/<project>.sif
```

The bind flag makes your workgroup storage visible inside the container at the same path. Singularity binds your home directory and `/tmp` on its own; anything else you want to see from the editor needs a `-B`.

The first time, the CLI asks how to sign in. Choose GitHub, open the printed `https://github.com/login/device` URL in your laptop's browser, and enter the one-time code. This links the tunnel to your GitHub account, and only that account can connect to it. On later runs the CLI reuses the stored credentials, as long as `TUNNEL_DATA_DIR` still exists. Because node-local scratch is wiped at the end of the job, you will re-authenticate when you use `$TMPDIR`; if you would rather not, keep the data directory under `/work/<workgroup>/<user>/.vscode-cli-<node>` instead.

Now on your laptop: open VS Code, open the Remote Explorer view, pick **Tunnels** in the dropdown, and connect to `<project>-gpu`. VS Code installs its server through the tunnel on first use, then opens a window whose terminal, Python interpreter, debugger, and notebooks all run inside the container on the compute node. Open the folder `/work/<workgroup>/<user>/<project>/code` and you are working on the cluster.

The tunnel stays up as long as the `singularity run` process, which ends with the allocation. When the time limit is reached the window disconnects, and to get back in you request a new node and start the tunnel again.

## Step 11: Keep the code in GitHub

The tunnel lets you edit directly on the cluster, and for quick fixes I do. For anything larger I edit on my laptop, push, and pull on the cluster. Local tools are faster and always available, including AI coding assistants such as Claude Code or Copilot, which work on a local checkout. And since every change passes through GitHub, the git history tells you which code ran when, which helps a lot when writing up results.

One-time setup on the cluster:

```bash
cd /work/<workgroup>/<user>/<project>
git clone git@github.com:<github-user>/<project>.git code
```

Add your cluster SSH key to your GitHub account so that pushes and pulls do not prompt for a password. Then the daily loop is:

1. Edit on the laptop. Run the fast checks that need no GPU: lint, type check, unit tests.
2. Commit and push.
3. In the tunnel terminal on the cluster, `git pull`, then run the experiment.
4. Results and logs land in `/work` or `/lustre`. Pull the small ones back with `rsync`; commit only figures and tables, never checkpoints.

Compute nodes have limited internet access. In my experience the tunnel connects fine from a compute node, but large downloads such as model weights, pretrained checkpoints, and datasets should happen on the login node, which has full access. If `git pull` stalls on a compute node, run it from a login-node shell instead. The container includes `git`, so the pull works from either the tunnel terminal or the host shell.

## Step 12: Batch jobs without a tunnel

The tunnel is for development. Once a script runs end to end, submit it as a batch job so it does not depend on your laptop being awake. The same image runs the script through `singularity exec`, with the same bind mounts.

```bash
#!/bin/bash
#SBATCH --job-name=<project>-train
#SBATCH --partition=_workgroup_
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=48G
#SBATCH --time=12:00:00
#SBATCH --output=/work/<workgroup>/<user>/<project>/logs/%x-%j.out

vpkg_require singularity
export SINGULARITY_CACHEDIR=/work/<workgroup>/<user>/.singularity

SIF=/work/<workgroup>/<user>/containers/<project>.sif
CODE=/work/<workgroup>/<user>/<project>/code

cd "$CODE"
singularity exec --nv \
  -B /work/<workgroup>:/work/<workgroup> \
  -B /lustre/scratch/<user>:/lustre/scratch/<user> \
  "$SIF" python train.py "$@"
```

Save it as `train.qs`, then:

```bash
workgroup -g <workgroup>
mkdir -p /work/<workgroup>/<user>/<project>/logs
sbatch train.qs
squeue -u <user>
tail -f /work/<workgroup>/<user>/<project>/logs/<project>-train-<jobid>.out
```

Anything after `sbatch train.qs` on the command line is forwarded to the script through `"$@"`, so Hydra overrides and flags pass straight through.

If you want to watch a long run without a batch job, launch it in the background inside the interactive allocation and keep the tunnel open next to it.

```bash
nohup singularity exec --nv -B /work/<workgroup>:/work/<workgroup> \
  /work/<workgroup>/<user>/containers/<project>.sif python train.py > train.log 2>&1 &
tail -f train.log
```

`nohup` survives a dropped SSH session but not the end of the allocation, so request enough time.

## Step 13: Update the environment

When a dependency changes, the loop is: edit `pyproject.toml`, rebuild, push with a new tag, pull the new tag on the cluster.

```bash
# laptop
docker build --platform linux/amd64 -t <dockerhub-user>/<project>:v2 container/
docker push <dockerhub-user>/<project>:v2

# caviness login node
cd /work/<workgroup>/<user>/containers
singularity pull <project>-v2.sif docker://<dockerhub-user>/<project>:v2
```

Keep the old `.sif` until you have re-run the verification script and a smoke test with the new one. Then update the path in your job scripts. If you want to pin exact versions after a successful build, freeze them from inside the image and commit the result next to `pyproject.toml`:

```bash
singularity exec <project>-v2.sif python -m pip freeze > container/requirements.lock
```

## Troubleshooting

- **`exec format error` on the cluster.** The image was built for ARM. Rebuild with `--platform linux/amd64`.
- **`error access singleton` when starting the tunnel.** Another tunnel, possibly on another node, holds the lock in the same data directory. Set `TUNNEL_DATA_DIR` per node, as in Step 4.
- **Disk quota exceeded during `singularity pull`.** The layer cache went to your home directory. Set `SINGULARITY_CACHEDIR` to `/work` and pull again.
- **`cuda available: RuntimeError: no GPU`.** Either you forgot `--nv`, or you are on the login node, or the wheels target a CUDA version newer than the driver supports. Check the `torch cuda build` line against 12.4.
- **Tunnel name rejected.** Names are 4 to 20 characters of lowercase letters, digits, and hyphens.
- **Locale warnings from Python or git.** Start the container with `LANG=C.UTF-8` in front of the `singularity` command.
- **VS Code shows the tunnel but cannot connect.** The allocation ended. Check `squeue -u <user>`; if the job is gone, request a new node and start the tunnel again.

<!-- ## Closing

The whole setup comes down to three files in a `container/` folder and about ten commands. In return, the environment is the same on every node and every rerun, the editor runs next to the GPU, and every change goes through GitHub whether I type it on the laptop or on the cluster. The recipe in this post is the one I use in my current projects, minus the project-specific dependencies. Copy it, edit the `pyproject.toml`, and it should work as is. -->
