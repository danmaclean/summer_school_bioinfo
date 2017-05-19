# Finding SNPs With An Alignment

## Learning Objectives

  * Understanding Pileups and VCFs
  * Calling reliable SNPs
  * Annotating SNPs with SNPEff

Many different programs can be used to call SNPs, including SAMTools [@Li:2009ka], Picard, GATK [@DeSumma:2017kr] and VarScan [@Koboldt:2009dk]. Some of these programs use the BAM alignment file directly, some use an MPileup file. MPileups and VCF files are both used to represent SNPs and associated scores.  

## MPileup

The MPileup format is a per-line base-by-base representation of the alignment. It provides a much more coherent way of viewing the alignment than the SAM file. 

Here's a sample: 

```

seq1 272 T 24  ,.$.....,,.,.,...,,,.,..^+. <<<+;<<<<<<<<<<<=<;<;7<&
seq1 273 T 23  ,.....,,.,.,...,,,.,..A <<<;<<<<<<<<<3<=<<<;<<+
seq1 274 T 23  ,.$....,,.,.,...,,,.,...    7<7;<;<<<<<<<<<=<;<;<<6
seq1 275 A 23  ,$....,,.,.,...,,,.,...^l.  <+;9*<<<<<<<<<=<<:;<<<<
seq1 276 G 22  ...T,,.,.,...,,,.,....  33;+<<7=7<<7<&<<1;<<6<
seq1 277 T 22  ....,,.,.,.C.,,,.,..G.  +7<;<<<<<<<&<=<<:;<<&<
seq1 278 G 23  ....,,.,.,...,,,.,....^k.   %38*<<;<7<<7<=<<<;<<<<<
seq1 279 C 23  A..T,,.,.,...,,,.,..... ;75&<<<<<<<<<=<<<9<<:<<

```

Each line represents a single nucleotide in the reference. The first three columns represent the reference name, position on the reference and the reference nucleotide. 

The next three columns are about the bases piled-up over that position, so are total read depth, read bases, and base qualities. 

At the read base column,

 * a dot `.` = a match to the reference base on the forward
 * a comma `,` = match on the reverse strand
 * any of `ACGTN` = a mismatch on the forward strand 
 * any of `acgtn` = mismatch on the reverse strand
 
### Different Flavours of PileUp
 
Pileup format has been extended at various times, so the exact format you get can vary a little, the description above is the core of them all and the extensions usually provide some further quality information. Recent versions of SAMtools add quite a few columns to this description. 

