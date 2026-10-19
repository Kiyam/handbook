# Working on clusters

Patterns I keep reusing across the Slurm clusters I work on. Hostnames, users and paths below are placeholders.

## Getting in

### Jump hosts

Most clusters sit behind a gateway, so you SSH to that first and then on to the login node. `ProxyJump` does both in one go, and chains: compute nodes are often only reachable from the login node.

```text title="~/.ssh/config"
Host <gateway>
    HostName gateway.example.ac.uk
    User <user>

Host <cluster>
    HostName login.cluster.example.ac.uk
    User <user>
    ProxyJump <gateway>
    ForwardAgent yes
    AddKeysToAgent yes

# A specific GPU node, reached through the login node
Host <gpu-node>
    HostName compute-gpu-0-6
    User <user>
    ProxyJump <cluster>
```

After that, `ssh <gpu-node>`, `scp`, `rsync` and VS Code Remote all go straight through. `ssh <gpu-node>` usually only works while you have a job running on that node.

### Ping before connecting

Some clusters only open SSH after the firewall has seen a specific packet, such as a ping of a certain size. A `ProxyCommand` sends it automatically before each connection:

```text title="~/.ssh/config"
Host <cluster>01
    HostName cluster01.example.fr
    User <user>
    ProxyCommand sh -c 'ping -c 1 -s 999 %h; exec nc %h %p'
```

If the cluster has more than one login node, one entry can try each in turn and connect to the first that answers. `HostKeyAlias` stops SSH complaining that the host key changed when it lands on a different node:

```text title="~/.ssh/config"
Host <cluster>
    User <user>
    HostKeyAlias <cluster>
    ProxyCommand sh -c 'for h in cluster01.example.fr cluster02.example.fr; do ping -c 1 -s 999 $h >/dev/null 2>&1; nc -z -G 3 $h %p 2>/dev/null && exec nc $h %p; done; echo "no node reachable" >&2; exit 1'
```

!!! note
    `nc -G 3` is the macOS connect timeout. On Linux, use `nc -w 3`.

### Short-lived certificates

Some national facilities don't take a static key at all. You log in through a browser and a CLI tool fetches a short-lived SSH certificate and writes a config file for you. [clifton](https://github.com/isambard-sc/clifton) is one example. Keep that generated file separate and pull it in at the top of your own config:

```text title="~/.ssh/config"
Include ~/.ssh/config_clifton
```

## Slurm jobs

### A job script header

```bash title="job.sbatch"
#!/usr/bin/env bash
#SBATCH -p <partition>
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=8      # MPI ranks per node
#SBATCH --cpus-per-task=8        # threads per rank
#SBATCH --exclusive              # whole node, no neighbours
#SBATCH --time 48:00:00
#SBATCH -o slurm_out/%x_%j.out   # %x = job name, %j = job ID
#SBATCH --mail-user=<email>
#SBATCH --mail-type=END

cd "$SLURM_SUBMIT_DIR"
hostname
```

!!! warning
    Slurm doesn't create the directory in `-o`. If `slurm_out/` doesn't exist, the job fails immediately and writes no log telling you why.

### Hybrid MPI + OpenMP

For codes that use MPI between processes and threads within each, ranks × threads should match the cores on the node (8 × 8 = 64 here). Pin each rank to its own block of cores, or the threads fight over the same cores:

```bash
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
mpirun -np "$SLURM_NTASKS" \
  -x OMP_NUM_THREADS \
  --map-by ppr:8:node:PE=8 \
  --bind-to core \
  python my_sim.py
```

`ppr:8:node:PE=8` means 8 processes per node with 8 processing elements (cores) each. `-x` passes the variable to every rank.

### One generic GPU script

Rather than writing a new `.sbatch` per experiment, I keep a single one that runs whatever command it is given. Resources can be overridden on the command line:

```bash title="scripts/gpu.sbatch"
#!/bin/bash
# sbatch scripts/gpu.sbatch python scripts/train.py --lr 1e-3
# sbatch --gres=gpu:H100:1 -t 2-00:00:00 scripts/gpu.sbatch python ...
#SBATCH -J gpu
#SBATCH -p <gpu-partition>
#SBATCH --gres=gpu:1
#SBATCH -c 8
#SBATCH --mem=64G
#SBATCH -t 08:00:00
#SBATCH -o logs/%x-%j.log

cd "$SLURM_SUBMIT_DIR"
source scripts/env.sh
echo "$(hostname) $(date -Is) :: $*"
srun "$@"
```

