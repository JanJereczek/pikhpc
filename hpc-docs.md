<!-- Source: https://www.pik-potsdam.de/en/institute/about/it-services/hpc/hpc-2024-documentation/hpc-2024-documentation
     (login-protected). Converted from a saved copy on 2026-09-29; re-convert if the page changes. -->

# HPC2024 User Guide

## Introduction

Access to the compute resources of a cluster is generally handled by workload management software. This software manages requests for compute resources by users' jobs (for example CPU cores, memory, run time, access to special hardware like extra RAM or GPUs), and places jobs in a priority- and resource-based queue.

On the PIK cluster, we use **Slurm**. This document outlines the essential steps in using Slurm for submitting and managing jobs. For more in-depth information, please see the References.

## Filesystems and Storage

The cluster provides several filesystems for different purposes:

| Filesystem | Path | Purpose | Backed up | Notes |
|---|---|---|---|---|
| Home | `/home/<username>` | Personal files, scripts, configurations | Yes | Limited quota, do not run jobs from here |
| Projects | `/p/projects/<project>` | Shared project data | Yes | Request access via ticket |
| Scratch | `/p/tmp/<username>` | Temporary job data, large datasets | No | Fast parallel filesystem, auto-cleanup of old files |
| Local NVMe | `/localscratch` | Ultra-fast local I/O (GPU nodes only) | No | ~3TB per node, not shared, cleared after job ends |

### Important guidelines

- **Run jobs from `/p/tmp`**: The parallel filesystem is designed for high I/O throughput. Running jobs that write significant output from `$HOME` can disrupt cluster operations.
- **Scratch is not backed up**: Files on `/p/tmp` are subject to automatic deletion after a period of inactivity. Keep important results elsewhere.
- **Check your quota**: Use `quota` or `lfs quota -u $USER /p` to see your current storage usage.

### Creating your scratch directory

```bash
mkdir -p /p/tmp/$USER
```

## Data Transfer

### rsync (recommended)

`rsync` is the recommended tool for transferring data to and from the cluster. It supports resumable transfers and only copies changed files:

```bash
# Upload to cluster
rsync -avzP /local/path/ hpc.pik-potsdam.de:/p/tmp/$USER/destination/

# Download from cluster
rsync -avzP hpc.pik-potsdam.de:/p/tmp/$USER/source/ /local/path/
```

Options explained:

- `-a`: Archive mode (preserves permissions, timestamps)
- `-v`: Verbose output
- `-z`: Compress during transfer
- `-P`: Show progress and allow resume

### scp

For simple file transfers:

```bash
# Upload
scp myfile.txt hpc.pik-potsdam.de:/p/tmp/$USER/

# Download
scp hpc.pik-potsdam.de:/p/tmp/$USER/results.tar.gz ./

# Upload directory
scp -r mydir/ hpc.pik-potsdam.de:/p/tmp/$USER/
```

## Basic Concepts

A *job* is an instance of your code that you want to run on the cluster. Jobs are submitted to the queue for scheduling and are run in order based on priority. Priority is a dynamic property based on (among other factors) how many jobs you have run recently and how many resources your job needs. A job is typically made up of a submission (batch) script and your code.

A *login node* is an interactive, shared server on the cluster where you land when you first log in. There are three of these: `login[01-03]`, each with 64 CPU cores and 768GB RAM. Which one you login to when you connect to `hpc.pik-potsdam.de` is more or less random - you will be directed to the node with the freest resources. These are intended for interactive work like editing your source code, compiling, short test runs and the like. Since these machines are shared amongst all cluster users, they are expressly *not* for large-scale parallel production runs.

A *compute node* is a server where your code will typically run. On the PIK cluster, there is one type of compute node equipped with 2x 64 core AMD Genoa processors and 768 GB RAM. The Slurm command `sinfo` shows the nodes of the cluster and their states.

A *partition* is a grouping of nodes, often for hardware compatibility but sometimes to provide different access restrictions. The command `sinfo` shows you what partitions are configured. There are currently two main partitions called `standard` and `priority` (partitions `io` and `gpu` are special purpose partitions and are described later).

A *QOS* (Quality Of Service, usually just `qos` in commands), is a set of limits and resources you can request. For example, your job may need more than one day to run, so you would request the `medium` QOS, which has a 7-day limit. Choosing the QOS with the most appropriate limits to accommodate your job will generally ensure the job doesn't wait longer than necessary in the queue. You can also request *fewer* resources than a QOS provides. This may allow the scheduler to move your job to the top of the queue more quickly. More on this below. The command `sacctmgr show qos` will show you more information about what limits each QOS has.

Not all partitions and QOSs can be combined. For example, asking for partition `io` and QOS `medium` won't work, because long running jobs must be placed on the compute nodes in the `standard` partition, whereas the `io` partition is confined to the three login (interactive) nodes, `login[01-03]`.

### QOS

The currently available QOSs on the CPU-only partitions, and their limits, are:

| QOS | Partition | Time limit | CPU limit (per job) | CPU limit (all jobs) |
|---|---|---|---|---|
| short | standard | 24 hours | 2048 | 18000 |
| medium | standard | 7 days | 1024 | 12000 |
| long | standard | 30 days | 32 | 6000 |
| priority | priority | 24 hours | (5 concurrent jobs) | 6144 |
| standby | priority | 7 days | 2048 | (unlimited) |
| standby_cancel | priority | 7 days | 2048 | 3072 |
| io | io | 3 days | 4 | 16 |

The currently available QOSs on the GPU partition and their limits are:

| QOS | Partition | Time limit | CPU limit (all jobs) | GPU limit (all jobs) |
|---|---|---|---|---|
| gpushort | gpu | 24 hours | 768 | 24 |
| gpumedium | gpu | 7 days | 512 | 16 |
| gpulong | gpu | 30 days | 256 | 8 |
| gpupreempt | gpu | 30 days | 768 | 24 |

#### `short`, `medium` and `long` QOSs.

These QOSs provide guaranteed runtime (in contrast to `standby`, see below) and unlimited concurrent jobs (in contrast to `priority`, see below). See the table above, or `sacctmgr show qos`, for details of the limits they apply. The default QOS is `short`, with a 24 hour time limit.

> Quick tip: you can override the QOS in a Slurm submission script on the command line like this: `sbatch -q priority slurm.sh`

This is useful if, for example, you see that the `short` queue is full and want to switch to the `priority` queue without editing the submit script.

#### `gpushort`, `gpumedium`, `gpulong` and `gpupreempt` QOSs.

These QOSs are for use on the GPU-equipped partition (`gpu`) and provide the same time-limits as the similarly-named CPU-only QOSs.

The `gpupreempt` QOS operates similarly to the `standby` CPU-only QOS (see below) in that jobs in this QOS may be cancelled by the appearance of jobs in one of the other three GPU QOSs which require resources. This QOS allows you to submit long-running jobs on all GPU nodes without blocking access to those nodes for other users with short-running jobs. If you have long-running jobs which need a guaranteed long runtime, use `gpumedium` or `gpulong` instead.

#### `priority` and `standby` QOSs

These QOSs have been implemented by IT Services following discussions within the Modelers Council as to how best to cope with long queue times on the cluster during periods of high load. Both are able to run only in the `priority` partition.

QOS `priority` provides a maximum of 64 CPU cores in up to 5 concurrent jobs. Queue time for jobs in this QOS is expected to be under 5 minutes. This QOS is well suited to workflows requiring fast turnaround, for example test or debug runs, or to get resources ahead of project deadlines.

QOS `standby` and `standby_cancel` are *preemptable*. This means that the system may cancel jobs running here if the resources are required for new jobs arriving with a `priority` QOS. (See the Slurm documentation on preemption for more details.) Jobs in the `standby` are cancelled or requeued if the sbatch option `--requeue` is set. Jobs in the `standby_cancel` are cancelled.

> Jobs are cancelled by Slurm by sending two signals to the application. The first, SIGTERM, can be handled in code, and gives an application an opportunity to save its state before terminating. The second signal (SIGKILL) is sent a short time later (currently set to 5 minutes), and directly terminates the application if it's still running. For applications running via `srun`, you can use `scancel -s <SIGNAL_NAME>` to trigger the signal in your code. Please check with the application developers if such signal handling has been implemented already. If you would like to learn more about signal handling in this scenario, a good introduction can be found [here](https://hpc-unibe-ch.github.io/slurm/checkpointing.html).

To make best use of the `standby` QOS, your code should be able to restart from an arbitrary point in its execution. Although this is not essential it does mean that already-spent computing resources won't be wasted in redoing the run from the start. Preempted jobs will be automatically requeued, starting from the beginning, and using the same Job ID. Note that if you're using the Job ID to create unique output files, your job submission will need to be able to detect this and handle existing (old) output appropriately.

