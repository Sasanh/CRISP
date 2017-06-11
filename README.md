# CRISP
Cancer-specific RNA-editing Identification using Somatic variation Pipeline (CRISP)


### Aligining raw RNA-Seq samples

We use [HISAT2](https://ccb.jhu.edu/software/hisat2/index.shtml), a fast and sensitive splice-aware aligner, to align the solid tumor cancer samples to Human Genome 19 (hg19) reference genome. (Here we assume that the proper preprocess of the RNA-Seq samples are already preformed.) 

```
hisat2 --dta -x hg19_index -1 rep1.fastq -2 rep2.fastq -S output.sam
```

### Preparing the aligned RNA-Seq sample for variant calling
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
