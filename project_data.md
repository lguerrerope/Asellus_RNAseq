# 04 — The project data

← [03 — Running jobs with PBS](03-running-jobs-with-pbs.md) · Next: [05 — Checking the data (MD5)](05-checking-the-data-md5.md)

---

Before running anything, you need to be able to look at a folder called `AASC_G1DD_P2`
and know instantly what animal that is. This chapter is the decoder ring. Come back to
it constantly.

## The experimental design

Three sites where *Asellus aquaticus* lives both on the surface and, independently,
inside a cave:

| Site | Country | Note |
|---|---|---|
| **Sušik** | Croatia | The most complete dataset — the only site with lab animals from *both* habitats |
| **Lummelunda** | Sweden | An independent cave colonisation |
| **Planina** | Slovenia | Fully troglomorphic — the most extreme cave form |

Two kinds of animal:

- **G0 — wild.** Collected from the field. Their expression reflects both their genes
  and a lifetime in their own habitat.
- **G1 — laboratory.** The next generation, born and raised in the lab under a
  controlled light regime. Because every G1 animal grew up in the same conditions apart
  from light, differences between G1 groups are the interesting ones.

Two light treatments for G1:

- **LD** — light/dark cycle, a normal day
- **DD** — constant darkness, cave-like

That is the heart of the whole project. Take a **surface** animal, raise it in **DD**,
and see whether its genes move in the same direction that **cave** animals have evolved.
If they do, plasticity may have paved the way for evolution.

## Reading a sample name

```
AASC_G1DD_P2
││││ │  │  │
││││ │  │  └─ P2   = pool replicate number (P1, P2, P3)
││││ │  └──── DD   = constant darkness   (LD = light/dark cycle)
││││ └─────── G1   = laboratory generation 1   (G0 = wild-caught)
│││└───────── C    = Cave habitat        (S = Surface)
││└────────── S    = Sušik site          (L = Lummelunda, P = Planina)
└┴─────────── AA   = Asellus aquaticus
```

So `AASC_G1DD_P2` = *Asellus aquaticus*, **S**ušik, **C**ave, lab generation 1, raised in
constant darkness, pool replicate 2.

> ### Every sample is a pool of three animals
> One "replicate" is not one individual — it is RNA from **three animals pooled
> together**. This matters enormously later: it is fine for expression, but it means we
> cannot genotype individuals, and any population-genetic analysis has to use methods
> built for pooled sequencing.

## What we have, and what we do not

n = 3 replicates for every group that exists.

| Site | Wild surface (G0) | Wild cave (G0) | Lab surface LD / DD | Lab cave LD / DD |
|---|---|---|---|---|
| **Sušik** | `AASS_G0` ✅ | `AASC_G0` ✅ | `AASS_G1LD` / `AASS_G1DD` ✅ | `AASC_G1LD` / `AASC_G1DD` ✅ |
| **Lummelunda** | `AALS_G0` ✅ | `AALC_G0` ✅ | `AALS_G1LD` / `AALS_G1DD` ✅ | ❌ not sequenced |
| **Planina** | `AAPS_G0` ✅ | `AAPC_G0` ✅ | ❌ none | ❌ none |

**36 *Asellus* samples in total.** Sušik is the only complete site; the "has plasticity
been lost in cave animals?" question can currently only be asked there.

> Note: `AAPS_G0` replicates are numbered **P2, P3, P4** — there is no P1. That is not a
> missing file, it is just how they were labelled. Three replicates as expected.

## Which delivery each sample came in

The data arrived in three batches, from three different sequencing runs. **Write this
table on a post-it**, because it explains an awkward problem below.

| Folder | When | Contains |
|---|---|---|
| `4. …March2022…` | 2022 | `AALC_G0`, `AALS_G0`, `AASC_G0`, `AASS_G0`, `AASS_G1LD`, `AASS_G1DD` (+ *Proasellus*) |
| `5. …November2022…` | 2022 | `AALS_G1LD`, `AALS_G1DD` |
| `6. …September2023…` | 2023 | `AAPC_G0`, `AAPS_G0`, `AASC_G1LD`, `AASC_G1DD` (+ *Proasellus*) |

### The batch effect problem

Samples sequenced in different runs differ slightly for purely technical reasons —
different library preparation day, different flow cell, different reagent lot. This is
called a **batch effect**, and it is indistinguishable from real biology if the batches
line up with your comparison.

