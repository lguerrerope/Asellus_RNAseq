# 99 — Command cheat sheet

← [Back to README](../README.md)

For when you know what you want but forgot how to spell it.

---

## Connect

```bash
ssh yourusername@padobran.srce.hr     # log in
exit                                   # log out (or Ctrl+D)
```

## Move around

```bash
pwd                  # where am I
ls -lh               # what is here, with readable sizes
cd /some/path        # go there
cd ..                # up one level
cd ~                 # home
cd -                 # back to where I just was
```

**Tab** completes names · **↑** recalls the last command · **Ctrl+C** cancels

## Look at files

```bash
less file.txt              # scroll (q to quit, /word to search)
head -20 file.txt          # first 20 lines
tail -20 file.txt          # last 20 lines
wc -l file.txt             # count lines
zcat file.fq.gz | head -4  # first read of a compressed FASTQ
```

## Find things

```bash
find . -name "*.fq.gz"            # find files by name
find . -name "*.fq.gz" | wc -l    # ... and count them
grep "FAILED" results.txt         # find lines containing a word
grep -c ": OK$" results.txt       # count matching lines
du -sh */                         # size of each folder
df -h /beegfs                     # free space on the disk
```

## Files and folders

```bash
mkdir -p path/to/newfolder   # create (‑p makes parents too)
cp file1 file2               # copy       (‑r for folders)
mv old new                   # rename / move
rm file                      # DELETE — permanent, no undo
chmod +x script.sh           # make a script executable
```

## Edit

```bash
nano file.sh     # Ctrl+O Enter = save · Ctrl+X = quit · Ctrl+W = search
```

Stuck in vim by accident? `Esc` then `:q!` then Enter.

## Software modules

```bash
module avail                            # everything installed
module avail 2>&1 | grep -i fastqc      # search
module load scientific/FastQC/0.12.1    # load
module list                             # what is loaded
module purge                            # unload all
```

## Jobs

```bash
qsub script.sh          # submit — prints a job ID
qstat -u $USER          # my jobs   (Q=queued R=running E=exiting)
qstat -x -u $USER       # ... including finished ones, with resources used
qstat -f 123456         # everything about one job
qdel 123456             # cancel
less jobname.o123456    # read the log
```

## Job script template

```bash
#!/bin/bash
#PBS -N step_name
#PBS -q cpu
#PBS -l walltime=06:00:00
#PBS -l select=1:ncpus=8:mem=16gb
#PBS -M lguerrer@irb.hr
#PBS -m ea
#PBS -j oe

cd "$PBS_O_WORKDIR"

module load scientific/SOMETHING

# work goes here
```

Queues: `cpu` (default) · `cpu_30` (long jobs) · `bigmem` (lots of RAM)

## Transfer files (run on **your own** computer)

```bash
scp user@padobran.srce.hr:/remote/file .          # server → laptop
scp file user@padobran.srce.hr:/remote/folder/    # laptop → server
scp -r ...                                        # whole folders
rsync -avh --progress src/ user@host:/dest/       # big or resumable transfers
```

---

## Project paths

```bash
/beegfs/home/lguerrer/Asellidae/            # raw data — READ ONLY, never modify
/beegfs/home/lguerrer/Asellidae/genome/     # reference genome GCA_964212115.1
/beegfs/home/lguerrer/asellus-pipeline/     # this guide and the scripts
```

## Sample name decoder

```
AA  S  C  _G1DD_ P2
│   │  │    │     └─ pool replicate 1–3
│   │  │    └─────── G0 wild · G1 lab · LD light/dark · DD constant dark
│   │  └──────────── habitat: C cave · S surface
│   └─────────────── site: S Sušik · L Lummelunda · P Planina
└─────────────────── Asellus aquaticus
```

## Expected file counts

| Batch | Samples | fq.gz files |
|---|---|---|
| 4 Asellus | 18 | 36 |
| 4 Proasellus | 10 | 20 |
| 5 | 6 | 12 |
| 6 | 31 | 124 (2 lanes each) |
| **Total** | **65** | **192** |

---

## Five rules

1. Quote every path — `"$RAWDIR"`, not `$RAWDIR`. The folder names contain spaces.
2. Absolute paths inside scripts, always.
3. Never write inside `Asellidae/`. Read from it, write elsewhere.
4. Test on one sample before submitting 65.
5. Read the log even when the job "worked".
