# Debugging jobs on HPC@UCD

Start with `scripts/job-postmortem.sh JOBID`; it automates most of this page. Use the sections
below to interpret what it prints and to handle cases it does not cover.

## Triage: what state is the job in?

```bash
squeue --me                       # still queued or running?
sacct -j JOBID -X --format=JobID,State,ExitCode,Elapsed,Timelimit,ReqMem,MaxRSS,NodeList,Reason
scontrol show job JOBID           # rich detail, only while Slurm still remembers the job (minutes to hours after it ends)
```

`sacct` shows one line per job plus one per step (`.batch`, `.extern`, `.0`). Memory
(`MaxRSS`) and CPU time appear on the step lines. Exit codes are `program:signal`.

## Submission rejected (`sbatch: error: ...`)

| Message | Meaning | Fix |
|---------|---------|-----|
| `Invalid account or account/partition combination specified` | no default account, or this account has no association with that partition | add `--account=` and pick a partition from `sacctmgr show assoc user=$USER format=account%20,partition%20,qos%40`. On Farm `publicgrp` only has `low`. |
| `QOSMaxCpuPerJobLimit`, `QOSMaxMemoryPerJob`, `QOSMaxGRESPerJob` | the request exceeds the per-job cap on this QOS (Hive free tier: 8 CPUs, 128 GB, 1 GPU) | shrink the request, use another account, or use `low` (no per-job caps, preemptible) |
| `Requested time limit is invalid (missing or exceeds some limit)` | `--time` above the partition/QOS max | see the cluster file for `MaxTime`; Hive free-tier `high` is 30 days, `low` is 7 |
| `Requested node configuration is not available` | no single node can satisfy the CPU/memory/GPU combination, or the GPU type is not in that partition | check `sinfo -p PART -N -o "%N %c %m %G"`; lower the request or change partition |
| `Job violates accounting/QOS policy (job submit limit, user's size and/or time limits)` | generic QOS refusal (`DenyOnLimit`) | run `sbatch --test-only` for the specific limit name; compare with `sacctmgr show qos ...` |
| `Invalid generic resource (gres) specification` | typo in `--gpus=TYPE:N` or type not defined on the cluster | list types with `sinfo -o "%G" --noheader \| sort -u` |
| `Invalid partition name specified` | partition does not exist on this cluster | partitions differ per cluster; see the cluster file |
| `Unable to open file` / `Batch script contains DOS line breaks` | wrong path or CRLF endings from a Windows editor | `dos2unix script.sh` or `sed -i 's/\r$//' script.sh` |

## Pending reasons (`squeue` NODELIST(REASON) column)

| Reason | What it means | What to do |
|--------|---------------|------------|
| `Priority` | others ahead in the queue | wait; smaller/shorter requests backfill sooner; check `sshare -U` |
| `Resources` | at the front, waiting for CPUs/memory/GPUs to free | wait, or shrink the request |
| `QOSGrpCpuLimit`, `QOSGrpMemLimit`, `QOSGrpGRES` | the account's purchased pool is fully used by group members (or Hive free-tier pool of 128 CPUs/2 TB/5 GPUs is full) | wait for lab mates' jobs, coordinate, or use `low`. On Hive this also appears erroneously for `--exclusive`; remove that flag |
| `QOSMaxCpuPerUserLimit`, `QOSMaxJobsPerUserLimit` | per-user cap on the QOS | wait or use another account |
| `AssocGrpCpuLimit`, `AssocMaxJobsLimit` | same idea at the association level | as above |
| `JobArrayTaskLimit` | array throttle `%N` is doing its job | nothing |
| `Dependency` | waiting on `--dependency` job | nothing |
| `DependencyNeverSatisfied` | the parent failed; this will never run | `scancel` it and resubmit after fixing the parent |
| `ReqNodeNotAvail, Reserved for maintenance` | `--time` runs into the maintenance reservation | shorten `--time` or wait; `scontrol show reservation` shows the window |
| `ReqNodeNotAvail, UnavailableNodes:...` | requested/constrained nodes are down or drained | drop constraints; `sinfo -R` lists drained nodes and reasons |
| `PartitionTimeLimit` | `--time` over the partition max (older Slurm) | lower it |
| `BadConstraints` | `--constraint` matches no node in the partition | check `sinfo -N -o "%N %f"` |
| `JobHeldUser` / `JobHeldAdmin` | held by `scontrol hold` or by staff | `scontrol release JOBID`; ask support if admin-held |
| `launch failed requeued held` | node failed to start the job; Slurm requeued and held it | `scontrol release JOBID` to retry; report if it repeats |
| `InvalidAccount`, `InvalidQOS` | association changed or expired | check `sacctmgr show assoc user=$USER` |
| `Nodes required for job are DOWN, DRAINED or reserved` | cluster-wide capacity issue | wait; check MOTD / maintenance page |

