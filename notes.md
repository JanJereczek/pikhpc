# Running Julia on the PIK cluster

**Scope of this folder.** `pikhpc-julia/` is shared across projects. It only records what is specific to running Julia on the PIK HPC2024 cluster. Do not put project-specific information here (package versions, scripts, results, benchmark numbers). Keep that in the project's own notes.

First written 2026-09-29. The general cluster guide (Slurm, QOSs, storage) is a separate document.

## Environment setup

- **Julia version.** Match the Julia version of the project's manifest (a `Manifest-v1.X.toml` is ignored by other versions, and Julia then re-resolves everything). Install extra versions with `juliaup add 1.12` and run them as `julia +1.12`. The juliaup default channel may be newer than the project needs.
- **`LD_LIBRARY_PATH` breaks Pkg.** The shell's loaded modules (hdf5, netcdf-c, curl/8.4.0, ...) put an old libcurl first on `LD_LIBRARY_PATH`. Julia then loads it instead of its own and Pkg fails with `curl_easy_setopt: 48`. Clear it for every Julia call: `LD_LIBRARY_PATH= julia +1.12 --project=.`. In job scripts, use `export LD_LIBRARY_PATH=`.
- **`startup.jl` per Julia version.** A new Julia version has its own global environment. If `~/.julia/config/startup.jl` loads packages such as OhMyREPL that are not installed there, Julia errors on startup. Either add them to that version's global environment or pass `--startup-file=no`.
- **Local dev packages.** A manifest stores the absolute path of `Pkg.develop`-ed packages. On a machine with a different home directory, run `Pkg.develop(path=...)` again.
- **Instantiate:**
  ```bash
  LD_LIBRARY_PATH= julia +1.12 --startup-file=no --project=. -e 'using Pkg; Pkg.instantiate()'
  ```

## No internet on compute nodes

Compute nodes cannot reach the internet. Only the `io` QOS (`--partition=io --qos=io`, on the login nodes) can. Do all package and artifact downloads on the login node. `Pkg.instantiate()` or `Pkg.add` on a compute node fails with `Failed to connect to pkg.julialang.org`.

Artifacts can depend on the node's hardware. `Reactant_jll`, for example, picks a different variant on a node with a GPU than on one without. The login node has an A40 GPU, so it downloads the CUDA variant, which the GPU nodes also use. A CPU-only compute node asks for a `gpu+none` variant that is not installed and cannot be downloaded. So `Pkg.precompile()` on a CPU-only node fails if Reactant is in the environment. Run it on a GPU node instead.

## Precompile on a GPU node before the first GPU run

A GPU job that seems to hang, with GPU utilisation at 0% and an empty log, is almost certainly precompiling. Julia keeps separate compile caches per CPU type (`-C native`). The login node (AMD EPYC 9354) and the compute nodes (EPYC 9554) don't share one. Precompiling a large stack (CUDA.jl, Reactant) on 4 cores takes far longer than a short job, so the run never gets to compute.

Warm the cache once, with many cores, on a GPU node:

```bash
srun --account=<account> --qos=gpushort --partition=gpu --gres=gpu:1 \
     --time=00:40:00 --ntasks=1 --cpus-per-task=32 \
     bash -c 'LD_LIBRARY_PATH= julia +1.12 --startup-file=no --project=. -e "using Pkg; Pkg.precompile()"'
```

For a CUDA.jl + Reactant + Makie project this took 191 s for 139 packages. Repeat it after changing the Julia version or updating packages.

To check whether a running job is stuck precompiling:

```bash
srun --jobid=<id> --overlap nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv
srun --jobid=<id> --overlap bash -c 'ps -o pid,ppid,etime,time,args -u $USER | grep [j]ulia'
```

A child `julia ... --output-ji .../compiled/v1.12/...` process means it is precompiling.

## Job setup

- **Account and QOS.** The account goes in `--account`; list yours with `sacctmgr show assoc where user=$USER`. GPU QOSs are `gpushort` (24 h), `gpumedium` (7 d), `gpulong` (30 d) and `gpupreempt`. Request GPUs with `--gres=gpu:N`, or the job gets a GPU node without a GPU. Each GPU node has 4× H100 80 GB, 128 cores and 1.5 TB RAM.
- **Quick tests.** `--qos=priority --partition=priority` is for short CPU test runs and starts within a few minutes.
- **Interactive runs.** `srun` with the same options runs a command on a compute node and blocks until it finishes.
- **Output location.** Write job logs to `/p/tmp/$USER`, not `$HOME`.
- **Skeleton of a Julia GPU job script:**
  ```bash
  #!/bin/bash
  #SBATCH --account=<account>
  #SBATCH --partition=gpu
  #SBATCH --qos=gpushort
  #SBATCH --gres=gpu:1
  #SBATCH --ntasks=1
  #SBATCH --cpus-per-task=4
  #SBATCH --time=01:00:00
  #SBATCH --output=/p/tmp/<user>/%x-%j.out

  export LD_LIBRARY_PATH=
  export JULIA_NUM_THREADS=$SLURM_CPUS_PER_TASK
  cd <project dir>
  julia +1.12 --startup-file=no --project=. <script>.jl
  ```

## Thread pinning

`ThreadPinning.pinthreads(:cores)` fails under Slurm when the job does not cover a whole node (`Couldn't set the affinity on this system`). Use `pinthreads(:affinitymask)` there, for example `pinthreads(haskey(ENV, "SLURM_JOB_ID") ? :affinitymask : :cores)`.

Set `JULIA_NUM_THREADS` (or `-t N`) to `$SLURM_CPUS_PER_TASK`. Julia otherwise sees the whole node's cores.

## Timing runs

The first call of anything in a Julia process carries JIT compilation (and XLA compilation with Reactant), which would otherwise land in the measured wall time. For timing, run a short warm-up of the same code path first and time only a second, fresh run in the same process. Measured on the cluster, this changed the steady-state time by up to about 20%. Reactant additionally keeps its compiled programs per simulation object, so a warm-up only helps if the compiled program is reused by the timed run.
