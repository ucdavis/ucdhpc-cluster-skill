---
name: ucd-hpc
description: Help people use the UC Davis HPC@UCD clusters Farm, Franklin, and Hive. Covers writing and fixing Slurm sbatch/srun job scripts with the right --account/--partition for that user (high vs preemptible low, GPU partitions, the free publicgrp tier and its per-job limits), requesting CPUs/memory/GPUs/time correctly, diagnosing pending or failed jobs (squeue reasons, sacct, OOM, TIMEOUT, preemption, QOS limits, "Invalid account or account/partition combination"), finding and loading software (environment modules on CVMFS, conda/mamba, Apptainer, R/RStudio, Jupyter, Open OnDemand), and storage and data transfer (20 GB home quota, /quobyte and /group PI shares, per-job scratch, Hive backups, rsync/Globus/scp). Use it whenever the user mentions farm, franklin, hive, hpc.ucdavis.edu, HPC@UCD, HPCCF, HiPPO, Quobyte, or Slurm/sbatch/srun/squeue/module on a UC Davis cluster, or is clearly working from a UC Davis login node, even if they never name the cluster or ask for a "job script".
---

# UC Davis HPC (Farm, Franklin, Hive)

You are helping a researcher, student, or lab staff member get work done on one of the three
HPC@UCD clusters. They may be new to Slurm, or an expert who just wants the right flags for
*this* cluster. The three clusters share one documentation set and one support team but differ
in partitions, limits, free-tier rules, storage paths, and software trees. Most user pain comes
from applying another cluster's assumptions, so the workflow below starts by pinning down which
cluster and which accounts the user actually has.

## Login-node discipline (applies to every command you run)

You are almost always working on a **login node**, which is a shared front end for hundreds of
people, not a workstation. A login node that bogs down blocks everyone's ability to submit and
monitor work, so the cost of a careless command is paid by the whole cluster, and staff will kill
processes that degrade the node even when they are inside the limits.

Ordinary users are capped, per user, at **2 CPUs (200%), 7.5% of RAM, 500 MB of swap and 512
processes**, plus 16,384 open files. Those numbers are set by
`/etc/security/systemd-user-limits.sh`, which PAM runs at every login; read that script for the
current values rather than trusting this list, and `/etc/security/limits.d/slurm.conf` for the
open-file limit.

**The one exemption: members of `hpccfgrp` (HPCCF staff) get no caps at all** — the same script
deletes any limits that were applied to them. So `id -nG | grep -qw hpccfgrp` tells you whether
the caps apply to the account you are running as.

This matters because of a trap: **never infer the policy from your own cgroup limits.** An agent
running as staff that inspects `systemctl show user-$(id -u).slice` or `/sys/fs/cgroup/...` sees
`infinity` everywhere and may conclude nothing would stop it from running work on the login node.
That conclusion is wrong twice over — the caps are real for the people you are usually helping,
and the policy (and staff killing offending processes) applies regardless of whether a cgroup
would have stopped you. Being exempt from enforcement is not permission; it just means you can do
more damage before anything intervenes.

- **Never run the analysis on the login node.** Compiles, data crunching, conda solves for large
  environments, `apptainer build`, indexing a genome, anything that reads a lot of data or runs
  for more than a few seconds of CPU: put it in `srun` (blocking, for interactive work) or
  `sbatch` (for anything long), and wait for the result. Waiting for a scheduled job is the
  correct behavior, not a delay to be engineered around.
- **Do not work around the resource limits.** Do not use `nice`/`ionice` to look polite while
  still consuming the node, do not split one job into many small processes or background them to
  stay under the per-process caps, do not run parallel transfer or `find` streams, and do not
  raise `ulimit`. The limits are the boundary of what belongs on a login node, not a quota to
  spend. If the work does not fit comfortably, it belongs in a job.
- **`find` is the most common way agents hurt a login node.** Every `find` walk pays metadata
  latency on CVMFS, Quobyte, or NFS, and an unbounded walk of `/quobyte`, `/group`, `/share`, or
  `/cvmfs` can hammer the file system for everyone. So:
  - Always pass `-maxdepth`. On a login node keep it at **2 or less**, and keep the starting
    point narrow (a project directory, not a share root).
  - For anything deeper or wider, run it in a job and wait:
    `srun --account=A --partition=P --time=15 --mem=2G find /quobyte/PIGRP -maxdepth 6 -name '*.bam'`
  - Prefer a targeted lookup over a walk: `module avail NAME` / `module search NAME` for
    software (never `find` for a binary), `sacct -j JOBID --format=WorkDir%200` to locate a job's
    logs, `ls` with globs when you already know the directory, `qinfo quota PATH` and `df -h` for
    space instead of `du` over a whole share. When you do need `du`, scope it
    (`du -sh ~/.conda ~/.cache`), not `du -sh /group/PIGRP`.
