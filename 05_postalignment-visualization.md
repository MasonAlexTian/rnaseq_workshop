# Alignment Visualization
The Integrative Genomics Viewer (IGV), developed by the Broad Institute, is a high-performance visualization tool for interactive exploration of large, integrated genomic datasets. It supports a wide variety of data types, including array-based and next-generation sequence data, and genomic annotations.

## Indexing BAM (.bai)
Before we can view our alignments in the IGV browser we need to index our BAM files. We will use `samtools` index for this purpose. For convenience later, index all bam files. Run the following command lines (or run `create_bai.sh`).

```bash
echo $RNA_ALIGN_DIR
cd $RNA_ALIGN_DIR
find *.bam -exec echo samtools index {} \; | sh
```

 An index file (.bai) will be generated for each BAM file.

## Visualize alignments
Start IGV on your laptop.

```bash
igv.sh &
```

Load the `UHR.bam` and `HBR.bam` files in IGV.
Make sure to set Human (hg38) as the reference genome sequence.

You can load the necessary files (`UHR.bam` and `HBR.bam`) in IGV directly using `File > Load from URL`. You may wish to customize the track names as you load them in to keep them straight. Do this by right-clicking on the alignment track and choosing `Rename Track`.

#### Q5.1 Go to an example gene locus on chr22: for example, *EIF3L*, *NDUFA6*, and *RBX1* have nice coverage

#### Q5.2 Differentially expressed genes,
- For example, *SULT4A1* and *GTSE1* are differentially expressed. Are they up-regulated or down-regulated in the brain (HBR) compared to cancer cell lines (UHR)?

- Mouse over some reads and use the read group (RG) flag to determine which replicate the reads come from. What other details can you learn about each read and its alignment to the reference genome.

### Exercise

Try to find a variant position in the RNAseq data:
- HINT: *DDX17* is a highly expressed gene with several variants in its 3 prime UTR.
- Other highly expressed genes you might explore are: *NUP50*, *CYB5R3*, and *EIF3L* (all have at least one transcribed variant).
- Are these variants previously known (e.g., present in dbSNP)?
- `File > Load from Server > Annotation > [v] Common SNPs 1.4.2`
- How should we interpret the allele frequency of each variant?
 - Remember that we have rather unusual samples here in that they are actually pooled RNAs corresponding to multiple individuals (genotypes).

## Note
- Save the session
- Take snapshots
- [tutorial](https://github.com/griffithlab/rnaseq_tutorial/wiki/IGV-Tutorial) for a comprehensive IGV usage
- http://software.broadinstitute.org/software/igv/book/export/html

### Up next
[Expression](06_expression.md)
