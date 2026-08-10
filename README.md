# Asellus Plasticity — Transcriptomics Pipeline

RNA-seq analysis of cave and surface populations of the isopod *Asellus aquaticus*,
asking how much of the evolved gene expression in cave animals can be explained by
plasticity in their surface ancestors.

**Ruđer Bošković Institute (IRB), Zagreb — Bilandžija lab.**

---

## Who this is for

You have just joined the project and you have **never used Linux, a terminal, or a
computing cluster before**. That is completely fine — this documentation assumes zero
prior knowledge and builds up from there.

Read the chapters **in order**. Each one ends with something you have actually done on
the server, and the next one builds on it. Do not skip ahead to the analysis chapters:
if you are not comfortable moving around the file system, the later steps will be
frustrating rather than difficult.

Expect chapters 00–03 to take you two or three days of poking around. That is normal
and it is time well spent.

---

## Table of contents

### Part 1 — Getting on the server

| # | Chapter | What you will be able to do |
|---|---------|------------------------------|
| 00 | [What is all this?](docs/00-what-is-all-this.md) | Explain what a cluster is and why we do not just use a laptop |
| 01 | [Connecting to Padobran](docs/01-connecting-to-padobran.md) | Log in to the server from your own computer |
| 02 | [Linux survival kit](docs/02-linux-survival-kit.md) | Move around, look at files, edit text, not lose your work |
| 03 | [Running jobs with PBS](docs/03-running-jobs-with-pbs.md) | Submit a job to the cluster and read its output |

### Part 2 — The data

| # | Chapter | What you will be able to do |
|---|---------|------------------------------|
| 04 | [The project data](docs/04-the-project-data.md) | Read any sample name and know exactly what animal it is |
| 05 | [Checking the data (MD5)](docs/05-checking-the-data-md5.md) | Prove that the 187 GB we received arrived intact |

### Part 3 — The analysis

| # | Chapter | Status |
|---|---------|--------|
| 06 | Quality control (FastQC + MultiQC) | *next* |
| 07 | Trimming (fastp) | planned |
| 08 | The reference: genome vs. transcriptome | planned |
| 09 | Mapping | planned |
| 10 | Counting reads per gene | planned |
| 11 | Differential expression (DESeq2) | planned |
| 12 | Functional enrichment (GO / KEGG) | planned |

### Reference

- [Command cheat sheet](docs/99-cheatsheet.md) — everything on one page, for when you
  know what you want but forgot how to spell it

---

## Repository layout

```
asellus-pipeline/
├── README.md                 <- you are here
├── docs/                     <- the written guide, one file per chapter
└── scripts/                  <- the actual job scripts, one folder per step
    └── 01-md5-check/
```

The **data itself is not in this repository** and never will be — it is 187 GB and
git is not built for that. The data lives on the server at
`/beegfs/home/lguerrer/Asellidae/`. This repository only holds text: instructions and
scripts.

---

## The scientific questions

Everything in this pipeline exists to answer these. Keep them in view; they are the
reason we run any of it.

1. **What gene expression changes underlie adaptation to caves?**
   Wild surface vs. wild cave animals, at each of three sites.
2. **How much of that is driven by plasticity?**
   Do surface animals raised in the dark shift their expression in the same direction
   that cave animals evolved?
3. **Is that plasticity adaptive or maladaptive?**
   Same direction as the evolved change, or opposite?
4. **Has plasticity been lost (canalised) in cave animals?**
   Do cave animals still respond to light/dark, or have they stopped responding?

There is a second set of questions about SNPs, population genetics and selection using
the same reads. Those come much later — do not worry about them yet.

---

## Where to ask for help

Ask Laura. Seriously — ask early rather than spending a day stuck. The one thing you
should *not* do is run commands you do not understand on the raw data. Raw data is the
one thing in this project that cannot be regenerated.
