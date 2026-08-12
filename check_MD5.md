# 05 — Checking the data (MD5)

← [04 — The project data](04-the-project-data.md) · Next: 06 — Quality control *(coming next)*

---

**This is the first real step of the pipeline, and the one people are most tempted to
skip. Do not skip it.**

## Why bother

A 5 GB file downloaded over the internet can arrive incomplete. The connection drops, the
download resumes badly, a disk hiccups. The result is a file that:

- has a perfectly normal name
- opens without complaint
- shows sensible reads if you look at the top
- and is silently missing the last 20% of its content

Nothing downstream will warn you. FastQC will happily report on a truncated file. The
aligner will happily map it. You will get a differential expression result, and it will
be quietly wrong, and you will not find out until someone asks why one sample has fewer
reads than the others.

> **This already happened in this project.** The transcriptome assembly
> `Trinity_24Ind.fasta` was copied to the server and looked fine. It is truncated: the
> file ends in the middle of a sequence, and it contains 1,390,947 contigs where the
> documentation says 1.9 million. It had no checksum, so nothing caught it. That is the
> failure mode this chapter exists to prevent.

## What a checksum is

An **MD5 checksum** is a 32-character fingerprint calculated from the entire content of a
file:

```
f4a8e8a0fd783bd52864d3b61d42fdc8  AALC_G0_P1_1.fq.gz
```

Change a single byte anywhere in that file and the fingerprint changes completely. The
sequencing company calculates it on their machine, before sending. We calculate it on
ours, after receiving. If the two match, the file crossed the internet intact — all
5 GB of it.

It is a very strong guarantee, and it costs nothing but time.

## Where our checksums live

Novogene supplies them at two levels, and **only one of them is usable**:

**❌ The `MD5.txt` at the top of each delivery folder — do not use.**
The file paths inside do not match how the data is actually stored on our server. The one
in batch 4 refers to a folder called `raw_data/` which does not exist here (ours is
called `X204SC22040339-Z01-F002_Asellus`), and the one in batch 6 lists samples called
`BJEL_*` that were never part of our delivery. Using these produces a wall of confusing
"No such file" errors.

**✅ The `MD5.txt` inside each sample folder — use this one.**
It lists plain file names with no path:

```
f4a8e8a0fd783bd52864d3b61d42fdc8  AALC_G0_P1_1.fq.gz
b83718444e8d2fc54470701b60364900  AALC_G0_P1_2.fq.gz
```

All 65 sample folders have one. Because the names have no path attached, you must be
*inside* the folder when you check — which is exactly what our scripts do.

## Doing it by hand, once

Before running anything on 65 samples, do one by hand so you understand what the script
automates:

```bash
cd "/beegfs/home/lguerrer/Asellidae/5. Hr_IRB_BilandzijaH_November2022_WBI_X204SC22101552-Z01-F001_DIRECTIONAL/01.RawData/AALS_G1DD_P1"

cat MD5.txt          # look at the expected fingerprints
md5sum -c MD5.txt    # calculate them and compare
```

After about a minute and a half:

```
AALS_G1DD_P1_1.fq.gz: OK
AALS_G1DD_P1_2.fq.gz: OK
```

`md5sum -c` means "check against this list". `OK` means the file on disk is byte-for-byte
what was sent. A `FAILED` line means it is not, and that file has to be downloaded again.

This one sample is 5.3 GB and took 79 seconds — about 67 MB/s, limited by disk speed
rather than CPU. All 187 GB will take roughly 45 minutes in total. That is why it is a
job and not something you run in your terminal.

## The scripts

Four scripts in [`scripts/md5_check/`](../scripts/md5_check/), one per delivery
folder:

| Script | Checks | Samples |
|---|---|---|
| `md5_batch4_asellus.sh` | batch 4 → `X204SC22040339-Z01-F002_Asellus` | 18 |
| `md5_batch4_proasellus.sh` | batch 4 → `X204SC22040339-Z01-F002_Proasellus` | 10 |
| `md5_batch5.sh` | batch 5 → `01.RawData` | 6 |
| `md5_batch6.sh` | batch 6 → `RAW_DATA` | 31 |

Total 65 — which matches the number of sample folders on disk.

They are deliberately four separate scripts rather than one clever one. The four files
are identical except for three lines: the job name, the input folder and the output file.
That makes each one readable on its own, and it means a failure in one batch does not
take the others down with it. **This is the pattern for the whole pipeline** — one step,
one folder, one script per group of samples.

### Reading the script

