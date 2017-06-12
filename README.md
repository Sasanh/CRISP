# CRISP
Cancer-specific RNA-editing Identification using Somatic variation Pipeline (CRISP)
<br />

### Aligining raw RNA-Seq samples

We use [HISAT2](https://ccb.jhu.edu/software/hisat2/index.shtml), a fast and sensitive splice-aware aligner, to align the solid tumor cancer samples to Human Genome 19 (hg19) reference genome. (Here we assume that the proper preprocess of the RNA-Seq samples are already preformed.) 

```
hisat2 --dta -x hg19_index -1 rep1.fastq -2 rep2.fastq -S output.sam
```
<br />

### Preparing the aligned RNA-Seq samples for variant calling
[Picard](https://broadinstitute.github.io/picard/) is used to prepare the aligned sample for variant calling.

#### (1) Sorting the aligned reads and fixing the ReadGroup information in the SAM header

```
java -jar picard.jar AddOrReplaceReadGroups I=output.sam O=rg_added_sorted.bam SO=coordinate RGID=id RGLB=library RGPL=platform RGPU=machine RGSM=sample
```

#### (2) Marking the duplicate reads

```
java -jar picard.jar MarkDuplicates I=rg_added_sorted.bam O=dedupped_temp.bam  CREATE_INDEX=true VALIDATION_STRINGENCY=SILENT M=output.metrics
```

#### (3) Re-Oredering the reads based on the order of chromosomes in the reference genome fasta file
```
java -jar picard.jar ReorderSam I=dedupped_temp.bam O=dedupped.bam  R=hg19.fa CREATE_INDEX=true
```
<br />

### Removing the Splice Region artifacts
[GATK SplitNCigarReads](https://software.broadinstitute.org/gatk/gatkdocs/3.6-0/org_broadinstitute_gatk_tools_walkers_rnaseq_SplitNCigarReads.php) is used to remove the portion of splice site regions that might contribute to false positive variant calls. 

```
java -jar GenomeAnalysisTK.jar -T SplitNCigarReads -R hg19.fa -I dedupped.bam -o split.bam -rf ReassignOneMappingQuality -RMQF 255 -RMQT 60 -U ALL
```
<br />

### Variant calling
We ran [GATK HaplotypeCaller](https://software.broadinstitute.org/gatk/documentation/tooldocs/current/org_broadinstitute_gatk_tools_walkers_haplotypecaller_HaplotypeCaller.php) to identify the potential RNA variants. 
```
java -jar GenomeAnalysisTK.jar -T HaplotypeCaller -R hg19.fa -I split.bam -dontUseSoftClippedBases -stand_call_conf 20.0 -o output_temp.vcf -nct 20 -U ALLOW_SEQ_DICT_INCOMPATIBILITY
```
<br />

### Variant Filtering

* <b><i>Removing Clusters of variants in a short region</i></b> : Clusters of three variants falling into a window of 35 bp are removed. Moreover, QualByDepth (QD) score should be >2.0 and FisherStrand score should be <30 (Suggested by GATK pipeline) . 
```
java -jar GenomeAnalysisTK.jar -T VariantFiltration -R hg19.fa -V output_temp.vcf -window 35 -cluster 3 -filterName FS -filter "FS > 30.0" -filterName QD -filter "QD < 2.0" -o output.vcf -U ALLOW_SEQ_DICT_INCOMPATIBILITY
```
* <b><i>Removing variants with 10 reads support</i></b> : [BCFTOOLS](https://samtools.github.io/bcftools/) is used to only include the single nucleotide variants with at least 10 total reads supporting the base. 
```
bcftools filter -O v -o sample.vcf  --include 'MIN(DP)>9 && TYPE="snp"' output.vcf

```
