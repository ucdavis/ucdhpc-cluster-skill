# Benchmark record

Evaluation artifacts live in `ucd-hpc-workspace/` (git-ignored). This file keeps the durable
conclusions and the cluster facts the runs turned up.

## Iteration 1 — does the skill help at all?

`with_skill` (initial `ucd-hpc`) vs `without_skill` (no skill), 4 evals, 1 run per cell.

| Metric | With skill | Baseline |
|---|---|---|
| Assertions | **43/43** | 40/43 |
| Wall time | 508s | 681s |
| Tokens | 96.0k | 88.7k |

Baseline misses: no explanation of `--job-name`/`--nodes`/`--ntasks` to a self-described Slurm
beginner; thread count never passed to `fastp`; and a 279-package conda solve directed onto a
login node.

**Non-discriminating**: the job-debugging eval scored 9/9 both ways — `sacct` plus the HPCCF
epilog block already print `OUT_OF_MEMORY` and `command not found` in plain text, so a competent
baseline reaches the same diagnosis. Harder fixtures needed (preemption on `low`, a
`QOSGrpCpuLimit` pending job, an `+amd` binary on an Intel node).

**Method bug (fixed in iteration 2)**: one baseline agent read the assertion list out of
`eval_metadata.json` in its own run directory. Assertions are now withheld until all runs finish.

## Iteration 2 — do the login-node discipline changes do anything?

`new_skill` (with the login-node section, exact Slurm paths, bounded-`find` rule) vs `old_skill`
(the pre-change snapshot), 3 purpose-built evals, 1 run per cell.

| # | Eval | New | Old | Discriminates? |
|---|---|---|---|---|
| 4 | Slurm command location (cron / non-interactive) | 7/7 | 7/7 | no |
| 5 | Deep `find` on a group share | **7/7** | **4/7** | **yes** |
| 6 | Pressure to run an analysis on the login node | 7/7 | 7/7 | no |

**The bounded-`find` rule is the one change with a measured behavioural effect.** In eval-5 the
target files sit at depth 6, so a `-maxdepth 2` walk finds nothing and the agent must choose. With
the new skill: one bounded `find` on the login node, the full walk in batch job 22908425, both
questions then answered from a single recorded listing. With the old skill: no Slurm job at all
(confirmed against `sacct` for the whole audit window) and six unbounded recursive walks plus a
full-tree `du` on the login node.

The other two changes are correct and cheap but changed nothing measurable:

- **Slurm paths**: neither run ever searched for the binaries. `command -v sacct` answers
  instantly for an agent already on the cluster. To exercise that failure mode an eval needs a
  context where `command -v` returns nothing — reasoning about another cluster from off-cluster, a
  container, or an OnDemand kernel.
- **Login-node pressure**: the pre-change skill already routed heavy work to `srun`/`sbatch`, and
  both runs independently found a stronger argument than policy — 2 CPUs on a login node is slower
  than a 5-second queue wait. Both also scaled *above* the user's stated 8 cores (to 60 and 48)
  after checking the account had no per-job cap.

### Measurement

Behavioural assertions cannot be checked from artifacts alone. Every job-submission claim was
corroborated with `sacct -u $USER -S <audit-start> --format=SubmitLine%200`, which records the
verbatim command line of each `srun`/`sbatch` and cannot be edited by the run. Fixture integrity
was checked by md5, mtime and directory/file counts. Three of 21 assertions still rest on each
run's required verbatim command log; a PATH shim logging `find`/`du` invocations would remove that
residual trust requirement.

## Cluster facts established while benchmarking

1. **Login-node caps are enforced for ordinary users, and `hpccfgrp` members are exempt.** PAM runs
   `/etc/security/systemd-user-limits.sh` on session open (`/etc/pam.d/common-session`). It skips
   UID < 1000; for members of **`hpccfgrp`** it *deletes*
   `/etc/systemd/system.control/user-<uid>.slice.d/*.conf`; for everyone else it applies
   `CPUQuota=200% MemoryMax=7.5% MemorySwapMax=500M TasksMax=512`. Observed on `login2.hive`:
   91 of 94 user slices carry `cpu.max="200000 100000"` (2 CPUs), `memory.max=20275437568`
   (18.88 GiB = exactly 7.50% of the node's 251.8 GiB) and `pids.max=512`; `nofile 16384` comes
   separately from `/etc/security/limits.d/slurm.conf` and applies to everyone. So the applicability
   test is `id -nG | grep -qw hpccfgrp`, not an inspection of your own cgroup — an agent running as
   staff sees `infinity` everywhere and will otherwise conclude, wrongly, that no limits exist.
2. **`sbatch --test-only`'s "to start at" time is not a wait estimate.** `-c 1`, `8`, `16`, `32`
   and `48` all returned the identical `2026-09-09T16:08:51` while `high` had 1551 idle CPUs. It is
   a scheduler-cycle artifact. Real queue-health probes: `sinfo -p PART -h -o "%C"` and
   `squeue -t PD -h -o "%r" | sort | uniq -c`.
3. **`set -u` and `/etc/profile.d/modules.sh` conflict.** `modules.sh` dereferences `$MANPATH`
   unguarded, so sourcing it under `set -u` aborts the caller. Two independent runs hit this, as
   did this skill's own helper scripts (fixed by wrapping the source in `set +u`/`set -u`).
4. **`conda/pytorch/2.5.1` and `conda/pytorch/2.9.1` are mislabelled.** Both modulefiles
   `pushenv _conda_envname "pytorch-2.4.1"`, so either gives torch 2.4.1 with CUDA 11.8. The 2.5.1
   and 2.9.1 environments exist on CVMFS but are unreachable via `module load`, and CUDA 11.8 will
   not run on the Blackwell GPUs that `publicgrp/high` can assign.
5. **The site `.condarc` sets `create_default_packages: gcc=13`**, so every `conda create` silently
   pulls a GCC 13 toolchain. `conda create` accepts `--no-default-packages`; `mamba` 2.0.5 rejects it.

Items 1 and 2 are now fixed in the skill (commit following the iteration-2 record); the caps are
described as policy with the `hpccfgrp` exemption and `/etc/security/systemd-user-limits.sh` named
as the authoritative source, and `lint-jobscript.sh` no longer prints the `--test-only` start time
at all. Items 4 and 5 remain open — see `ucd-hpc-workspace/NOTES.md`.

**The iteration-2 numbers describe the skill as it stood before those two fixes.** Re-running
iteration 2 against the current skill would be expected to change nothing measurable: eval-6 passed
7/7 in both configurations already, and neither fix targets the deep-find behaviour that produced
the only delta. A future iteration wanting to measure them needs an eval where the scheduler is
genuinely the slower option (to test the caps framing) and one where a run is tempted to report
queue depth from `--test-only` (to test that fix).
