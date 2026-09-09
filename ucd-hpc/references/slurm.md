# Slurm on HPC@UCD clusters

Shared behavior for Farm, Franklin, and Hive (all run Slurm 26.x with `select/cons_tres`,
`CR_CPU_MEMORY`, cgroup enforcement, backfill scheduling, and partition-priority preemption).
Cluster-specific partition tables live in `hive.md`, `farm.md`, `franklin.md`.

## Where the commands live

All three clusters get Slurm from the same Spack view on CVMFS, at this exact path:

```
/cvmfs/hpc.ucdavis.edu/sw/spack/environments/core/view/generic/slurm/bin/
```

It holds `sinfo`, `squeue`, `sbatch`, `srun`, `salloc`, `sacct`, `sacctmgr`, `scontrol`,
`scancel`, `sstat`, `sshare`, `sprio`, `sreport`, `sbcast`, and the rest. Verified identical on
Hive, Farm, and Franklin (2026-09). Franklin also has an older Slurm 23.02 tree under
`/share/apps/22.04/spack/opt/core/.../slurm-23-02-6-1/bin`; ignore it, the CVMFS view is what the
cluster runs.

Interactive logins load the `slurm` module from `/etc/profile.d/modules.sh`, so the commands are
on `PATH` already. A non-interactive shell (`ssh host 'sinfo'`, a script, a cron job, a job step
after `module purge`) may not have them, giving `sinfo: command not found`. Two fixes, in order of
preference:

```bash
source /etc/profile.d/modules.sh          # loads the slurm module; also defines `module`
export PATH=/cvmfs/hpc.ucdavis.edu/sw/spack/environments/core/view/generic/slurm/bin:$PATH
```

Do **not** search the file system for these binaries. A `find` over `/cvmfs` or a share is slow,
hammers the metadata servers, and is never necessary — the path above is fixed, and
`command -v sinfo` answers the question in a millisecond. See *Login-node discipline* in
`SKILL.md`. `module avail slurm` lists the available versions (`26-05-4-1` is current on all
three); `module load slurm` gives the default.

## Accounts, associations, QOS

- Access is granted through **associations**: user + account + partition + QOS. Accounts are
  usually a PI group named `<pi-login>grp` (e.g. `jrigrp`), sometimes a department or
  `publicgrp` (free tier).
- See yours: `sacctmgr show assoc user=$USER format=account%20,partition%20,qos%40`.
  On all three clusters `/opt/hpccf/bin/slurm-show-resources.py` prints the same as a table with
  the group pool ("Account Limits") and per-job caps ("Limits Per Job"); it also runs at login.
  The account marked `⁺` is the default; everyone else needs `--account=...`.
- Limits are on the QOS, not the partition: `sacctmgr show qos NAME format=name,maxtres,grptres,maxtrespu,maxwall -P`.
  `MaxTRES` = per job, `GrpTRES` = the whole account's pool, `MaxTRESPU` = per user. All QOS use
  `DenyOnLimit`, so a request over a per-job cap is rejected at submit time rather than queued.
- Accounts you were just added to in HiPPO appear within about an hour and always need
  `--account=NEWGRP` explicitly.

## Partition types

| Kind | Typical name | Who | Preemption | Notes |
|------|--------------|-----|------------|-------|
| Priority | `high`, `bmh`, `*-h`, `<group>-gpu` | owners of purchased resources (and Hive free tier) | none | starts within minutes if the group's pool has room |
| Scavenger | `low`, `bml` | everyone with an association (free) | `REQUEUE` | runs on idle owned nodes; killed and requeued when owners need them; 7 d cap (14 d Franklin) |
| GPU | `gpu-a100`, `gpu-a6000`, `gpu-6000-blackwell`, `gpuh`, ... | owning groups | none | `low` also exposes idle GPUs on Farm and Hive |

Preempted jobs go back to PENDING and restart from the top of the script, so `low` suits short,
restartable, or checkpointed work. The grace period between the signal and the kill is short
(Hive 130 s, Franklin 60 s, Farm 0). Trap `SIGTERM` if the program can flush state.

## Requesting resources

### CPUs, tasks, nodes

- `--cpus-per-task/-c N`: CPUs kept together on one node. Use for threads/OpenMP.
- `--ntasks/-n N`: separate tasks, may land on different nodes. Use for MPI or many processes.
  `srun` inside the script launches one copy per task.
- `--nodes/-N N` (or a range `1-4`): minimum (or range of) nodes. Default packing fills a node
  before spilling to the next.
- SMT is on: 2 hardware threads per core. Odd CPU requests round up; `-c 1` alone yields 2
  tasks unless you also pass `-n 1`.
- Read the granted count in the script: `$SLURM_CPUS_PER_TASK`, `$SLURM_NTASKS`,
  `$SLURM_JOB_NUM_NODES`. Programs that spawn more threads than CPUs slow down sharply.

