# Farm

College of Agricultural and Environmental Sciences (CA&ES) condo cluster; PIs from any college
can buy in. Docs: <https://docs.hpc.ucdavis.edu/farm/>. Support: `farm-hpc@ucdavis.edu`.

- **Login**: `ssh USER@farm.hpc.ucdavis.edu`, **SSH key only** (no passwords). Open OnDemand:
  <https://ondemand.farm.hpc.ucdavis.edu>.
- **Accounts**: HiPPO <https://hippo.ucdavis.edu/Farm>. Free tier = sponsor `CA&ES free tier`
  (`publicgrp`), which grants **`low` only**; `publicgrp` + `high` is rejected with
  `Invalid account or account/partition combination specified`. Hardware and storage are bought
  through HiPPO's product catalog (five-year lifecycle; rack fee for BYO equipment).
- **Cluster name file**: `/opt/hpccf/etc/cluster_name.conf` → `farm`. Slurm 26.05.
- Login prints the Slurm resources table and, if applicable, an **expired storage** warning
  (`farm-show-expired-storage.py`): storage past five years is unsupported and unbacked-up.

## Partitions (verified 2026-09)

| Partition | Max time | Default mem/CPU | Preempt | Nodes / purpose |
|-----------|----------|-----------------|---------|-----------------|
| `low` | 7 d | 2000 M | `REQUEUE` | ~98 nodes: all parallel + bigmem + every GPU node when idle; free for everyone |
| `high` | 150 d | 2000 M | none | ~66 parallel nodes (64 CPU/256 GB class and newer 128-CPU); purchased shares |
| `bml` | 7 d | 4096 M | `REQUEUE` | 22 bigmem nodes (`bm*`, up to 128 CPU/2 TB) when idle |
| `bmh` | 150 d | 4096 M | none | same bigmem nodes; purchased |
| `bgpu` | 150 d | 3072 M | none | 2 nodes: `gpu:a5500:4`, `gpu:v100:1`; purchased |
| `gpuh` | 7 d | 15360 M | none | 2 nodes with `gpu:titan` (3 and 5); purchased (high) |
| `gpum` | 7 d | 15360 M | none | same titan nodes, medium priority |
| `gpu-a100-h` | 34 d | 7168 M | none | 3 nodes, `gpu:a100` (4 and 8), `MaxNodes=1` |
| `gpu-6000_ada-h` | 30 d | 12000 M | none | 2 nodes, `gpu:6000_ada:4` |
| `gpu-h100-h` | 30 d | 4000 M | none | 1 node, `gpu:h100:4` |
| `burnin` | | | | staff |

Naming: `-h` = high priority for the owning group, `-m`/`gpum` = medium, `l` = low/scavenger.
`DefaultTime` unset: always pass `--time`.

## GPUs

Types for `--gpus=TYPE:N` (all also appear in `low` when idle):
`a100`, `h100`, `6000_ada`, `a5500`, `v100`, `titan`. Node features: `gpu:a100,nvlink:4`,
`gpu:h100,nvlink:4`, `gpu:6000ada`, `gpu:a5500,bgpu`, `gpu:v100,bgpu`, `gpu:titan`. Check
current inventory: `sinfo -p low -o "%G|%D" --noheader | column -s'|' -t`.

```bash
#SBATCH --account=publicgrp     # or PIGRP
#SBATCH --partition=low         # free tier's only GPU route; preemptible
#SBATCH --gpus=a100:1
#SBATCH --time=2-00:00:00
```

Owning groups use `gpu-a100-h`, `gpu-h100-h`, `gpu-6000_ada-h`, `bgpu`, `gpuh`.

## Hardware

Farm III: ~60 parallel nodes (64 CPU / 256 GB class, newer 128-CPU nodes), 27 bigmem nodes
(up to 128 CPU / 2 TB, `bm` feature), GPU nodes above; EDR/100 Gb and 200 Gb interconnect;
~15k CPU threads, ~66 TB RAM total. Node features: `cpu`, `bm`, `gpu`, plus GPU models.

## Storage

- Home `/home/$USER`: 20 GB, autofs NFS, **no backup**.
- Group shares: `/group/<pi>grp` (NFS file servers, e.g. `nas-6-0`) for most groups; newer
  purchases on Quobyte at `/quobyte/<pi>grp` (same one-writer-per-file rule as Hive;
  `qinfo quota PATH`). `ls /group` shows only mounted shares; `cd` into yours by name
  (`id -Gn`). Several `/group` shares are near 100% full; check `df -h /group/<pi>grp` before
  large writes.
- Per-job `/tmp` and `/scratch` (`$TMPDIR=/tmp`, ~1.8 TB local), deleted at job end (since the
  June 2026 maintenance).
- **No backups of anything on Farm.** Users arrange their own (Box via rclone, Globus, external).
- Globus: `UC Davis Farm home`; PI shares exported on request, writable at
  `/group/<pi>grp/globus-write/<login>/`.

## Software

Same CVMFS module tree and central conda as Hive (`module avail`, `module load conda`,
`conda/*`, `R/*`, `matlab/r2024a`). Defaults loaded: `slurm`, `openmpi/5.0.5`. Apptainer
available. Open OnDemand apps: JupyterLab, RStudio Server, VS Code, desktop. See `software.md`.

## Example: array over samples on a lab account

```bash
#!/bin/bash
#SBATCH --job-name=fastp
#SBATCH --account=PIGRP
#SBATCH --partition=high
#SBATCH --time=04:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=16G
#SBATCH --array=1-96%24
#SBATCH --output=logs/fastp-%A_%a.out
#SBATCH --error=logs/fastp-%A_%a.err

source /etc/profile.d/modules.sh
module load fastp/0.23.4

SAMPLE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" samples.txt)
IN=/group/PIGRP/reads; OUT=/group/PIGRP/$USER/trimmed
mkdir -p "$OUT" logs
fastp -w "$SLURM_CPUS_PER_TASK" \
  -i "$IN/${SAMPLE}_R1.fastq.gz" -I "$IN/${SAMPLE}_R2.fastq.gz" \
  -o "$OUT/${SAMPLE}_R1.fastq.gz" -O "$OUT/${SAMPLE}_R2.fastq.gz" \
  -j "$OUT/${SAMPLE}.json" -h "$OUT/${SAMPLE}.html"
```