## Is the queue actually busy?

Two commands answer this; `sbatch --test-only` does not.

```bash
sinfo -p PARTITION -h -o "%C"                        # allocated/idle/other/total CPUs
squeue -t PD -h -o "%r" | sort | uniq -c | sort -rn  # the reasons pending jobs are pending
```

`sinfo`'s idle count is the capacity actually available now. The pending-reason histogram tells you
whether the backlog is competition or bookkeeping: `Resources` and `Priority` mean real contention,
while `QOSGrp*`, `Dependency`, `DependencyNeverSatisfied` and `JobArrayTaskLimit` mean those jobs
are blocked by their own groups' limits or their own throttles and are not standing between you and
a node.

**`sbatch --test-only`'s "Job N to start at HH:MM" is not a wait estimate.** It is the result of one
scheduler pass, and it does not scale with the size of the request: on Hive, `-c 1`, `-c 8`,
`-c 16`, `-c 32` and `-c 48` all returned the identical timestamp while 1,551 CPUs sat idle in
`high`, and that timestamp was over an hour and a half in the future. Use `--test-only` for one
thing only — whether the scheduler accepts the request. Reading it as a queue depth is how a user
ends up believing the cluster is congested and running their work on a login node instead.

## The job ended badly

| `sacct` State / evidence | Cause | Fix |
|--------------------------|-------|-----|
| `OUT_OF_MEMORY`, ExitCode `0:125`, log has `oom-kill event(s)` or `Out Of Memory` | exceeded `--mem`/`--mem-per-cpu` | look at `MaxRSS`; request ~1.3× the observed peak; for multi-node jobs remember `--mem` is per node |
| `TIMEOUT`, log has `CANCELLED AT ... DUE TO TIME LIMIT` | hit `--time` | raise `--time` within the partition max, move to a longer partition, or checkpoint/split the work |
| `PREEMPTED` then `PENDING`/`REQUEUED`, `Restarts>0` in scontrol | `low`/`bml` job displaced by owner work | expected on scavenger partitions; use `high` (own account) or make the job restart-safe |
| `NODE_FAIL` | node crashed or lost contact | resubmit; with `--requeue` (default) Slurm may already have; report if it repeats |
| `CANCELLED by 0` | cancelled by root/staff (usually policy or emergency) | check email/MOTD; ask support |
| `CANCELLED by <uid>` | the user (or a script) ran `scancel` | `id UID` to see who |
| `FAILED` ExitCode `127:0`, log `command not found` | program not on PATH: module not loaded, conda env not activated, or `module` itself missing in the script | add `source /etc/profile.d/modules.sh` and the right `module load`; check `module avail NAME` |
| `FAILED` `126:0` | `Permission denied` on the executable or interpreter | `chmod +x`; check the shebang |
| `FAILED` `1:0`, `2:0` | the program reported an error | read stderr; run the same command interactively via `srun --pty` |
| `FAILED` `0:9` or `137:0` (SIGKILL) | killed externally, often OOM at the process level or `scancel` | check for oom messages; check memory |
| `FAILED` `0:11` or `139:0` (SIGSEGV) | program crashed; sometimes an architecture mismatch (`+amd` build on an Intel node on Franklin, `zen2` module on other CPUs) | use the generic build or add `--constraint` |
| `COMPLETED` but results wrong/missing | logic error, or a later step silently failed | add `set -euo pipefail` to scripts so failures stop the job and show up in `State` |

