# 03 — Running jobs with PBS

← [02 — Linux survival kit](02-linux-survival-kit.md) · Next: [04 — The project data](04-the-project-data.md)

---

Everything real we do runs as a **job**: a small text file describing the work, handed
to the scheduler, executed later on a compute node.

## Why not just run the command?

You could type `fastqc somefile.fq.gz` on the login node and it would work. You must not,
because:

- The login node is shared by everyone at SRCE. Heavy work there slows down dozens of people.
- If your connection drops, your command dies with it. A job survives — you can shut your laptop.
- A job gets guaranteed CPUs and memory. On the login node you are competing for scraps.
- You get a log file, so in six months you can prove what you ran.

**Rule of thumb:** if it takes more than a few seconds or reads more than a few hundred
MB, it is a job.

## Anatomy of a job script

A job script is an ordinary shell script with special comment lines at the top that PBS
reads. Here is the whole thing, annotated:

```bash
#!/bin/bash
#PBS -N md5_batch5
#PBS -q cpu
#PBS -l walltime=06:00:00
#PBS -l select=1:ncpus=1:mem=4gb
#PBS -M lguerrer@irb.hr
#PBS -m bea
#PBS -j oe

cd "$PBS_O_WORKDIR"

module load scientific/FastQC/0.12.1

# ... the actual work goes here ...
```

| Line | Meaning |
|---|---|
| `#!/bin/bash` | This file is a bash script. Always the first line. |
| `-N md5_batch5` | Job name. Shows up in `qstat` and in the log file name. Keep it short and descriptive. |
| `-q cpu` | Which queue. See below. |
| `-l walltime=06:00:00` | Maximum real time (HH:MM:SS). **If the job exceeds it, PBS kills it dead.** |
| `-l select=1:ncpus=1:mem=4gb` | 1 node, 1 CPU core, 4 GB RAM. |
| `-M ...@irb.hr` | Where to email. |
| `-m bea` | Email at **b**egin, **e**nd, and **a**bort. Use `-m ea` once the novelty wears off. |
| `-j oe` | Join normal output and error messages into one log file. Always want this. |

The `#PBS` lines look like comments to bash, which is exactly why they work: bash ignores
them, PBS reads them.

**`cd "$PBS_O_WORKDIR"`** — a job starts in your home directory, not where you submitted
it from. `$PBS_O_WORKDIR` is a variable PBS sets to "the folder the job was submitted
from". This line puts you back there. Quotes for the usual reason.

**`module load`** — the compute node starts with no software loaded. Whatever the job
needs, the job must load itself.

## Choosing walltime, CPUs and memory

New users get this wrong in both directions. The trade-off:

- **Ask for too little** → the job is killed halfway and you wasted the time it did run.
- **Ask for too much** → you wait longer in the queue, because the scheduler has to find
  a big enough gap.

Practical advice:

- **Walltime:** estimate, then double it. Overshooting costs you queue time; undershooting
  costs you the whole job. For a first run of something new, be generous.
- **CPUs:** only ask for more than 1 if the program can actually use them (it will have a
  `-t`, `-p` or `--threads` option). `md5sum` cannot, so we ask for 1. `fastp` and
  `HISAT2` can, so we ask for 8–10.
- **Memory:** most steps here need only a few GB. Genome indexing and assembly need a lot
  (30–100 GB). When unsure, start at 8 GB and look at what the job actually used.

The queues on Padobran:

| Queue | Use it for |
|---|---|
| `cpu` | Almost everything. The default choice. |
| `cpu_30` | Longer jobs (up to 30 days) — assembly, big genome indexing |
| `bigmem` | Jobs that need unusual amounts of RAM |

## The four commands you need

```bash
qsub myscript.sh        # submit. Prints a job ID like 123456.padobran
qstat -u $USER          # what are my jobs doing?
qdel 123456             # cancel a job
qstat -f 123456         # everything PBS knows about one job
```

In `qstat`, the **S** column is the status:

- `Q` — queued, waiting for a free slot
- `R` — running
- `E` — exiting, nearly done
- `F` — finished (only shown with `qstat -x`)

To see finished jobs, including how long they actually took and how much memory they
really used:

```bash
qstat -x -u $USER
```

That last one is how you learn to request the right resources next time.

## Reading the output

Because we used `-j oe`, PBS writes one log file into the folder you submitted from,
named after the job:

```
md5_batch5.o123456
```

`o` for output, then the job number. Read it with `less`:

```bash
less md5_batch5.o123456
```

**Always read the log, even when the job "worked".** A job that finishes is not a job
that succeeded — a program can fail, print an error, and still exit politely. The log is
where the truth is.

## Your first job

Make a file called `hello.sh` in your home directory:

```bash
cd ~
nano hello.sh
```

Type this in (Ctrl+O, Enter, Ctrl+X to save and quit):

```bash
#!/bin/bash
#PBS -N hello
#PBS -q cpu
#PBS -l walltime=00:05:00
#PBS -l select=1:ncpus=1:mem=1gb
#PBS -j oe

cd "$PBS_O_WORKDIR"

echo "Hello from compute node: $(hostname)"
echo "The date is: $(date)"
echo "I am running as user: $USER"
echo "I was submitted from: $PBS_O_WORKDIR"
```

Submit it, watch it, read it:

```bash
qsub hello.sh
qstat -u $USER
ls
less hello.o*
```

The hostname it prints will be a compute node, not `padobran`. That is the whole point —
your commands ran on a different machine.

## Habits worth building now

1. **Name jobs meaningfully.** `md5_batch5` tells you something in a week. `test2` does not.
2. **One step, one script, one folder.** Do not build a single script that does trimming
   and mapping and counting. When something breaks — and it will — you want to rerun one
   step, not all of them.
3. **Absolute paths inside scripts.** Always.
4. **Never overwrite the raw data.** Read from `Asellidae/`, write to your own output folder.
5. **Test on one sample before submitting 65.** Ten minutes of caution saves a day of queue time.
6. **Keep the scripts in the git repository**, not only on the server. In six months the
   scripts are the only record of what you actually did.

---

Next: [04 — The project data](04-the-project-data.md)