### Memory

- `--mem=SIZE` per node (total for a single-node job), `--mem-per-cpu=SIZE`, or
  `--mem-per-gpu=SIZE`; mutually exclusive. Units `K|M|G|T`, default megabytes: `--mem=32`
  means 32 MB.
- No memory request means `DefMemPerCPU × CPUs` for that partition (see cluster files).
- Exceeding the request kills the step: `slurmstepd: error: Detected N oom-kill event(s)`.
  Compare `MaxRSS` from `sacct` with `ReqMem` to right-size.

### GPUs

- `--gpus=N` or `--gpus=TYPE:N` (`--gres=gpu:TYPE:N` is equivalent). Also `--gpus-per-task`,
  `--gpus-per-node`, `--cpus-per-gpu`, `--mem-per-gpu`.
- Types per partition: `sinfo -p PART -o "%G|%D" --noheader | column -s'|' -t`. Hive types
  include `a100`, `a6000`, `6000_blackwell`, `l40s`, `5000_ada`; Farm `a100`, `h100`,
  `6000_ada`, `a5500`, `v100`, `titan`; Franklin has untyped GPUs (`gpu:8`), select by
  `--constraint=amd|intel`.
- Inside the job `$CUDA_VISIBLE_DEVICES` is set; `nvidia-smi` shows only the granted GPUs.
- Ask for CPUs and RAM alongside the GPU; GPU partitions default to more memory per CPU
  precisely because GPU jobs need it.

### Time

`--time=D-HH:MM:SS` (`--time=90` is 90 minutes, `--time=2:00:00` two hours, `1-00` one day).
Over the partition's `MaxTime` the submission fails with
`Requested time limit is invalid (missing or exceeds some limit)`. Before a maintenance window,
jobs whose limit reaches into the window sit `PENDING (ReqNodeNotAvail, Reserved for maintenance)`
until it ends; shorten `--time` to start sooner.

### Node features / constraints

`--constraint=FEATURE` with `|`, `&`, and parentheses, e.g. `--constraint='(zen3|zen4)'` on
Hive or `--constraint=amd` on Franklin. List features: `sinfo -N -o "%N %f" --noheader | sort -u`.
Each constraint narrows the eligible nodes and lengthens the wait.

## Job scripts

```bash
#!/bin/bash
#SBATCH --job-name=NAME
#SBATCH --account=ACCOUNT
#SBATCH --partition=PARTITION
#SBATCH --time=D-HH:MM:SS
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=N
#SBATCH --mem=SIZE             # or --mem-per-cpu
#SBATCH --output=NAME-%j.out
#SBATCH --error=NAME-%j.err
# optional: --gpus=1  --array=1-100%10  --mail-type=END,FAIL --mail-user=you@ucdavis.edu
# optional: --dependency=afterok:JOBID  --constraint=...  --nodes=1-4

source /etc/profile.d/modules.sh
module load PKG/VERSION

cd "$SLURM_SUBMIT_DIR"
# work
```

- `#SBATCH` lines must precede the first command. Command-line flags to `sbatch` override them.
- Filename patterns: `%j` job id, `%x` job name, `%u` user, `%N` first node, `%A` array job
  id, `%a` array task id. Default output is `slurm-%j.out` in the submit directory.
- `source /etc/profile.d/modules.sh` (or `#!/bin/bash -l`) makes `module` available regardless
  of the submitting shell. `module purge` also unloads `slurm`; `module load slurm` afterwards
  if the script calls `srun`, `sacct`, or `scontrol`.
- Useful variables: `SLURM_JOB_ID`, `SLURM_ARRAY_JOB_ID`, `SLURM_ARRAY_TASK_ID`,
  `SLURM_CPUS_PER_TASK`, `SLURM_NTASKS`, `SLURM_MEM_PER_NODE` (MB), `SLURM_SUBMIT_DIR`,
  `SLURM_JOB_NODELIST`, `TMPDIR` (per-job local scratch, see `storage.md`).
