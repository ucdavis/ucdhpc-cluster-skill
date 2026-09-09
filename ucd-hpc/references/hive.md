# Hive

Centrally managed campus cluster (HPC@UCD), open to all UC Davis staff, faculty, and graduate
students. HPC2's hardware was folded into Hive in July 2026; Peloton's legacy `waltz` storage is
mounted read-only. Docs: <https://docs.hpc.ucdavis.edu/hive/>.

- **Login**: `ssh USER@hive.hpc.ucdavis.edu` (lands on `login1`/`login2`); SSH key **or** campus
  passphrase. Open OnDemand: <https://ondemand.hive.hpc.ucdavis.edu>.
- **Accounts**: HiPPO <https://hippo.ucdavis.edu/Hive>. Free tier = sponsor
  `HPC@UCD Sponsored Public Access (publicgrp)`. PI groups buy CPUs/GPUs/TB at
  <https://hpc.ucdavis.edu/rates#hive>.
- **Cluster name file**: `/opt/hpccf/etc/cluster_name.conf` → `hive`. Slurm 26.05.
- Every login prints "Slurm resources available to you" (`/opt/hpccf/bin/slurm-show-resources.py`).

## Partitions (verified 2026-09)

| Partition | Max time | Default mem/CPU | Preempt | Contents / who |
|-----------|----------|-----------------|---------|----------------|
| `high` | 30 d | 4000 M | none | ~150 CPU nodes + 12 general A6000/Blackwell GPUs; purchased shares and the free tier |
| `low` | 7 d | 4000 M | `REQUEUE`, 130 s grace | **all** nodes incl. every GPU (86 GPUs) when idle; free for everyone |
| `gpu-a100` | 30 d | 8000 M | none | 9 nodes, 44 A100 (80 GB SXM/PCIe); owning groups |
| `gpu-a6000` | 30 d | 16000 M | none | 1 node, 4 A6000; owning group |
| `gpu-6000-blackwell` | 30 d | 16000 M | none | 4 nodes, 14 RTX PRO 6000 Blackwell; owning groups |
| `gpu-a100-40gb` | 30 d | 2000 M | none | 1 node, 4 A100 40 GB |
| `gpu-5000-ada`, `gpu-l40s` | 30 d | 2000 M | none | 1 node each, 4 GPUs |
| `gpu-h100` | 30 d | | none | H100 nodes for a specific group (also reachable via `low`) |
| `hpccf`, `burnin` | | | | staff/testing; ignore |

`DefaultTime` is unset everywhere: **always pass `--time`**.

## Free tier (`publicgrp`)

| Partition | Per job | Group pool (all free-tier users together) |
|-----------|---------|-------------------------------------------|
| `high` | 8 CPUs, 128 GB, 1 GPU (A6000), 30 days | 128 CPUs, 2000 GB, 5 GPUs |
| `low` | no per-job caps, 7 days, preemptible | everything idle |

Exceeding a per-job cap fails at submit time with `QOSMaxCpuPerJobLimit`,
`QOSMaxMemoryPerJob`, `QOSMaxGRESPerJob`, or `Requested time limit is invalid`. A full pool shows
as `(QOSGrpCpuLimit)`/`(QOSGrpMemLimit)`/`(QOSGrpGRES)` in `squeue`. Free-tier users who need
more than 8 CPUs or 128 GB use `--partition=low` and accept preemption, or join a PI group.

```bash
#SBATCH --account=publicgrp
#SBATCH --partition=high      # ≤8 CPUs, ≤128G, ≤1 GPU, ≤30-00
#SBATCH --gpus=1              # optional; A6000 in high
```

## Hardware and GPU selection

Node shapes: 80× 64 CPU/256 GB (zen2), 24× 128/512 GB (zen2), 15× 224/1.5 TB (zen4),
11× 128/2 TB (zen3), plus GPU nodes. CPU features: `zen`, `zen2`, `zen3`, `zen4`, `zen5`,
`icelake`; also `gpu`, `ib`, `nvlink`, and GPU model features.

GPU type strings for `--gpus=TYPE:N` (as of 2026-09; recheck with
`sinfo -p low -o "%G|%D" --noheader | column -s'|' -t`):