Check each comparison against the table above:

| Comparison | Verdict |
|---|---|
| Cave vs. surface, wild, any site | ✅ **Clean.** Both groups are always in the same batch. |
| Sušik surface LD vs. DD | ✅ Clean — both in batch 4 |
| Lummelunda surface LD vs. DD | ✅ Clean — both in batch 5 |
| Lummelunda wild vs. Lummelunda lab | ⚠️ Batch 4 vs. batch 5 |
| **Sušik surface LD/DD vs. Sušik cave LD/DD** | 🔴 **Confounded.** Surface is batch 4, cave is batch 6. |

That last row is the canalisation question, and there is no statistical trick that fully
rescues it: "cave" and "batch 6" are the same thing in this dataset. We will still run
it, include batch in the model where we can, and be explicit about the limitation when
we write it up. Knowing this *now* is much better than discovering it in the discussion
section.

## File structure

Each sample is a folder holding paired-end reads and a checksum file:

```
AALS_G1DD_P1/
├── AALS_G1DD_P1_1.fq.gz     <- read 1 ("forward")
├── AALS_G1DD_P1_2.fq.gz     <- read 2 ("reverse")
└── MD5.txt                  <- checksums for those two files
```

`_1` and `_2` are the two ends of the same DNA fragment. They always travel together and
must be given to every program as a pair, in the same order.

**Batch 6 is different: it has four files per sample**, because those libraries were
spread across two sequencing lanes:

```
AAPC_G0_P1_EKDL230012386-1A_HJHM5DSX7_L4_1.fq.gz    <- lane L4, read 1
AAPC_G0_P1_EKDL230012386-1A_HJHM5DSX7_L4_2.fq.gz    <- lane L4, read 2
AAPC_G0_P1_EKDL230012386-1A_HJJTFDSX7_L3_1.fq.gz    <- lane L3, read 1
AAPC_G0_P1_EKDL230012386-1A_HJJTFDSX7_L3_2.fq.gz    <- lane L3, read 2
```

The lanes belong to one sample and get combined — but **not yet**. Check them first
(chapter 05), then merge them (chapter 06). Never merge before checking: if one lane is
corrupt, merging hides which one.

## Technical details

- **Paired-end, 150 bp** reads
- **Directional / stranded** libraries — the protocol preserves which DNA strand the RNA
  came from. This is genuinely useful information and we must tell the counting software
  about it, or we throw it away.
- **Libraries were prepared in-house**, not by the company. One more reason batch matters.
- Roughly 3–5 GB of compressed data per sample; **187 GB in total**

## The other animals in the folder

Two sets of samples are *not* part of this project's questions:

- `PCV_*`, `PCVR_*` — *Proasellus* light/dark experiments (10 samples, batch 4)
- `PaMO_CUT_*`, `PC_CUT_*`, `PH_CUT_*`, `PK_CUT_*` — *Proasellus* "CUT" samples, several
  species (19 samples, batch 6)

They live in the same delivery folders and appear in the same MD5 checks, but they do not
belong to the *Asellus* plasticity analysis. **Do not include them in any expression
comparison.** Ask Laura before touching them.

## The reference

We have two possible references, and this is an open decision:

1. **A chromosome-level genome**, `GCA_964212115.1` (qmAseAqua29.1), in
   `Asellidae/genome/`. Downloaded and checksum-verified. Excellent quality — but it
   ships **without gene annotation**, so there is no ready-made list of where the genes
   are.
2. **A de novo transcriptome**, `Trinity_24Ind.fasta`, assembled from 24 of our own
   samples. It has genes but no genomic coordinates — and **the copy on the server is
   truncated** and has to be fetched again.

Chapter 08 will work through the choice. For now just know that both exist and neither is
ready to use yet.

## Quick self-test

Without scrolling up, say what these are:

1. `AALC_G0_P3`
2. `AASS_G1LD_P1`
3. `AAPC_G0_P2`

<details>
<summary>Answers</summary>

1. *Asellus aquaticus*, Lummelunda, **cave**, wild-caught, replicate 3
2. *Asellus aquaticus*, Sušik, **surface**, lab generation 1, light/dark cycle, replicate 1
3. *Asellus aquaticus*, Planina, **cave**, wild-caught, replicate 2

</details>

---

Next: [05 — Checking the data (MD5)](05-checking-the-data-md5.md)