Automatic requeuing can be disabled on a per-job basis by adding the following line to your submit script:

`#SBATCH --no-requeue`

#### `io` QOS - jobs requiring internet access

The `io` QOS is intended for small, short-running jobs which require internet access (for example to download datasets from an external site) or to access software only available on these nodes. These jobs run on the login nodes, and as such are limited to 4 CPU cores and a maximum of 5 concurrent jobs (amongst all users).

To use this configuration, you must specify both QOS and partition in your submission script:

```bash
#SBATCH --partition=io
#SBATCH --qos=io
```

The limits for this QOS can be seen with the command:

```bash
sacctmgr show qos io
```

### Time limits

The Slurm scheduler uses an algorithm known as "backfilling" to start low priority jobs before higher priority jobs if they won't delay the start of the higher priority jobs. This assessment is done on the basis of the requested time limit.

> The priority of your jobs depends on a few factors, most notably how much compute time you've used recently, which QOS you're requesting and how many compute resources you need.

You may see much better job turnaround if you supply a realistic time limit for your job (with the `--time` parameter to `sbatch`) closer to your expected runtime rather than allowing it to default to the QOS runtime limit.

Acceptable time formats include "minutes", "minutes:seconds", "hours:minutes:seconds", "days-hours", "days-hours:minutes" and "days-hours:minutes:seconds".

For example, if you know your job will take at most 3 hours to run, you would submit to QOS `short`, but specify `--time=03:00:00`. (Also on the `sbatch` command line)

Using only a slightly shorter time limit than the default (e.g. `23:30:00` instead of `24:00:00`) will have almost no effect on the backfill scheduler. It would only have an effect on competing jobs scheduled at exactly the same time with the same priority.

### Node types

There are three node types on the cluster. The following table outlines their main characteristics:

| Type | CPU Architecture | socket | cores/sock. | cores/node | RAM/node | Nodes | Partition |
|---|---|---|---|---|---|---|---|
| CPU | AMD EPYC 9354 | 2 | 32 | 64 | 768GB | 3 | io |
| CPU | AMD EPYC 9554 | 2 | 64 | 128 | 768GB | 180 | standard |
| CPU | AMD EPYC 9554 | 2 | 64 | 128 | 768GB | 60 | priority |
| GPU | AMD EPYC 9554 | 2 | 64 | 128 | 1536GB | 12 | gpu |

## Fair Share

This section explains, step by step, why your job sometimes waits in the queue while somebody else's job starts immediately, and how to use the two commands `sprio` and `sshare` to find out what is going on.

### The basic idea

The cluster is a shared machine. There is never enough hardware for everybody at once, so jobs wait in a queue. The queue is **not** first-come-first-served. Instead, every waiting job gets a **priority** number, and Slurm always starts the waiting job with the highest priority first.

*Fair share* is the part of that priority number that answers one question:

> **How much of the cluster has your project used recently, compared to how much it is entitled to?**

- If your project has used **less** than its entitlement recently, your fair-share value goes **up**, your jobs get a higher priority and they start sooner.
- If your project has used **more** than its entitlement recently, your fair-share value goes **down**, your jobs get a lower priority and they wait longer.

So heavy users are slowed down automatically, and occasional users are not starved out by them. Nothing is ever taken away from you permanently: fair share only changes the *order* of the queue.

> Important: fair share does **not** reject your jobs and it does **not** give you a monthly quota that can run out. If the cluster is empty, your job starts right away no matter how much you have used.

### Who shares with whom

Every user belongs to one or more **accounts**. An account is normally a PIK project (`--account=<my account>` in your submit script). The accounts are arranged in a tree:

```
root                            the whole cluster
 |
 +-- project-a  (share 1)       a project account
 |     +-- user1
 |     +-- user2                <- you
 |
 +-- project-b  (share 1)       another project account
       +-- user3
```

Two things follow from this:

1. **All project accounts on the cluster currently get an equal share** (every account has `RawShares=1`). With around 200 accounts, each account is entitled to roughly 1/200 = 0.5% of the cluster over time.
2. **Usage is shared within your account.** If a colleague in your project runs thousands of core-hours, *your* jobs also get a lower priority, because fair share is computed for the project first. This is intentional - the project is the unit that shares.

### What counts as "usage"

Usage is measured in **CPU-seconds**: the number of cores your job was *allocated*, multiplied by how long it ran. Note that this counts the cores you **reserved**, not the cores you actually kept busy - a job that asks for 128 cores and uses only one is charged for all 128. Memory is not charged separately on this cluster.

Old usage is forgotten gradually, with a **half-life of 7 days** (`PriorityDecayHalfLife`). So:

| Time since the job ran | How much of it still counts |
|---|---|
| today | 100% |
| 1 week ago | 50% |
| 2 weeks ago | 25% |
| 4 weeks ago | ~6% |

In practice, a big run hurts your priority for about two weeks and is then essentially forgotten.

### The full priority formula

Fair share is the largest, but not the only, ingredient of a job's priority. Slurm multiplies each factor (a number between 0 and 1) by a weight and adds everything up:

```
Priority =   Age       * 10000
           + FairShare * 50000
           + JobSize   *  1000
           + QOS       * 10000
```

| Factor | Weight |
|---|---|
| `FairShare` | 50000 |
| `Age` | 10000 |
| `QOS` | 10000 |
| `JobSize` | 1000 |

What the four factors mean:

- **`FairShare`** - your account's recent usage compared with its entitlement (see above). By far the most important factor.
- **`Age`** - how long the job has already been waiting. Reaches its maximum after **1 day** (`PriorityMaxAge`) and then stops growing.
- **`QOS`** - a fixed bonus depending on the requested QOS (see the table below).
- **`JobSize`** - slightly favours jobs asking for many cores. Usually a very small contribution.

The QOS bonus is fixed per QOS and currently works out as follows (you can check it at any time with `sacctmgr show qos format=name%16,priority`):

| QOS | Points |
|---|---|
| `priority` | 10000 |
| `gpushort`, `gpupreempt` | 5000 |
| `medium`, `gpumedium` | 2500 |
| `standby`, `standby_cancel` | 1250 |
| `long`, `gpulong` | 500 |
| `io` | 250 |
| `short` (the default) | 0 |

The maximum possible priority is therefore about **71000**. You can always print the current weights yourself:

```bash
sprio -w
```

```
          JOBID PARTITION   PRIORITY       SITE        AGE  FAIRSHARE    JOBSIZE        QOS
        Weights                               1      10000      50000       1000      10000
```

Because the fair-share weight (50000) is five times the age weight (10000), **you cannot simply out-wait a user with a better fair share.** Waiting longer helps only up to one day.

### `sprio` - why is *this job* ranked where it is?

`sprio` shows the priority of **pending (waiting) jobs** and breaks it down into the individual factors. Running jobs are not shown, because their priority no longer matters.

Show the breakdown for your own waiting jobs:

```bash
sprio -u $USER
```

```
          JOBID PARTITION     USER   PRIORITY       SITE        AGE  FAIRSHARE    JOBSIZE        QOS
        1875732 standard     user1      19674          0      10000       4667          8       5000
        1875737 standard     user1      19674          0      10000       4667          8       5000
```

How to read the line for job `1875732`:

- `PRIORITY 19674` - the total. This is the number Slurm sorts the queue by.
- `AGE 10000` - the maximum, so this job has already been waiting a day or more. It will not gain any more priority from waiting.
- `FAIRSHARE 4667` - out of a possible 50000. This is the interesting part: the account has used a lot recently. **This is where the priority is being lost.**
- `QOS 5000` - out of a possible 10000, from the requested QOS.
- `JOBSIZE 8` - a small job, so almost nothing here.

Other useful forms:

```bash
sprio                        # all pending jobs on the cluster
sprio -j 1875732             # one specific job
sprio -l                     # long format: also shows USER, ACCOUNT and QOSNAME
sprio -n                     # show the factors as fractions between 0 and 1
                             # instead of already-weighted points
```

A quick way to see who is ahead of you, sorted by priority:

```bash
sprio | sort -k3 -nr | head -20
```

To see priorities together with the reason a job is waiting:

```bash
squeue -o "%.10i %.9P %.8u %.9q %.10Q %.20R" --states=PENDING
```

> If `squeue` gives the reason `Priority`, it means exactly this: other jobs have a higher priority number. `Resources` instead means your job is next in line and is only waiting for nodes to become free.

### `sshare` - what is my fair share?

