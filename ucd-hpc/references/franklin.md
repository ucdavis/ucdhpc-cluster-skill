# Franklin

College of Biological Sciences cluster (Center for Neuroscience, MMG, MCB, cryo-EM facility,
approved collaborators). Genomics, proteomics, cryo-EM structure determination, computational
neuroscience. Docs: <https://docs.hpc.ucdavis.edu/franklin/>.

- **Login**: `ssh USER@franklin.hpc.ucdavis.edu`, **SSH key only**. Add `-Y` for X11 (Relion
  GUI). Open OnDemand: <https://ondemand.franklin.hpc.ucdavis.edu>.
- **Accounts**: HiPPO <https://hippo.ucdavis.edu/Franklin>. **No free/public tier**; every user
  is sponsored by a PI or department. Access to `low` comes through department QOS such as
  `mcbdept-low-qos`.
- **Cluster name file**: `/opt/hpccf/etc/cluster_name.conf` → `franklin`. Slurm 26.05.
- Login prints the Slurm resources table (`/opt/hpccf/bin/slurm-show-resources.py`).

## Partitions (verified 2026-09)

| Partition | Max time | Default mem/CPU | Preempt | Nodes |
|-----------|----------|-----------------|---------|-------|
| `low` (**default partition**) | 14 d | 2000 M | `REQUEUE`, 60 s grace | all 16 nodes: 7 CPU + 9 GPU nodes when idle |
| `high` | 60 d | 2000 M | none | 7 CPU nodes (`c-7-70`, `c-8-*`): 256 logical CPUs, 1 TB each |
| `<group>-gpu`: `jawdatgrp-gpu`, `jalettsgrp-gpu`, `mmgdept-gpu`, `mcbdept-gpu`, `cashjngrp-gpu`, `mmaldogrp-gpu`, `cnsdept-gpu`, `ajfishergrp-gpu` | unlimited | (node default) | none | the owning group's GPU node(s), 7–8 GPUs each |

`low` is the default partition, so a script that omits `--partition` lands on preemptible
nodes; say so explicitly. `DefaultTime` unset: always pass `--time`.

## GPUs

GPUs are **untyped** (`Gres=gpu:8`), so `--gpus=N` only; there is no `--gpus=TYPE:N`. Choose the
node class by CPU feature:

| Constraint | Nodes | CPUs / RAM | GPUs |
|------------|-------|------------|------|
| `--constraint=amd` | `gpu-7-42/50/54/58`, `gpu-9-58/66` | 256 / 1 TB | 8× RTX A4000 / A5000 / 6000 Ada (varies by owner) |
| `--constraint=intel` | `gpu-7-62`, `gpu-9-18`, `gpu-9-26` | 40 / 384 GB | 7–8× RTX 2080 Ti (Al-Bassam lab) |

Group members use their `<group>-gpu` partition (no time limit); others reach idle GPUs via
`low`. Use `+amd` module variants on AMD nodes and `+intel` on Intel nodes.

```bash
#SBATCH --account=PIGRP
#SBATCH --partition=PIGRP-gpu     # or low (preemptible) for non-owners
#SBATCH --gpus=2
#SBATCH --constraint=amd
#SBATCH --cpus-per-task=16
#SBATCH --mem=64G
#SBATCH --time=3-00:00:00
```

## Hardware

7 CPU nodes: 2× AMD EPYC, 128 physical / 256 logical cores, 1 TB RAM. 9 GPU nodes, 72 GPUs
total. Features: `amd,cpu`, `amd,gpu`, `intel,gpu`. ~3 PB of ZFS storage.

## Storage

- Home `/home/$USER`: 20 GB, no backup.
- Lab storage `/group/<pi>grp` (autofs; appears only after you access it by name; `id -Gn`
  shows your group). Purchased per lab; **no backups**.
- Per-job `$TMPDIR=/tmp` (~0.9 TB local disk); there is **no `/scratch`** on Franklin.
- `/share/databases`: `alphafold` (uniref30/90, uniprot, mgnify, pdb; `$ALPHAFOLD_DB_ROOT`),
  `blast` (nr, refseq_protein, swissprot, pdbaa/pdbnt, 16S/18S/28S, nt_prok, nt_viruses, mito,
  taxdb; weekly updates), `relion`, `cryolo`, `deepemhancer`, `medic`. Request more by ticket.
- Cryo-EM facility Windows machines (Warp, K3) can mount lab storage via Samba
  (`\\172.16.1.4\<pi>grp`) with a password provisioned into `~/samba-password` on request.
- Globus: `UC Davis Franklin home`; PI shares on request, writable at
  `/group/<pi>grp/globus-write/<login>/`.

## Software

Environment Modules 5.5 (docs call it lmod; commands are the same). Trees:
`/share/apps/22.04/modulefiles/spack/core` (gcc/7.5.0, 11.4.0, 13.2.0; aocc/4.1.0;
intel-oneapi-compilers/2023.2.1; cuda/8.0.61, 11.2.2, 11.7.1; openmpi/4.1.5{,+amd,+intel};
slurm), `/share/apps/22.04/modulefiles/spack/software` (bioinformatics, cryo-EM, libraries;
`+amd`/`+intel` variants), `/share/apps/franklin/modulefiles` (conda environments), plus the
CVMFS `sw/modulefiles` (matlab, fsl, phenix). Defaults loaded: `slurm`, `openmpi/default`, `ucx`.

- **Relion**: `relion/cpu/4.0.1+amd`, `relion/gpu/4.0.1+amd`, `relion/gpu/4.0.1+intel`, plus
  3.1.3 and 5.0-beta variants. `module load relion/gpu` gives the default. GUI via `ssh -Y`,
  submits Slurm jobs itself with pre-filled paths. Use `relion-helper` (`conda/relion-helper`)
  when switching Relion versions within a project.
- **AlphaFold**: `module load alphafold/2.3.2`; run `alphafold-wrapped --output_dir=... --fasta_paths=... --max_template_date=... --use_gpu_relax=true`
  (fills database paths from `$ALPHAFOLD_DB_ROOT`). Needs a GPU node.
- **Cryo-EM conda envs**: `conda/cryolo/1.9.3-cuda-11`, `conda/topaz/0.2.5`,
  `conda/cryodrgn`, `conda/deepemhancer/0.16`, `conda/pyem`, `conda/warp`,
  `conda/scipion/3.0.12`, `conda/sphire`, `conda/gpu-isac`, `conda/spisonet`, `conda/medic`.
  Other cryo-EM modules: `motioncor2/1.5.0`, `ctffind/4.1.14+amd|+intel`, `gctf`.
- **Bioinformatics**: abyss, angsd, minimap2, mmseqs2, mothur, mummer4, muscle, ... see
  `module avail`. `module load conda` for the central miniforge (`mamba` warns about
  `MAMBA_ROOT_PREFIX`; harmless).
