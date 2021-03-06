ChIP-Pipeline
==============
My Custom Snakemake ChIP-Seq Pipeline
--
```
"""
Author: Mahesh Bachu
Aim: My Snakemake workflow to process single-end ChIP-seq data
Run: snakemake -s ChIP.snake
"""

#Custom ChIP-Seq Pipeline Mahesh Bachu
#Modules to load
#bowtie
#samtools
#bedtools
#bamtools
#fastqc
#trim_galore (cutadapt)
#macs2
#sicer
#homer
#deeptools


#bash safe mode
shell.executable("/bin/bash")
shell.prefix("set -euo pipefail; ")
#configfile: "config.yaml"
#
from os.path import join

# Globals ---------------------------------------------------------------------


# Globals ---------------------------------------------------------------------

# Full path to a FASTA file.
GENOME = '/home/maheshb/reference_genomes/Homo_sapiens_Ensembl_GRCh37/Bowtie2Index/genome.fa'

# Full path to a folder that holds all of your FASTQ files.
FASTQ_DIR = './Fastq-Files/'

# A Snakemake regular expression matching the forward mate FASTQ files.
SAMPLES, = glob_wildcards(join(FASTQ_DIR, '{sample,*.R1.fastq.gz'))

# Patterns for the 1st mate and the 2nd mate using the 'sample' wildcard.
PATTERN_R1 = '{sample}.R1.fastq.gz'
#PATTERN_R2 = '{sample}.R2.fastq.gz'

################################################################################################################

# Rules -----------------------------------------------------------------------

rule all:
    input:
        'test.txt'

ruleorder: fastqc > clean_fastq > bowtie2_mapping > flagstat_bam > bedtools_bed > tagdirectory_homer > call_broad_peaks > bigwig_deeptools_1x > bigwig_deeptools_RPKM

rule final:
input:
    expand("cleaned/{sample}_trimmed.fq.gz", sample=SAMPLES),
            expand("qc/{sample}_fastqc.html", sample=SAMPLES),
            expand("qc/{sample}_fastqc.zip", sample=SAMPLES),
            expand("out/{sample}/{sample}_trimmed.sorted.bam", sample=SAMPLES),
            expand("out/{sample}/{sample}_trimmed.sorted.bai", sample=SAMPLES),
            expand("out/{sample}_vs_{frag}.bamcompare.bigwig", zip,sample=CASES,frag=CONTROLS),
            expand("out/{sample}/{sample}_vs_{frag}_peaks.narrowPeak",zip,sample=CASES,frag=CONTROLS),
    expand("out/{sample}/{sample}_vs_{frag}_peaks.broadPeak",zip,sample=CASES,frag=CONTROLS),

################################################################################################################
rule fastqc:
    input:  "r1 = join(FASTQ_DIR, PATTERN_R1)"
    output: "FQC-Report/{chip}_fastqc.zip", "FQC-Report/{chip}_fastqc.html"
    log:    "00log/{chip}_fastqc"
    threads: 8
    resources: mem_mb=4
    message: "fastqc {input}: {threads} / {resources.mem_mb}"
    shell:
        "fastqc -o FQC-Report -f fastq --noextract {input[0]}"

## use trimmomatic to trim low quality bases and adaptors
rule clean_fastq:
    input:   "{chip}.fastq.gz"
    output:  "clean_fastq/{chip}_clean.fastq.gz"
    log:     "00log/{chip}_clean_fastq"
    threads: 8
    resources: mem_mb=16
    message: "clean_fastq {input}: {threads} threads / {resources.mem_mb}"
    shell:
        """
        trimmomatic SE {input} {output} \
        ILLUMINACLIP:Truseq_adaptor.fa:2:30:10 LEADING:3 \
        TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36 2> {log}
        """

rule bowtie2_mapping:
    input: 
        "clean_fastq/{chip}_clean.fastq.gz"
    genome = GENOME
    output:"mapped_reads/{chip}.bam"
    params:idx = "/home/maheshb/reference_genomes/Homo_sapiens_Ensembl_GRCh37/Bowtie2Index/genome",
    threads: 8
    resources: mem_mb=16
    shell:
        """"
        bowtie2 --sensitive-local -p {threads} --no-unal -x {params.idx} -U ${input.fq} | samtools view -q30 -Sb - > {output}
        cd mapped_reads
        samtools sort -T /tmp/${chip}.bam -o ${chip}_sorted.bam ${chip}.bam && rm ${chip}.bam && samtools rmdup -s ${chip}_sorted.bam ${chip}_sorted_rmdup.bam && rm ${chip}_sorted.bam && samtools index ${j}_sorted_rmdup.bam"
        """"

rule flagstat_bam:
    input:  "mapped_reads/{chip}_sorted_rmdup.bam"
    output: "mapped_reads/{chip}_sorted_rmdup.bam.flagstat"
    log:    "00log/{chip}_sorted_rmdup.flagstat_bam"
    threads: 8
    resources: mem_mb=16
    message: "flagstat_bam {input}: {threads} threads / {resources.mem_mb}"
    shell:
        ""
        cd mapped_reads
        samtools flagstat {input} > {output} 2> {log}
        ""
rule bedtools_bed:
        input:"mapped_reads/{chip}_sorted_rmdup.bam"
        output: "../Bed_Files/{chip}_sorted_rmdup.bed"
        log:    "00log/{chip}_sorted_rmdup_bed_log"
        threads: 8
        resources: mem_mb=8
        message: "bedtools_bed {input}: {threads} threads / {resources.mem_mb}"
        shell:
            ""
            cd mapped_reads
            bedtools bamtobed -i ${input} > {output}
            ""
rule tagdirectory_homer:
        input:"Bed_Files/{chip}_sorted_rmdup.bed"
        output: "{chip}_sorted_rmdup"
        log:    "00log/{chip}_sorted_rmdup_mkdir_log"
        threads: 8
        resources: mem_mb=8
        message: "tagdirectory_homer {input}: {threads} threads / {resources.mem_mb}"
        shell:
            ""
            cd Bed_Files
            makeTagDirectory ${output}/ ${input} -format bed
            ""
rule bigwig_deeptools_1x:
    input:"mapped_reads/{chip}_sorted_rmdup.bam"
    output: "../DeepTools-BigWigs-1xDepth/{chip}_sorted_rmdup.SeqDepthNorm.bw"
    log: "00log/{chip}_sorted_rmdup_bigwig_log"
    threads: 8
    resources: mem_mb=16
    message: "bigwig_deeptools_1x {input}: {threads} threads / {resources.mem_mb}"
    shell:
        ""
        cd mapped_reads
        bamCoverage --bam ${input} -of bigwig -o ${output} --scaleFactor 1 --binSize 10 --normalizeTo1x 2150570000 --ignoreForNormalization chrX chrM --extendReads 150 --centerReads --smoothLength 30
        ""

rule bigwig_deeptools_RPKM:
    input:"mapped_reads/{chip}_sorted_rmdup.bam"
    output: "../DeepTools-BigWigs-RPKM/{chip}_sorted_rmdup.RPKMNorm.bw"
    log: "00log/{chip}_sorted_rmdup_bigwig_RPKM_log"
    threads: 32
    resources: mem_mb=16
    message: "bigwig_deeptools_1x {input}: {threads} threads / {resources.mem_mb}"
    shell:
        ""
        cd mapped_reads
        bamCoverage --bam ${input} -of bigwig -o ${output} --scaleFactor 1 --binSize 10 --normalizeUsingRPKM --ignoreForNormalization chrX chrM --extendReads 150 --centerReads --smoothLength 30
        ""
```
