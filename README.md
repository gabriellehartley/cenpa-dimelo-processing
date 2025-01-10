      !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
      !!   CENPA Dimelo-seq Processing   !!
      !!            Workflow             !!
      !!        Gabrielle Hartley        !!
      !!   gabrielle.hartley@uconn.edu   !!
      !!          R O'Neill Lab          !!
      !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

Process raw ONT reads to generate filtered CENPA Dimelo-seq tracks. Final track takes **CENPA (middle track)** and **Negative (top track)**, and generates a **filtered CENPA track (bottom track)** that reduces background noise associated with spurious 6mA basecalls. 

![Example Tracks](link)

## 1. Basecall reads with Dorado. 
If barcoded, include --kit-name. If not, this flag can be omitted and the dorado demux step can be skipped.

```
module load Dorado/0.8.2

dorado basecaller --min-qscore 10 --kit-name SQK-NBD114-24 sup,6mA /path/to/pod5/ > CENPA.Dimelo.bam
dorado basecaller --min-qscore 10 --kit-name SQK-NBD114-24 sup,6mA /path/to/pod5/ > Negative.Dimelo.bam

dorado demux --output-dir negatives --no-classify Negative.Dimelo.bam
```

## 2. Align basecalled reads to genome of interest.

```
module load Dorado/0.8.2

dorado aligner assembly.fasta CENPA.Dimelo.bam > CENPA.Dimelo.aligned.bam
dorado aligner assembly.fasta Negative.Dimelo.bam > Negative.Dimelo.aligned.bam 
```

## 3. Process Alignment Files for Modbam2bed 

```
module load samtools/1.19.2

samtools sort CENPA.Dimelo.aligned.bam > CENPA.Dimelo.aligned.sorted.bam
samtools index CENPA.Dimelo.aligned.sorted.bam

samtools sort Negative.Dimelo.aligned.bam > Negative.Dimelo.aligned.sorted.bam
samtools index Negative.Dimelo.aligned.sorted.bam
```

## 4. Convert modified bam to methylated bed file using modbam2bed.
Use --mod_threshold=0.8 per suggestion from Nick Altemose.

```
modbam2bed \
     -e -m 6mA -t 10 \
     assembly.fasta \
     --mod_threshold=0.8 \
     CENPA.Dimelo.aligned.sorted.bam > CENPA.Dimelo.aligned.sorted.0.8.bed

modbam2bed \
     -e -m 6mA -t 10 \
     assembly.fasta \
     --mod_threshold=0.8 \
     Negative.Dimelo.aligned.sorted.bam > Negative.Dimelo.aligned.sorted.0.8.bed
```

## 5. Process methylated bed to viewable bedgraph.

```
awk 'BEGIN {OFS="\t"}; {print $1,$2,$3,$11}' CENPA.Dimelo.aligned.sorted.0.8.bed > CENPA.Dimelo.aligned.sorted.0.8.bedgraph
grep -v "nan" CENPA.Dimelo.aligned.sorted.0.8.bedgraph > CENPA.Dimelo.aligned.sorted.0.8.filtered.bedgraph

awk 'BEGIN {OFS="\t"}; {print $1,$2,$3,$11}' Negative.Dimelo.aligned.sorted.0.8.bed > Negative.Dimelo.aligned.sorted.0.8.bedgraph
grep -v "nan" Negative.Dimelo.aligned.sorted.0.8.bedgraph > Negative.Dimelo.aligned.sorted.0.8.filtered.bedgraph
```

## 6. Sort bedgraphs.

```
LC_COLLATE=C
sort -k1,1 -k2,2n CENPA.Dimelo.aligned.sorted.0.8.bedgraph > CENPA.Dimelo.aligned.sorted.0.8.sorted.bedgraph
sort -k1,1 -k2,2n Negative.Dimelo.aligned.sorted.0.8.filtered.bedgraph > Negative.Dimelo.aligned.sorted.0.8.filtered.bedgraph
```

## 7. Combine bedgraphs using bedtools.

```
module load bedtools/2.31.1
unionBedGraphs -g genome.fasta -i CENPA.Dimelo.aligned.sorted.0.8.sorted.bedgraph Negative.Dimelo.aligned.sorted.0.8.filtered.bedgraph > CENPA_Negative_Union.bedgraph
```

## 8. Subtract negative basecalls from CENPA basecalls.

```
awk 'BEGIN {OFS="\t"}; {print $1,$2,$3,$5-$4}' CENPA_Negative_Union.bedgraph > CENPA-Negative_Subtract.bedgraph
```

## 9. Sort bedgraph.
Sometimes sorting again is not necessary, other times error is thrown.

```
LC_COLLATE=C
sort -k1,1 -k2,2n CENPA-Negative_Subtract.bedgraph > CENPA-Negative_Subtract.sorted.bedgraph
```

## 10. Generate bigwig for UCSC Genome Browser. 

```
module load GenomeBrowser
bedGraphToBigWig CENPA-Negative_Subtract.sorted.bedgraph assembly.fasta CENPA-Negative_Subtract.sorted.bigwig
```
