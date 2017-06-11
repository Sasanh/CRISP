# CRISP
Cancer-specific RNA-editing Identification using Somatic variation Pipeline (CRISP)


### Aligining raw RNA-Seq samples

We use [HISAT2](https://ccb.jhu.edu/software/hisat2/index.shtml), a fast and sensitive splice-aware aligner, to align the solid tumor cancer samples to Human Genome 19 (hg19) reference genome. (Here we assume that the proper preprocess of the RNA-Seq samples are already preformed.) 

```
hisat2 --dta -x hg19_index -1 rep1.fastq -2 rep2.fastq -S output.sam
```


#### Sorting the aligned reads and fixing the ReadGroup in the SAM header

[Picard](https://broadinstitute.github.io/picard/) is used to insert the missing ReadGroup information in the SAM file and sort the aligned reads based on the genomic coordinate. The output of this phase is sotred as BAM format. 

```
java -jar picard.jar AddOrReplaceReadGroups I=output.sam O=rg_added_sorted.bam SO=coordinate RGID=id RGLB=library RGPL=platform RGPU=machine RGSM=sample
```



```
java -jar picard.jar MarkDuplicates I=rg_added_sorted.bam O=dedupped_temp.bam  CREATE_INDEX=true VALIDATION_STRINGENCY=SILENT M=output.metrics
```

```
java -jar picard.jar ReorderSam I=dedupped_temp.bam O=dedupped.bam  R=hg19.fa CREATE_INDEX=true
```