`sshare` shows the accounting side: the shares, the accumulated usage and the resulting fair-share factor. Start with your own associations:

```bash
sshare -U
```

```
Account                    User  RawShares  NormShares    RawUsage  EffectvUsage  FairShare
-------------------- ---------- ---------- ----------- ----------- ------------- ----------
project-a                 user1          1    0.200000           0      0.000000   1.000000
project-b                 user1          1    0.500000   823008020      1.000000   0.093336
project-c                 user1          1    0.050000           0      0.000000   0.773389
```

This user is a member of three accounts and has a **different fair share in each one**. Submitting with `--account=project-a` would give a much better priority than submitting with `--account=project-b` - but only use accounts you are actually entitled to use for that work.

The columns:

- **`RawShares`** - the entitlement assigned by the administrators. All accounts currently have `1`, i.e. every project is treated equally.
- **`NormShares`** - the same thing as a fraction, normalised within one level of the tree. `0.5` on a user line means "one of two users in this account".
- **`RawUsage`** - accumulated, decayed usage in CPU-seconds (see the half-life above). Not a running total - it shrinks over time.
- **`EffectvUsage`** - the account's share of the usage of its parent in the tree.
- **`FairShare`** - **the result, between 0 and 1.** This is the number that goes into the priority formula. Higher is better.

`FairShare` is a **ranking**: Slurm sorts all users by how well their usage matches their entitlement, and turns the position in that list into a number between 0 and 1.

- `1.000000` - best-placed user(s) on the cluster; you have used much less than your share.
- `0.500000` - right in the middle of the field.
- `0.093336` - near the bottom; your account has used far more than its share recently.

And this is exactly how `sprio` and `sshare` fit together:

```
FairShare factor (sshare)  x  PriorityWeightFairShare  =  FAIRSHARE points (sprio)
     0.093336              x          50000            =        4667
```

More detail with the long format:

```bash
sshare -l -U
```

The extra column worth knowing is `LevelFS` = `NormShares / NormUsage`:

- `LevelFS > 1` - you have used **less** than your share. Good.
- `LevelFS < 1` - you have used **more** than your share.
- `LevelFS = inf` - no usage at all recorded (yet).

Other useful forms:

```bash
sshare                       # the whole tree: all accounts on the cluster
sshare -A <account>          # one account and its own usage
sshare -a -A <account>       # one account including every user in it
sshare -u user1,user2        # specific users
```

Use `sshare -a -A <account>` to find out **which colleague** in your project is using the shared budget:

```bash
sshare -a -A project-b
```

```
# example output
Account                    User  RawShares  NormShares    RawUsage  EffectvUsage  FairShare
-------------------- ---------- ---------- ----------- ----------- ------------- ----------
project-b                                1    0.004902   823008020      0.041565
 project-b               user1           1    0.500000   800000000      0.972000   0.093336
 project-b               user2           1    0.500000    23008020      0.028000   0.480000
```

Here `user1` has used almost the whole project's budget, which is why both users in the project now have a low `FairShare`.

### Seeing your usage in real numbers

`RawUsage` is decayed and therefore hard to interpret. For actual core-hours, use `sreport`:

```bash
sreport -t hourper cluster AccountUtilizationByUser start=2026-09-01 end=now account=project-b
```

```
# example output
  Cluster         Account     Login     Proper Name                 Used               Energy
--------- --------------- --------- --------------- -------------------- --------------------
   pikhpc       project-b                                 430342(2.34%)        1374366(0.01%)
   pikhpc       project-b     user1  Example User A        392000(2.13%)              0(0.00%)
   pikhpc       project-b     user2  Example User B         38342(0.21%)        1374366(0.01%)
```

`Used` is in CPU hours. Adjust `start=` and `end=` to the period you are interested in, and drop `account=` to see all the accounts you are a member of.

### What you can actually do about it

Fair share itself is not something you can tune - but these things help:

1. **Request a realistic `--time`.** The backfill scheduler can slot short jobs into gaps ahead of higher-priority jobs. This is often the single most effective change you can make. See Time limits.
2. **Do not ask for more cores than you need.** You are charged for every core you reserve, whether you use it or not, and that lowers your fair share for the next two weeks. The same goes for `--time`: a job that is cancelled at its time limit is charged for the whole time it ran.
3. **Use the right QOS.** The default QOS `short` gets no QOS bonus at all. For quick test or debug runs, `priority` has the largest bonus and jobs there are expected to start within about 5 minutes. See QOS.
4. **Use `standby` for work that is not urgent.** Jobs there may be preempted, but they can fill the cluster when it is otherwise idle.
5. **Submit early, do not hold jobs back.** Waiting in the queue costs you nothing and earns up to 10000 priority points over the first day.
6. **Spread large campaigns out over time** instead of submitting everything in one week. Because of the 7-day half-life, the same total amount of computing hurts your priority less if it is spread over a month.
7. **Talk to your project colleagues.** You share one fair-share budget, so coordinating big runs within the project helps everybody in it.

### Frequently asked questions

**My job says `Priority` and never moves. Is it stuck?** No. Run `sprio -j <jobid>`. If `AGE` is already at 10000 and `FAIRSHARE` is low, your account simply has a worse fair share than the accounts ahead of you. It will improve as your usage decays.

**I submitted two identical jobs, why does one start first?** Their priority is identical, so Slurm falls back on the job ID - the older job wins.

**Why did a job submitted after mine start before mine?** It very likely belongs to an account with a better fair share, or it requested a QOS with a higher bonus, or it was small enough to be backfilled into a gap.

**Does cancelling a pending job give me priority back?** Pending jobs cost no usage at all, so there is nothing to give back. Only jobs that actually ran count.

**My `FairShare` is 0. Am I blocked?** No. A very low fair share only puts you at the back of the queue. Your jobs still run, and your value recovers automatically as the decay works.

## Intel Compiler and AMD CPUs

The Intel compiler with the compiler options `-x...` may produce binaries not runnable on our AMD CPUs; `-xHost`,`-xCORE-AVX2`,`-xCORE-AVX512` are examples of such options. If you're using the Intel compiler, please avoid these options.

The error message looks like this:

> Please verify that both the operating system and the processor support Intel(R) AVX2, AVX512, or AVX-512F, AVX-512CD, AVX-512ER, AVX-512PF, AVX-512VL, and AVX-512BW instruction sets.

The solution is to replace the `-x...` options with `-ax...` options. For example, `-xCore-AVX2` becomes `-axCore-AVX2`. `-xHost` becomes `-march=native`.

## Simple Job Submission

### Submitting via a script

#### A simple submission script

A Slurm job submission script is simply a shell script with scheduler-specific directives (lines beginning `#SBATCH`) coming before any other commands. The job is submitted to the queuing system with the command `sbatch <scriptname>`, which allocates resources and starts the job on those resources. This is the recommended method.

