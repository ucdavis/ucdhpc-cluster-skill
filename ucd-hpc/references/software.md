# Software on HPC@UCD

## The landscape

| Mechanism | Farm & Hive | Franklin |
|-----------|-------------|----------|
| Spack-built modules | `/cvmfs/hpc.ucdavis.edu/sw/spack/modulefiles/main/.../{core,lang,general}` (identical on both) | `/share/apps/22.04/modulefiles/spack/{core,software}` |
| Central conda environments | `/cvmfs/hpc.ucdavis.edu/sw/conda/modulefiles` (`conda/NAME/VERSION`, `R/VERSION`) | `/share/apps/franklin/modulefiles` (`conda/NAME/VERSION`) |
| Manually installed | `/cvmfs/hpc.ucdavis.edu/sw/modulefiles` (matlab, fsl, phenix) | same CVMFS tree is mounted |
| Containers | `module load apptainer` (also `/usr/bin/apptainer`) | same |
| Web apps | Open OnDemand: JupyterLab, RStudio Server, VS Code, desktop | same |

Everything is Environment Modules (`envmod` 5.x), even on Franklin (its docs say lmod; the
commands are identical). Release notes for the CVMFS tree:
`/cvmfs/hpc.ucdavis.edu/sw/RELEASES.md`. Check before asserting anything exists:

```bash
module avail NAME          # exact or prefix match
module search NAME         # searches descriptions too
module -t avail conda/     # central conda environments
module whatis NAME         # one-line description
module show NAME/VERSION   # what it changes (PATH, LD_LIBRARY_PATH, $NAME_ROOT ...)
```

## Module naming

- `name/version` with `(default)` marking what a bare `module load name` gives you.
- **Compiler suffix** (CVMFS): `hdf5/1.14.5%oneapi@2025.0.0` was built with Intel oneAPI; the
  unsuffixed module is the GCC 13.2 build. Mixing compilers within one job usually breaks links.
- **MPI variants**: `hpl/2.3+openmpi-5.0.5` vs `+mpich-4.3.2`; load the matching MPI.
- **Franklin arch variants**: `ctffind/4.1.14+amd` (default, AMD nodes), `+intel` (only for
  the Intel RTX 2080 Ti nodes), unsuffixed = generic x86-64-v3. `+amd` binaries fail on Intel
  nodes and vice versa.
- **Nested names**: `relion/gpu/4.0.1+amd`, `conda/pytorch/2.9.1`: load at least
  `name/variant` (`module load relion/gpu`); `module load relion` alone fails.
- **Extra trees** (CVMFS): `module load dev` exposes pre-release software (may change or break);
  `module load zen2` exposes AMD-optimized builds (AOCC, amdblis/libflame, openmpi) for
  zen2 nodes; pair with `--constraint=zen2`.
- Every module sets `$<NAME>_ROOT` to its install prefix, useful for `-I`/`-L` flags.

## Modules in job scripts

The `module` command is a shell function defined by `/etc/profile.d/modules.sh`. Batch scripts
submitted from zsh, cron, OnDemand, or with a scrubbed environment do not have it. Put this at
the top of scripts:

```bash
source /etc/profile.d/modules.sh
module load bwa/0.7.17 samtools/1.19.2
```

`#!/bin/bash -l` also works (sources `/etc/profile`). On Farm/Hive the profile purges modules
then loads `slurm` and `openmpi`; `module purge` in a script therefore removes `srun` from PATH,
so follow it with `module load slurm` when the script uses Slurm commands.

## Conda and Python

- `module load conda` (loads `conda/base/latest`, a miniforge install with `mamba`, and does
  the `conda init` shell hook for you). Never run `conda init` or install Miniconda/Anaconda
  in `$HOME`; a private install clashes with the central one, breaks RStudio/Jupyter in
  OnDemand (`ERROR: CONDA_EXE is currently defined`), and is unsupported. To migrate, delete the
  `# >>> conda initialize >>>` block from `~/.bashrc`/`~/.bash_profile`/`~/.zshrc`, log out and in.
- Environments go on **group storage**, not the 20 GB home:

  ```bash
  module load conda
  mamba create --prefix /quobyte/PIGRP/$USER/envs/myenv python=3.12 numpy pandas   # Hive
  mamba create --prefix /group/PIGRP/$USER/envs/myenv ...                            # Farm/Franklin
  conda activate /quobyte/PIGRP/$USER/envs/myenv
  ```

  Register the directory so names work: `conda config --add envs_dirs /quobyte/PIGRP/$USER/envs`.
  Move the package cache too: `conda config --add pkgs_dirs /quobyte/PIGRP/$USER/conda-pkgs`,
  and reclaim space with `conda clean --all`.
- Big solves (tensorflow, bioinformatics stacks) exceed login-node limits; run them in
  `srun --account=A --partition=P --time=1:00:00 --cpus-per-task=2 --mem=16G --pty bash -l`.
