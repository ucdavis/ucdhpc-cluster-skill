# Storage and data movement on HPC@UCD

## Where things live

| Space | Hive | Farm | Franklin | Notes |
|-------|------|------|----------|-------|
| Home | `/home/$USER`, 20 GB, NFS, **backed up nightly** | `/home/$USER`, 20 GB, autofs NFS, no backup | `/home/$USER`, 20 GB, no backup | Check with `df -h ~`. Dotfiles, small scripts only. |
| Group / PI share | `/quobyte/<pi>grp` (Quobyte parallel FS) | `/group/<pi>grp` (NFS, older) and/or `/quobyte/<pi>grp` (newer purchases) | `/group/<pi>grp` (ZFS/NFS, autofs) | Where data, conda envs, containers, and results go. Named after the PI's login ID. |
| Per-job local scratch | `/tmp` and `/scratch` (same disk, ~1.6 TB), `$TMPDIR=/tmp` | `/tmp` and `/scratch` (~1.8 TB), `$TMPDIR=/tmp` | `/tmp` (~0.9 TB), `$TMPDIR=/tmp`, no `/scratch` | Private to the job, fastest I/O, **deleted when the job ends**. Copy results out before exit. |
| Network scratch | `/nfs/hive/scratch` (22 TB, shared, not purged automatically, not backed up) | none | none | For intermediates shared between jobs. Clean up yourself; abuse gets accounts locked. |
| Special | `/nfs/hive/scratch-mpi-io` (needs `mpi-io-grp`; dir named after job id or it is purged), `/nfs/peloton/waltz` (legacy Peloton storage, read access) | `/share/apps` | `/share/databases` (alphafold, blast, relion, ...), `/share/apps` | |

Franklin (and Farm `/group`) mounts are on demand: `ls /group` may not list a share until you
`cd` into it. Find the group name with `id -Gn` or `groups` (ends in `grp`), then
`ls /group/NAMEgrp` or `ls /quobyte/NAMEgrp`.

## Walking these file systems costs everyone

Quobyte, the NFS group shares, and CVMFS are network file systems: every directory a `find`,
`du`, `ls -R`, or `rsync --dry-run` descends into is a round trip to a metadata server shared by
the whole cluster. A recursive walk of a share root from a login node is one of the fastest ways
to make the cluster feel broken for everybody, and Python environments or single-cell datasets
(tens of thousands of small files each) make it far worse.

- On a login node, bound every walk: `find /group/PIGRP/project -maxdepth 2 -name '*.bam'`.
  Anything deeper belongs in a job: `srun --account=A --partition=P --time=15 --mem=2G find ... -maxdepth 6 ...`.
- Ask a cheaper question first: `df -h PATH` and `qinfo quota PATH` for space, `ls` with a glob
  when the directory is known, `sacct -j JOBID --format=WorkDir%200` to find a job's output,
  `module avail NAME` for software.
- Scope `du` to a suspect directory (`du -sh ~/.conda ~/.cache ~/.apptainer`) rather than a share.
- When a full inventory really is needed, produce it once in a job, write the listing to a file,
  and read the file afterwards instead of re-walking.

## Quotas and cleaning up

```bash
df -h ~                                  # home usage vs 20 GB
qinfo quota /quobyte/PIGRP               # Quobyte quota (Hive, Farm); may print "No effective quotas"
du -sh ~/.conda ~/.cache ~/.apptainer ~/.local ~/.singularity 2>/dev/null | sort -h
```

`Disk quota exceeded` in home almost always means conda/pip/Apptainer caches or environments.
Move them to the group share (see `software.md`: `conda config --add envs_dirs/pkgs_dirs`,
`APPTAINER_CACHEDIR`, `R_LIBS_USER`, `PIP_CACHE_DIR`), then `conda clean --all` and delete the
old copies. Extra home space is not sold; PIs buy group storage instead.

## Quobyte rule: one writer per file

On Hive (all `/quobyte`) and Farm `/quobyte` shares, several nodes appending to the same file
causes lock contention that blocks the file, hangs `slurmd`, drops the node from the cluster, and
needs admin intervention. Offending jobs are killed and accounts may be locked. So:

- `--output`/`--error` must contain `%j` (single node), `%A_%a` (arrays), and add `%N` for
  multi-node jobs (`slurm-%j_%N.out`).
- Array tasks and MPI ranks write to distinct files: `out-${SLURM_JOB_ID}_${SLURM_ARRAY_TASK_ID}.txt`,
  `$(hostname)-${SLURM_JOB_ID}.results`.
