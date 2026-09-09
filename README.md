# ucd-hpc — a Claude skill for UC Davis HPC users

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that teaches Claude how the
UC Davis HPC@UCD clusters (**Farm**, **Franklin**, **Hive**) actually work: which partitions and
accounts a user has, the free-tier limits, how to request CPUs/memory/GPUs/time, why jobs fail or
sit pending, where software lives (modules, conda, Apptainer, Open OnDemand), and how storage and
data transfer work.

The skill is `ucd-hpc/`. It bundles:

- `SKILL.md` — the workflow Claude follows (identify cluster → probe live → read references → verify).
- `references/` — distilled, verified facts: shared Slurm behavior, debugging tables, software,
  storage, and one file per cluster (`hive.md`, `farm.md`, `franklin.md`).
- `scripts/cluster-context.sh` — read-only snapshot of the user's accounts, QOS limits, partitions,
  GPUs, storage, and software trees.
- `scripts/job-postmortem.sh JOBID` — explains a pending/failed job from `sacct`/`scontrol` and its logs.
- `scripts/lint-jobscript.sh SCRIPT` — catches cluster-specific mistakes and runs `sbatch --test-only`.

## Install

```bash
git clone https://github.com/ucdavis/ucdhpc-cluster-skill ~/ucdhpc-cluster-skill   # or wherever
mkdir -p ~/.claude/skills
ln -s ~/ucdhpc-cluster-skill/ucd-hpc ~/.claude/skills/ucd-hpc
```

Works best when Claude Code runs on a cluster login node (live probes), but the references
also let it help from a laptop.

## Sources

Prose is derived from <https://docs.hpc.ucdavis.edu> (source: <https://github.com/ucdavis/hpccf-docs>);
partition, QOS, GPU, and storage facts were verified against the live clusters in September 2026.
Re-verify after maintenance windows: `sinfo -s`, `scontrol show partition`, `sacctmgr show qos`.

## Development

`ucd-hpc/evals/evals.json` holds test prompts. Iteration outputs go to `ucd-hpc-workspace/`
(git-ignored).