The simplest job submission script looks like this (let's call the file `submit.sh`):

```bash
#!/bin/bash

#SBATCH --qos=short
#SBATCH --job-name=test_job
#SBATCH --account=<my account>
#SBATCH --output=test-%j.out
#SBATCH --error=test-%j.err
#SBATCH --chdir=/p/tmp/<your username>/<some directory>

hostname
```

> Quick tip: the parameters to `#SBATCH` can also be set on the `sbatch` command line at submit time and will override those in the script itself. This is useful in that you can run with different parameters without editing the submit script.

Let's look at each line and find out what's going on:

This line just sets this file up to be a Bash script:

```bash
#!/bin/bash
```

The next line selects the QOS:

```bash
#SBATCH --qos=short
```

The command `sacctmgr show qos` lists the available QOSs and their limits (principally allowed runtime and maximum number of CPUs). Choosing the QOS with the shorter runtime limits often means that your job will spend less time in the queue.

Next, we can give our job an easily-identifiable name, so we can distinguish between similar jobs in the queue:

```bash
#SBATCH --job-name=test_job
```

Now tell Slurm under which account, or project group, your job is to run:

```bash
#SBATCH --account=<myaccount>
```

To find out which accounts you belong to, run this: `sacctmgr show assoc where user=$USER`. You can omit this line and Slurm will use your default account (see `sacctmgr show user $USER`)

If you belong to several accounts, please use the most appropriate one for your runs. This helps us manage resources more effectively over time.

The next two lines set up the location for standard output and standard error from our program:

```bash
#SBATCH --output=test-%j.out
#SBATCH --error=test-%j.err
```

The `%j` in the filename is a placeholder for the job number. In this way, multiple runs of the same program will produce separate output files, distinguished by job number. See the section *filename pattern* in the manpage for `sbatch` for a full list (`man sbatch`). A useful filename pattern to use is `%x-%j`. This produces filenames of the format `jobname-jobid`.

If you leave out the `--error` line, both standard output and standard error will be sent to the file given by `--output`. If you don't use a job-specific identifier for the output files, multiple submissions of the same job will overwrite each other's output.

```bash
#SBATCH --chdir=/p/tmp/<your username>/<some directory>
```

The `--chdir` parameter tells Slurm where the output and error files are to be written. It is essential to use one of the parallel filesystems for major job output (more than a few megabytes), since `$HOME` filesystems (`/home/<your username>`) are not intended for the high load which can be generated by cluster jobs, which can disrupt operation of the entire cluster.

Make sure that the directory specified by `--chdir` exists before submitting your run, otherwise the job will silently fail.

`man sbatch` will give you more information about the options available to you.

#### Submitting the job

The script is submitted to be run on the cluster with the Slurm command sbatch thus:

```bash
$ sbatch submit.sh
Submitted batch job 303
```

The number reported here (`303`) is the job ID, which you can use in several Slurm commands to check the status of your job (`squeue`, `scontrol show job`, `sacct` etc.)

This will run one program (`hostname` in our simple script) on one CPU core.

### Submitting interactively

It's also possible to use `srun` interactively and submit exactly the same jobs as above. But note: interactive submission means your shell/terminal blocks until the job completes, including the time spent queuing! This may be OK for quick running jobs, but be aware your job will quit if the connection to the cluster drops during longer running jobs (e.g. you shut down your laptop). `Ctrl-C` (twice) will end the job. If your connection to the cluster fails, the job will end also.

Jobs can be run in the background with a trailing '&' (or `Ctrl-Z`, `bg`). The job will still be cancelled if the connection is lost.

```bash
srun --qos=short --job-name=test_job --output=test-%j.out --error=test-%j.err \
 --nodes=1 --ntasks-per-node=128 hostname
```

## Seeing the state of your jobs (`squeue`)

You can use the command `squeue` to see the current state of the job queue and `squeue -u $USER` to see only your own jobs.

```bash
JOBID PARTITION     NAME     USER ST  TIME  NODES NODELIST(REASON)

  529  standard sumprime   mbehr PD  0:00      1 (Priority)
  530  standard sumprime   mbehr PD  0:00      1 (Priority)
  527  standard sumprime   mbehr  R  0:02      1 csm14c181
```

JOBID is the unique number that identifies the job.

`ST` means State, here one job is `Running` and two are `PenDing`.

`NODELIST(REASON)` tells you either the nodes on which your job is running, or if it is not running, the reason for the current state is given in parentheses. Here, for example, jobs 529 and 530 cannot start because their priority is low.

Some common Job Reason Codes:

- `Resources` : your job is waiting for resources to become available
- `Priority` : one or more jobs in the queue have a higher priority
- `QOSGrpCpuLimit` : the maximum number of active CPUs for this QOS has been reached (i.e. the QOS is full)
- `Dependency` : your job is waiting for its predecessor to complete
- `Licenses` : your job is waiting for a license to become available (e.g. Matlab)
- `JobArrayTaskLimit` : if you're using job arrays, you have reached the limit of the number of tasks in an array. Slurm will wait for tasks to complete before starting these.

You can modify the output of `squeue` with the `--format` option (or the `$SQUEUE_FORMAT` environment variable) if you want to see fewer (or different) fields. See `man squeue`. Many Slurm commands take the `--format` option or environment variables to customise output.

## Cancelling jobs (`scancel`)

You can cancel a single job like this:

```bash
scancel <jobid>
```

If you want to cancel all your jobs at once, use `-u`:

```bash
scancel -u <username>
```

## Job Arrays

Job arrays provide a convenient way to submit many similar jobs with a single command. Each job in the array runs independently with its own task ID.

### Basic job array

```bash
#!/bin/bash
#SBATCH --qos=short
#SBATCH --job-name=my_array
#SBATCH --array=0-99
#SBATCH --output=output_%A_%a.out

# %A = master job ID, %a = array task ID
echo "Task ID: $SLURM_ARRAY_TASK_ID"

# Use the task ID to select input files or parameters
python process_data.py --input data_${SLURM_ARRAY_TASK_ID}.csv
```

### Limiting concurrent tasks

To avoid overloading resources or licenses, limit concurrent tasks with `%`:

```bash
#SBATCH --array=0-99%10    # Run at most 10 tasks simultaneously
```

### Non-contiguous arrays

```bash
#SBATCH --array=1,3,5,7,9           # Specific values
#SBATCH --array=1-100:2             # Every second value (1,3,5,...,99)
```

### Using array index in your code

**Python:**

```python
import os
task_id = int(os.environ.get('SLURM_ARRAY_TASK_ID', 0))
input_file = f"data_{task_id}.csv"
```

**R:**

```r
task_id <- as.integer(Sys.getenv("SLURM_ARRAY_TASK_ID"))
input_file <- sprintf("data_%d.csv", task_id)
```

**Bash:**

```bash
INPUT_FILES=(file1.txt file2.txt file3.txt file4.txt)
CURRENT_FILE=${INPUT_FILES[$SLURM_ARRAY_TASK_ID]}
```

## Parallel code

There are a few different models of code parallelism. Common ones are threaded (OpenMP, for example), multiprocess (where an application starts several independent workers, like in Python's `multiprocessing` or `dask` modules) and Message Passing Interface (MPI), typically implemented on our cluster with the Intel MPI library (sometimes via Python's `mpi4py` or R's `Rmpi` packages).

> MPI jobs are the **only** way to have parallel code span multiple compute nodes. Threaded and multiprocess codes are limited to single nodes, but can be combined with MPI (so-called hybrid model).

### MPI

MPI jobs are submitted using a) `--ntasks` or b) `--nodes` plus `--ntasks-per-node` parameters in the submission script. The run itself is started with `mpirun`.

Here is an example of an MPI job submission script:

```bash
#!/bin/bash

#SBATCH --qos=short
#SBATCH --job-name=sumprimes
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=128
#SBATCH --output=%x-%j.out
#SBATCH --error=%x-%j.err
#SBATCH --time=30

echo "------------------------------------------------------------"
echo "SLURM JOB ID: $SLURM_JOBID"
echo "$SLURM_NTASKS tasks"
echo "------------------------------------------------------------"
# shortcut $SLURM_NTASKS is equal to 512 (4*128)
# See http://slurm.schedmd.com/sbatch.html for a full list of 
# input and output variables

module load intel/oneAPI
mpirun $HOME/cluster-examples/sumprimes/mpi/sumprimes 0 10000000
```

This example makes use of `--time`, which tells Slurm that my job will need no more than 30 minutes to run (instead of the standard 24 hours for QOS `short`). This gives Slurm a hint to move this job to the top of the queue, if doing so won't delay the start of other jobs.

### mpi4py: MPI and Python

By default, Anaconda installs its own MPI library (MPICH), however at this stage we only support the Intel MPI library. To check which MPI library is being used, run `mpirun --version` on the command line. You should see an Intel-specific version something like this:

```bash
Intel(R) MPI Library for Linux* OS, Version 2021.14 Build 20240911 (id: b3fc682)
Copyright 2003-2024, Intel Corporation.
```

If not, you should re-install `mpi4py` from the Intel Anaconda channel: `conda install -c intel mpi4py`. This will ensure best performance of the MPI library.

For more on Python and `conda` on the cluster, see our documentation "Python on the cluster" and "Python Environments".

### Threaded or multiprocess jobs (e.g. OpenMP, Python's `multiprocessing`, `dask` or `joblib`)

Code running on multiple threads requires careful use of tasks and task placement to prevent overbooking and suboptimal performance.

A threaded job with N threads is regarded by Slurm as a 1-task, N CPUs-per-task job. We must then request N CPUs from Slurm on which to run the N threads. Thus:

```bash
# 1. make sure all cores we get are on one node 
#    jobs of this type may not span nodes.
#SBATCH --nodes=1  

# 2. we're a 1-task job
#SBATCH --ntasks=1

# 3. this is how many CPUs (threads) our task will need 
#SBATCH --cpus-per-task=4

# if you're using OpenMP for threading:
export OMP_PROC_BIND=true # make sure our threads stick to cores
export OMP_NUM_THREADS=4  # matches how many cpus-per-task we asked for

./my_threaded_program
```

> If you requested fewer CPU cores than threads you intend to run, the effect may not immediately be obvious. The most common effect is code running much slower than you expect.

### Layout of parallel jobs on nodes

