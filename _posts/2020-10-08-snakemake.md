---
title: 'An introduction to Snakemake for pipeline management'
date: 2020-10-07
permalink: /posts/2020/10/snakemake/
layout: single

# table of contents
toc: true
toc_label: "Contents"
toc_icon: "dna"
toc_sticky: true
---

A quick-start guide for Snakemake

Motivation
---

There are few (if any) scientific questions that you can answer by running a single program or script. Calling variants involves aligning reads, sorting reads, indexing reads, running a variant caller, filtering those calls, etc. Running a simulation will invariably require changing parameters or including new combinations of those parameters.

And there are few (if any) programs or scripts that will run/compile correctly on the first try, due to either user error, code bugs, dependency conflicts, improperly configured environments, or any combination therein.

For these reasons, it's often crucial to package the various steps of your analysis into a "pipeline." Ideally, this pipeline would accept a file(s) as input, do some stuff with that file, and generate an output file(s). For example, your pipeline might take a FASTQ file and reference genome FASTA as input, and output an aligned, sorted, and indexed BAM.

In theory, a pipeline could just be a Bash script in which you enumerate each step of the process. But what if you want to run the pipeline on hundreds of samples' FASTQ files? And what if the final step in your Bash script fails? You'll have to re-run **the entire pipeline all over again.**

This is where Snakemake comes in. Snakemake is a flexible Python-based pipeline manager, and it's even tuned for running on the Sage Grid Engine (or pretty much any other compute environment).

As an example, let's imagine that we want to take paired-end FASTQ from 3 different *E. coli* samples (SRR2589044, SRR2584863, and SRR2584866) and generate a preliminary set of variant calls for each sample.

To start, let's imagine we only want to process one sample: SRR2589044.

Every step of the pipeline gets its own "rule"
---

