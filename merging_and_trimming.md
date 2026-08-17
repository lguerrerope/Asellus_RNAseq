# 07 — Merging lanes and trimming reads

← [06 — Quality control with FastQC](06-quality-control.md) · Next: 08 — MultiQC *(coming next)*

---

FastQC told us what is wrong with the reads. This chapter fixes it.

Two things happen here, and **the order matters**:

1. **Merge** — put every sample into one pair of files with one naming scheme
2. **Trim** — cut off adapters, polyG and low-quality tails

This is the first step that **creates new data**. Everything so far only looked. From now
on, each step reads from the previous folder and writes to a new one, and the raw data in
the delivery folders is never touched.

```
Asellidae/
├── 4. …  5. …  6. …        raw data, read-only, never changes
├── 00_fastqc/               reports
├── 01_merged/               ← this chapter: one pair of files per sample
└── 02_trimmed/              ← this chapter: cleaned reads
```

---

## Part 1 — Why merge

Our samples do not all have the same shape:

| Batch | Files per sample | Why |
|---|---|---|
| 4 and 5 | 2 (`_1`, `_2`) | one sequencing lane |
| 6 | 4 | the library was split across two lanes |

A "lane" is a physical channel on the flow cell. Splitting one library across two lanes is
routine — it spreads the sample out so a local problem does not ruin it. The two lanes are
**the same library, the same RNA, the same animals**. They are not replicates and must
never be treated as such. They get joined back together.

If we left that difference in place, every later step would need two versions of itself.
Instead we spend ten minutes now and end up with **36 samples, all shaped identically**:

```
01_merged/
├── AALC_G0_P1_1.fq.gz
├── AALC_G0_P1_2.fq.gz
├── AALS_G0_P1_1.fq.gz
└── …                        72 files = 36 samples × 2 reads
```

### Merging is just `cat`

The surprising part: gzip-compressed files can be stuck end to end with plain `cat`, and
the result is a valid gzip file. No decompressing, no special tool.

```bash
cat sample_lane1_1.fq.gz sample_lane2_1.fq.gz > sample_1.fq.gz
```

This works because the gzip format is a series of independent blocks; a reader simply
carries on into the next one. It was verified on our data before writing the script:
10,000 reads + 20,000 reads gave exactly 30,000 reads, and `gzip -t` reported the merged
file as valid.

### The one way to get this badly wrong

**Read 1 and read 2 must be concatenated in the same lane order.**

Paired-end files are matched by position: the 4th read of `_1.fq.gz` is the partner of the
4th read of `_2.fq.gz`. If you merge lane A then lane B for read 1, but lane B then lane A
for read 2, every pair after the join point is mismatched. Nothing crashes. No error
appears. The aligner simply finds that most pairs do not fit together, and your mapping
rate quietly collapses.

The script avoids this by using the same `*` pattern for both, because `*` always expands
in alphabetical order:

```bash
cat "$SAMPLE"/*_1.fq.gz > "$OUTDIR/${NAME}_1.fq.gz"
cat "$SAMPLE"/*_2.fq.gz > "$OUTDIR/${NAME}_2.fq.gz"
```

Same pattern, same order, both times. That is the whole safeguard.

### Batches 4 and 5: shortcuts instead of copies

Those 24 samples have nothing to merge. Copying them would duplicate ~90 GB for no reason,
so instead we create **symbolic links** — shortcuts that point at the original file:

```bash
ln -sf "$SAMPLE/${NAME}_1.fq.gz" "$OUTDIR/${NAME}_1.fq.gz"
```

`ln -s` makes the link, `-f` replaces one that already exists. To any program that opens
it, the link behaves exactly like the file. It costs nothing and takes no space.

Two rules: the target must be an **absolute path** (a relative one breaks as soon as
anything moves), and you must never delete "through" a link expecting to tidy up a copy —
`rm` on the link removes only the shortcut, but any program that *writes* to it writes to
the original. We only ever read, so we are safe.

Check for broken links with:

```bash
find /beegfs/home/lguerrer/Asellidae/01_merged -xtype l
```

Silence is good.

### Running step 01

```bash
cd /beegfs/home/lguerrer/Asellidae/01_merged

qsub merge_batch6.sh              # 12 samples to concatenate — a real job
bash link_batch4_batch5.sh        # 24 samples to link — instant, no job needed
```

Note that one is submitted and the other is just run. Merging reads and rewrites ~35 GB,
so it belongs on a compute node. Making 48 shortcuts touches no data at all — sending
that to the queue would be silly. Part of learning the cluster is knowing which is which.

When both are done:

```bash
ls /beegfs/home/lguerrer/Asellidae/01_merged/*.fq.gz | wc -l    # expect 72
```

The scripts also skip any sample that is already merged, so if a job runs out of walltime
you can resubmit it and it continues instead of starting over.

---

## Part 2 — Trimming with fastp

### What we are removing, and why

From [chapter 06](06-quality-control.md):