Depending on your application, it may be desirable to either run as many tasks as possible on a node ("blocking", or "packed", for example in the case of parallel jobs with high process-to-process communication) or as few as possible ("non-blocking" or "non-packed", for example where processes need to share as little bandwidth to disk or memory with other processes as possible).

There are several ways to achieve this.

1. You don't care about process placement. This simply gets assigned CPUs from wherever they are available, possibly fully packed nodes, possibly as few as one CPU per node.

```bash
#SBATCH --ntasks=n
```

1. Blocked: You want to run on as few nodes as possible, to minimise the slower inter-node communication:

```bash
#SBATCH --nodes=N
#SBATCH --ntasks-per-node=128
```

This gives you an allocation of Nx128 CPUs.

1. Non-blocked: You want to run with a specific number of CPUs per node:

```bash
#SBATCH --nodes=N
#SBATCH --ntasks-per-node=M
```

Where 1<=M<=128. This gives you MxN CPUs in total. Note that this does not leave the remaining (128-M) CPUs unoccupied on the node. If you're using this to make use of the full disk or RAM bandwidth of the node, you will also need to request exclusive access to prevent other jobs (yours or another user's) from being allocated here:

```bash
#SBATCH --exclusive
```

## Requesting a GPU node

The cluster has 12 GPU nodes, each equipped with 4x NVIDIA Hopper H100 GPUs (80GB HBM3 each), 128 CPU cores, and 1.5TB RAM.

To access GPUs, use the Slurm generic resource argument ("GRES") and request the `gpu` partition:

```bash
#SBATCH --partition=gpu
#SBATCH --qos=gpushort
#SBATCH --gres=gpu:1
```

Replace `gpu:1` with `gpu:2`, `gpu:3`, `gpu:4` if you want to use more GPUs during the job.

*Do not forget to specify the number of GPUs with `--gres=gpu:<number>`, otherwise you allocate a node without GPUs.*

### GPU software stack

CUDA and related libraries are available as modules:

```bash
module load cuda/12.6.0
module load cudnn/9.5.1
```

### Checking GPU status

To see GPU utilization on a node (requires an interactive session or running job):

```bash
nvidia-smi
```

### PyTorch with GPU

```bash
module load anaconda/2025
conda create -n pytorch_gpu python=3.11
source activate pytorch_gpu
conda install pytorch pytorch-cuda=12.4 -c pytorch -c nvidia
```

Example GPU job script for PyTorch:

```bash
#!/bin/bash
#SBATCH --partition=gpu
#SBATCH --qos=gpushort
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=16
#SBATCH --mem=64G
#SBATCH --time=04:00:00
#SBATCH --job-name=pytorch_train
#SBATCH --output=%x-%j.out

module load anaconda/2025
source activate pytorch_gpu

python train_model.py
```

### TensorFlow with GPU

```bash
module load anaconda/2025
conda create -n tf_gpu python=3.11
source activate tf_gpu
pip install tensorflow[and-cuda]
```

### Multi-GPU jobs

For jobs using multiple GPUs on a single node:

```bash
#SBATCH --gres=gpu:4
#SBATCH --cpus-per-task=64
```

Ensure your code is set up for multi-GPU training (e.g., using PyTorch's `DistributedDataParallel` or TensorFlow's `MirroredStrategy`).

### Local NVMe scratch storage (GPU nodes only)

Each GPU node has a fast local NVMe scratch directory:

| Path | Capacity | Purpose |
|---|---|---|
| `/localscratch` | ~3 TB | Fast temporary storage for I/O-intensive GPU workloads |

This local storage is **significantly faster** than the parallel filesystem (`/p/tmp`) for random I/O and is ideal for:

- Training datasets that are read repeatedly
- Checkpoint files during training
- Temporary files with high I/O frequency

#### Usage example

```bash
#!/bin/bash
#SBATCH --partition=gpu
#SBATCH --qos=gpushort
#SBATCH --gres=gpu:1
#SBATCH --job-name=gpu_training
#SBATCH --output=%x-%j.out

# Create a job-specific directory on local scratch
LOCAL_SCRATCH="/localscratch/$USER/$SLURM_JOB_ID"
mkdir -p $LOCAL_SCRATCH

# Copy dataset to local scratch for faster access
cp -r /p/tmp/$USER/dataset $LOCAL_SCRATCH/

# Run training with data from local scratch
module load anaconda/2025
source activate pytorch_gpu
python train.py --data-dir $LOCAL_SCRATCH/dataset --checkpoint-dir $LOCAL_SCRATCH/checkpoints

# Copy results back to parallel filesystem
cp -r $LOCAL_SCRATCH/checkpoints /p/tmp/$USER/results_$SLURM_JOB_ID/

# Clean up local scratch
rm -rf $LOCAL_SCRATCH
```

> **Important**: Data on `/localscratch` is **not preserved** after your job ends and is **not shared** between nodes. Always copy important results back to `/p/tmp` or `/p/projects` before your job completes.

## Requesting more memory (RAM) on a node

It is possible to get access to extra memory on regular compute nodes, either by directly requesting an amount of memory or by requesting an allocation for extra CPU cores. In the first case:

```bash
#SBATCH --mem=6000
```

It's recommended to request just enough memory for your job to run (with some headroom), rather than the maximum possible. Requesting more resources generally leads to unnecessarily longer queue times and fewer resources for other users.

Note that requesting extra RAM or cores means that the extra resources are reserved to your account. This has implications for the priority of your jobs in the case that the machine is full (or close to full).

## Reserving resources in advance

Slurm includes a feature for reserving resources for a specified future time period. This can be as little as one CPU core, or up to many full compute nodes. A reservation identifies the resources available and the time period, or validity, of the reservation plus the user(s) and account(s) which may access it.

> Reservations can only be activated by a cluster administrator, and only in the case of impending project deadlines.

At the start of the reserved time period, the reservation becomes available for the named users, or members of the named account, to use in their job submissions. Accounting for unused reservations is made by dividing the resource allocation amongst the named users, or the named account. The reservation is booked as a used resource regardless of whether it ran jobs or not. At the end of the validity period the reservation is cancelled automatically.

To request a reservation, please email cluster support, with the following information:

- username(s) who shall be able to run jobs in this reservation OR the project group (`--account` in the Slurm submit script).
- Start time of the reservation
- Duration (or end time) of the reservation.
- Number of CPU cores (or number of nodes) required for the reservation, or other resources (e.g. GPU)

Once the reservation has been created (and you've been informed of the reservation name), jobs can be submitted to it with the command

```bash
sbatch --reservation=<reservation name> <submit script>
```

Until the reservation becomes active, however, jobs will remain in the "pending" state.

We sometimes use reservations to hold back resources in the case of hardware maintenance or other planned downtime. In these cases, the resources can still be used if your job will complete before the start of the reservation.

Full documentation on how reservations are used is in the Slurm documentation.

## When your run finishes (for advanced cases)

### Cleaning up

In some cases you may create temporary files or directories during the course of a run. Often these can grow quite large so it's good practice wherever possible to remove these files. This snippet shows a simple batch script (omitting the `#SBATCH` lines) which makes some temporary directories and cleans them up on exit:

```bash
# create a uniquely-named temporary directory, by default under /tmp
mytmpdir=$(mktemp -d)

# define a cleanup function to be called at the end of the run
cleanup() {
  rm -r $mytmpdir
  exit
}

# By trapping the EXIT signal, we also account for cases where the job 
# ends abnormally (e.g. is cancelled)
trap cleanup EXIT

# this is our "job" (it just sleeps)
sleep 60 &
wait
```

### Other post-run actions

As well as deleting temporary data, the `cleanup()` function can be used to write data, copy files etc. Be aware that these actions should complete within the `UnkillableStepTimeout` limit set by Slurm, otherwise the job will be forcibly removed.

`UnkillableStepTimeout` is currently set to 60 seconds (see `scontrol show config`)

## Slurm and Environment Modules

Slurm submissions can make use of `module load` commands at submit time and affect the environment under which the job runs. You can use this to set up the appropriate compiler, MPI environment or libraries prior to a run.

For more information about environment modules, see our associated documentation.

## Useful commands

Here's a list of the most important Slurm commands. `man` pages are also available for more information.

`sbatch <scriptname>` to submit a batch script `scriptname` to the queue for execution

`scontrol show job <id>` shows detailed information about a running or recently finished job.

`scancel` will cancel one or many jobs

`sinfo` to see partition and node information

`squeue` to see the state of the queue

`sacct` see information about your completed jobs. Use `--format` to see extra information, e.g.

```bash
sacct -j 200 --format=JobID,JobName,MaxRSS,Elapsed,AveCPUFreq
```

`sprio` shows the priority of your pending (waiting) jobs, broken down into its individual factors. See Fair Share.

`sshare` shows your fair-share entitlement and your account's recent usage. See Fair Share.

`sreport` produces usage reports in core-hours, e.g.

```bash
sreport -t hourper cluster AccountUtilizationByUser start=2026-09-01 end=now account=<my account>
```

`sview` is a graphical user interface to most of the commands above. If you have X Forwarding enabled on your SSH connection to the cluster, this will show you a nice window with the job, node and partition state information. Note: some admin commands (like draining a node, resuming a node, shutting down a node) are visible, but inactive for non-admin users.

### Customising your commands

Most Slurm commands take custom format strings, so you can change the output to suit what you're interested in. For example, with `squeue`, the following format string shows `TIME_LEFT`, `TIME_LIMIT` and `QOS` which don't appear in the standard `squeue` output: `export SQUEUE_FORMAT="%.18i %.9P %.8j %.8u %.2t %.10M %.10L %.10l %.6D %.6C %.8q %R"`

Or on the command line, or as an alias:

```bash
squeue --format="%.18i %.9P %.8j %.8u %.2t %.10M %.10L %.10l %.6D %.6C %.8q %R"
```

### Getting an interactive shell on a compute node.

It's possible to launch an interactive (Bash) shell on a compute node. This is useful for, for example, checking the existence of a particular system library (which may differ from what's on a login node) or looking at the capabilities of a node.

