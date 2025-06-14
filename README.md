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

On the first stage of analysis we observed a statistically significant difference in the frequency of high-impact variants between ancient and modern samples, with a notably higher burden in ancient populations.

The difference likely reflects false-positive variants introduced by postmortem damage in ancient DNA:


![](images/boxplots.png)

To count SNPs with each type of substitution (A->T, A->C etc.) we used a bash script:

    output_file="substitution.txt"
    echo -n "" > "$output_file"  

    for file in *.vcf; do
        echo "File: $file" >> "$output_file"
        
        total_snps=$(grep -v "^#" "$file" | awk 'length($4)==1 && length($5)==1' | wc -l)

        if [ "$total_snps" -eq 0 ]; then
            echo "  No SNPs found." >> "$output_file"
            echo "" >> "$output_file"
            continue
        fi

        for ref in A C G T; do
            for alt in A C G T; do
                if [ "$ref" != "$alt" ]; then
                    count=$(grep -v "^#" "$file" | awk -v r="$ref" -v a="$alt" '$4==r && $5==a' | wc -l)
                    percent=$(awk -v c="$count" -v t="$total_snps" 'BEGIN { printf "%.2f", (c/t)*100 }')
                    echo "  $ref->$alt: $count SNPs (${percent}%)" >> "$output_file"
                fi
            done
        done

        echo "" >> "$output_file"
    done

The output file substitution.txt can be found in this repository.

We observed higher C->T (and G->A) substitutions % in ancient samples: 

![](images/C-T_substitution_optimized.png)

Postmortem C->T ang G->A substitutions lead to increase in false-positive impact variants, so we filtered them out.

To count VEP annotated variants without C->T and G->A substitution we run the following commands:

    awk '!/^##/ && !($4 == "C" && $5 == "T") && !($4 == "G" && $5 == "A")' sample_name.vcf | wc -l
    awk '$0 ~ /HIGH/ && !($4 == "C" && $5 == "T") && !($4 == "G" && $5 == "A")' sample_name.vcf | wc -l
    awk '$0 ~ /MODERATE/ && $0 !~ /HIGH/ && !($4 == "C" && $5 == "T") && !($4 == "G" && $5 == "A")' sample_name.vcf | wc -l
    awk '$0 ~ /LOW/ && $0 !~ /HIGH/ && $0 !~ /MODERATE/ && !($4 == "C" && $5 == "T") && !($4 == "G" && $5 == "A")' sample_name.vcf | wc -l
    awk '$0 ~ /MODIFIER/ && $0 !~ /HIGH/ && $0 !~ /MODERATE/ && $0 !~ /LOW/ && !($4 == "C" && $5 == "T") && !($4 == "G" && $5 == "A")' sample_name.vcf | wc -l

Then we counted heterozigous SNP excluding C->T and G->A:

    for file in *.vcf; do
        count=$(awk -F'\t' '($4 != "C"  $5 != "T") && ($4 != "G"  $5 != "A") && /0\/1/' "$file" | wc -l)
        echo "$file: $count"
    done

Het/Hom ratio metric supports that all the SNPs in cropped reads, excluding C->T and G->A, can be considered true:
![](images/het_hom_comparison.png)

#### Сropping ancient reads

To crop ancient fasts reads from 5' and 3' ends by 7 bp, we used [chopper](https://github.com/wdecoster/chopper) tool:

        chopper --headcrop 7 --tailcrop 7 -i sample_name.fastq > sample_name_cropped.fastq

The cropped fastq files were processed as the full ones (as described above)

Even after cropping and filtering out C->T and G->A there is difference between modern and ancient samples:

![](images/variant_impact_comparison_95-100.png)

## Conclusion 

In our analysis, we detected a significantly higher proportion of protein-altering genetic variants in the ancient genomes compared to contemporary ones. While we detected and eliminated the effect of well-known false calls C->T and G-A caused by C->U postmortem changes, we still observed a higher proportion of high-impact genetic variants in ancient samples compared to modern populations. That contradicts the relaxed selection hypothesis, which states the  increasing genetic load of high impact variants in the modern human population.According to this hypothesis, modern humans have a higher genetic load (increased frequency of harmful mutations) compared to ancient populations, where harsher selective pressures would have more efficiently removed such variants. 

The presence of  high-impact variant rates in ancient genomes suggests several possible explanations:

- There are still technical artifacts among the final set of variant calls, resulting from ancient DNA treatment, contamination, degradation, or postmortem changes.
- Ancient populations may have experienced stronger genetic drift due to smaller effective population sizes, allowing some harmful alleles to reach higher frequencies despite selection.
  
To test both explanations, we plan to validate our results on additional ancient datasets and examine the dependency of genetic load on the type of aDNA treatment, sample age, and population.

## References

1. Orlando, L., Allaby, R., Skoglund, P., Der Sarkissian, C., Stockhammer, P. W., Ávila-Arcos, M. C., ... & Warinner, C. (2021). Ancient DNA analysis. Nature reviews methods primers, 1(1), 14.
2. Mallick, S., Micco, A., Mah, M., Ringbauer, H., Lazaridis, I., Olalde, I., ... & Reich, D. (2024). The Allen Ancient DNA Resource (AADR) a curated compendium of ancient human genomes. Scientific Data, 11(1), 182.
3. Margaryan, A., Lawson, D. J., Sikora, M., Racimo, F., Rasmussen, S., Moltke, I., ... & Willerslev, E. (2020). Population genomics of the Viking world. Nature, 585(7825), 390-396.
4. Yates, J. A. F., Lamnidis, T. C., Borry, M., Valtueña, A. A., Fagernäs, Z., Clayton, S., ... & Peltzer, A. (2021). Reproducible, portable, and efficient ancient genome reconstruction with nf-core/eager. PeerJ, 9, e10947.