- **Never search the file system for Slurm commands.** They are on `PATH` on all three clusters
  once the `slurm` module is loaded, and always at this exact path:
  `/cvmfs/hpc.ucdavis.edu/sw/spack/environments/core/view/generic/slurm/bin/` (`sinfo`, `squeue`,
  `sbatch`, `srun`, `salloc`, `sacct`, `sacctmgr`, `scontrol`, `scancel`, `sstat`, `sshare`,
  `sprio`, `sreport`). If `sinfo: command not found` in a non-interactive shell, run
  `source /etc/profile.d/modules.sh` (which loads the `slurm` module) or prepend that directory
  to `PATH` — do not go looking for it.
- **Interactive sessions**: run them inside `tmux` on the login node so a dropped SSH connection
  does not kill the job, and use `srun --jobid=JOBID --overlap --pty bash -l` to look inside a
  running job rather than sshing to the node (which is not permitted).

## Workflow

### 1. Identify the cluster

On a login node, `cat /opt/hpccf/etc/cluster_name.conf` prints `farm`, `franklin`, or `hive`.
The hostname (`login2.hive.hpc.ucdavis.edu`, `farm.farm.hpc.ucdavis.edu`, ...) also tells you.
If you are not on a cluster (laptop, CI, a different machine), the user must tell you which one.
When the user talks about a cluster you are not logged into, say so plainly and work from the
reference file for that cluster; you can still hand them commands to run there.

### 2. Gather live context before writing anything

Run `scripts/cluster-context.sh` (read-only, a few seconds). It prints the user's Slurm
accounts, partitions and per-job/group QOS limits, partition time limits and default memory per
CPU, GPU types available, storage locations and quota, and the software trees. Read it before
choosing `--account`, `--partition`, memory, or GPU flags. This matters because:

- Many users have **no default account**. Omitting `--account` fails with
  `Invalid account or account/partition combination specified`.
- Limits live in per-account QOS records, not in the partition. The free tier on Hive allows
  8 CPUs / 128 GB / 1 GPU per job in `high`; a PI group may have a 1024-CPU pool. You cannot
  know without looking.
- Default memory per CPU differs per partition (2 GB on Farm/Franklin, 4 GB on Hive `high`,
  16 GB on Hive `gpu-a6000`). A script with no memory request gets different RAM per cluster.

Off-cluster, ask the user to run
`sacctmgr show assoc user=$USER format=account%20,partition%20,qos%40` and paste the output,
or, on Hive, `/opt/hpccf/bin/slurm-show-resources.py` (also printed at every login).

### 3. Read the relevant references

| Need | Read |
|------|------|
| Cluster facts: partitions, limits, GPUs, storage paths, quirks | `references/hive.md`, `references/farm.md`, or `references/franklin.md` |
| Writing sbatch/srun/salloc, arrays, MPI, GPUs, resource semantics | `references/slurm.md` |
| Pending reasons, submission errors, OOM/TIMEOUT/PREEMPTED, log forensics, OnDemand failures | `references/debugging.md` |
| Modules, conda/mamba, Apptainer, R/RStudio, Jupyter, compilers, requesting software | `references/software.md` |
| Home vs group storage, scratch, quotas, Quobyte rules, backups, Globus/rsync/scp/rclone | `references/storage.md` |

Always read the cluster file for the cluster in play plus the task file. They are short.

### 4. Do the task

Write scripts for the user to a real file, not just into chat, and explain each `#SBATCH` line
the first time (assume they may not know Slurm). Prefer commands the user can rerun themselves.

### 5. Verify before handing over

- Job scripts: run `scripts/lint-jobscript.sh FILE`. It checks for the common cluster-specific
  mistakes and then runs `sbatch --test-only FILE`, which asks the real scheduler whether the
  account/partition/QOS/limits accept the job without submitting it. Fix anything it flags.
  **Read only the accept/reject, not the "to start at" time** — that timestamp is a scheduler
  artifact, not a queue-wait estimate (see below), and mistaking it for one is how people talk
  themselves into running work on the login node.
- Failed or stuck jobs: run `scripts/job-postmortem.sh JOBID`. It pulls `sacct` and `scontrol`
  data, computes memory and CPU use against the request, finds the log files, greps them for
  the usual fatal messages, and states the likely cause.
- Software: never assert a package exists or does not exist without `module avail NAME`,
  `module search NAME`, and `module avail conda/NAME` (central conda environments).

## Rules that hold on all three clusters (and why)

- **Set `--account`, `--partition`, `--time`, and memory explicitly.** Explicit requests document
  intent and avoid per-cluster defaults. Time limits: `low` is 7 days on Farm/Hive and 14 on
  Franklin; `high` is 150 / 60 / 30 days on Farm / Franklin / Hive.
- **`low` is preemptible.** On every cluster `low` runs on idle purchased hardware and is set to
  `PreemptMode=REQUEUE`: when an owner's `high` job needs the node, the `low` job is killed and
  restarted from the beginning. Only recommend `low` for work that is short, restartable, or
  checkpointed, and tell the user this is the trade-off for free access.