In this example, we'll be working with publicly available data from Richard Lenski's long-term *E. coli* experimental evolution project (an example dataset I stole from [Data Carpentry](https://datacarpentry.org/wrangling-genomics/aio.html)).

The first step of our pipeline will be to download the FASTQ data. In Snakemake each step of the pipeline is defined as a rule:

```
rule download_fastq:
	input: 
	output: 
		"SRR2589044_1.fastq.gz",
		"SRR2589044_2.fastq.gz"
	shell:
		"""
		wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR258/004/SRR2589044/SRR2589044_1.fastq.gz
		wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR258/004/SRR2589044/SRR2589044_2.fastq.gz
		"""
```

This rule, which we've named "download_fastq," doesn't take any input, since we're just downloading sequence data directly.

We specify that the expected output of this rule is two gzipped FASTQ files, which have fairly self-explanatory names.

After `shell:`, we simply list the commands we'd normally type at the command line to produce the specified output.

The next step of our pipeline will be to download an *E. coli* reference genome so that we can align these reads.

```
rule download_reference:
	input: 
	output:
		"GCA_000017985.1_ASM1798v1_genomic.fna.gz"
	shell:
		"""
		wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/017/985/GCA_000017985.1_ASM1798v1/GCA_000017985.1_ASM1798v1_genomic.fna.gz
		"""
```

Now that we have both FASTQ and and reference genome, we can align the reads using BWA-MEM.

```
rule bwa_align:
	input: 
		ref = "GCA_000017985.1_ASM1798v1_genomic.fna.gz"
		fq1 = "SRR2589044_1.fastq.gz",
		fq2 = "SRR2589044_2.fastq.gz"
	output:
		"SRR2589044.bam"
	shell:
		"""
		bwa mem -t 4 {input.ref} {input.fq1} {input.fq2} | samtools view -b > {output}
		"""
```

This rule takes as input a reference genome and two FASTQ files. As you can see, it's possible to name individual input or output files (using python variable assignment) so that we can access particular files in our shell command.


Snakemake will only run a rule if it has to
---

Let's take a look at the full pipeline we've defined so far.

```
rule all:
	input:
		"SRR2589044.bam"

rule download_fastq:
	input: 
	output: 
		"SRR2589044_1.fastq.gz",
		"SRR2589044_2.fastq.gz"
	shell:
		"""
		wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR258/004/SRR2589044/SRR2589044_1.fastq.gz
		wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR258/004/SRR2589044/SRR2589044_2.fastq.gz
		"""

rule download_reference:
	input: 
	output:
		"GCA_000017985.1_ASM1798v1_genomic.fna.gz"
	shell:
		"""
		wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/017/985/GCA_000017985.1_ASM1798v1/GCA_000017985.1_ASM1798v1_genomic.fna.gz
		"""

rule bwa_align:
	input: 
		ref = "GCA_000017985.1_ASM1798v1_genomic.fna.gz"
		fq1 = "SRR2589044_1.fastq.gz",
		fq2 = "SRR2589044_2.fastq.gz"
	output:
		"SRR2589044.sorted.bam"
	shell:
		"""
		bwa mem -t 4 {input.ref} {input.fq1} {input.fq2} | sambamba view -S -f bam /dev/stdin | sambamba sort -o {output} /dev/stdin
		"""
```

**A huge advantage of Snakemake is that it will only run a rule if its output is needed by a downstream rule.**

You can see that I've added a rule (called `all`) to the top of the pipeline. This is because Snakemake runs in a "bottom-up" fashion.

**The `all` rule tells Snakemake what the final output of the entire pipeline should be.** In this case, we want the final output to be an aligned BAM.

In this example, Snakemake finds the rule that outputs `SRR2589044.bam` (which is `bwa_align`), and checks to see if that rule has access to all of its necessary inputs (a reference and two FASTQ files). If not, Snakemake finds the rules that produce those files, and runs them. And so on. Once `bwa_align` has all of the inputs it needs, Snakemake runs it to produce the final output.

And let's say that for whatever reason, `bwa_align` fails when its run. When we re-run the pipeline, Snakemake will check that its input files are present. Since all of the previous rules will have been run before invoking `bwa_align`, Snakemake will see that its inputs are present and won't re-run any of the upstream steps!

Using `expand` to run a pipeline on many samples or with many parameters
---

In the previous example, we were only running the pipeline on a single sample. But to generalize the pipeline to run on any list of samples, we can make use of the `expand` feature, as well as Snakemake "wildcards."

Here's an example of what our pipeline would look like with wildcard placeholders instead of explicit sample names.

```
samples = ["SRR2589044", "SRR2584863", "SRR2584866"]

rule all:
	input:
		expand("{sample}.bam", sample=samples)

rule download_fastq:
	input: 
	output: 
		"{sample}_1.fastq.gz",
		"{sample}_2.fastq.gz"
	shell:
		"""
		wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR258/004/{sample}/{sample}_1.fastq.gz
		wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR258/004/{sample}/{sample}_2.fastq.gz
		"""

rule download_reference:
	input: 
	output:
		"GCA_000017985.1_ASM1798v1_genomic.fna.gz"
	shell:
		"""
		wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/017/985/GCA_000017985.1_ASM1798v1/GCA_000017985.1_ASM1798v1_genomic.fna.gz
		"""

rule bwa_align:
	input: 
		ref = "GCA_000017985.1_ASM1798v1_genomic.fna.gz"
		fq1 = "{sample}_1.fastq.gz",
		fq2 = "{sample}_2.fastq.gz"
	output:
		"{sample}.sorted.bam"
	shell:
		"""
		bwa mem -t 4 {input.ref} {input.fq1} {input.fq2} | sambamba view -S -f bam /dev/stdin | sambamba sort -o {output} /dev/stdin
		"""
```

We've just replaced every instance of a sample name with a `{sample}` wildcard. And in the `all` rule, I've 

Visualizing the full pipeline with a DAG
---

We can actually visualize the full pipeline using Snakemake.

After putting the full pipeline in a file called `Snakefile`, we can run:

```
snakemake --dag | dot -Tsvg > dag.svg
```

This will produce the following image, showing us exactly what steps Snakemake will during execution.


