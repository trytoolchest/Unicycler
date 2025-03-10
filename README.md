<p align="center"><img src="misc/logo.png" alt="Unicycler" width="600"></p>

Unicycler is an assembly pipeline for bacterial genomes. It can assemble [Illumina](http://www.illumina.com/)-only read sets where it functions as a [SPAdes](http://cab.spbu.ru/software/spades/)-optimiser. It can also assembly long-read-only sets ([PacBio](http://www.pacb.com/) or [Nanopore](https://nanoporetech.com/)) where it runs a [miniasm](https://github.com/lh3/miniasm)+[Racon](https://github.com/lbcb-sci/racon) pipeline. For the best possible assemblies, give it both Illumina reads _and_ long reads, and it will conduct a hybrid assembly.

Read more about Unicycler here:
> [__Wick RR, Judd LM, Gorrie CL, Holt KE.__ Unicycler: resolving bacterial genome assemblies from short and long sequencing reads. _PLoS Comput Biol_ 2017.](http://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1005595)

And read about how we use it to complete bacterial genomes here:
> [__Wick RR, Judd LM, Gorrie CL, Holt KE.__ Completing bacterial genome assemblies with multiplex MinION sequencing. _Microb Genom_ 2017.](http://mgen.microbiologyresearch.org/content/journal/mgen/10.1099/mgen.0.000132)



# A note on Trycycler

[Trycycler](https://github.com/rrwick/Trycycler/wiki) is a newer tool that in many cases is a better choice than Unicycler. Here is a quick guide on whether you should use Unicycler or Trycycler to assemble your bacterial genome:
* If you only have short reads, use Unicycler (Trycycler does not do short-read assembly).
* If you only have long reads, Trycycler is a better choice. While Unicycler can do long-read-only assembly, its approach is somewhat out-of-date. Something else to consider is the depth of your long reads, as Trycycler prefers deeper read sets. If your long-read set is particularly shallow (~25× or less), then [Flye](https://github.com/fenderglass/Flye) might be your best option.
* If you have both short and long reads (i.e. are doing a hybrid assembly), then Unicycler and [Trycycler+polishing](https://github.com/rrwick/Trycycler/wiki/Polishing-after-Trycycler) are both viable options. If you have lots of long reads (~100× depth or more), use Trycycler+polishing. If you have sparse long reads (~25× or less), use Unicycler. If your long-read depth falls between those values, it might be worth trying both approaches.

You can read more on Trycycler's FAQ page: [Should I use Unicycler or Trycycler to assemble my bacterial genome?](https://github.com/rrwick/Trycycler/wiki/FAQ-and-miscellaneous-tips#should-i-use-unicycler-or-trycycler-to-assemble-my-bacterial-genome)



# Table of contents

* [Introduction](#introduction)
* [Requirements](#requirements)
* [Installation](#installation)
    * [Install from source](#install-from-source)
    * [Build and run without installation](#build-and-run-without-installation)
* [Quick usage](#quick-usage)
* [Background](#background)
    * [Assembly graphs](#assembly-graphs)
    * [Limitations of short reads](#limitations-of-short-reads)
    * [SPAdes graphs](#spades-graphs)
* [Method: Illumina-only assembly](#method-illumina-only-assembly)
    * [Read correction](#read-correction)
    * [SPAdes assembly](#spades-assembly)
    * [Multiplicity](#multiplicity)
    * [Overlap removal](#overlap-removal)
    * [Bridging](#bridging)
* [Method: long-read-only assembly](#method-long-read-only-assembly)
    * [miniasm assembly](#miniasm-assembly)
    * [Racon polishing](#racon-polishing)
* [Method: hybrid assembly](#method-hybrid-assembly)
    * [Long read plus contig assembly](#long-read-plus-contig-assembly)
    * [Direct long read bridging](#direct-long-read-bridging)
    * [Bridge application](#bridge-application)
    * [Finalisation](#finalisation)
* [Conservative, normal and bold](#conservative-normal-and-bold)
* [Options and usage](#options-and-usage)
    * [Standard options](#standard-options)
    * [Advanced options](#advanced-options)
* [Output files](#output-files)
* [Tips](#tips)
    * [Running time](#running-time)
    * [Necessary read length](#necessary-read-length)
    * [Poretools](#poretools)
    * [Nanopore: 1D vs 2D](#nanopore-1d-vs-2d)
    * [Porechop](#porechop)
    * [Bad Illumina reads](#bad-illumina-reads)
    * [Very short contigs](#very-short-contigs)
    * [Chromosomes and plasmid depth](#chromosomes-and-plasmid-depth)
    * [Known contamination](#known-contamination)
    * [Manual multiplicity](#manual-multiplicity)
    * [Manual completion](#manual-completion)
    * [Using an external long-read assembly](#using-an-external-long-read-assembly)
* [Other included tools](#other-included-tools)
* [Paper](#paper)
* [Acknowledgements](#acknowledgements)
* [License](#license)



# Introduction

As input, Unicycler takes one of the following:
* Illumina reads from a bacterial isolate (ideally paired-end, but unpaired works too)
* A set of long reads (either PacBio or Nanopore) from a bacterial isolate (uncorrected long reads are fine, though corrected long reads should work too)
* Illumina reads and long reads from the same isolate (best case)

Reasons to use Unicycler:
* It circularises replicons without the need for a separate tool like [Circlator](http://sanger-pathogens.github.io/circlator/).
* It handles plasmid-rich genomes.
* It can use long reads of any depth and quality in hybrid assembly. 10x or more may be required to complete a genome, but Unicycler can make nearly-complete genomes with far fewer long reads.
* It produces an assembly _graph_ in addition to a contigs FASTA file, viewable in [Bandage](https://github.com/rrwick/Bandage).
* It has very low misassembly rates.
* It can cope with very repetitive genomes, such as [_Shigella_](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC153260/).
* It's easy to use: runs with just one command and usually doesn't require tinkering with parameters.

Reasons to __not__ use Unicycler:
* You're assembling a eukaryotic genome or a metagenome (Unicycler is designed exclusively for bacterial isolates).
* Your Illumina reads and long reads are from different isolates (Unicycler struggles with sample heterogeneity).
* You're impatient (Unicycler is thorough but not especially fast).



# Requirements

* Linux or macOS
* [Python](https://www.python.org/) 3.4 or later
* C++ compiler with C++14 support:
    * [GCC](https://gcc.gnu.org/) 4.9.1 or later
    * [Clang](http://clang.llvm.org/) 3.5 or later
    * [ICC](https://software.intel.com/en-us/c-compilers) also works (though I don't know the minimum required version number)
* [setuptools](https://packaging.python.org/installing/#install-pip-setuptools-and-wheel) (only required for installation of Unicycler)
* For short-read or hybrid assembly:
  * [SPAdes](http://bioinf.spbau.ru/spades) v3.6.2 – v3.13.0 (`spades.py`)
* For long-read or hybrid assembly:
  * [Racon](https://github.com/lbcb-sci/racon) (`racon`)
* For polishing
  * [Pilon](https://github.com/broadinstitute/pilon/wiki) (`pilon1.xx.jar`)
  * [Java](https://www.java.com/download/) (`java`)
  * [Bowtie2](http://bowtie-bio.sourceforge.net/bowtie2/) (`bowtie2-build` and `bowtie2`)
  * [Samtools](http://www.htslib.org/) v1.0 or later (`samtools`)
* For rotating circular contigs:
  * [BLAST+](https://www.ncbi.nlm.nih.gov/books/NBK279690/) (`makeblastdb` and `tblastn`)

Some of these dependencies can be removed by turning off parts of the Unicycler pipeline (e.g. `--no_polish`). Unicycler expects these tools to be available in `$PATH`. If they aren't, you can specify their location using Unicycler options (e.g. `--samtools_path`).

[Bandage](https://github.com/rrwick/Bandage) isn't required to run Unicycler, but it is very helpful for manually investigating assemblies (the graph images in this README were made with Bandage). Make sure to get the [latest release](https://github.com/rrwick/Bandage/releases) for full compatibility with Unicycler.


# Installation

### Install from source
These instructions install the most up-to-date version of Unicycler:
```bash
git clone https://github.com/rrwick/Unicycler.git
cd Unicycler
python3 setup.py install
```

Notes:
* If the last command complains about permissions, you may need to run it with `sudo`.
* If you want a particular version of Unicycler, download the source from the [releases page](https://github.com/rrwick/Unicycler/releases) instead of cloning from GitHub.
* Install just for your user: `python3 setup.py install --user`
    * If you get a strange 'can't combine user with prefix' error, read [this](http://stackoverflow.com/questions/4495120).
* Install to a specific location: `python3 setup.py install --prefix=$HOME/.local`
* Install with pip (local copy): `pip3 install path/to/Unicycler`
* Install with pip (from GitHub): `pip3 install git+https://github.com/rrwick/Unicycler.git`
* Install with specific Makefile options: `python3 setup.py install --makeargs "CXX=icpc"`


### Build and run without installation

This approach compiles Unicycler code, but doesn't copy executables anywhere:
```bash
git clone https://github.com/rrwick/Unicycler.git
cd Unicycler
make
```
Now instead of running `unicycler`, you instead use `path/to/unicycler-runner.py`.



# Quick usage

__Illumina-only assembly:__<br>
`unicycler -1 short_reads_1.fastq.gz -2 short_reads_2.fastq.gz -o output_dir`

__Long-read-only assembly:__<br>
`unicycler -l long_reads_high_depth.fastq.gz -o output_dir`

__Hybrid assembly:__<br>
`unicycler -1 short_reads_1.fastq.gz -2 short_reads_2.fastq.gz -l long_reads.fastq.gz -o output_dir`

If you don't have any reads of your own, take a look in the [`sample_data`](sample_data/) directory for links to some small read sets.



# Background

### Assembly graphs

To understand what Unicycler is doing, you need to know about assembly graphs. For a thorough introduction, I'd suggest [this tutorial](http://homolog.us/Tutorials/index.php?p=1.1&s=1) or the [Velvet paper](http://genome.cshlp.org/content/genome/18/5/821.full.html). But in short, an assembly graph is a data structure where contigs aren't disconnected sequences but can have connections to each other:
```
Just contigs:               Assembly graph:

TCGAAACTTGACGCGAGTCGC                             CTTGTTTA
TGCTACTGCTTGATGATGCGG                            /        \
TGTCCATT                    TCGAAACTTGACGCGAGTCGC          TGCTACTGCTTGATGATGCGG
CTTGTTTA                                         \        /
                                                  TGTCCATT
```

Most assemblers use graphs internally to produce their assemblies, but users often ignore the graph in favour of the conceptually simpler FASTA file of contigs. When a genome assembly is 100% complete, we have one contig per chromosome/plasmid and there's no real need for the graph. But most assemblies are not complete (especially short-read assemblies), and a graph can describe an incomplete assembly much better than contigs alone.


### Limitations of short reads

The main reason we can't get a complete assembly from short reads is that DNA usually contains _repeats_ – the same sequence occurring two or more times in the genome. When a repeat is longer than the reads (or for paired-end sequencing, longer than the insert size), it forms a single contig in the assembly graph with multiple connections in and multiple connections out.

Here is what happens to a simple bacterial assembly graph as you add repeats to the genome:
<p align="center"><img src="misc/repeats_in_graph.png" alt="Repeats in graph"></p>

As repeats are added, the graph becomes increasingly tangled (and real assembly graphs get a lot more tangled than that).

To complete a bacterial genome assembly (i.e. find the one correct sequence for each chromosome/plasmid), we need to resolve the repeats. This means finding which way into a repeat matches up with which way out. Short reads don't have enough information for this but _long reads_ do.


### SPAdes graphs

Assembly graphs come in many different varieties, but we are particularly interested in the kind produced by SPAdes, because that is what Unicycler uses.

SPAdes graphs are made by performing a De Bruijn graph assembly with a range of different k-mer sizes, from small to large (see the [SPAdes paper](http://online.liebertpub.com/doi/abs/10.1089/cmb.2012.0021)). Each assembly builds on the previous one, which allows SPAdes to get the advantages of both small k-mer assemblies (a more connected graph) and large k-mer assemblies (ability to resolve repeats). Two contigs in a SPAdes graph that connect will overlap by their k-mer size (more info on the [Bandage wiki page](https://github.com/rrwick/Bandage/wiki/Assembler-differences)).

After producing the graph, SPAdes can perform further repeat resolution by using paired-end information. Since two reads in a pair are close to each other in the original DNA, SPAdes can use this to trace paths in the graph to form larger contigs (see [their paper on ExSPAnder](http://bioinformatics.oxfordjournals.org/content/30/12/i293.short)). However, the SPAdes contigs with repeat resolution do not come in graph form – they are only available in a FASTA file.



# Method: Illumina-only assembly

When assembling just Illumina reads, Unicycler functions mainly as a SPAdes optimiser. It offers a few benefits over using SPAdes alone:
* Tries a wide range of k-mer sizes and automatically selects the best.
* Filters out low-depth parts of the assembly to remove contamination.
* Applies SPAdes repeat resolution to the graph (as opposed to disconnected contigs in a FASTA file).
* Rejects low-confidence repeat resolution to reduce the rate of misassembly.
* Trims off graph overlaps so sequences aren't repeated where contigs join.

More information on the Illumina-only assembly process is described in the steps below.


### Read correction

Unicycler uses SPAdes' built-in read correction step before assembling the Illumina reads. This can be disabled with `--no_correct` if your Illumina reads are very high quality or you've already performed read QC.


### SPAdes assembly

<img align="right" src="misc/k-mer_plot.png" width="156" height="179">

Unicycler uses SPAdes to assemble the Illumina reads into an assembly graph. It tries assemblies at a wide range of k-mer sizes, evaluating the graph at each one. It chooses the graph which best minimises both contig count and dead end count. If the Illumina reads are good, it produces an assembly graph with long contigs but few to no dead ends ([more info here](#bad-illumina-reads)). Since a typical bacterial genome has no dead ends (the sequences are circular) an ideal assembly graph won't either.

A raw SPAdes graph can also contain some 'junk' sequences due to sequencer artefacts or contamination, so Unicycler performs some graph cleaning to remove these. Therefore, small amounts of contamination in the Illumina reads should not be a problem.


### Multiplicity

To scaffold the graph, Unicycler must distinguish between single copy contigs and repeats. It does this with a greedy algorithm that uses both read depth and graph connectivity:

<p align="center"><img src="misc/multiplicity.png" alt="Multiplicity assignment" width="700"></p>

This process does _not_ assume that all single copy contigs have the same read depth, which allows it to identify single copy contigs from plasmids as well as the chromosome. After it has determined multiplicity, Unicycler chooses a set of 'anchor' contigs. These are sufficiently-long single-copy contigs suitable for bridging in later steps.


### Overlap removal

To reduce redundancy and allow for neatly circularised contigs, Unicycler removes all overlap in the graphs:
<pre>
Before:                                       After:
                   <b>GACGCGT</b>TGACAAGGAAAT                           TGACAAGGAAAT
                  /                                             /
TTGACTACCCA<b>GACGCGT</b>                            TTGACTACCCAGACGCGT
                  \                                             \
                   <b>GACGCGT</b>CCTCTCATTCTA                           CCTCTCATTCTA
</pre>


### Bridging

At this point, the assembly graph does not contain the SPAdes repeat resolution. To apply this to the graph, Unicycler builds bridges between single copy contigs using the information in the SPAdes `contigs.paths` file.

<p align="center"><img src="misc/short_read_bridging.png" alt="Short read bridging" width="600"></p>

Bridges are given a quality score, most importantly based on the length of the bridge compared to the length of the paired end insert size, so bridges which span a long repeat are given a low score. Since paired-end sequencing cannot resolve repeats longer than the insert size, bridges which attempt to span long repeats cannot be trusted. This selectivity helps to reduce the number of misassemblies.



# Method: long-read-only assembly

When assembling just long reads, Unicycler uses a miniasm+Racon pipeline. It offers a couple advantages over using other long-read-only assemblers:
* Multiple rounds of Racon polishing give a good final sequence accuracy.
* Circular replicons (like most bacterial chromosomes and plasmids) assemble into circular replicons with no start-end overlap.

More information on the long-read-only assembly process is described in the steps below.


### miniasm assembly

Unicycler uses minimap and miniasm to assemble the long reads in essentially the same manner as described in the [miniasm README](https://github.com/lh3/miniasm). This produces an uncorrected assembly which is made directly of pieces of reads – the assembly error rate will be similar to the read error rate.

The version of miniasm that comes with Unicycler is slightly modified in a couple of ways. The first modification is to help circular replicons assemble into circular string graphs. The other modification only applies to hybrid assembly, so I'll come back to that!


### Racon polishing

After miniasm assembly, Unicycler carries out multiple rounds of polishing with [Racon](https://github.com/lbcb-sci/racon) to improve the sequence accuracy. It will polish until the assembly stops improving, as measured by the agreement between the reads and the assembly. Circular replicons are 'rotated' (have their starting position shifted) between rounds of polishing to ensure that no part of the sequence is left unpolished.



# Method: hybrid assembly

Hybrid assembly (using both Illumina read and long reads) is where Unicycler really shines. Like with the Illumina-only pipeline described above, Unicycler will produce an Illumina assembly graph. It then uses long reads to build bridges, which often allows it to resolve all repeats in the genome, resulting in a complete genome assembly.

In hybrid assembly, Unicycler carries out all the steps in the Illumina-only pipeline, plus the additional steps below:


### Long read plus contig assembly

This step uses miniasm and Racon, and is very much like the [long-read-only assembly method](#method-long-read-only-assembly) described above. Here however, the assembly is not just on long reads but a mixture of long reads and anchor contigs from the Illumina-only assembly. Since these anchor contigs can often be much longer than long reads (sometimes hundreds of kbp), they can significantly help the assembly. This takes advantage of the other modification to miniasm which was teased above. In Unicycler's miniasm, contigs and long reads are treated slightly differently in the string graph manipulations to better perform this step.

After the assembly is finished, Unicycler finds anchor contigs in the assembled sequence and uses the intervening sequences to create bridges:
```
assembled sequence:                 TATGGTCTCGCATGTTAATTCTACTCCCGAACTTGGCCCATCCCCGGCTAGGCTGGGCACTAGACGGTGGAT
anchor contigs:                         GTCTCGCATGTTAA    ACTCCCGAACTTGGCCCATCCCCGGC       GGCACTAGACGGTGG
intervening sequences for bridges:                    TTCT                          TAGGCTG
```


### Direct long read bridging

Unicycler also attempts to make long-read bridges directly by semi-globally aligning the long reads to the assembly graph. For each pair of single copy contigs which are linked by read alignments, Unicycler uses the read consensus sequence to find a connecting path and creates a bridge.

<p align="center"><img src="misc/long_read_bridging.png" alt="Long read bridging"></p>

This step and the previous step are somewhat redundant, as both use long reads to build bridges between short-read contigs. They are both included because they have different strengths. The previous approach can tolerate low long-read depth but requires a good short-read assembly graph (i.e. few dead ends). This step requires decent long-read depth but can tolerate poor short-read assembly graphs. By using the two strategies together, Unicycler can successfully handle many types of input.


### Bridge application

At this point of the pipeline there can be many bridges, some of which may conflict. Unicycler therefore assigns a quality score to each based on all available evidence (e.g. read alignment quality, graph path match, read depth consistency). Bridges are then applied in order of decreasing quality so whenever there is a conflict, only the most supported bridge is used. A minimum quality threshold prevents the application of low evidence bridges (see [Conservative, normal and bold](#conservative-normal-and-bold) for more information).

<p align="center"><img src="misc/bridge_application.png" alt="Application of bridges"></p>


### Finalisation

If the above steps have resulted in any simple, circular sequences, then Unicycler will attempt to rotate/flip them to begin at a consistent starting gene. By default this is [dnaA](http://www.uniprot.org/uniprot/?query=gene_exact%3AdnaA&sort=score) or [repA](http://www.uniprot.org/uniprot/?query=gene_exact%3ArepA&sort=score), but users can specify their own with the `--start_genes` option.

Finally, Unicycler does multiple rounds of short-read polishing using [Pilon](https://github.com/broadinstitute/pilon/wiki).



# Conservative, normal and bold

Unicycler can be run in three modes: conservative, normal (the default) and bold, set with the `--mode` option. Conservative mode is least likely to produce a complete assembly but has a very low risk of misassembly. Bold mode is most likely to produce a complete assembly but carries greater risk of misassembly. Normal mode is intermediate regarding both completeness and misassembly risk.

If the structural accuracy of your assembly is paramount to your research, conservative mode is recommended. If you want a completed genome, even if it contains a mistake or two, then use bold mode.

The specific differences between the three modes are as follows:

Mode         | Invocation                      | Short read bridges | Bridge quality threshold | Contig merging
------------ | ------------------------------- | ------------------ | ------------------------ | -------------------------------------
conservative | `‑‑mode conservative`           | not used           | high (25)                | contigs are only merged with bridges
normal       | `‑‑mode normal`<br>(or nothing) | used               | medium (10)              | contigs are merged with bridges and when their multiplicity is 1
bold         | `‑‑mode bold`                   | used               | low (1)                  | contigs are merged wherever possible

<p align="center"><img src="misc/conservative_normal_bold.png" alt="Conservative, normal and bold" width="550"></p>

In the above example, the conservative assembly is incomplete because some bridges fell below the quality threshold and were not applied. Its contigs, however, are extremely reliable. Normal mode nearly gave a complete assembly, but a couple of unmerged contigs remain. Bold mode completed the assembly, but since lower confidence regions were bridged and merged, there is a larger risk of error.



# Options and usage

### Standard options

Run `unicycler --help` to view the program's most commonly used options:

```
usage: unicycler [-h] [--help_all] [--version] [-1 SHORT1] [-2 SHORT2] [-s UNPAIRED] [-l LONG] -o OUT
                 [--verbosity VERBOSITY] [--min_fasta_length MIN_FASTA_LENGTH] [--keep KEEP] [-t THREADS]
                 [--mode {conservative,normal,bold}] [--linear_seqs LINEAR_SEQS] [--vcf]

       __
       \ \___
        \ ___\
        //
   ____//      _    _         _                     _
 //_  //\\    | |  | |       |_|                   | |
//  \//  \\   | |  | | _ __   _   ___  _   _   ___ | |  ___  _ __
||  (O)  ||   | |  | || '_ \ | | / __|| | | | / __|| | / _ \| '__|
\\    \_ //   | |__| || | | || || (__ | |_| || (__ | ||  __/| |
 \\_____//     \____/ |_| |_||_| \___| \__, | \___||_| \___||_|
                                        __/ |
                                       |___/

Unicycler: an assembly pipeline for bacterial genomes

Help:
  -h, --help                     Show this help message and exit
  --help_all                     Show a help message with all program options
  --version                      Show Unicycler's version number

Input:
  -1 SHORT1, --short1 SHORT1     FASTQ file of first short reads in each pair (required)
  -2 SHORT2, --short2 SHORT2     FASTQ file of second short reads in each pair (required)
  -s UNPAIRED, --unpaired UNPAIRED
                                 FASTQ file of unpaired short reads (optional)
  -l LONG, --long LONG           FASTQ or FASTA file of long reads (optional)

Output:
  -o OUT, --out OUT              Output directory (required)
  --verbosity VERBOSITY          Level of stdout and log file information (default: 1)
                                   0 = no stdout, 1 = basic progress indicators, 2 = extra info,
                                   3 = debugging info
  --min_fasta_length MIN_FASTA_LENGTH
                                 Exclude contigs from the FASTA file which are shorter than this
                                 length (default: 100)
  --keep KEEP                    Level of file retention (default: 1)
                                   0 = only keep final files: assembly (FASTA, GFA and log),
                                   1 = also save graphs at main checkpoints,
                                   2 = also keep SAM (enables fast rerun in different mode),
                                   3 = keep all temp files and save all graphs (for debugging)
  --vcf                          Produce a VCF by mapping the short reads to the final assembly
                                 (experimental, default: do not produce a vcf file)

Other:
  -t THREADS, --threads THREADS  Number of threads used (default: 8)
  --mode {conservative,normal,bold}
                                 Bridging mode (default: normal)
                                   conservative = smaller contigs, lowest misassembly rate
                                   normal = moderate contig size and misassembly rate
                                   bold = longest contigs, higher misassembly rate
  --linear_seqs LINEAR_SEQS      The expected number of linear (i.e. non-circular) sequences in
                                 the underlying sequence (default: 0)
```

### Advanced options

Run `unicycler --help_all` to see a complete list of the program's options. These allow you to turn off parts of the pipeline, specify the location of tools (only necessary if they are not in PATH) and adjust various settings:

```
SPAdes assembly:
  These options control the short-read SPAdes assembly at the beginning of the Unicycler
  pipeline.

  --spades_path SPADES_PATH      Path to the SPAdes executable (default: spades.py)
  --no_correct                   Skip SPAdes error correction step (default: conduct SPAdes error
                                 correction)
  --min_kmer_frac MIN_KMER_FRAC  Lowest k-mer size for SPAdes assembly, expressed as a fraction of
                                 the read length (default: 0.2)
  --max_kmer_frac MAX_KMER_FRAC  Highest k-mer size for SPAdes assembly, expressed as a fraction
                                 of the read length (default: 0.95)
  --kmers KMERS                  Exact k-mers to use for SPAdes assembly, comma-separated
                                 (example: 22,33,44, default: automatic)
  --kmer_count KMER_COUNT        Number of k-mer steps to use in SPAdes assembly (default: 10)
  --depth_filter DEPTH_FILTER    Filter out contigs lower than this fraction of the chromosomal
                                 depth, if doing so does not result in graph dead ends (default:
                                 0.25)
  --largest_component            Only keep the largest connected component of the assembly graph
                                 (default: keep all connected components)
  --spades_tmp_dir SPADES_TMP_DIR
                                 Specify SPAdes temporary directory using the SPAdes --tmp-dir
                                 option (default: make a temporary directory in the output
                                 directory)

miniasm+Racon assembly:
  These options control the use of miniasm and Racon to produce long-read bridges.

  --no_miniasm                   Skip miniasm+Racon bridging (default: use miniasm and Racon to
                                 produce long-read bridges)
  --racon_path RACON_PATH        Path to the Racon executable (default: racon)
  --existing_long_read_assembly EXISTING_LONG_READ_ASSEMBLY
                                 A pre-prepared long read assembly for the sample in GFA format.
                                 If this option is used, Unicycler will skip the miniasm/Racon
                                 steps and instead use the given assembly (default: perform long
                                 read assembly using miniasm/Racon)

Assembly rotation:
  These options control the rotation of completed circular sequence near the end of the
  Unicycler pipeline.

  --no_rotate                    Do not rotate completed replicons to start at a standard gene
                                 (default: completed replicons are rotated)
  --start_genes START_GENES      FASTA file of genes for start point of rotated replicons
                                 (default: start_genes.fasta)
  --start_gene_id START_GENE_ID  The minimum required BLAST percent identity for a start gene
                                 search (default: 90.0)
  --start_gene_cov START_GENE_COV
                                 The minimum required BLAST percent coverage for a start gene
                                 search (default: 95.0)
  --makeblastdb_path MAKEBLASTDB_PATH
                                 Path to the makeblastdb executable (default: makeblastdb)
  --tblastn_path TBLASTN_PATH    Path to the tblastn executable (default: tblastn)

Pilon polishing:
  These options control the final assembly polish using Pilon at the end of the Unicycler
  pipeline.

  --no_pilon                     Do not use Pilon to polish the final assembly (default: Pilon is
                                 used)
  --bowtie2_path BOWTIE2_PATH    Path to the bowtie2 executable (default: bowtie2)
  --bowtie2_build_path BOWTIE2_BUILD_PATH
                                 Path to the bowtie2_build executable (default: bowtie2-build)
  --samtools_path SAMTOOLS_PATH  Path to the samtools executable (default: samtools)
  --pilon_path PILON_PATH        Path to a Pilon executable or the Pilon Java archive file
                                 (default: pilon)
  --java_path JAVA_PATH          Path to the java executable (default: java)
  --min_polish_size MIN_POLISH_SIZE
                                 Contigs shorter than this value (bp) will not be polished using
                                 Pilon (default: 10000)

VCF:
  These options control the production of the VCF of the final assembly.

  --bcftools_path BCFTOOLS_PATH  Path to the bcftools executable (default: bcftools)

Graph cleaning:
  These options control the removal of small leftover sequences after bridging is complete.

  --min_component_size MIN_COMPONENT_SIZE
                                 Graph components smaller than this size (bp) will be removed from
                                 the final graph (default: 1000)
  --min_dead_end_size MIN_DEAD_END_SIZE
                                 Graph dead ends smaller than this size (bp) will be removed from
                                 the final graph (default: 1000)

Long read alignment:
  These options control the alignment of long reads to the assembly graph.

  --contamination CONTAMINATION  FASTA file of known contamination in long reads
  --scores SCORES                Comma-delimited string of alignment scores: match, mismatch, gap
                                 open, gap extend (default: 3,-6,-5,-2)
  --low_score LOW_SCORE          Score threshold - alignments below this are considered poor
                                 (default: set threshold automatically)
```



# Output files

Unicycler's most important output files are `assembly.gfa`, `assembly.fasta` and `unicycler.log`. These are produced by every Unicycler run. Which other files are saved to its output directory depends on the value of `--keep`:
* `--keep 0` retains only the important files. Use this setting to save drive space.
* `--keep 1` (the default) also saves some intermediate graphs.
* `--keep 2` also retains the SAM file of long-read alignments to the graph. This ensures that if you rerun Unicycler with the same output directory (for example changing the mode to conservative or bold) it will run faster because it does not have to repeat the alignment step.
* `--keep 3` retains all files and saves many intermediate graphs. This is for debugging purposes and uses a lot of space. Most users should probably avoid this setting.

All files and directories are described in the table below. Intermediate output files (everything except for `assembly.gfa`, `assembly.fasta` and `unicycler.log`) will be prefixed with a number so they are in chronological order.

File                           | Description                                                                                       | `--keep` level
------------------------------ | ------------------------------------------------------------------------------------------------- | --------------
spades_assembly/               | directory containing all SPAdes files and each k-mer graph                                        | 3
best_spades_graph.gfa          | the best SPAdes short-read assembly graph, with a bit of graph clean-up                           | 1
overlaps_removed.gfa           | overlap-free version of the SPAdes graph, with some more graph clean-up                           | 3
miniasm_assembly/              | directory containing miniasm string graphs and unitig graphs                                      | 3
read_alignment/                | directory containing `long_read_alignments.sam`                                                   | 3
bridges_applied.gfa            | bridges applied, before any cleaning or merging                                                   | 1
cleaned.gfa                    | redundant contigs removed from the graph                                                          | 3
merged.gfa                     | contigs merged together where possible                                                            | 3
final_clean.gfa                | more redundant contigs removed                                                                    | 1
rotated.gfa                    | circular replicons rotated and/or flipped to a start position                                     | 1
polished.gfa                   | after a round of Pilon polishing                                                                  | 1
__assembly.gfa__               | final assembly in [GFA v1](https://github.com/GFA-spec/GFA-spec/blob/master/GFA1.md) graph format | 0
__assembly.fasta__             | final assembly in FASTA format (same contigs as in assembly.gfa)                                  | 0
__unicycler.log__              | Unicycler log file (same info as stdout)                                                          | 0



# Tips

### Running time

Unicycler is thorough and accurate, but not particularly fast. The [direct long read bridging](#direct-long-read-bridging) step of the pipeline can take a while to complete. Two main factors influence the running time: the number of long reads (more reads take longer to align) and the genome size/complexity (finding bridge paths is more difficult in complex graphs).

Unicycler may only take an hour or so to assemble a small, simple genome with low depth long reads. On the other hand, a complex genome with many long reads may take 12 hours to finish or more. If you have a very high depth of long reads, you can make Unicycler run faster by subsampling for only the longest reads.

Using a lot of threads (with the `--threads` option) can make Unicycler run faster too. It will only use up to 8 threads by default, but if you're running it on a big machine with lots of CPU and RAM, feel free to use more!

Unicycler also works with [PyPy](https://pypy.org/) which can speed up parts of its pipeline. However, some of Unicycler's slowest steps are when it calls other tools (like SPAdes) or uses C++ code, so PyPy may not help much. I haven't tested this thoroughly – if you try it, let me know how you go!


### Necessary read length

The length of a long read is very important, typically more than its accuracy, because longer reads are more likely to align to multiple single copy contigs, allowing Unicycler to build bridges.

Consider a sequence with a 2 kb repeat:
<p align="center"><img src="misc/read_length.png" alt="Long read length"></p>

In order to resolve the repeat, a read must span it by aligning to some sequence on either side. In this example, the 1 kb reads are shorter than the repeat and are useless. The 2.5 kb reads _can_ resolve the repeat, but they have to be in _just the right place_ to do so. Only one out of the six in this example is useful. The 5 kb reads, however, have a much easier time spanning the repeat and all three are useful.

So how long must your reads be for Unicycler to complete an assembly? _Longer than the longest repeat in the genome._ Depending on the genome, that might be a 1 kb insertion sequence, a 6 kb rRNA operon or a 50 kb prophage. If your reads are just a bit longer than the longest repeat, you'll probably need a lot of them. If they are much longer, then fewer reads should suffice. But in any scenario, _longer is better!_


### Poretools

[Poretools](http://poretools.readthedocs.io/en/latest/) can turn your Nanopore FAST5 reads into a FASTQ file appropriate for Unicycler. Here's an example command:
```bash
poretools fastq --type best --min-length 1000 path/to/fast5/dir/ > nanopore_reads.fastq
```
If you have 2D reads, `--type best` makes Poretools give only one FASTQ read per FAST5 file (if you have 1D reads, you can exclude that option). Adjust the `--min-length 1000` parameter to suit your dataset – a larger value would be appropriate if you have lots of long reads.


### Nanopore: 1D vs 2D

Since Unicycler can tolerate low accuracy reads, [Oxford Nanopore 1D sequencing](https://nanoporetech.com/applications/dna-nanopore-sequencing) is preferable to 2D, as it can provide twice as many reads. Unicycler will of course work with 2D reads, but you can get more out of your flow cell with 1D.

Update: 2D reads are now a thing of the past, replaced by 1D<sup>2</sup>. I haven't tried 1D<sup>2</sup> yet, thought I suspect the above advice still holds.


### Porechop

If you are using Oxford Nanopore reads, then I'd recommend using [Porechop](https://github.com/rrwick/Porechop) before assembly. This will trim adapters off read ends and split some chimeric reads, both of which make Unicycler's job a little bit easier.


### Bad Illumina reads

Unicycler prefers decent Illumina reads as input – ideally with uniform read depth and 100% genome coverage. Bad Illumina read sets can still work in Unicycler, but greater long-read depth will be required to compensate.

You can look at the `best_spades_graph.gfa` file (the first graph Unicycler saves to file) in Bandage to get a quick impression of the Illumina read quality:

<p align="center"><img src="misc/illumina_graph_comparison.png" alt="Graphs of varying quality" width="750"></p>

__A__ is an very good Illumina read graph – the contigs are long and there are no dead ends. This read set is ideally suited for use in Unicycler and shouldn't require too many long reads to complete (10–15x would probably be enough).

__B__ is also a good graph. The genome is more complex, resulting in a more tangled structure, but there are still very few dead ends (you can see one in the lower left). This read set would also work well in Unicycler, though more long reads may be required to get a complete genome (maybe 20x or so).

__C__ is a disaster! It is broken into many pieces, probably because parts of the genome got no read depth at all. This genome may take lots of long reads to complete in Unicycler, possibly 30x or more. The final assembly will probably have more small errors (SNPs and indels), as parts of the genome cannot be polished well with Illumina reads.


### Very short contigs

Confused by very small (e.g. 2 bp) contigs in Unicycler assemblies? Unlike a SPAdes graph where neighbouring sequences overlap by their k-mer size, Unicycler's final graph has no overlaps and the sequences adjoin directly. This means that contigs in complex regions can be quite short. They may be useless as stand-alone contigs but are still important in the graph structure.

<p align="center"><img src="misc/short_contigs.png" alt="Short contigs in assembly graph"></p>

If short contigs are a problem for your downstream analysis, you can use the `--min_fasta_length` to exclude them from Unicycler's FASTA file (they will still be included in the GFA file).


### Chromosomes and plasmid depth

Unicycler normalises the depth of contigs in the graph to the median value. This typically means that the chromosome has a depth near 1x and plasmids may have different (typically higher) depths.

<p align="center"><img src="misc/depth.png" alt="Plasmid depths"></p>

In the above graph, the chromosome is at the top (you can only see part of it) and there are two plasmids.  The plasmid on the left occurs in approximately 4 or 5 copies per cell. For the larger plasmid on the right, most cells probably had one copy but some had more. Since sequencing biases can affect read depth, these per cell counts should be interpreted loosely.


### Known contamination

If your long reads have known contamination, you can use the `--contamination` option to give Unicycler a FASTA file of the contaminant sequences. Unicycler will then discard any reads for which the best alignment is to the contaminant.

For example, if you've sequenced two isolates in succession on the same Nanopore flow cell, there may be residual reads from the first sample in the second run. In this case, you can supply a reference/assembly of the first sample to Unicycler when assembling the second sample.

Some Oxford Nanopore protocols include a lambda phage spike-in as a control. Since this is a common contaminant, you can simply use `--contamination lambda` to filter these out (no need to supply a FASTA file).


### Manual multiplicity

If Unicycler makes a serious mistake during its multiplicity determination, this can have detrimental effects on the rest of the assembly. I've seen this happen mainly in two scenarios:
* when the Illumina graph is badly fragmented (multiplcity determination has few graph connections to work with)
* there are multiple very similar plasmids in the genome (shared sequences between plasmids can be huge, 10s of kbp)

If you believe this has happened in your assembly, you can manually assign multiplicities and try the assembly again. Here's the process:
* View the short read assembly (`001_best_spades_graph.gfa`) in Bandage and view the region in question. Note that Unicycler's graph colour scheme uses green for single-copy segments and yellow/orange/red for multi-copy segments.
* For any segments where you disagree with Unicycler's multiplicity, add a `ML` tag to the GFA segment line in `001_best_spades_graph.gfa`. Examples:
  * If Unicycler called segment 50 single-copy but you think it's actually a 2-copy repeat, add `ML:i:2` to the end of the GFA line starting with `S    50`.
  * If Unicycler called segment 107 multi-copy but you think it's actually single-copy, add `ML:i:1` to the end of the GFA line starting with `S    107`.
* Run Unicycler again, pointing to the same output directory (with your modified `001_best_spades_graph.gfa` file). It will take your manually assigned multiplicities into account and hopefully do better!


### Manual completion

If Unicycler doesn't complete your bacterial genome assembly on its own, you may be able to complete it manually with a bit of bioinformatics detective work. There's no single, straight-forward procedure for doing so, but I've put together [a few examples on the Unicycler wiki](https://github.com/rrwick/Unicycler/wiki/Tips-for-finishing-genomes) which may be helpful.


### Using an external long-read assembly

If you have a long-read assembly that you've prepared outside Unicycler and trust (e.g. with Canu), you can give it to Unicycler with `--existing_long_read_assembly`. Unicycler will then skip its miniasm/Racon step and use this assembly instead.




# Other included tools

Unicycler also comes with a few other tools which may be of interest:

* [Unicycler-align](docs/unicycler-align.md): semi-global alignment of long reads
* [Unicycler-polish](docs/unicycler-polish.md): hybrid assembly polishing
* [Unicycler-scrub](docs/unicycler-scrub.md): read scrubber for trimming ends and splitting chimeras
* [Unicycler-check](docs/unicycler-check.md): misassembly detection and alignment visualisation

These tools may be experimental, incomplete or no longer under development, so use with caution!



# Paper

An open access article describing Unicycler is available from _PLOS Computational Biology_:<br>
[Unicycler: Resolving bacterial genome assemblies from short and long sequencing reads](http://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1005595)

If you use Unicycler in your research, a citation would be appreciated!



# Acknowledgements

Unicycler would not have been possible without [Kat Holt](https://holtlab.net/), my fellow researchers in her lab and the many other people I work with at the University of Melbourne's [Bio21 Molecular Science & Biotechnology Institute](http://www.bio21.unimelb.edu.au/). In particular, [Margaret Lam](https://scholar.google.com.au/citations?user=cWmhzUIAAAAJ), [Kelly Wyres](https://scholar.google.com.au/citations?user=anwFM9oAAAAJ), [David Edwards](https://scholar.google.com.au/citations?hl=en&user=rZ1RJK0AAAAJ) and [Claire Gorrie](https://scholar.google.com.au/citations?user=mSO9WPUAAAAJ) worked with me on many challenging genomes during Unicycler's development. [Louise Judd](https://scholar.google.com.au/citations?user=eO22mYUAAAAJ) is a whizz with the MinION and produced many of the long reads I have used when developing Unicycler.

Unicycler uses [SeqAn](https://www.seqan.de/) to perform alignments and other sequence manipulations. The authors of this library have been very helpful during Unicycler's development and I owe them a great deal of thanks! It also uses [minimap](https://github.com/lh3/minimap) for alignment and [miniasm](https://github.com/lh3/minimap) for long read assembly, and so I'd like to thank [Heng Li](https://github.com/lh3) for these tools. Finally, Unicycler uses [nanoflann](https://github.com/jlblancoc/nanoflann), a delightfully fast and lightweight nearest neighbour library, to perform its line-finding in semi-global alignment.



# License

[GNU General Public License, version 3](https://www.gnu.org/licenses/gpl-3.0.html)