Parameter sweeps then become a plain loop, with `-J` giving each job a readable name in `squeue`:

```bash title="scripts/submit_sweep.sh"
#!/bin/bash
cd "$(dirname "$0")/.."
for n in 256 512 1024; do
    for seed in 0 1 2 3 4; do
        sbatch -J "sweep_n${n}_s${seed}" scripts/gpu.sbatch \
            python scripts/train.py --n "$n" --seed "$seed" --out "results/n$n/s$seed"
    done
done
```

### Interactive GPU shell

For debugging, `srun --pty` drops you into a shell on a compute node with the environment already loaded:

```bash title="scripts/gpu_shell.sh"
#!/bin/bash
# scripts/gpu_shell.sh --gres=gpu:H100:1 -t 8:00:00
cd "$(dirname "$0")/.."
exec srun -p <gpu-partition> --gres=gpu:1 -c 8 --mem=64G -t 04:00:00 "$@" \
    --pty bash -c "source scripts/env.sh && exec bash -i"
```

Extra arguments come before `--pty`, so they override the defaults.

## GPU Python environments

When not using a [container](containers.md), I use a uv venv plus the cluster's CUDA module. The script below is sourced by both the batch script and the interactive shell above:

```bash title="scripts/env.sh"
# CUDA/NCCL come from the cluster module; only cuDNN comes from the venv (jax[cuda12-local]).
type module &>/dev/null || source /etc/profile.d/modules.sh  # non-login shells (sbatch)
module load cuda/12.9
_root=$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)
export LD_LIBRARY_PATH="$(echo "$_root"/.venv/lib/python3.*/site-packages/nvidia/cudnn/lib):$LD_LIBRARY_PATH"
export XLA_FLAGS="--xla_gpu_cuda_data_dir=$CUDA_PATH${XLA_FLAGS:+ $XLA_FLAGS}"
source "$_root/.venv/bin/activate"
unset _root
# Inline plots in the terminal for interactive sessions; batch jobs keep the Agg default.
[[ -t 1 ]] && export MPLBACKEND=module://sfm.termplot
```

Some notes on what's in it:

- **`module: command not found` inside a job.** Batch jobs run non-login shells, which don't set up environment modules. Sourcing `/etc/profile.d/modules.sh` first fixes that.
- **`jax[cuda12-local]`** installs JAX *without* bundling CUDA, so it uses the cluster's CUDA. That matches the installed driver and saves a few GB in the venv. cuDNN still comes from pip, hence the `LD_LIBRARY_PATH` line.
- **`XLA_FLAGS`** tells XLA where the CUDA toolkit is (`ptxas`, libdevice). Without it, JAX may fail or silently fall back to slow paths.
- **`[[ -t 1 ]]`** is true only when output goes to a terminal. Interactive sessions get plots drawn inline in the terminal, while batch jobs keep writing files.

## Living on old clusters

Older clusters run things like CentOS 7, with zsh 5.0 and a tmux too old for plugins, and you have no root. I handle that in three ways:

- **Per-host [chezmoi](../terminal/terminal-setup.md#chezmoi) templates.** The same dotfiles work everywhere, with host checks that turn features off where they'd break:

    ```text
    {{- if eq .chezmoi.hostname "<old-cluster>" }}
    ZSH_THEME="gnzh"                    # simple theme, no fancy prompt
    TERM=xterm
    zstyle ':omz:update' mode disabled  # newer oh-my-zsh breaks on zsh 5.0
    {{- else }}
    zstyle ':omz:update' mode auto
    {{- end }}
    ```

    The same blocks load the cluster's conda and `module load` lines, and skip tools that aren't installed (fzf, carapace).

- **No `chsh`? Hand over from bash.** On clusters where accounts come from NIS/LDAP, you can't change your login shell. A few lines at the top of `.bashrc` start zsh instead:

    ```bash
    # Skipped when the parent is zsh, so typing `bash` still gives you bash.
    if [ -x /usr/bin/zsh ] && [ "$(ps -o comm= -p $PPID 2>/dev/null)" != zsh ]; then
      exec /usr/bin/zsh -l
    fi
    ```

- **Build into `~/.local`.** For a tool the cluster lacks, the usual `./configure --prefix=$HOME/.local && make && make install` works. Expect to build its dependencies (ncurses, libevent and so on) the same way first. This is what made tmux a pain (see [tmux & Herdr](../terminal/tmux-herdr.md)). For anything bigger than one tool, a [container](containers.md) is less work.