Other frequent log lines: `CUDA out of memory` (GPU RAM, not `--mem`: smaller batch or a bigger
GPU), `Disk quota exceeded` (20 GB home; see `storage.md`), `No space left on device` (per-job
`/tmp` or a full group share), `ModuleNotFoundError` (wrong conda environment),
`Illegal instruction` (binary built for a newer CPU; add `--constraint`), `srun: error: ... task
0: Exited with exit code N` (the step failed; look above it for the program's own message).

## Finding the log files

- `scontrol show job JOBID` prints `StdOut=` and `StdErr=` while the job is remembered.
- Otherwise the default is `slurm-JOBID.out` in the submit directory
  (`sacct -j JOBID -X --format=WorkDir%80`), or whatever `--output` named, with `%j`
  expanded. Search: `ls -t "$(sacct -j JOBID -X -n --format=WorkDir%200 | xargs)"/*JOBID*`.
- For Open OnDemand sessions, click the long `Session ID` on the card and read `output.log`.

## Right-sizing requests after a run

```bash
sacct -j JOBID --format=JobID,Elapsed,Timelimit,ReqMem,MaxRSS,AllocCPUS,TotalCPU -P
```

Memory efficiency is `MaxRSS / ReqMem`; CPU efficiency is `TotalCPU / (Elapsed × AllocCPUS)`.
Very low CPU efficiency with many CPUs means the program is single-threaded or was never told
how many threads to use (pass `$SLURM_CPUS_PER_TASK`). Over-requesting delays start and wastes
the group's pool. (`seff` is not installed on these clusters; compute it from `sacct`.)

## Inspecting a running job

`srun --jobid=JOBID --overlap --pty bash -l` opens a shell on the job's node inside its cgroup:
run `top -u $USER`, `nvidia-smi`, `ls $TMPDIR`, `cat /sys/fs/cgroup/memory.max`. Direct `ssh
node` is not allowed.

## Login node problems

Per-user caps on login nodes: 2 CPUs (`CPUQuota=200%`), 7.5% of RAM, 500 MB swap, 512 processes,
and 16,384 open files. Symptoms: processes vanish silently (OOM), `Too many open files`,
`fork: retry: Resource temporarily unavailable`. Count processes with `pgrep --count --uid $USER`;
move the work into `srun --pty bash`. Staff kill offending processes, and jobs that hammer a shared
file system can get an account locked. The fix is always to run the work in a job and wait for it,
never to shrink or stagger it until it squeezes under the caps — see *Login-node discipline* in
`SKILL.md`.

**Who is capped, and how to check.** PAM runs `/etc/security/systemd-user-limits.sh` at each login
(wired in `/etc/pam.d/common-session`). It skips system accounts (UID < 1000), skips — and actively
removes limits for — members of **`hpccfgrp`**, and for everyone else applies
`CPUQuota=200% MemoryMax=7.5% MemorySwapMax=500M TasksMax=512` to that user's slice. So:

```bash
cat /etc/security/systemd-user-limits.sh          # the authoritative current values
grep nofile /etc/security/limits.d/slurm.conf     # the open-file limit (set separately)
id -nG | grep -qw hpccfgrp && echo "exempt (staff)" || echo "capped"
ls /etc/systemd/system.control/user-$(id -u).slice.d/   # the drop-in, absent when exempt
systemctl show "user-$(id -u).slice" -p CPUQuotaPerSecUSec -p MemoryMax -p TasksMax
```

If a user reports being throttled, confirm with the last two commands. If *you* are exempt because
you are running as staff, the caps still apply to the person you are helping — never generalise
from your own slice to "there are no limits here".

A related self-inflicted case: `sinfo`/`sbatch`/`sacct: command not found` in a non-interactive
shell or after `module purge`. The commands are always at
`/cvmfs/hpc.ucdavis.edu/sw/spack/environments/core/view/generic/slurm/bin/`; run
`source /etc/profile.d/modules.sh` (or prepend that directory to `PATH`). Do not `find` for them.

## Open OnDemand app failures

Open the failed card, click the `Session ID`, read `output.log`:

- `EnvironmentNameNotFound: Could not find conda environment` → typo in the conda env field.
- `R version change [4.2.3 -> 4.4.2] detected` or `install.packages()` failing → clear the
  RStudio cache: `~/.RData`, `~/.local/share/rstudio*`, and if needed `~/.cache/rstudio*`,
  `~/.config/rstudio*`; disable "Restore .RData into workspace".
- `oom_kill events` / kernel died / "abnormally terminated" → request more memory in the form.
- `CANCELLED ... DUE TO TIME LIMIT` → request more hours.
- `sbatch: error: QOSMaxMemoryPerJob` or similar → the form asked for more than the
  account/partition allows (Hive free tier: 8 CPUs, 128 GB, 1 GPU).
- `ERROR: CONDA_EXE is currently defined: /home/.../conda/bin/conda` → a self-installed conda
  conflicts with the central one; remove the `conda initialize` block from `~/.bashrc`.
- Desktop browsers lose their profiles each session by design.

## SSH and access

- Prompted for a *password* on Farm/Franklin (which are key-only) → the key is not being
  offered: `chmod 600 ~/.ssh/id_rsa`, or `ssh -i ~/.ssh/KEY user@cluster.hpc.ucdavis.edu`, or
  the public key in HiPPO is not the one on this machine. Hive also accepts the campus
  passphrase.
- `REMOTE HOST IDENTIFICATION HAS CHANGED` → compare against the fingerprints in
  <https://docs.hpc.ucdavis.edu/general/access/>; if they match, run the `ssh-keygen -R`
  command shown; if not, open a ticket.
- Account requested but cannot log in → PI approval plus ~1 hour provisioning; HiPPO syncs
  every 15 minutes.

## Writing a good ticket

Send to `hpc-help@ucdavis.edu` (Farm: `farm-hpc@ucdavis.edu`): username, cluster, sponsor
account, exact commands and the directory, pasted error text (attachments are often stripped;
give full paths instead), and the job ID. Emailing staff directly is not answered by policy.
