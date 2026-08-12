# 06 — Quality control with FastQC

← [05 — Checking the data (MD5)](05-checking-the-data-md5.md) · Next: 07 — Trimming *(coming next)*

---

MD5 proved the files arrived **intact**. That is a different question from whether the
data is **good**. A file can be a perfect copy of a bad sequencing run.

FastQC answers the second question. It reads every sequencing file and produces a report
about read quality, length, adapter contamination and a few other things. It does not
change your data — it only looks. That makes it completely safe to run and re-run.

## First: how software works on this cluster

You met `module load` in [chapter 02](02-linux-survival-kit.md). Here is the part that
trips people up on Padobran.

### Finding a program

```bash
module avail                              # everything (very long list)
module avail 2>&1 | grep -i fastqc        # search for one thing
```

The `2>&1` is needed because `module avail` prints its list to the error stream rather
than the normal one, so without it `grep` would see nothing. Searching gives:

```
scientific/FastQC/0.12.1
```

That full string — including the version — is the module name. Load it:

```bash
module load scientific/FastQC/0.12.1
```

Check what you have loaded, and unload everything if things get confusing:

```bash
module list
module purge
```

> **A module loaded in your terminal does not follow your job.** A compute node starts
> with nothing loaded. That is why every job script contains its own `module load` line.
> Forgetting it is the single most common reason a job dies instantly with
> `command not found`.

### The container twist

On most clusters, `module load` puts the program directly in your PATH and you just type
its name. On Padobran, several programs — FastQC among them — are shipped as
**apptainer containers** instead.

A container is a single file (`.sif`) holding the program and everything it needs to run:
its own libraries, its own version of Java, its own miniature operating system. It solves
a real problem — programs that need conflicting versions of the same library can coexist
— but it means you cannot simply type `fastqc`.

What `module load scientific/FastQC/0.12.1` actually does is set a variable pointing at
the container file:

```bash
module show scientific/FastQC/0.12.1
```
```
prepend_path("PATH","/apps/scientific/FastQC/0.12.1")
setenv("IMAGE_PATH","/apps/scientific/FastQC/0.12.1/fastqc.sif")
```

To run something inside the container you use `apptainer exec`:

```bash
apptainer exec --bind /beegfs /apps/scientific/FastQC/0.12.1/fastqc.sif fastqc --version
```
```
FastQC v0.12.1
```

Reading that command left to right:

| Part | Meaning |
|---|---|
| `apptainer exec` | run a command inside a container |
| `--bind /beegfs` | make the `/beegfs` disk visible from inside |
| `.../fastqc.sif` | which container |
| `fastqc --version` | what to run inside it |

**`--bind` is the part to remember.** A container cannot see your files by default — it
is sealed off. If you forget it, FastQC starts happily and then reports that your
perfectly existing fastq files do not exist. Whenever a containerised program claims a
file is missing and you can see it with `ls`, the answer is almost always a missing bind.

Not every program on Padobran is a container. `samtools`, for example, loads normally and
you just type `samtools`. When in doubt, `module show` tells you: if it sets `IMAGE_PATH`,
it is a container.

## What FastQC does

It reads a fastq file and produces two outputs per file:

- **`SAMPLE_fastqc.html`** — the report you look at, with graphs
- **`SAMPLE_fastqc.zip`** — the same numbers as plain text, which MultiQC reads later

FastQC processes each fastq file independently, which means read 1 and read 2 of the same
sample get separate reports. That is deliberate — the two reads of a pair often differ in
quality, and read 2 is usually the worse one.

## Running it

The scripts are in [`scripts/00_fastqc/`](../scripts/00_fastqc/), one per delivery batch,
and they are already copied to `Asellidae/00_fastqc/`. All four write their reports into
that same folder, so everything ends up in one place.

```bash
cd /beegfs/home/lguerrer/Asellidae/00_fastqc

qsub fastqc_batch5.sh              # 12 files, the quickest
qsub fastqc_batch4_asellus.sh      # 36 files
qsub fastqc_batch6.sh              # 124 files (two lanes per sample)
qsub fastqc_batch4_proasellus.sh   # 20 files — Proasellus, optional
```

Watch them with `qstat -u $USER`. The batch 6 job is the long one.

