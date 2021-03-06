# CRISP
Cancer-specific RNA-editing Identification using Somatic variation Pipeline (CRISP)
<br />

![](Figure1.png)
<br />

---
### Aligining raw RNA-Seq samples


We use [HISAT2](https://ccb.jhu.edu/software/hisat2/index.shtml), a fast and sensitive splice-aware aligner, to align the solid tumor cancer samples to Human Genome 19 (hg19) reference genome. (Here we assume that the proper preprocess of the RNA-Seq samples are already preformed.) 

```
hisat2 --dta -x hg19_index -1 rep1.fastq -2 rep2.fastq -S output.sam
```
<br />

---
### Preparing the aligned RNA-Seq samples for variant calling
[Picard](https://broadinstitute.github.io/picard/) is used to prepare the aligned sample for variant calling.

1) Sorting the aligned reads and fixing the ReadGroup information in the SAM header

```
java -jar picard.jar AddOrReplaceReadGroups I=output.sam O=rg_added_sorted.bam SO=coordinate RGID=id RGLB=library RGPL=platform RGPU=machine RGSM=sample
```

2) Marking the duplicate reads

```
java -jar picard.jar MarkDuplicates I=rg_added_sorted.bam O=dedupped_temp.bam  CREATE_INDEX=true VALIDATION_STRINGENCY=SILENT M=output.metrics
```

3) Re-Oredering the reads based on the order of chromosomes in the reference genome fasta file
```
java -jar picard.jar ReorderSam I=dedupped_temp.bam O=dedupped.bam  R=hg19.fa CREATE_INDEX=true
```
<br />

---
### Removing the Splice Region artifacts
[GATK SplitNCigarReads](https://software.broadinstitute.org/gatk/gatkdocs/3.6-0/org_broadinstitute_gatk_tools_walkers_rnaseq_SplitNCigarReads.php) is used to remove the portion of splice site regions that might contribute to false positive variant calls. 

```
java -jar GenomeAnalysisTK.jar -T SplitNCigarReads -R hg19.fa -I dedupped.bam -o split.bam -rf ReassignOneMappingQuality -RMQF 255 -RMQT 60 -U ALL
```
<br />

---
### Variant calling
We ran [GATK HaplotypeCaller](https://software.broadinstitute.org/gatk/documentation/tooldocs/current/org_broadinstitute_gatk_tools_walkers_haplotypecaller_HaplotypeCaller.php) to identify the potential RNA variants. 
```
java -jar GenomeAnalysisTK.jar -T HaplotypeCaller -R hg19.fa -I split.bam -dontUseSoftClippedBases -stand_call_conf 20.0 -o output_temp.vcf -nct 20 -U ALLOW_SEQ_DICT_INCOMPATIBILITY
```
<br />

---
### Variant Filtering

* <b><i>Removing Clusters of variants in a short region</i></b> : Clusters of three variants falling into a window of 35 bp are removed. Moreover, QualByDepth (QD) score should be >2.0 and FisherStrand score should be <30 (Suggested by GATK pipeline) . 
```
java -jar GenomeAnalysisTK.jar -T VariantFiltration -R hg19.fa -V output_temp.vcf -window 35 -cluster 3 -filterName FS -filter "FS > 30.0" -filterName QD -filter "QD < 2.0" -o output.vcf -U ALLOW_SEQ_DICT_INCOMPATIBILITY
```
* <b><i>Removing variants with 10 reads support</i></b> : [Bcftools](https://samtools.github.io/bcftools/) is used to only include the single nucleotide variants with at least 10 total reads supporting the base. 
```
bcftools filter -O v -o sample.vcf  --include 'MIN(DP)>9 && TYPE="snp"' output.vcf

```
<br />

---
### Removing variants falling into repeats and low complexity sequences
1) Repeat regions for the desired human reference genome are obtained from [RepeatMasker database](http://www.repeatmasker.org/species/hg.html)
2) Regions labeled "Simple_repeat", "Low_Complexity" and "Satellite" are extracted.
```
awk '$11=="Simple_repeat" || $11=="Low_complexity" || $11=="Satellite"{print $5,$6,$7,$11}' hg19.fa.out > bad_repeats.txt
```
3) Removing variants overlap with repeat regions extracted from the previous step. 
```
java -jar Trim_RepeatMasker.jar Input_Folder_VCF Output_Folder_VCF bad_repeats.txt

usage:
Input_Folder_VCF       Folder containing input VCF files
Output_Folder_VCF      Folder that the trimmed VCF files will be generated.
bad_repeats.txt        Repeat regions to be removed
```
<br />

---
### Keeping the variants located in unique genomic regions
1) Econde Uniqueness track for genomic regions of 35 bp (bigwig file) is obtained from [UCSC annotations download](http://hgdownload.cse.ucsc.edu/downloads.html)
2) The uniqueness track file is converted into bedgraph and unique regions with score of "1" are extracted
```
awk '$4 ==1' wgEncodeDukeMapabilityUniqueness35bp.bedGraph > unique.bed
```
3) Retaining the variants overlapped with unique regions.  
```
java -jar Include_Uniqe_Regions_Only.jar Input_Folder_VCF Output_Folder_VCF unique.bed

usage:
Input_Folder_VCF       Folder containing input VCF files
Output_Folder_VCF      Output Folder that the trimmed VCF files will be generated.
unique.bed.txt         Unique regions in the genome with uniqueness score of "1"
```
<br />

---
### Removing germline and somatic DNA variations (SNPs)  
1) Obtaining germline and somatic mutations (in VCF format) from dbSNP, 1000Genomes, Cosmic, ... (ftp://ftp.ensembl.org/pub/release-89/variation/vcf/homo_sapiens/)
2) Obtaining the cancer specific mutations from The Cancer Genome Atlas (TCGA) (https://portal.gdc.cancer.gov) 
```
java -jar Remove_SNP.jar Variants_Folder Input_Folder_VCF Output_Folder_VCF

usage:
Variants_Folder        Folder containing somatic and germline DNA variants (VCF format)
Input_Folder_VCF       Folder containing input VCF files
Output_Folder_VCF      Output Folder that the trimmed VCF files will be generated.
```
<br />

---

### Performance
Performance of the CRISP pipeline by applying the inclusive filters. 
<br />

![](CRIPS_Performance.png)
<br />

