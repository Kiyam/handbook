# Containers with Apptainer

Clusters run old operating systems, the module system never has the version you need, and you don't have root. A container skips all of that. You build the whole software stack once and copy it to the cluster as a single `.sif` file. Then it runs the same way on every machine.

**Apptainer** is the HPC container runtime. It used to be called **Singularity**, and most clusters have one of the two installed: the `singularity` command is usually an alias. Unlike Docker, it needs no root daemon, runs as your own user, and mounts `$HOME` and the current directory automatically.

```bash
apptainer --version   # or: singularity --version
```

## The definition file

An image is built from a `.def` file. This is a trimmed version of the one I use for a JAX + PyTorch project. It works on both **x86_64** and **aarch64** (NVIDIA GH200 Grace Hopper) machines.

```text title="myproject.def"
Bootstrap: docker
From: nvcr.io/nvidia/cuda:12.6.3-devel-ubuntu22.04

# The NGC CUDA image is multi-arch, so the same file builds on x86_64 and aarch64.

%files
    codes/mylib      /opt/codes/mylib
    constraints.txt  /opt/constraints.txt

%environment
    export PATH=/opt/venv/bin:/usr/local/cuda/bin:$PATH
    export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
    export XLA_FLAGS="--xla_gpu_cuda_data_dir=/usr/local/cuda"
    export CUDA_HOME=/usr/local/cuda

%post
    export DEBIAN_FRONTEND=noninteractive
    export PIP_ROOT_USER_ACTION=ignore
    echo ">>> Building for architecture: $(uname -m)"

    # Build-time system packages
    apt-get update && apt-get install -y \
        wget git cmake ninja-build g++ software-properties-common

    # Python 3.13 from the deadsnakes PPA
    add-apt-repository -y ppa:deadsnakes/ppa
    apt-get update && apt-get install -y python3.13 python3.13-dev python3.13-venv

    python3.13 -m venv /opt/venv
    export PATH=/opt/venv/bin:$PATH
    pip install --no-cache-dir --upgrade pip

    # JAX ships its own CUDA libraries, with wheels for both architectures
    pip install --no-cache-dir "jax[cuda12]==0.5.3"

    # On aarch64 the default PyPI torch wheel is CPU-only, so always use the CUDA index
    pip install --no-cache-dir torch --index-url https://download.pytorch.org/whl/cu126

    pip install --no-cache-dir -c /opt/constraints.txt matplotlib numpyro healpy
    pip install --no-cache-dir -c /opt/constraints.txt /opt/codes/mylib

    # Remove build-only packages and files to shrink the image
    apt-get remove --purge -y cmake ninja-build g++ git wget \
        python3.13-dev software-properties-common
    apt-get autoremove --purge -y && apt-get clean
    rm -rf /var/lib/apt/lists/* /opt/codes /opt/constraints.txt

%runscript
    exec python "$@"

%labels
    Base   nvcr.io/nvidia/cuda:12.6.3-devel-ubuntu22.04 (multi-arch)
    Python 3.13

%help
    Python 3.13 with JAX (CUDA 12) and PyTorch (CUDA).
      apptainer exec --nv myproject.sif python my_script.py
```

The sections:

| Section | Runs | Purpose |
|---|---|---|
| `Bootstrap` / `From` | — | Base image. `docker` pulls from any OCI registry, here NVIDIA's NGC. |
| `%files` | before `%post` | Copies local files into the image, such as your own code or a constraints file. |
| `%post` | at build time, as root | Installs everything. This is where most of the file goes. |
| `%environment` | every time the container starts | Sets `PATH`, CUDA variables and so on. |
| `%runscript` | `apptainer run` or `./myproject.sif` | Default command. Here `./myproject.sif script.py` runs Python. |
| `%labels` / `%help` | — | Metadata, shown with `apptainer inspect` and `apptainer run-help`. |

!!! tip "Things that made a difference"
    - **Pin versions.** Pass a `constraints.txt` (`pip install -c`) so rebuilding in six months gives you the same environment.
    - **Clean up at the end of `%post`.** Compilers, `-dev` headers and copied source code are only needed while building. Purging them and the apt lists saves hundreds of MB.
    - **`--no-cache-dir` on every `pip install`.** Otherwise the pip cache ends up inside the image.
    - **A CPU-only variant** from `From: ubuntu:22.04` without the CUDA wheels is *much* smaller: about 0.6 GB, against about 7.7 GB for the GPU image. Use it for CPU partitions and quick tests.

## Building

`apptainer build` needs root or fakeroot. Many clusters allow `--fakeroot` on login nodes. If yours doesn't, build on your own Linux machine and copy the `.sif` across.

```bash
# Native build, on either architecture
apptainer build myproject.sif myproject.def

# Without root, where the cluster allows it
apptainer build --fakeroot myproject.sif myproject.def

# Cross-build an aarch64 image on an x86_64 machine (needs QEMU + binfmt_misc)
apptainer build --arch arm64 myproject_aarch64.sif myproject.def
```

!!! warning "Cross-building is slow"
    Building for arm64 under QEMU emulation works, but anything that compiles C++ takes many times longer than a native build. If you have access to an aarch64 node, build there.

Apptainer caches downloaded layers in `~/.apptainer/cache`. That fills up a small home quota fast, so point it at scratch:

```bash
export APPTAINER_CACHEDIR=/scratch/$USER/.apptainer-cache
apptainer cache clean   # when it gets too big
```

## Running

```bash
# Run a script with GPU access (--nv binds the host's NVIDIA driver into the container)
apptainer exec --nv myproject.sif python my_script.py

# Interactive shell inside the container
apptainer shell --nv myproject.sif

# Use the %runscript
./myproject.sif my_script.py

# Make another directory visible inside the container
apptainer exec --nv --bind /scratch/$USER:/data myproject.sif python train.py --data /data
```

In a Slurm job you put the container in front of the command you'd normally run:

```bash title="job.sbatch"
#!/bin/bash
#SBATCH --gres=gpu:1
#SBATCH -c 8
#SBATCH -t 08:00:00
#SBATCH -o logs/%x-%j.log

srun apptainer exec --nv /scratch/$USER/myproject.sif python run.py "$@"
```

!!! note
    - The `.sif` is a single read-only file. Keep it on shared or scratch storage, not in `$HOME`, where a few multi-GB images eat the quota. Every node can read it from there.
    - The container uses the **host's** GPU driver, so the CUDA version in the image must be one the driver supports. Check with `nvidia-smi` on a compute node.
    - Your `$HOME` is mounted inside, so a `~/.local` full of pip packages can leak into the container's Python. If imports behave strangely, try `--no-home` or `PYTHONNOUSERSITE=1`.