#### Example 1 - starting a login shell

`srun --pty bash`

Note that this locks resources (which show up in `squeue`) until you quit the shell manually or the time limit is reached.

#### Example 2 - running a command to check GPU parameters in a shell (no login shell opens and the command returns):

```bash
srun --pty --partition gpu --time=5 --gres=gpu:1 nvidia-smi
```

#### Example 3 - attach to a running job

This command will open a shell on a node in your job allocation. This might be useful for checking which job processes are running, or how much free memory there is.

```bash
srun --jobid=<job id of existing allocation> --pty bash
```

### To see the currently configured QOSs and their limits

```bash
sacctmgr show qos format=name%15,priority,maxwall,mintres,maxtres,grptres
```

or make an alias in your shell's profile:

```bash
alias ssq="sacctmgr show qos format=name,priority,maxwall,mintres,maxtres,grptres"
```

## VS Code Remote Development

Visual Studio Code can connect directly to the cluster for development, editing, and debugging.

### Setup

1. Install the **Remote - SSH** extension in VS Code
2. Open the Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`)
3. Select "Remote-SSH: Connect to Host..."
4. Enter `hpc.pik-potsdam.de` (or use your SSH config host alias)

### SSH config (recommended)

Add this to your `~/.ssh/config` for easier connections:

```
Host hpc
    HostName hpc.pik-potsdam.de
    User <your-username>
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
```

Then connect using just `hpc` as the hostname.

### Important notes

- VS Code runs on login nodes - do **not** run compute-intensive tasks directly
- Use VS Code for editing and light testing, submit heavy work as batch jobs
- Consider using the Jupyter extension for interactive notebooks (see below)

## Jupyter Notebooks

You can run Jupyter notebooks on compute nodes and access them from your local browser.

### Method 1: Interactive session with port forwarding

**Step 1:** Start an interactive job and launch Jupyter:

```bash
# On the cluster
srun --qos=short --time=04:00:00 --cpus-per-task=4 --mem=16G --pty bash

# Note the node name (e.g., csm14c181)
hostname

# Load modules and start Jupyter
module load anaconda/2025
source activate myenv
jupyter notebook --no-browser --ip=0.0.0.0 --port=8888
```

**Step 2:** Create an SSH tunnel from your local machine:

```bash
# On your local machine
ssh -L 8888:<nodename>:8888 hpc.pik-potsdam.de
```

**Step 3:** Open `http://localhost:8888` in your browser and use the token shown in the terminal.

### Method 2: Batch job with Jupyter

Create a script `jupyter_job.sh`:

```bash
#!/bin/bash
#SBATCH --qos=short
#SBATCH --time=08:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --output=jupyter-%j.log

module load anaconda/2025
source activate myenv

# Get the node name and a random port
NODE=$(hostname)
PORT=$(shuf -i 8000-9000 -n 1)

echo "=========================================="
echo "Jupyter is running on: $NODE:$PORT"
echo "Create tunnel with: ssh -L $PORT:$NODE:$PORT hpc.pik-potsdam.de"
echo "Then open: http://localhost:$PORT"
echo "=========================================="

jupyter notebook --no-browser --ip=0.0.0.0 --port=$PORT
```

Submit with `sbatch jupyter_job.sh`, then check the log file for connection instructions.

### JupyterLab

Replace `jupyter notebook` with `jupyter lab` for the more modern JupyterLab interface.

## Apptainer

Apptainer (formerly known as Singularity) is a container platform designed for high-performance computing (HPC) environments. It allows users to run applications in a portable and reproducible manner.

### Basic Usage

To pull a container image from an external repository like Docker Hub:

```bash
apptainer pull docker://alpine
```

This command will download the Alpine Linux container image and save it as alpine_latest.sif.

Run a command within the container:

```bash
apptainer exec alpine_latest.sif echo "Hello from Alpine!"
```

Get an interactive shell within the container:

```bash
apptainer shell alpine_latest.sif
```

You can create your own container from a definition file. Below is an example definition file named example.def:

```bash
Bootstrap: docker
From: ubuntu:20.04

%post
    apt-get update && apt-get install -y python3
```

Build the container image:

```bash
apptainer build example.sif example.def
```

Run the custom container with a Python script:

```bash
apptainer exec example.sif python3 -c "print('Hello from the custom container!')"
```

To create a writable directory in the container that persists across runs:

```bash
apptainer build --sandbox my_writable_container example.def
apptainer exec --writable my_writable_container touch /newfile
```

## Software and Programming Languages

### Python / Conda

The recommended Conda distribution on the cluster is Miniforge. For compatibility with existing workflows, load it with:

```bash
module load anaconda/2025
```

#### Anaconda licensing and module changes

We no longer provide Anaconda modules due to licensing restrictions. The Anaconda Terms of Service prohibit use in many institutional and HPC environments without a paid license, and continuing to offer Anaconda would place the cluster in violation of those terms. As a result, the `anaconda/2023` module has been removed and cannot be reinstated.

A local installation of Anaconda is also not feasible on this system, as it would be subject to the same licensing restrictions.

Instead, we provide Miniforge, a fully open-source Conda distribution based on `conda-forge` that does not carry these licensing limitations. In most cases, it is a drop-in replacement for Anaconda environments. For convenience, the Lmod module `anaconda/2025` is still available, but it loads Miniforge.

#### Creating a conda environment

```bash
# Create a new environment
conda create -n myenv python=3.12

# Activate it
source activate myenv

# Install packages
conda install numpy pandas scipy matplotlib
```

> **Note**: Use `source activate` instead of `conda activate` on the cluster. See our detailed Python documentation for more information.

#### Using Python in a job script

```bash
#!/bin/bash
#SBATCH --qos=short
#SBATCH --job-name=python_job
#SBATCH --output=%x-%j.out
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4

module load anaconda/2025
source activate myenv

python my_script.py
```

#### MPI with Python (mpi4py)

For parallel Python jobs using MPI, install `mpi4py` from the Intel channel:

```bash
conda install -c intel mpi4py
```

This ensures compatibility with the Intel MPI library used on the cluster. See our detailed Python documentation for more information.

### R

R is available on the cluster:

```bash
module load R/4.3.2
R
```

#### R in a batch job

```bash
#!/bin/bash
#SBATCH --qos=short
#SBATCH --job-name=r_job
#SBATCH --output=%x-%j.out
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4

module load R/4.3.2

# Set number of threads for parallel operations
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

Rscript my_analysis.R
```

#### Installing R packages

R packages can be installed to your home directory:

```r
install.packages("tidyverse", repos="https://cloud.r-project.org")
```

### Julia

Julia is available in multiple versions:

```bash
module load julia/1.11.7
julia
```

#### Julia tips

Set `JULIA_NUM_THREADS=N` or `BLAS.set_num_threads(N)` if you're using package LinearAlgebra. This package tries to run threads on half the visible CPU cores, not just those allocated by Slurm. Therefore `N` should match the number of CPU cores allocated by Slurm. If you want to do multithreaded BLAS, set `--ntasks=1` and `--cpus-per-task=N` in Slurm, and `BLAS.set_num_threads(N)`

Check other packages for potential threading under the hood.