| Problem | Where | How bad |
|---|---|---|
| Illumina Universal Adapter | all 124 batch 6 files | up to **42.9%** of reads |
| PolyG | batch 6 | up to **16.8%** |
| Low-quality tails | everywhere, mildly | minor |

**Adapter** is the sequence the lab attaches to each RNA fragment so the machine can grab
it. It is not biology. It gets sequenced whenever the fragment is shorter than the read
length: the machine reaches the end of the real fragment and keeps reading into the
adapter on the far side.

We measured this directly. Running fastp on `AAPS_G0_P4` — the worst sample — reports:

```
Insert size peak (evaluated by paired-end reads): 143
```

The fragments are ~143 bp and the reads are 150 bp. Every read overshoots by a few bases,
and many fragments are shorter still. That is the entire explanation for the FastQC
result, and it is a property of the library, not a mistake in the run.

**PolyG** is different — it is a machine artefact. The NovaSeq detects bases with two
colours, and "no signal at all" is encoded as `G`. When a cluster stops producing signal,
the instrument dutifully writes a long run of Gs that never existed.

If we skipped trimming, none of this would align to the genome. Batch 6 would show a
markedly lower mapping rate than batches 4 and 5 — and because batch 6 contains **all the
Planina samples and all the Sušik cave lab samples**, that purely technical difference
would sit exactly on top of the comparisons we care about and look like biology.

### The script

One job, looping over all 36 samples:
[`scripts/02_fastp/fastp_all_samples.sh`](../scripts/02_fastp/fastp_all_samples.sh)

```bash
module load scientific/fastp/0.23.4

INDIR="/beegfs/home/lguerrer/Asellidae/01_merged"
OUTDIR="/beegfs/home/lguerrer/Asellidae/02_trimmed"
REPORTS="$OUTDIR/reports"

for R1 in "$INDIR"/*_1.fq.gz; do

    NAME=$(basename "$R1" _1.fq.gz)      # ".../AASS_G0_P1_1.fq.gz" -> "AASS_G0_P1"
    R2="$INDIR/${NAME}_2.fq.gz"

    if [ -f "$OUTDIR/${NAME}_2.trimmed.fq.gz" ]; then
        echo "already trimmed, skipping"
        continue
    fi

    fastp \
        -i "$R1"  -I "$R2" \
        -o "$OUTDIR/${NAME}_1.trimmed.fq.gz" \
        -O "$OUTDIR/${NAME}_2.trimmed.fq.gz" \
        --detect_adapter_for_pe \
        --trim_poly_g --poly_g_min_len 10 \
        --trim_poly_x --poly_x_min_len 10 \
        --length_required 36 \
        --thread 8 \
        --html "$REPORTS/${NAME}_fastp.html" \
        --json "$REPORTS/${NAME}_fastp.json"
done
```

**`fastp` is not a container.** Unlike FastQC, `module load scientific/fastp/0.23.4` puts
the program straight in your PATH and you just type `fastp`. Use `module show` to tell
which kind you are dealing with: if it sets `IMAGE_PATH`, it is a container and needs
`apptainer exec`.

**`basename "$R1" _1.fq.gz`** strips both the folder and the given suffix, turning a full
path into a clean sample name. That name is then used to build the read 2 path and all
four output names — one variable, everything consistent.

**Lowercase `-i/-o` is read 1, uppercase `-I/-O` is read 2.** fastp uses case to
distinguish them. Swapping them silently produces mismatched output.

### What each option does

| Option | What it does | Why here |
|---|---|---|
| `--detect_adapter_for_pe` | Works out the adapter by overlapping read 1 and read 2 | More reliable than trusting a list. On our data it found `AGATCGGAAGAGC…` on its own |
| `--trim_poly_g` | Removes runs of G at the 3' end | The NovaSeq artefact, up to 16.8% |
| `--poly_g_min_len 10` | Only if the run is ≥10 bases | Short G runs may be real sequence |
| `--trim_poly_x` | Removes any single-base run at the 3' end | Mostly polyA tails of mRNA |
| `--length_required 36` | Discards reads shorter than 36 bp after trimming | After removing 40% adapter some reads are stubs, and short reads map ambiguously all over the genome |
| `--thread 8` | 8 threads | Must match `ncpus` in the PBS header |
| `--json` | Machine-readable report | MultiQC reads these to summarise all 36 samples |

fastp also applies its **default quality filter** without being asked: a read is dropped if
more than 40% of its bases are below Q15. Our data is clean enough that this removes very
little.

### What we deliberately do NOT do

**We do not trim the first 12 bases.** The old `Larvae_RNA` script had `--trim_front1 12`,
and that was right *there*, because those libraries carried a 12-base UMI at the start of
each read. Our libraries have no UMIs.

It is tempting to add it anyway, because FastQC's "Per base sequence content" is red for
every sample and trimming the front makes it turn green. Resist. That bias comes from
random priming; it is present in every RNA-seq dataset ever made, and aligners handle it
without difficulty. Trimming 12 of 150 bases throws away 8% of the data to make a graph
look nicer.