```bash
#!/bin/bash
#PBS -N md5_batch5
#PBS -q cpu
#PBS -l walltime=06:00:00
#PBS -l select=1:ncpus=1:mem=4gb
#PBS -M lguerrer@irb.hr
#PBS -m bea
#PBS -j oe

# Go to the folder the job was submitted from
cd "$PBS_O_WORKDIR"

# Folder that holds the sample subfolders of this batch
RAWDIR="/beegfs/home/lguerrer/Asellidae/5. Hr_IRB_BilandzijaH_November2022_WBI_X204SC22101552-Z01-F001_DIRECTIONAL/01.RawData"

# File where the results are written (absolute path)
OUTPUT="/beegfs/home/lguerrer/Asellidae/md5_batch5_results.txt"

# Start from an empty results file
> "$OUTPUT"

# Check one sample folder at a time
for SAMPLE in "$RAWDIR"/*/; do
    echo "=== $(basename "$SAMPLE") ===" >> "$OUTPUT"
    # Each sample folder has its own MD5.txt with plain file names,
    # so we move into the folder before running the check
    ( cd "$SAMPLE" && md5sum -c MD5.txt ) >> "$OUTPUT" 2>&1
done

# Print the failures, if any
echo "Failed files:"
grep "FAILED" "$OUTPUT"

echo "Done. Results in $OUTPUT"
```

Four things in there are worth understanding, because they recur in every script we write:

**`for SAMPLE in "$RAWDIR"/*/; do ... done`**
A loop. `*/` matches every *folder* inside `$RAWDIR` (the trailing slash is what restricts
it to folders). The body runs once per sample, with `$SAMPLE` set to that folder's path.
Sixty-five samples, one instruction.

**`( cd "$SAMPLE" && md5sum -c MD5.txt )`**
The round brackets run the commands in a **subshell** — a temporary copy of the shell.
The `cd` inside it does not affect the rest of the script, so the loop does not need to
find its way back afterwards. We need the `cd` because `MD5.txt` lists bare file names,
which only make sense from inside the folder.

**`>` and `>>`**
`>` sends output to a file, replacing whatever was there. `>>` appends to the end. The
lone `> "$OUTPUT"` before the loop is a trick to empty the results file, so re-running the
script does not stack new results on top of old ones. Inside the loop everything uses
`>>`, otherwise each sample would erase the previous one.

**`2>&1`**
Programs write to two separate streams: normal output (1) and errors (2). `2>&1` means
"send errors to the same place as normal output", so a problem ends up in the results file
instead of vanishing.

And, as always, **every path is in double quotes** — without them the spaces in
`4. HR_IRB_Bilandzija…` break the script apart.

## Running them

```bash
cd /beegfs/home/lguerrer/asellus-pipeline/scripts/md5_check

qsub md5_batch5.sh              # start with this one, it is the smallest
```

Watch it:

```bash
qstat -u $USER
```

When it disappears from `qstat`, submit the rest:

```bash
qsub md5_batch4_asellus.sh
qsub md5_batch4_proasellus.sh
qsub md5_batch6.sh
```

You can submit all three at once — PBS will run them when there is room.

## Reading the results

Two things to look at. First the PBS log, which ends with a summary:

```bash
less md5_batch5.o123456
```

Then the results file itself:

```bash
less /beegfs/home/lguerrer/Asellidae/md5_batch5_results.txt
```

The important checks:

```bash
# How many files passed?
grep -c ": OK$" /beegfs/home/lguerrer/Asellidae/md5_batch5_results.txt

# Did anything fail? (no output = nothing failed = good)
grep "FAILED" /beegfs/home/lguerrer/Asellidae/md5_batch5_results.txt
```

Expected counts when everything is intact:

| Batch | Files that must say OK |
|---|---|
| batch 4 Asellus | 36 |
| batch 4 Proasellus | 20 |
| batch 5 | 12 |
| batch 6 | 124 |
| **Total** | **192** |

**Count the `OK` lines — do not just check that nothing said FAILED.** A file that was
never checked also never fails. If the numbers do not match the table, something was
skipped and you need to find out what.

## If something fails

Do not panic and do not delete anything.

1. Note exactly which file failed.
2. Tell Laura — that specific file has to be re-downloaded from Novogene.
3. Re-run the check for that sample only, by hand, using the manual method above.
4. **Do not proceed to analysis with a failed file.** Its results would be wrong in a way
   nobody would ever spot.

## What "done" looks like

You are finished with this chapter when:

- all four jobs have completed
- the `OK` counts add up to 192
- no line anywhere says `FAILED`

Write the date and the counts in your lab notebook. From this point on, we can state that
every read we analyse is exactly what came off the sequencer — and that is a claim worth
being able to make.

---

Next: 06 — Quality control with FastQC and MultiQC *(coming next)*