### Python (Snakemake)

Snakemake thread count doesn't always match the Slurm-allocated core count. Check the documentation, or look at the [documentation](https://snakemake.readthedocs.io/en/stable/executing/cluster.html) for Slurm clusters. It may be worth creating a configuration profile of your own.

## Best Practices

### Resource requests

1. **Request realistic resources**: Overestimating leads to longer queue times and wastes resources. Use `sacct` to check actual usage of past jobs: `sacct -j <jobid> --format=JobID,Elapsed,MaxRSS,MaxVMSize,TotalCPU`
2. **Use the right QOS**: Jobs needing only a few hours should use `short`, not `medium` or `long`.
3. **Set accurate time limits**: Use `--time` to specify expected runtime. This helps the scheduler backfill your job earlier.
4. **Request memory appropriately**: Default is ~6GB per core. Only use `--mem` if you need significantly more or less.

### Job efficiency

Check job efficiency after completion:

```bash
seff <jobid>
```

This shows CPU and memory efficiency. Aim for:

- CPU efficiency > 70%
- Memory efficiency > 50%

### File I/O

1. **Work from `/p/tmp`**: Use the parallel filesystem for job data
2. **Avoid many small files**: Pack them into archives or use HDF5/NetCDF
3. **Don't write to `$HOME` from jobs**: This can slow down the entire cluster

### Checkpointing

For long-running jobs, implement checkpointing to save progress periodically:

```python
# Example: Save checkpoint every N iterations
if iteration % checkpoint_interval == 0:
    torch.save({
        'epoch': epoch,
        'model_state': model.state_dict(),
        'optimizer_state': optimizer.state_dict(),
    }, f'checkpoint_{iteration}.pt')
```

This is especially important for jobs in `standby` or `gpupreempt` QOSs, which may be cancelled.

### Login node etiquette

- **Do not** run compute-intensive tasks on login nodes
- Keep interactive sessions short
- Use `srun --pty bash` for interactive compute work
- Compiling is OK, but large parallel builds should be done in jobs

## Energy Accounting

The cluster records the electrical energy consumed by your jobs. This is useful for estimating the carbon footprint of your computations, comparing the efficiency of different code versions or input sizes, and reporting energy use in publications. There are two complementary ways to obtain energy data: via Slurm's built-in accounting (recommended for normal users) and, for low-level measurements, directly via the kernel's `sysfs` power-capping (RAPL) interface.

### Energy accounting with Slurm

Slurm collects energy data automatically. On this cluster the energy is sampled at the **node level** via the server's IPMI/BMC sensors and stored as a tracked resource (TRES) in the accounting database. No changes to your submission script are required - every job already has its energy recorded.

The relevant site configuration (visible via `scontrol show config`) is:

| Setting | Value | Meaning |
|---|---|---|
| `AcctGatherEnergyType` | `acct_gather_energy/ipmi` | Energy is read from the node's baseboard management controller (BMC) |
| `AcctGatherNodeFreq` | `30 sec` | The energy sensor is polled every 30 seconds |
| `AccountingStorageTRES` | `...,energy,...` | Energy is stored as a trackable resource per job |

#### Viewing energy use of finished jobs

Use `sacct` with the `ConsumedEnergy` (human-readable, in joules with SI prefixes) and/or `ConsumedEnergyRaw` (raw value in joules) fields:

```bash
sacct -j <jobid> --format=JobID,JobName,Elapsed,ConsumedEnergy,ConsumedEnergyRaw
```

Example output:

```bash
JobID            JobName    Elapsed ConsumedEnergy ConsumedEnergyRaw
------------ ----------- ---------- -------------- -----------------
414912               LLM 27-02:30:43          3.54G        3540542420
840649       interactive   05:41:11         40.29M          40291590
841640       interactive   00:11:31        910.24K            910240
```

Here `3.54G` means roughly 3.54 gigajoules (983 kWh) and `40.29M` means 40.3 megajoules (11.2 kWh). To convert joules to kilowatt-hours, divide by 3,600,000:

```bash
# kWh = ConsumedEnergyRaw / 3.6e6
echo "scale=3; 3540542420 / 3600000" | bc
```

> Use `sacct -X` to show one line per job (the job allocation) rather than separate lines for each job step.

#### Viewing energy use of running jobs

For a job that is still running, use `sstat`:

```bash
sstat -j <jobid> --format=JobID,ConsumedEnergy,ConsumedEnergyRaw
```

#### Caveats and interpretation

Energy accounting is convenient but has important limitations you should understand before drawing conclusions:

- **Whole-node measurement.** The IPMI sensor reports the power draw of the **entire physical node** (CPUs, memory, fans, NICs, etc.), not just your process. The value is attributed to the job(s) running there.
- **Shared nodes split the reading.** If several jobs share a node, the node's energy is divided between them, so a single job's figure on a shared node is an estimate, not an exact measurement. For accurate, reproducible energy figures, run on a dedicated node using `#SBATCH --exclusive`.
- **Sampling resolution.** Because the sensor is polled only every 30 seconds (`AcctGatherNodeFreq`), very short jobs (seconds) yield coarse or near-zero readings. Energy figures are most meaningful for jobs running at least several minutes.
- **Idle baseline included.** A node's baseline (idle) power is included in the reading, so even a lightly loaded job will accrue energy proportional to its wall-clock time.

### Energy accounting using sysfs (RAPL) files

For fine-grained, code-level measurements you can read the CPU's energy counters directly from the Linux power-capping (RAPL - *Running Average Power Limit*) interface under `/sys/class/powercap`. Unlike the IPMI value above, RAPL reports the energy consumed by the **processor package(s)** (and sub-domains such as cores), not the whole node.

#### The powercap hierarchy

Each socket (processor package) is exposed as a top-level RAPL *zone*, with optional nested *sub-zones* for sub-domains. On a dual-socket compute node you will typically see:

```bash
$ ls -l /sys/class/powercap/
intel-rapl:0     ->  package 0 (socket 0)
intel-rapl:0:0   ->  a sub-domain of socket 0 (e.g. "core")
intel-rapl:1     ->  package 1 (socket 1)
intel-rapl:1:0   ->  a sub-domain of socket 1 (e.g. "core")
```

> The directories are named `intel-rapl` for historical reasons; this is the generic kernel power-capping framework and is used on our AMD EPYC nodes as well. The `name` file inside each zone tells you what it represents (e.g. `package-0`, `core`).

Each zone provides these files:

| File | Meaning |
|---|---|
| `name` | The domain name, e.g. `package-0`, `core` |
| `energy_uj` | Cumulative energy consumed since boot, in **microjoules** |
| `max_energy_range_uj` | The maximum counter value; the counter **wraps around** to 0 after this |

#### Permissions

On the **compute nodes** the `energy_uj` files are readable by every user, so you can take RAPL measurements directly from within your jobs:

```bash
$ cat /sys/class/powercap/intel-rapl:0/energy_uj
12345678901
```

On the **login nodes**, however, these files are restricted to root (`-r-------- 1 root root`), so a read there fails with `Permission denied`. This is intentional: high-resolution energy readings can be used as a side channel, and the login nodes are shared interactive servers. Since RAPL is meant for measuring your *own* workload, do this inside a job on a compute node anyway:

```bash
# Read RAPL from a compute node, e.g. via an interactive allocation
srun --qos=short --time=10 --cpus-per-task=1 \
  cat /sys/class/powercap/intel-rapl:0/energy_uj
```

> For everyday per-job energy figures, the Slurm accounting method (`sacct`) above is usually sufficient and requires no extra effort. Reach for the `sysfs`/RAPL method when you need package-level resolution or measurements around a specific code section.

#### Measuring an interval (handling counter wraparound)

The counter is **monotonic but wraps** when it exceeds `max_energy_range_uj` (here 65,532 J per socket). To measure the energy used by a command, read the counter before and after and compute the delta, accounting for a possible wrap and summing over both sockets:

```bash
#!/bin/bash
# Measure package energy consumed by a command via RAPL sysfs.
# Run this inside a job on a compute node, where energy_uj is readable by all users.

zones=(/sys/class/powercap/intel-rapl:0 /sys/class/powercap/intel-rapl:1)

read_total() {
    local sum=0
    for z in "${zones[@]}"; do
        sum=$(( sum + $(cat "$z/energy_uj") ))
    done
    echo "$sum"
}

# Sum of the wrap limits across all packages
max_total=0
for z in "${zones[@]}"; do
    max_total=$(( max_total + $(cat "$z/max_energy_range_uj") ))
done

start=$(read_total)

# ---- the workload you want to measure ----
./my_program
# ------------------------------------------

end=$(read_total)

# Correct for a counter wrap during the interval
delta=$(( end - start ))
if (( delta < 0 )); then
    delta=$(( delta + max_total ))
fi

# energy_uj is in microjoules: convert to joules and kWh
echo "Package energy: $(echo "scale=3; $delta / 1000000" | bc) J"
echo "             = $(echo "scale=6; $delta / 3600000000000" | bc) kWh"
```

