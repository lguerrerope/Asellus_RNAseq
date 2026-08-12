# 02 — Linux survival kit

← [01 — Connecting to Padobran](01-connecting-to-padobran.md) · Next: [03 — Running jobs with PBS](03-running-jobs-with-pbs.md)

---

This is the smallest set of commands that lets you work without being dangerous. Do not
try to memorise it. Do the examples on the real project folders — you cannot break
anything by *looking*.

## The shape of a command

```
command  -options  arguments
```

For example:

```bash
ls -l /beegfs/home/lguerrer/Asellidae
```

`ls` is the command, `-l` is an option that changes how it behaves, and the path is what
it acts on. Options usually start with `-` (short) or `--` (long). Spaces separate
things — which is why filenames with spaces cause trouble later in this chapter.

## Where am I, what is here

```bash
pwd                  # print working directory — where am I right now?
ls                   # list what is here
ls -l                # ... with size, date, permissions
ls -lh               # ... with human sizes (GB instead of 1759311019)
ls -lh /some/path    # list somewhere else without going there
```

`ls -lh` is the one you will use most.

## Moving around

```bash
cd /beegfs/home/lguerrer      # go to an exact place
cd Asellidae                  # go into a folder that is inside the current one
cd ..                         # go up one level
cd ~                          # go home
cd -                          # go back to where I just was
```

**Absolute vs. relative paths.** A path starting with `/` is absolute — it means the
same thing from anywhere, like a full postal address. Anything else is relative to where
you currently are, like "second door on the left". In scripts, **always use absolute
paths**. It is more typing and it will save you hours.

## The Tab key is your best friend

Type the first few letters of a name and press **Tab**. The shell completes it. Press
Tab twice to see all the possibilities.

This is not a nicety. It prevents typos, and with folder names like
`4. HR_IRB_BilandzijaH_March2022_RNAseq_WBI - X204SC22040339-Z01-F002_DIRECTIONAL` it is
the only sane way to work.

Press the **↑ arrow** to bring back the previous command and edit it instead of retyping.

## Looking inside files

Our files are huge. Never open one blindly.

```bash
less somefile.txt    # scroll through it: arrows to move, q to quit, /word to search
head somefile.txt    # first 10 lines
tail somefile.txt    # last 10 lines
head -50 somefile.txt   # first 50 lines
wc -l somefile.txt   # how many lines?
```

`less` is safe on any size of file because it only loads the part you are looking at.
`cat` prints the whole thing to the screen — fine for a 10-line file, catastrophic for a
5 GB one.

**Compressed files** (`.gz`) are not readable directly. Use the `z` versions:

```bash
zless file.fq.gz
zcat file.fq.gz | head -4     # look at the first read of a FASTQ file
```

The `|` is a "pipe": it feeds the output of the left command into the right one. Here it
means "decompress, but only keep the first 4 lines". Without `head`, you would print 5 GB
of sequence into your terminal.

Try it on real data:

```bash
zcat "/beegfs/home/lguerrer/Asellidae/5. Hr_IRB_BilandzijaH_November2022_WBI_X204SC22101552-Z01-F001_DIRECTIONAL/01.RawData/AALS_G1DD_P1/AALS_G1DD_P1_1.fq.gz" | head -4
```

You will get four lines: a header starting with `@`, the DNA sequence, a `+`, and a line
of quality scores. That is one read. There are about 35 million of them in that file.

## Finding things

```bash
find /beegfs/home/lguerrer/Asellidae -name "*.fq.gz" | wc -l      # count all fastq files
find /beegfs/home/lguerrer/Asellidae -name "MD5.txt"              # where are the checksum files
grep "FAILED" results.txt                                          # find lines containing a word
grep -c ">" transcripts.fasta                                      # count matching lines
du -sh /beegfs/home/lguerrer/Asellidae/*                           # how big is each folder
```

`grep` searches inside files, `find` searches for files. You will use both constantly.

## Spaces in filenames — read this twice

Our sequencing folders came from Novogene with spaces and dots in their names:

```
4. HR_IRB_BilandzijaH_March2022_RNAseq_WBI - X204SC22040339-Z01-F002_DIRECTIONAL
```

The shell splits commands on spaces, so it reads that as **five separate arguments**.
This breaks:

```bash
cd /beegfs/home/lguerrer/Asellidae/4. HR_IRB...        # ✗ error
```

This works:

```bash
cd "/beegfs/home/lguerrer/Asellidae/4. HR_IRB_BilandzijaH_March2022_RNAseq_WBI - X204SC22040339-Z01-F002_DIRECTIONAL"    # ✓
```

**Rule: put double quotes around every path, always.** Also around variables that hold
paths: `"$RAWDIR"`, not `$RAWDIR`. This single habit prevents the most common and most
confusing class of bug you will hit on this project.

When you create your *own* folders, never use spaces. Use `_`.

## Making and removing things

```bash
mkdir myfolder              # make a folder
mkdir -p a/b/c              # make a whole nested path at once
cp file1 file2              # copy
cp -r folder1 folder2       # copy a folder
mv oldname newname          # rename or move
rm file                     # delete — permanently
```

> ### There is no undo and no recycle bin
> `rm` deletes immediately and forever. There is no trash to recover from.
>
> Before pressing Enter on any `rm`, read the line once more. If it contains `*`, read
> it twice. If it points anywhere inside `Asellidae/`, do not run it — ask Laura.

## Editing text on the server

You need to edit job scripts. Use **nano**, which is the friendly one:

```bash
nano myscript.sh
```

The commands are listed at the bottom of the screen; `^` means Ctrl.

- **Ctrl+O** then Enter — save ("write Out")
- **Ctrl+X** — exit
- **Ctrl+K** — cut the current line
- **Ctrl+W** — search

You will also hear about `vim`. It is more powerful and much harder. If you ever open it
by accident, press `Esc` then type `:q!` and Enter to escape.

## Loading software

Analysis programs are not available by default. The cluster keeps them in **modules**
that you switch on when needed:

```bash
module avail                       # list everything installed (it is a long list)
module avail 2>&1 | grep -i fastqc # search for one thing
module load scientific/FastQC/0.12.1
module list                        # what do I have loaded right now
module purge                       # unload everything
```

A module you load in your terminal disappears when you log out. That is why **every job
script must load its own modules** — the compute node starts with nothing loaded.

Programs relevant to us include `FastQC`, `fastp`, `multiqc`, `HISAT2`, `STAR`,
`salmon`, `samtools`, `trinity`, `transdecoder`, `busco`, `EggNOG-mapper`,
`orthofinder`, `paml`, `hyphy`. All already installed — we do not have to compile
anything.

## Things that will happen to you

| Situation | What to do |
|---|---|
| A command is taking forever and you want out | **Ctrl+C** |
| The terminal prints gibberish forever | Ctrl+C, then `reset` |
| `command not found` | You forgot `module load`, or you misspelled it |
| `No such file or directory` | Wrong path. Check with `ls` on the parent folder — usually a missing quote around a space |
| `Permission denied` on a script | `chmod +x script.sh` makes it executable |
| You are lost | `pwd`, then `cd ~` and start again |

## Practise before moving on

Do all of these. They only read, they change nothing.

```bash
# 1. Go to the data and look around
cd /beegfs/home/lguerrer/Asellidae
ls -lh

# 2. How big is each delivery?
du -sh */

# 3. How many sequencing files are there in total?
find . -name "*.fq.gz" | wc -l          # expect 192

# 4. Look at one sample folder (use Tab completion to type the path!)
ls -lh "5. Hr_IRB_BilandzijaH_November2022_WBI_X204SC22101552-Z01-F001_DIRECTIONAL/01.RawData/AALS_G1DD_P1"

# 5. Read its checksum file
cat "5. Hr_IRB_BilandzijaH_November2022_WBI_X204SC22101552-Z01-F001_DIRECTIONAL/01.RawData/AALS_G1DD_P1/MD5.txt"

# 6. Look at the first read of that sample
zcat "5. Hr_IRB_BilandzijaH_November2022_WBI_X204SC22101552-Z01-F001_DIRECTIONAL/01.RawData/AALS_G1DD_P1/AALS_G1DD_P1_1.fq.gz" | head -4
```

If step 6 prints four lines ending in a row of `F`s, you are ready for the next chapter.

---

Next: [03 — Running jobs with PBS](03-running-jobs-with-pbs.md)