- `mamba` prints harmless lock-file warnings on shared caches, and on Franklin a
  `MAMBA_ROOT_PREFIX` warning; both can be ignored.
- Central environments: `module load conda/pytorch/2.9.1` = `module load conda` +
  `conda activate pytorch-2.9.1`. `conda env list` shows them. Unloading `conda` deactivates
  everything.
- `pip install` inside an activated conda env is fine; `pip install --user` lands in
  `~/.local` and eats home quota. Plain `module load python/3.11.9` + `python -m venv` is an
  alternative for pure-Python work.
- In job scripts: `source /etc/profile.d/modules.sh; module load conda; conda activate ENV`.
  (`conda activate` works after the module; `source activate` is obsolete.)

## R and RStudio

`module load R/4.4.2` (conda-based R on CVMFS; `R/4.3.3` also present). RStudio Server runs
through Open OnDemand; pick the R version in the form. User libraries default to
`~/R/x86_64-pc-linux-gnu-library/4.4`; point `R_LIBS_USER` at group storage in `~/.Renviron` if
home fills up. Session problems (version-change errors, `install.packages()` failing) are fixed
by clearing `~/.RData` and `~/.local/share/rstudio*` and disabling workspace restore.

## Jupyter, VS Code, desktops

Open OnDemand at `https://ondemand.<cluster>.hpc.ucdavis.edu` launches JupyterLab (choose a
conda env; the name must exist in `conda env list`), RStudio Server, VS Code Server, and an
XFCE desktop as Slurm jobs; the form's account/partition/CPUs/memory/hours map directly to
`sbatch` flags and the same QOS limits apply. `module load conda/jupyterlab/4` exists for
manual `jupyter lab --no-browser --ip=$(hostname)` inside an `srun` session with an SSH tunnel.

## Apptainer (Singularity)

```bash
module load apptainer
export APPTAINER_CACHEDIR=/quobyte/PIGRP/$USER/apptainer-cache   # keeps ~/.apptainer small
apptainer build tf.sif docker://tensorflow/tensorflow:latest-gpu   # once, ideally in an srun session
apptainer exec --nv tf.sif python train.py                          # --nv exposes the granted GPU(s)
apptainer shell tf.sif
export APPTAINER_BIND=/quobyte/PIGRP,/nfs/hive/scratch               # extra paths inside the container
```

Home and the current directory are bound automatically. Docker itself is not available (no
root); any OCI image works through Apptainer. Build `.sif` files once and reuse them, not in
every job.

## Compilers, MPI, CUDA

Defaults loaded at login (Farm/Hive): `slurm/26-05-4-1`, `openmpi/5.0.5`. Available:
`gcc/13.2.0` (default), `gcc/11.4.0`, `gcc/9.5.0`, `aocc/5.0.0`, `clang/19.1.3`,
`oneapi/2025.0.0`, `nvhpc/24.9`, `cuda/13.3.0` (default) plus 12.6, 12.3, 11.x; `mpich/4.3.2`;
`intel-oneapi-mkl`, `amdfftw`, `fftw`, `hdf5`, `netcdf-*`, `boost`, `eigen`, `cmake/3.28.1`.
Franklin: `gcc/13.2.0`, `aocc/4.1.0`, `intel-oneapi-compilers/2023.2.1`, `cuda/11.7.1`,
`openmpi/4.1.5{,+amd,+intel}`. CUDA modules are for compiling; GPU nodes carry the driver.
Build on a compute node of the target architecture when using `-march=native`.

## Cluster-specific software notes

- **Franklin** (cryo-EM/structural biology): `relion/{cpu,gpu}/VERSION+arch` with `relion-helper`
  for switching versions inside a project (GUI needs `ssh -Y`); `alphafold/2.3.2` with the
  `alphafold-wrapped` script that fills in `--*_database_path` from `$ALPHAFOLD_DB_ROOT`
  (`/share/databases/alphafold`); conda envs `cryolo`, `topaz`, `cryodrgn`, `deepemhancer`,
  `pyem`, `warp`, `scipion`; databases in `/share/databases/{alphafold,blast,relion,cryolo,...}`.
- **Hive/Farm** (CVMFS general tree): broad bioinformatics (bwa, bwa-mem2, bowtie2, star,
  salmon, samtools, bcftools, gatk, blast-plus, diamond, kraken2, spades, canu, busco, ...),
  physics/chemistry (gromacs, lammps, quantum-espresso, amber/26+cuda, gaussian/16, geant4,
  root), climate/geo (cdo, nco, gdal, gmt), `alphafold/2.3.2`, `relion/5.0.0`, `matlab/r2024a`,
  `julia/1.12.1`, `rust`, `go`, `rstudio`, `code-server`.

## Requesting software

Read <https://hpc.ucdavis.edu/software-installation-policy> and submit the request form linked
there. Packages already in Spack (<https://packages.spack.io>) or conda-forge/bioconda are
approved fastest. Meanwhile users can self-serve with conda on group storage or Apptainer.