**We do not deduplicate.** FastQC flagged high duplication in 50 files. In RNA-seq that is
expected: a highly expressed gene *must* produce identical reads. Removing them would
flatten exactly the signal we are measuring. Deduplication is for DNA, or for RNA-seq with
UMIs.

### Running it

```bash
cd /beegfs/home/lguerrer/Asellidae/02_trimmed
qsub fastp_all_samples.sh
```

Roughly 5–10 minutes per sample, so **3–6 hours** for all 36. The script skips samples
that are already finished, so a job that hits its walltime can simply be resubmitted.

> ### A real failure worth learning from
> This job was once submitted **before** step 01 had been run, so `01_merged/` was empty.
> The log said:
>
> ```
> === * ===
> ERROR: Failed to open file: /beegfs/home/lguerrer/Asellidae/01_merged/*_1.fq.gz
> ```
>
> The clue is `=== * ===`. When a `*` pattern matches nothing, bash does not complain — it
> hands the program the literal text `*_1.fq.gz`, and the program tries to open a file
> with a star in its name. **Any time you see a bare `*` in your output, a pattern matched
> nothing.**
>
> No damage was done: fastp writes output only after opening its input. The script now
> checks its input folder first and stops with a message naming the step you skipped.

---

## Part 3 — Reading the fastp output

Every sample produces three things: a summary in the job log, an HTML report, and a JSON
file. Learn to read the first one.

### The summary block

Here is the real output from the `AAPS_G0_P4` test run, annotated:

```
Read1 before filtering:
total reads: 50000                     ← what came in
total bases: 7500000                   ← 50000 × 150 bp
Q20 bases: 7186245(95.8166%)
Q30 bases: 6855883(91.4118%)

Read1 after filtering:
total reads: 48124                     ← what survived
total bases: 6001204                   ← far fewer bases: reads got SHORTER
Q20 bases: 5882461(98.0213%)
Q30 bases: 5679424(94.6381%)           ← quality went UP
```

Two numbers matter and they behave differently:

- **total reads** dropped 3.8%. This is how many sequences were thrown away entirely.
  Small is good.
- **total bases** dropped 20%. This is how much sequence was cut off the ends. Large is
  fine — that is the adapter being removed, and it is the point of the exercise.

**Q30 rose from 91.4% to 94.6%** because the trimmed material was the worst material.
Quality improving after trimming is exactly what should happen.

### The filtering breakdown

```
Filtering result:
reads passed filter: 96248             ← 96.2%
reads failed due to low quality: 2102
reads failed due to too many N: 44
reads failed due to too short: 1606    ← shorter than 36 bp after trimming
reads with adapter trimmed: 50890      ← 51% of reads had adapter!
bases trimmed due to adapters: 2308710
reads with polyX in 3' end: 2659
bases trimmed in polyX tail: 104053
Duplication rate: 3.242%
Insert size peak: 143
```

`reads with adapter trimmed: 50890` out of 100,000 — **half the reads in this sample ended
in adapter**, and now they do not. That single line is the justification for this entire
chapter.

Note that only **1,606 reads** were lost for being too short. Trimming this aggressively
removes a lot of *sequence* while losing almost no *reads*, which is the good outcome.

### What good looks like

| Number | Healthy | Worry if |
|---|---|---|
| reads passed filter | >90% | <80% — investigate that sample |
| reads too short | <5% | >10% — the insert size was very small |
| Q30 after | higher than before | it went down (should be impossible) |
| adapter trimmed | anything | batch 4/5 should be near zero, batch 6 high |
| duplication rate | <20% for RNA-seq | >50% — possibly a low-complexity library |
| insert size peak | >150 for batch 4/5 | ~140 for batch 6, as we saw |

**Expect the batches to look different**, and that is fine: batch 6 should show massive
adapter trimming, batches 4 and 5 almost none. That is the problem being fixed, not a new
one appearing.

### The HTML report

Open one in VS Code, or copy it to your laptop:

```bash
scp yourusername@padobran.srce.hr:/beegfs/home/lguerrer/Asellidae/02_trimmed/reports/AAPS_G0_P4_fastp.html .
```

It shows before/after side by side. The graph worth studying is **quality along the read**:
before trimming it sags at the end, after trimming the sag is gone — because that part of
the read no longer exists.

### The JSON

You will never read these by hand. They exist so MultiQC can pull the same numbers out of
all 36 samples and put them in one table — which is the next chapter, and the point at
which you can finally compare samples at a glance instead of one report at a time.

---

## Checklist

- [ ] `01_merged/` holds 72 files and `find … -xtype l` reports no broken links
- [ ] fastp job finished; `02_trimmed/` holds 72 `.trimmed.fq.gz` files
- [ ] `02_trimmed/reports/` holds 36 HTML and 36 JSON files
- [ ] Every sample passed >90% of its reads
- [ ] Batch 6 samples show heavy adapter trimming; batches 4 and 5 show almost none
- [ ] No sample lost an unusual number of reads compared with the rest

Count the outputs, do not assume them. A loop that skips a sample does it quietly.

---

Next: 08 — Summarising everything with MultiQC *(coming next)*