- Validate without submitting: `sbatch --test-only script.sh`. A rejection prints the exact
  scheduler error, which is the point of the command. Its success line ("Job N to start at HH:MM
  ... on nodes X") is **not a wait-time estimate** — the same timestamp comes back regardless of
  how many CPUs you ask for, and it can be hours ahead while the partition is largely idle. For
  queue health use `sinfo -p PART -h -o "%C"` and `squeue -t PD -h -o "%r" | sort | uniq -c`
  (see `debugging.md`, *Is the queue actually busy?*).

## Job arrays

```bash
#SBATCH --array=1-300%20          # 300 tasks, at most 20 running at once
#SBATCH --output=NAME-%A_%a.out
#SBATCH --error=NAME-%A_%a.err

# map the task index to an input, e.g. line N of a manifest
SAMPLE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" samples.txt)
# or: FILES=(data/*.fastq.gz); F=${FILES[$SLURM_ARRAY_TASK_ID-1]}
```

- Resource flags apply per task. `MaxArraySize` is 50000.
- The `%N` throttle shows as `(JobArrayTaskLimit)` in `squeue`; it is intended, not an error.
- Every task needs its own output file and its own result paths (Quobyte rule).
- Cancel one task `scancel JOBID_5`, a range `scancel JOBID_[10-20]`, or all `scancel JOBID`.

## Dependencies and chaining

`--dependency=afterok:ID` (start after success), `afterany:ID`, `afternotok:ID`,
`singleton` (one job with this name per user at a time). Capture ids with
`JOB=$(sbatch --parsable step1.sh)`. A parent that fails leaves children
`(DependencyNeverSatisfied)` forever; `scancel` them.

## Running one command in a job and waiting for it

Anything that does real work belongs off the login node (see *Login-node discipline* in
`SKILL.md`). The two blocking patterns, both of which return only when the work is done:

```bash
# one command, output straight back to your terminal; blocks until it finishes
srun --account=A --partition=P --time=15 --cpus-per-task=2 --mem=4G \
     find /quobyte/PIGRP -maxdepth 6 -name '*.bam' -printf '%s\t%p\n'

# a script, with logs on disk; --wait blocks until the job leaves the queue
sbatch --wait --account=A --partition=P --time=2:00:00 --mem=8G job.sh; echo "exit=$?"
```

`srun` suits a single short probe (a listing, a `du`, a version check, a quick conversion) and is
the right tool when you would otherwise be tempted to run it on the login node. `sbatch --wait`
suits anything long, and its exit status reflects the job. Without `--wait`, capture the id and
poll instead of busy-looping:

```bash
JOB=$(sbatch --parsable job.sh)
squeue -j "$JOB" -h -o "%T %R"          # STATE and, if pending, the reason
sacct -j "$JOB" -X --format=State,ExitCode,Elapsed,MaxRSS   # after it leaves the queue
```

A pending job is not a problem to route around: `PENDING (Priority)` or `(Resources)` means the
scheduler is doing its job. Check the reason once, tell the user what it means (see
`debugging.md`), and wait. Never mirror the work onto the login node "while we wait".

## Interactive work

- Shell on a compute node: `srun --account=A --partition=P --time=2:00:00 --cpus-per-task=4 --mem=16G --pty bash -l`.
  Add `--gpus=1` for a GPU. `salloc` with the same flags reserves resources and returns a
  shell on the login node from which `srun CMD` runs steps.
- Attach to a running batch job (top, nvidia-smi, inspect files):
  `srun --jobid=JOBID --overlap --pty bash -l`. You share the job's limits.
- Run inside `tmux` on the login node so a dropped SSH connection does not kill the session.
- Graphical apps: JupyterLab, RStudio Server, VS Code, and a desktop are one click away in
  Open OnDemand (`https://ondemand.<cluster>.hpc.ucdavis.edu`); those sessions are Slurm jobs
  too and take the same account/partition/resource choices.

## MPI

- One task per rank: `--ntasks=128`, not `--cpus-per-task=128`. Hybrid MPI+OpenMP:
  `--ntasks=16 --cpus-per-task=8`, then `export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK`.
- Launch with `srun ./prog` (uses PMIx) or `mpirun`. Match the MPI module used at build time
  (`openmpi/5.0.5` is loaded by default on Farm/Hive, `openmpi/default` on Franklin).
- Hive extras: `--constraint='(zen2|zen3|zen4)'` to stay on one CPU generation,
  `--nodes=1-4` to limit spread, `--distribution=block,pack`, and `--switches=1@1-00` to stay
  within one InfiniBand switch (waiting at most one day for it). Output files need `%N`.

## Monitoring and control

```bash
squeue --me                                            # my jobs, state, reason
squeue --me -o "%.10i %.12P %.20j %.3t %.12M %.12l %.5C %.10m %R"   # + time used/limit, CPUs, mem
scontrol show job JOBID                                # everything while the job is in memory
sacct -j JOBID --format=JobID,JobName%20,State,ExitCode,Elapsed,Timelimit,ReqMem,MaxRSS,AllocCPUS,TotalCPU,NodeList
sacct -u $USER -S 2026-09-01 -X --format=JobID,JobName%20,Partition,State,ExitCode,Elapsed  # history
scancel JOBID          scancel -u $USER          scancel --name=NAME --state=PENDING
scontrol hold JOBID    scontrol release JOBID    scontrol update JobId=ID TimeLimit=2-00   # shrink only, mostly
sshare -U              # fair-share standing per account
sinfo -s               # partitions with node counts allocated/idle/other/total
sinfo -p PART -N -l    # node-level CPUs, memory, features, state
```

`squeue` state codes: `PD` pending, `R` running, `CG` completing, `S` suspended. The
`NODELIST(REASON)` column explains pending jobs; see `debugging.md` for the code meanings.