### The script, line by line

```bash
# Load FastQC (on this cluster it runs inside an apptainer container)
module load scientific/FastQC/0.12.1

# Folder that holds the sample subfolders of this batch
RAWDIR="/beegfs/home/lguerrer/Asellidae/5. Hr_IRB_.../01.RawData"

# Folder where all the reports are collected
OUTDIR="/beegfs/home/lguerrer/Asellidae/00_fastqc"

# FastQC does not create the output folder itself, so we make it first
mkdir -p "$OUTDIR"

apptainer exec --bind /beegfs \
    /apps/scientific/FastQC/0.12.1/fastqc.sif \
    fastqc "$RAWDIR"/*/*.fq.gz \
    -o "$OUTDIR" \
    -t 8
```

Three things worth noticing:

**`"$RAWDIR"/*/*.fq.gz`** — two stars. The first matches every sample folder, the second
every fastq inside it. One command, all the files of the batch.

**`mkdir -p "$OUTDIR"`** — FastQC refuses to start if the output folder does not exist,
rather than creating it. `-p` means "and don't complain if it already exists", which is
what lets all four scripts share one output folder.

**`-t 8`** — process 8 files simultaneously. This must match the `ncpus` in the PBS header;
each thread needs roughly 250 MB of RAM. Asking for 8 cores and then using `-t 4` wastes
half of what the scheduler reserved for you.

## Looking at the reports

The reports are HTML files on the server, and a terminal cannot draw graphs. Two options:

**In VS Code** (easiest, if you use the Remote-SSH extension): just click the `.html` file
and choose to preview it.

**Or copy one to your own computer** — run this in a terminal on *your laptop*, not on the
server:

```bash
scp yourusername@padobran.srce.hr:/beegfs/home/lguerrer/Asellidae/00_fastqc/AASS_G0_P1_1_fastqc.html .
```

Then open it by double-clicking.

You will not open 172 reports by hand. That is what MultiQC is for, and it is the next
thing we do. But open three or four now, because you need to know what the individual
graphs look like before a summary of them means anything.

## Reading a report

Eleven modules, each with a green tick, orange `!` or red ✗.

> ### The traffic lights lie about RNA-seq
> FastQC's thresholds were designed with whole-genome DNA sequencing in mind. RNA-seq
> legitimately breaks several of its assumptions. **Some red crosses are completely
> expected here and are not a problem.** Never drop a sample because FastQC showed red —
> understand which module is red and why.

| Module | What it checks | What to think for RNA-seq |
|---|---|---|
| **Basic Statistics** | read count, length, %GC | Sanity check. Note the read count. |
| **Per base sequence quality** | quality along the read | **The important one.** Should stay in the green zone. Slight decline at the end is normal. |
| **Per sequence quality scores** | overall quality per read | Should peak high. A second low peak means a subset of bad reads. |
| **Per base sequence content** | A/C/G/T balance per position | **Red is expected.** Random-primed RNA-seq is biased in the first ~12 bases. Ignore. |
| **Per sequence GC content** | GC distribution vs. theory | The theoretical curve assumes a genome; transcriptomes are not that. Mild warnings are fine; a sharp extra peak can mean contamination. |
| **Per base N content** | unreadable bases | Must be flat at zero. |
| **Sequence Length Distribution** | read lengths | All 150 bp before trimming. |
| **Sequence Duplication Levels** | repeated reads | **Red is expected.** Highly expressed genes produce identical reads by design. |
| **Overrepresented sequences** | very frequent sequences | Usually rRNA or a very abundant transcript. Worth a BLAST if one dominates. |
| **Adapter Content** | leftover adapter sequence | **Take this one seriously.** It tells you how hard to trim. |
| **Per tile sequence quality** | bad regions of the flow cell | A few warnings are normal. Persistent failures across many samples suggest a run problem. |

## Our results

172 files analysed (batches 4-Asellus, 5 and 6; the *Proasellus* batch was not run).
Here is what came out.

### The good news

| Module | Result across 172 files |
|---|---|
| Per base sequence quality | **172 PASS**, zero warnings |
| Per sequence quality scores | **172 PASS** |
| Per base N content | **172 PASS** |
| Sequence Length Distribution | **172 PASS** — all reads 150 bp |

