# svpack
Tools for filtering, comparing, and annotating structural variant (SV) calls in VCF format.

## Getting support
Bugs and feature requests are welcome. Please use GitHub Issues to submit. Pull requests are also appreciated. We are not able to provide scientific support for interpreting individual variants or cases.

## Related data resources
### Human population control SV callsets
* `resources/HPRC_GIAB.GRCh38.pbsv.vcf.gz` - pbsv calls using HiFi reads for 103 humans from [Genome in a Bottle](http://doi.org/10.1038/sdata.2016.25) and [Human Pangenome Reference Consortium](https://humanpangenome.org/data-use-protocol/)
* [`EEE_SV-Pop_1.ALL.sites.20181204.vcf.gz`](http://ftp.1000genomes.ebi.ac.uk/vol1/ftp/data_collections/hgsv_sv_discovery/working/20181025_EEE_SV-Pop_1/VariantCalls_EEE_SV-Pop_1/) - SV calls using PacBio reads for 15 humans from [Audano 2019](https://doi.org/10.1016/j.cell.2018.12.019)
* [`nstd166.GRCh38.variant_call.vcf.gz`](https://www.ncbi.nlm.nih.gov/dbvar/studies/nstd166/) - gnomAD-SV SV calls using short reads for 10,847 humans from [Collins 2020](https://www.nature.com/articles/s41586-020-2287-8)

### Gene annotations
* `resources/ensembl.GRCh38.101.reformatted.gff3.gz` - [Ensembl gene annotations](http://ftp.ensembl.org/pub/current_gff3/homo_sapiens/) for GRCh38 in GFF3 with chromosome names reformatted to have chr prefix. Unzip to be accepted by `svpack consequence`.

## Examples

### Filter for rare SVs that impact a gene
One approach to prioritize SVs for interpretation is to use a series of `svpack` calls to filter for:
1. confident SV calls (PASS calls)
2. SV calls not seen in population controls (rare variants)
3. SV calls that impact a coding gene

```
# `examples/HG005_chr17_spikein.GRCh38.pbsv.vcf.gz` is a callset for the Genome
# in a Bottle control sample HG005 with a known pathogenic deletion in *PRKAR1A* added at chr17:68,514,334.

zcat examples/HG005_chr17_spikein.GRCh38.pbsv.vcf.gz                  |  # Input                      1,783 SV calls (chr17)
  svpack filter --pass-only --min-svlen 50 -                          |  # PASS & longer than 50 bp     860 SV calls
  svpack match -v - resources/HPRC_GIAB.GRCh38.pbsv.vcf.gz            |  # & rare (not in controls)     144 SV calls
  svpack consequence --require-csq -  \
      <(zcat resources/ensembl.GRCh38.101.reformatted.gff3.gz)        |
  grep  "sv:cds" | cut -f1-3,8                                          # & impacting a coding exon      1 SV call

# Recovers the known variant.
# chr17  68514334  pbsv.DEL.M1  SVTYPE=DEL;END=68516517;SVLEN=-2183;BCSQ=sv:cds|PRKAR1A||protein_coding|+||
```


## Commands

### Filter an SV callset
Filter SVs in `A.vcf`, writing SVs that satisfy all specified filters to `stdout`.

```
-v, --invert          Invert filter, return variants that fail at least one criterion
-p, --pass-only       Only retain variants with FILTER of PASS or .
--require-svtype      Only retain variants with SVTYPE INFO field
-l N, --min-svlen N   Minimum |SVLEN| for variants with SVLEN INFO field
```

#### Examples
```
# Return PASS SVs at least 50 bp long.
# SVs like BND that do not have an SVLEN attribute are considered to satisfy "--min-svlen"
svpack filter --pass-only --min-svlen 50 A.vcf```

# Return PASS SVs shorter than 50 bp.  Chain two svpack filter commands since if
# "--invert", "--pass-only", and "--min-svlen 50" were specified together then svpack
# would return SVs that do not satisfy either "--pass-only" or "--min-svlen".
svpack filter --pass-only A.vcf | svpack filter --invert --min-svlen 50
```

### Match variants between two structural variant callsets
Identify SVs in `A.vcf` that match SVs in `B.vcf`.  Consider two SVs as match when the SVs are:
* similar SVTYPE: allowing INS & DUP to match
* nearby, with start position within --max-pos-diff
* similar length, with difference in length within --max-svlen-diff

```
-v, --invert              Invert match, return variants in A that do not match a
                          variant in B
-i INFO, --info INFO      Output all records and annotate with INFO field(s)
                          from best match in B. Overrides -v.
-p N, --max-pos-diff N    Maximum difference in POS to consider variants to match [100]
-l N, --max-svlen-diff N  Maximum difference in SVLEN to consider variants to match [100]
```

#### Examples
```
# Identify variants in A.vcf that are also seen in B.vcf
svpack match A.vcf B.vcf

# Identify variants in A.vcf that are not seen in B.vcf
svpack match -v A.vcf B.vcf

# Identify variants in KID.vcf that are not seen in MOM.vcf or DAD.vcf
svpack match -v KID.vcf MOM.vcf | svpack match -v - DAD.vcf
```

### Annotate SVs that impact genes
Add a `BCSQ` INFO tag to SVs that impact a gene in `genes.gff`.  A DEL, DUP, INS, INV, or CNV variant
is considered to impact a gene if it overlaps an exon.  A BND variant is considered to impact a gene
if it has a breakend between the gene transcription start and end.  Currently the Ensembl GFF3 gene set
format is supported.

#### Examples
```
# Add the BCSQ tag to A.vcf.  The annotated VCF is written to stdout.
svpack consequence A.vcf ensembl.gff3.gz
```

## DISCLAIMER
THIS WEBSITE AND CONTENT AND ALL SITE-RELATED SERVICES, INCLUDING ANY DATA, ARE PROVIDED "AS IS," WITH ALL FAULTS, WITH NO REPRESENTATIONS OR WARRANTIES OF ANY KIND, EITHER EXPRESS OR IMPLIED, INCLUDING, BUT NOT LIMITED TO, ANY WARRANTIES OF MERCHANTABILITY, SATISFACTORY QUALITY, NON-INFRINGEMENT OR FITNESS FOR A PARTICULAR PURPOSE. YOU ASSUME TOTAL RESPONSIBILITY AND RISK FOR YOUR USE OF THIS SITE, ALL SITE-RELATED SERVICES, AND ANY THIRD PARTY WEBSITES OR APPLICATIONS. NO ORAL OR WRITTEN INFORMATION OR ADVICE SHALL CREATE A WARRANTY OF ANY KIND. ANY REFERENCES TO SPECIFIC PRODUCTS OR SERVICES ON THE WEBSITES DO NOT CONSTITUTE OR IMPLY A RECOMMENDATION OR ENDORSEMENT BY PACIFIC BIOSCIENCES.