| `--gpus=` | GPU | Where |
|-----------|-----|-------|
| `a6000:1` | RTX A6000 48 GB | `high`, `low`, `gpu-a6000` |
| `6000_blackwell:1` | RTX PRO 6000 Blackwell 96 GB | `high` (some), `low`, `gpu-6000-blackwell` |
| `a100:1` | A100 80 GB (some nodes typed `nvidia_a100-sxm4-80gb` / `nvidia_a100_80gb_pcie`) | `low`, `gpu-a100` |
| `l40s:1`, `5000_ada:1` | L40S / RTX 5000 Ada | `low`, own partitions |
| `1` (no type) | any idle GPU | any GPU-bearing partition |

Some nodes carry long vendor type names (`nvidia_a100-sxm4-80gb`, `nvidia_l40s`,
`nvidia_rtx_5000_ada_generation`); `--gpus=a100:1` only matches nodes typed exactly `a100`, so
for the widest match use `--gpus=1` plus `--partition=gpu-a100`, or a `--constraint=gpu:a100`
feature. Verify with `sbatch --test-only`.

## Hive-specific rules

- **Quobyte everywhere**: group storage is `/quobyte/<pi>grp`. Unique `--output`/`--error`
  per writer (`%j`, `%A_%a`, `%N`) is mandatory; violations knock nodes offline and lock accounts.
- **Do not use `--exclusive`**: it never schedules and shows a bogus `(QOSGrpCpuLimit)`.
- **MPI**: `--ntasks=N`, `--nodes=1-4`, `--constraint='(zen2|zen3|zen4)'`,
  `--distribution=block,pack`, `--switches=1@1-00`; output with `%N`. MPI-IO workloads use
  `/nfs/hive/scratch-mpi-io/<jobid>/` after joining `mpi-io-grp` via HiPPO.
- `high` normally starts within minutes; if not, the group's pool is full (`QOSGrp*`), the
  request needs a big contiguous chunk, or a maintenance reservation or DB backup is running.

## Storage

- Home `/home/$USER`: 20 GB, NFS, backed up nightly (7 daily/5 weekly/3 monthly snapshots).
  Restore with `/quobyte/backups/bin/restore.sh` from `~`.
- Group `/quobyte/<pi>grp`: purchased capacity; backups only for `BACKED-UP/` and only if
  bought. `qinfo quota /quobyte/<pi>grp` for quota. Globus writes land in `globus-write/<login>/`.
- Per-job `/tmp` and `/scratch` (`$TMPDIR=/tmp`, ~1.6 TB local NVMe/SSD), wiped at job end.
- `/nfs/hive/scratch`: shared network scratch, persists between jobs, not purged automatically,
  not backed up, user must clean up. `/scratch/nfs` no longer exists.
- `/nfs/peloton/waltz`: legacy Peloton storage (since 2025-09). LSSC0 (Genome Center) data was
  migrated into `/quobyte/<pi>grp`; the transfer node `transfer.hive.hpc.ucdavis.edu` exists
  for remaining home-directory pulls.

## Software

Full CVMFS tree (`module avail`: core, lang, general sections; `conda/*` and `R/*` environments;
`matlab/r2024a`, `fsl`, `phenix`). Defaults loaded at login: `slurm`, `openmpi/5.0.5`.
`module load dev` (pre-release) and `module load zen2` (AMD-optimized) add trees. Apptainer,
Open OnDemand (JupyterLab, RStudio, VS Code, desktop). See `software.md`.

## Example: free-tier GPU training job

```bash
#!/bin/bash
#SBATCH --job-name=train
#SBATCH --account=publicgrp
#SBATCH --partition=high
#SBATCH --time=1-00:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=64G
#SBATCH --gpus=1
#SBATCH --output=train-%j.out
#SBATCH --error=train-%j.err

source /etc/profile.d/modules.sh
module load conda
conda activate /quobyte/PIGRP/$USER/envs/torch     # or: module load conda/pytorch/2.9.1

cd "$SLURM_SUBMIT_DIR"
nvidia-smi --query-gpu=name,memory.total --format=csv
python train.py --workers "$SLURM_CPUS_PER_TASK"
```