You can see more here [http://samtools.sourceforge.net/pileup.shtml](http://samtools.sourceforge.net/pileup.shtml)

`SAMtools` is usually used to generate an MPileup, the `mpileup` command can do this. More recent versions of `SAMtools mpileup` and most other SNP callers can also generate another type of SNP describing file, a VCF [Variant Call Format](https://samtools.github.io/hts-specs/VCFv4.2.pdf) file. 


The related VCF file takes a slightly different approach and looks like this:

```
##FORMAT=<ID=DP,Number=1,Type=Integer,Description="Read Depth">
##FORMAT=<ID=HQ,Number=2,Type=Integer,Description="Haplotype Quality">
#CHROM POS ID REF ALT QUAL FILTER INFO FORMAT NA00001 NA00002 NA00003
20 14370 rs6054257 G A 29 PASS NS=3;DP=14;AF=0.5;DB;H2 GT:GQ:DP:HQ 0|0:48:1:51,51 1|0:48:8:51,51 1/1:43:5:.,.
20 17330 . T A 3 q10 NS=3;DP=11;AF=0.017 GT:GQ:DP:HQ 0|0:49:3:58,50 0|1:3:5:65,3 0/0:41:3
20 1110696 rs6040355 A G,T 67 PASS NS=2;DP=10;AF=0.333,0.667;AA=T;DB GT:GQ:DP:HQ 1|2:21:6:23,27 2|1:2:0:18,2 2/2:35:4
20 1230237 . T . 47 PASS NS=3;DP=13;AA=T GT:GQ:DP:HQ 0|0:54:7:56,60 0|0:48:4:51,51 0/0:61:2
20 1234567 microsat1 GTC G,GTCT 50 PASS NS=3;DP=9;AA=G GT:GQ:DP 0/1:35:4 0/2:17:2 1/1:40:3
```

The first thing you notice is that it describes only variant positions **NOT** **EVERY** position like Pileup.

The lines starting with `##` are the meta-data description lines. They define the labels for SNP data and contain key-value pairs separated by an '=' sign (e.g. number=1, Type=Integer, Description="Some description"). 

The header line starts with a single `#`, and has the column headings, these being `ALT` and `QUAL`.  

  1. `CHROM` = chromosome
  2. `POS` = position 
  3. `ID` = an ID (if given) 
  4. `REF` = reference sequence nucleotide
  5. `ALT` = SNP nucleotide
  6. `FILTER` = a filter defined in the metadata
  7. `INFO` = Summary information about this SNP
  8. `FORMAT` = Format of the sample column(s)
  9. Sample information for one sample formatted according to `FORMAT`
  10. More sample info (if needed), also according to `FORMAT`

`INFO` and `FORMAT` are where the VCF format really makes use of its meta-data. In the INFO column there will be a combination of key=value pairs, separated by
semi-colons. An entry will read something like:

```
	ADP=27;WT=0;HET=2;HOM=0;NC=0
```

Each key (e.g. `ADP`) will have its meaning explained in the metadata `INFO` fields. 

The `FORMAT` column reads slightly different from the `INFO` field. 

```
	GT:GQ:SDP:DP:RD:AD:FREQ:PVAL:RBQ:ABQ:RDF:RDR:ADF:ADR. 
```

The meaning of these keys can be found in the meta-data in the
`FORMAT` lines. Some common ones that are useful are:

 * `DP` = coverage depth
 * `GT` =  genotype, e.g `0/0`, `1/1` and `0/1` 

Genotype values have the following meanings: 

 * `0/0` - Homozygous to the reference (`REF`) 
 * `1/1` - Homozygous to the alternate non-reference allele (`ALT`) 
 * `0/1` - Heterozygous (`0/2` represents a heterozygote with two alternate alleles)

Deciding on the right parameters for SNP calling is very case specific. Often SNP calling pipelines will need to be optimised to ensure that the level of false positive calls is acceptable. Here are some parameters in SNP callers to look out for

 * Minimum coverage - the number of read bases that contribute to the SNP call, more is better
 * Minimum variant allele - the fraction of bases in the pileup that are needed for a SNP call (1/100 = bad, 50/100 = believable)
 * P-value threshold - or some other probability based measure of SNP accuracy.
 * Strand filter - discard SNPs where most reads come from one strand - helpful with certain sequencing errors
 * Base quality - discard bases in the pileup that don't have good enough individual Phred scores

Whichever tool you use, make sure that you get a good idea of what it's options for making high-quality SNPs are.

## Exercises 

### MPileup 

You are provided with a new BAM file for _Arabidopsis_ chromosome 4 in the shared data library `SNP Calling`, use it to run `HTS SAMtools .. SAMtools mpileup`. Remember you'll need to load a reference genome, the `TAIR_10_chr4.fasta` file should be imported to your history for this purpose.  

Please complete the section quiz at [https://goo.gl/forms/YvCzG7JfYxj7zntz2](https://goo.gl/forms/YvCzG7JfYxj7zntz2)

  1. How do you make `SAMtools mpileup` output VCF or Mpileup?
  2. What are differences in information between the two? 
  3. Generate an MPileup file and select appropriate `advanced options` to make sure you have a good enough mapping quality (~20) and base quality (~30) for reliable SNP calls.
  4. What is the point of adding a maximum read depth?
  5. How does this run compare in execution time with earlier ones in this course? Why is there a difference?

These are the largest datasets we use in this workshop The reference is a whole 20 Mb _Arabidopsis_ chromosome, with full 30 deep covereage. The laptop VM takes a while to chug through it.
If you are getting bored, you can restrict the amount of the reference that SAMtools will churn through. Try looking in `Advanced Options` for `Select regions to call`. You can paste or type in a region in the format `Chr4:1-100` - replacing the 1 and 100 with suitable start and stop coordinates. Try doing just 2000000.

### VarScan

From the mpileup file you created in the challenge above, use `VarScan Mpileup` in `Finding Variants` to filter the positions to find the SNPs and make the criteria a bit more stringent. 

  1. Inspect the pileup (or run some SAMtools stats) to determine suitable values for the depth and quality parameters.
  2. Our purpose is to clearly separate Homozygous and Heterozygous SNPs, which filters do you think we should use? What frequency values should we set to get many, good homozygous calls (hint: 100% frequency rules out some SNPs where a single miscalled read or base messes things up) 
  3. How long does this take? What factors do you think affect the run time?
