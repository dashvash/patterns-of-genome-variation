# Comparing patterns of genome variation in contemporary and ancient humans
Bioinformatics Institute 2024/25 course project

## Introduction

In this study we aim to trace differences of genome variation in contemporary and ancient humans. For those purposes we use all the detected and annotated SNPs to compare genetic load of high-impact variants. We expected to detect evidence of relaxed selection -  This hypothesis suggests that as nowadays natural selection weakens in urbanising environments due to healthcare, better nutrition, etc. genetic mutations  with high functional impact accumulate in the population instead of being efficiently purged by natural selection.

## Obtaining data

### Ancient DNA
- FASTQ files from the Population genomics of the Viking world project: [ENA PRJEB37976](https://www.ebi.ac.uk/ena/browser/view/PRJEB37976)
### Modern DNA
- FASTQ files from 1000 Genomes project [IGSR Portal](https://www.internationalgenome.org/data-portal/sample)

## Pipeline set up

For this project we choose a Nextflow pipeline [nf-core/eager](https://nf-co.re/eager/2.5.2) v.2.5.2.
Nextflow version 22.10.6 is required, the installetion guide can be found [here](https://www.nextflow.io/docs/latest/install.html#installation.)

Command to run the pipeline:

    nextflow run nf-core/eager -r 2.5.2 -c bwa.config  -params-file nf-params.json  -profile docker

params-file nf-params.json contain paths to regerence genome, analysing fastq-files and specifies the reqiered tools and their parameters. The examples of bwa.config and nf-params.json can be found in this repository.

As a result of genotyping we got vcf file for every sample and annotated them with [VEP](https://www.ensembl.org/info/docs/tools/vep/script/vep_download.html#docker) using Docker:

    docker run -v $HOME/vep_data:/data ensemblorg/ensembl-vep vep -i vcf/sample_name.unifiedgenotyper.vcf  --vcf -o sample_name.vep.vcf --cache --offline

## VCF file processing

After running VEP we got vcf file with annotated variants' impact levels: MODIFIER, LOW, MODERATE, HIGH.

In the initial step, we quantified the total number of SNPs and the count of each of the four impact variants type:

        grep -v '##' sample_name.vcf | wc -l
        grep 'HIGH' sample_name.vcf | wc -l
        grep 'MODERATE' sample_name.vcf | grep -v 'HIGH'| wc -l
        grep 'LOW' sample_name.vcf | grep -v -e 'HIGH' -e 'MODERATE' | wc -l
        grep 'MODIFIER' sample_name.vcf | grep -v -e 'HIGH' -e 'MODERATE' -e 'LOW' | wc -l

### Сropping ancient reads

To crop ancient fasts reads from 5' and 3' ends by 7 bp, we used [chopper](https://github.com/wdecoster/chopper) tool:

        chopper --headcrop 7 --tailcrop 7 -i sample_name.fastq > sample_name_cropped.fastq