The base quality is uniformly excellent. That is the single most important thing on this
page, and it means no sample has to be discarded for quality.

**Sequencing depth** — all 36 *Asellus* samples (lanes summed) fall between **14.7 and
36.7 million read pairs**, median 23.6 M. For differential expression you want at least
~15–20 M, so every sample is usable. The shallowest is `AAPS_G0_P2` at 14.7 M — fine, but
worth remembering if that sample ever looks like an outlier later.

GC content sits between 38% and 46%, consistent across samples. Nothing suggests
contamination with a foreign organism.

### The one real finding: adapters in batch 6

| Batch | Adapter Content result |
|---|---|
| batch 4 Asellus (36 files) | **all PASS** — highest adapter level 0.23% |
| batch 5 (12 files) | **all PASS** |
| **batch 6 (124 files)** | **all 124 FAIL** — up to **42.9%** in *Asellus* files |

This is not random. Every single file from the September 2023 delivery is affected and no
file from the earlier ones is. The adapter detected is the Illumina Universal Adapter, and
the worst *Asellus* file is `AAPS_G0_P4` at 42.9%.

The explanation is insert size: if the RNA fragments in that library were shorter than
150 bp, the sequencer reads to the end of the fragment and continues into the adapter on
the other side. Nearly half the reads in some batch 6 files end in adapter sequence.

**This is fixable and expected to be fixed by trimming**, which is the next step. But note
what it would do if we skipped trimming: adapter sequence does not match the genome, so
those reads would fail to align, batch 6 samples would show artificially low mapping
rates, and — because batch 6 contains all the Planina samples and all the Sušik cave lab
samples — that technical artefact would masquerade as biology in exactly the comparisons
we care about. It also reinforces the batch-effect warning from
[chapter 04](04-the-project-data.md).

There is also **PolyG contamination up to 16.8%**. This is a known artefact of two-colour
Illumina chemistry (NovaSeq): when nothing is detected, the machine reads it as `G`. The
trimming step has a specific option for it.

### The expected "failures"

| Module | Result | Verdict |
|---|---|---|
| Per base sequence content | 99 FAIL, 73 WARN, 0 PASS | **Normal.** Random priming bias in the first bases. Every RNA-seq dataset looks like this. |
| Sequence Duplication Levels | 50 FAIL, 13 WARN | **Normal.** Highly expressed genes generate duplicate reads. Do *not* deduplicate RNA-seq without UMIs. |
| Overrepresented sequences | 5 FAIL, 103 WARN | Expected — mostly rRNA and very abundant transcripts. |
| Per sequence GC content | 7 FAIL, 91 WARN | Expected — the reference curve assumes a genome. |

### Worth keeping an eye on

**Per tile sequence quality: 10 files FAIL**, all from batch 4:

```
AALS_G0_P1_1   AALS_G0_P3_1   AALS_G0_P3_2   AALS_G1DD_P3_1
AASS_G0_P1_1   AASS_G0_P1_2   AASS_G0_P2_1   AASS_G0_P2_2
AASS_G1DD_P3_1 AASS_G1DD_P3_2
```

This means part of the flow cell performed poorly — a bubble, a smudge. Since overall
per-base quality still passes in all of them, the affected reads are a small minority and
trimming will remove most. No action needed, but if one of these samples behaves oddly in
the final analysis, come back to this list.

## Checklist before moving on

- [x] All batches have reports — count them: `ls /beegfs/home/lguerrer/Asellidae/00_fastqc/*.html | wc -l`
- [x] No sample fails **Per base sequence quality**
- [x] Every sample has enough reads (>15 M pairs)
- [x] Adapter contamination measured, so we know what to tell the trimmer
- [ ] Run the *Proasellus* batch too, if those samples will be used
- [ ] Summarise all reports with MultiQC

Counting the reports is a real check, not a formality: FastQC skips a file it cannot read
and carries on without failing the job. If the count is not what you expect, something was
silently skipped.

## What we learned, in one line

The sequencing is high quality throughout; the only real problem is heavy adapter and
polyG contamination confined to the 2023 batch, which the next step removes.

---

Next: 07 — Trimming with fastp *(coming next)*