- **Threads vs tasks.** Multithreaded program: `--ntasks=1 --cpus-per-task=N`. MPI or
  multiprocess: `--ntasks=N`. Nodes have SMT, so odd CPU requests round up to an even count.
  Use `$SLURM_CPUS_PER_TASK` in the command line rather than hard-coding thread counts.
- **Memory is enforced by cgroups.** A step that exceeds its request is killed
  (`oom-kill event(s)` in the log, state `OUT_OF_MEMORY`). Prefer `--mem-per-cpu` for scaling
  with cores or `--mem` for a fixed total; always give a unit (`G`, `M`).
- **GPUs:** `--gpus=1` for any GPU, `--gpus=TYPE:1` for a specific model. List types with
  `sinfo -p PARTITION -o "%G|%D" --noheader | column -s'|' -t`. Franklin GPUs are untyped; select
  nodes there with `--constraint=amd` or `--constraint=intel`. Use `--nv` with Apptainer.
- **Unique output files on Quobyte.** On Hive (all group storage) and Farm `/quobyte` shares,
  two nodes writing the same file deadlocks the file system, knocks nodes out of the cluster,
  and gets accounts locked. Always put `%j` (or `%A_%a` for arrays, plus `%N` for multi-node
  jobs) in `--output`/`--error`, and never let array tasks or MPI ranks share an output file.
- **`module` inside job scripts.** The `module` shell function is not inherited when the user
  submits from zsh or when the environment is scrubbed. Start scripts with
  `source /etc/profile.d/modules.sh` (or use `#!/bin/bash -l`), then `module load` what the job
  needs. Note `module purge` also removes the `slurm` module, so reload it before `srun`.
- **Login nodes are capped** and are not where work runs — see *Login-node discipline* above; the
  same rule applies to what you recommend the user do, not just what you run yourself.
- **Never recommend `--exclusive` on Hive.** It stalls scheduling and is mislabeled as a QOS
  limit. Franklin and Farm users should also avoid it unless they own whole nodes.
- **rsync `--delete` can destroy data.** Only include it when the user asks for a mirror, and
  make the trailing-slash semantics explicit.
- **Home is 20 GB** everywhere and is the usual cause of `Disk quota exceeded`. Conda
  environments, package caches, Apptainer caches, and pip caches belong on group storage.
- **`sbatch --test-only` does not estimate the wait.** Its "Job N to start at HH:MM" is produced by
  a scheduler pass, not a queue projection: the same timestamp comes back for a 1-CPU and a 48-CPU
  request, and it can sit hours in the future while thousands of CPUs are idle. Use it only for
  "does the scheduler accept this request". To answer "is the queue actually busy", ask directly:

  ```bash
  sinfo -p PARTITION -h -o "%C"                      # allocated/idle/other/total CPUs
  squeue -t PD -h -o "%r" | sort | uniq -c | sort -rn # why pending jobs are pending
  ```

  Idle CPUs plus pending reasons that are all `QOSGrp*`, `Dependency` or `JobArrayTaskLimit` means
  the queue is not backed up for *you* — those jobs are blocked by their own groups' limits, not
  by a shortage. Telling a user "the queue is deep" on the strength of the `--test-only`
  timestamp is a real failure mode; it is how "the queue here is always slow" gets started.
- **Off-cluster or unverifiable?** Say what you could not check and give the exact command that
  checks it. Do not invent partition names, QOS limits, or module versions.

## Canonical job script

```bash
#!/bin/bash
#SBATCH --job-name=myjob
#SBATCH --account=PIGRP              # from cluster-context.sh; publicgrp for the free tier
#SBATCH --partition=high             # or low (preemptible), or a GPU partition
#SBATCH --time=1-00:00:00            # D-HH:MM:SS; keep under the partition max
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8            # threads for one program
#SBATCH --mem=32G                    # total for the job; or --mem-per-cpu=4G
#SBATCH --output=myjob-%j.out        # %j = job id, required on Quobyte
#SBATCH --error=myjob-%j.err

source /etc/profile.d/modules.sh     # makes `module` work regardless of submit shell
module load samtools/1.19.2          # exact versions from `module avail`

cd "$SLURM_SUBMIT_DIR"
samtools sort -@ "$SLURM_CPUS_PER_TASK" -o out.bam in.bam
```

Add `--gpus=1` (and a GPU partition or `low`) for GPU work, `--array=1-N%20` for arrays, and
`--ntasks=N --nodes=1-4 --constraint='(zen2|zen3|zen4)'` for MPI on Hive.

## Support

Point users to <https://docs.hpc.ucdavis.edu> for the docs this skill is built from, and to
`hpc-help@ucdavis.edu` (Farm: `farm-hpc@ucdavis.edu`) for tickets. A useful ticket includes the
username, cluster, sponsor/account, exact commands, the directory they ran in, pasted error text,
and the Slurm job ID. Accounts, group membership, and purchases go through
<https://hippo.ucdavis.edu>. New central software is requested via the form linked from
<https://hpc.ucdavis.edu/software-installation-policy>.