> Because the counter can wrap roughly every 65 kJ per socket (a matter of minutes under load), for long measurements you must poll `energy_uj` frequently enough to detect each wrap, or simply rely on Slurm's accounting which handles this for you.

## Slurm REST API

In addition to the command-line tools, the cluster exposes Slurm's functionality through a RESTful HTTP API (`slurmrestd`). This is useful for programmatic job submission and monitoring from scripts, web services, or workflow engines, without needing an SSH session for every action.

The API is reachable at:

```
https://hpcsapi.pik-potsdam.de
```

### API versions and the OpenAPI specification

The API is versioned. The current supported version is **`v0.0.43`**; older versions (e.g. `v0.0.40`) have been removed. There are two endpoint namespaces:

| Path prefix | Backed by | Purpose |
|---|---|---|
| `/slurm/v0.0.43/...` | `slurmctld` | Live cluster operations: submit, query and cancel jobs, node/partition state |
| `/slurmdb/v0.0.43/...` | `slurmdbd` | Accounting database: historical jobs, associations, QOS, energy data |

The full, machine-readable list of endpoints is served from the OpenAPI specification at `/openapi/v3`, which you can use to generate a client in your language of choice.

### Authentication (JWT)

The API uses **JWT (JSON Web Token)** authentication. You generate a token on a login node with `scontrol token` and send it, together with your username, in two HTTP headers:

| Header | Value |
|---|---|
| `X-SLURM-USER-NAME` | your cluster username (`$USER`) |
| `X-SLURM-USER-TOKEN` | the JWT from `scontrol token` |

Generate a token (default lifespan is short; extend it with `lifespan=<seconds>`):

```bash
# Sets the SLURM_JWT environment variable for the current shell
export $(scontrol token lifespan=3600)
echo $SLURM_JWT
```

> Treat the token like a password: anyone holding it can act as you until it expires. Prefer short lifespans, and do not commit tokens to scripts or version control.

### Examples

**Ping the controller** (a good first connectivity/auth test):

```bash
export $(scontrol token lifespan=600)

curl -s \
  -H "X-SLURM-USER-NAME: $USER" \
  -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  https://hpcsapi.pik-potsdam.de/slurm/v0.0.43/ping
```

A successful response confirms the controllers are up:

```json
{
  "pings": [
    { "hostname": "cws01", "pinged": "UP", "responding": true, "mode": "primary",  "primary": true  },
    { "hostname": "cws02", "pinged": "UP", "responding": true, "mode": "backup",   "primary": false }
  ]
}
```

**List your jobs** from the accounting database (including energy, by adding `--format`-equivalent fields client-side or parsing the JSON):

```bash
curl -s \
  -H "X-SLURM-USER-NAME: $USER" \
  -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  "https://hpcsapi.pik-potsdam.de/slurmdb/v0.0.43/jobs?users=$USER"
```

**Query the live queue / a single job**:

```bash
# All queued/running jobs
curl -s -H "X-SLURM-USER-NAME: $USER" -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  https://hpcsapi.pik-potsdam.de/slurm/v0.0.43/jobs

# A specific job
curl -s -H "X-SLURM-USER-NAME: $USER" -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  https://hpcsapi.pik-potsdam.de/slurm/v0.0.43/job/<jobid>
```

**Submit a job.** Send a JSON document describing the job to the `job/submit` endpoint:

```bash
curl -s -X POST \
  -H "X-SLURM-USER-NAME: $USER" \
  -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  -H "Content-Type: application/json" \
  https://hpcsapi.pik-potsdam.de/slurm/v0.0.43/job/submit \
  -d '{
        "job": {
          "name": "rest-test",
          "qos": "short",
          "account": "<my account>",
          "partition": "standard",
          "current_working_directory": "/p/tmp/'"$USER"'",
          "environment": { "PATH": "/bin:/usr/bin" },
          "tasks": 1,
          "time_limit": 10
        },
        "script": "#!/bin/bash\nhostname\n"
      }'
```

The response contains the new `job_id`, which you can then poll via the `job/<jobid>` endpoint above.

**Cancel a job**:

```bash
curl -s -X DELETE \
  -H "X-SLURM-USER-NAME: $USER" \
  -H "X-SLURM-USER-TOKEN: $SLURM_JWT" \
  https://hpcsapi.pik-potsdam.de/slurm/v0.0.43/job/<jobid>
```

> The JSON schema for job submission (and every other endpoint) is defined precisely in the OpenAPI spec at `/openapi/v3`. When in doubt about field names or structure, consult that spec or the upstream [Slurm REST API documentation](https://slurm.schedmd.com/rest_api.html).

## Troubleshooting

### Job won't start - checking why

```bash
# See pending reason
squeue -u $USER

# Common reasons:
# - Priority: Higher priority jobs ahead (see the Fair Share section)
# - Resources: Waiting for nodes to free up
# - QOSMaxCpuLimit: QOS limit reached
# - Dependency: Waiting for another job
```

### Job started but failed immediately

Check the error file:

```bash
cat <jobname>-<jobid>.err
```

Common causes:

- Module not loaded
- File/directory not found
- Out of memory (OOM)

### Out of Memory (OOM) errors

If your job is killed with `Killed` or `oom-kill`:

1. Check actual memory usage of past jobs: `sacct -j <jobid> --format=MaxRSS`
2. Request more memory: `#SBATCH --mem=64G`
3. Or reduce memory usage in your code (smaller batch sizes, streaming data)

### Job hangs / no output

1. Check if it's actually running: `squeue -j <jobid>`
2. Attach to the job: `srun --jobid=<jobid> --pty bash top -u $USER`
3. Common causes:
    - Waiting for I/O
    - Deadlock in parallel code
    - Waiting for network resource

### Module conflicts

If you see library errors after loading modules:

```bash
module purge
module load <only-what-you-need>
```

### Permission denied on /p/tmp

Create your directory first:

```bash
mkdir -p /p/tmp/$USER
```

### SSH connection drops

Add to your `~/.ssh/config`:

```
Host hpc
    HostName hpc.pik-potsdam.de
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

## Commonly encountered error messages

```bash
login01:UCM:4835:1297c700: 527 us(527 us): open_hca: device mlx5_0 not found
[0] MPI startup(): dapl fabric is not available and fallback fabric is not enabled
```

This error occurs when trying to run MPI code on the login nodes (`login[01-03]`) in the default configuration. The network devices available to MPI are different on login and compute nodes, so setting MPI to run via shared memory will fix this error:

```bash
export I_MPI_FABRICS=shm:shm
```

Note: the login nodes are not intended for production runs of MPI (or any other code), so this workaround is intended for short, small-scale test runs only.

To reset to the default configuration for the compute nodes:

```bash
export I_MPI_FABRICS=shm:ofi
```

## Known issues

For issues relating to the Intel tools (compilers, MPI etc.) please see our [Intel-specific documentation](https://www.pik-potsdam.de/en/institute/about/it-services/hpc/user-guides/slurm/intel-compilers-and-tools).

### Parallel IO (MPI-IO) stalls

Under some circumstances, MPI-IO to the parallel filesystems (`/p/tmp`, `/p/projects`) may stall, without any warning or error messages. This can be circumvented by explicitly specifying the filesystem type to MPI:

```bash
I_MPI_EXTRA_FILESYSTEM=on
I_MPI_EXTRA_FILESYSTEM_LIST=gpfs
```

## Getting Help

If you have problems, need advice or have questions about running jobs on the cluster, open a ticket. If you're getting errors or unexpected behaviour with runs, please let us know the Job ID and where we can see the submission script and any output or errors files. Please send us as much detail as you can about observed vs. expected behaviour.

There is also a set of FAQs on our ticket system (OTRS) covering a few common questions [here](https://otrs.pik-potsdam.de/otrs/customer.pl?Action=CustomerFAQExplorer;CategoryID=3)

## References

### Official Slurm documentation:

[http://slurm.schedmd.com/documentation.html](http://slurm.schedmd.com/documentation.html)

### PIK GitLab repository of cluster example code and scripts

[https://gitlab.pik-potsdam.de/mbehr/cluster-examples](https://gitlab.pik-potsdam.de/mbehr/cluster-examples)