- Programs that genuinely need shared writes must use file locking; pointing them at `$TMPDIR`
  and merging afterwards is usually simpler.

## Permissions and sharing inside a group

Group shares are group-owned; new files inherit the group when the directory has the setgid
bit (`chmod g+s dir`). To let lab mates read: `chmod -R g+rX dir`; to let them write:
`chmod -R g+rwX dir` (capital X keeps files non-executable). Finer control: `setfacl -m u:LOGIN:rx dir`.
Do not `chmod 777`.

## Backups

- **Hive only.** Home directories are backed up nightly (keep 7 daily, 5 weekly, 3 monthly).
  Group shares are backed up only inside `/quobyte/PIGRP/BACKED-UP/` and only if the PI has
  purchased backup space; nothing outside that directory is backed up. Symlinks inside
  `BACKED-UP/` are stored as links, not the targets.
- Restore: `cd ~` (or `cd /quobyte/PIGRP`), run `/quobyte/backups/bin/restore.sh`, copy files
  from the mounted `snapshots/` directory in a second shell, then Ctrl-C to unmount.
- **Farm and Franklin have no backups.** Farm hardware older than five years triggers a login
  warning; data on it is at risk. Users are responsible for off-site copies (Box via rclone,
  Globus to another site, external drives).
- Backups are on campus but not off-site; the backup server itself is not replicated.

## Data transfer

Hostnames: `farm.hpc.ucdavis.edu`, `franklin.hpc.ucdavis.edu`, `hive.hpc.ucdavis.edu`
(`transfer.hive.hpc.ucdavis.edu` exists for LSSC0 migrations only). Farm/Franklin require SSH
keys; Hive also takes the campus passphrase. Transfers run on login nodes; that is expected use,
but avoid dozens of parallel streams.

- **rsync** (resumable, preserves metadata). Trailing slashes matter: `src/` copies contents,
  `src` copies the directory itself.

  ```bash
  rsync --archive --one-file-system --info=progress2 ./data/ USER@hive.hpc.ucdavis.edu:/quobyte/PIGRP/USER/data/
  rsync --archive --info=progress2 USER@farm.hpc.ucdavis.edu:/group/PIGRP/results/ ./results/
  ```

  Rerunning resumes. `--delete`/`--delete-after` makes the destination match the source and
  **permanently deletes** anything extra; only use when a mirror is explicitly wanted and paths
  are triple-checked. Cluster to cluster: run rsync on one login node with the other as remote.
- **scp** for a few files: `scp -rp localdir USER@farm.hpc.ucdavis.edu:/group/PIGRP/USER/`.
- **Globus** (v5, free tier): collections `UC Davis <Cluster> home` exist for all three; PI
  shares are exported on request (`UC Davis <Cluster> <pi>grp`) and are writable only under
  `/globus-write/<login>/` (→ `/quobyte/PIGRP/globus-write/<login>/` on Hive,
  `/group/PIGRP/globus-write/<login>/` on Farm/Franklin). Both endpoints need a Globus login or a
  paid subscription on the far side.
- **Box**: `module load rclone`, create a Box custom OAuth app (redirect `http://localhost:53682/`),
  `rclone config` → type `box`, authorize via an OnDemand desktop browser, then
  `rclone copy/sync ./dir mybox:/Folder --progress`.
- **Open OnDemand Files app**: browser upload/download for files under a few hundred MB.
- **GUI clients** (FileZilla, Cyberduck, WinSCP) work over SFTP but are unsupported by HPC@UCD.
- **From LSSC0 (Genome Center)**: SSH to `transfer.hive.hpc.ucdavis.edu` and rsync from
  `LSSC0-ID@barbera.hpc.genomecenter.ucdavis.edu:` into `/quobyte/PIGRP/...`; group migrations
  are done by staff via ticket.
- **Public datasets**: pull with `aria2c`, `wget`, `awscli`, `sratoolkit`, `aspera-cli`
  (all modules) from inside an `srun` session or job on a compute node; the login-node process
  caps apply otherwise.

## Using scratch well

```bash
cd "$TMPDIR"                                   # per-job local disk, private, fast
cp /quobyte/PIGRP/USER/input.bam .
tool --threads "$SLURM_CPUS_PER_TASK" input.bam > output.bam
cp output.bam /quobyte/PIGRP/USER/results/output-${SLURM_JOB_ID}.bam   # before the job exits!
```

Local scratch is destroyed at job end, so copy-out is part of the script, ideally under a
`trap ... EXIT`. Many small files (conda envs, Python imports, single-cell tools) are far faster
from local scratch than from network shares.
