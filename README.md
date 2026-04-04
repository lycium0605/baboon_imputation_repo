# Baboon Genotype Imputation Test Data

Test dataset for reproducing the genotype imputation pipeline described in:

> **Genotype imputation from low-coverage whole genome sequencing data in wild baboons**

This repository contains a minimal subset of the full dataset — a 5 Mb region of baboon chromosome 20 (NC_044995.1:1-5,000,000) with 7 validation samples — sufficient to run the complete GLIMPSE2 imputation and post-imputation filtering pipeline end-to-end.

## Data Summary

| File | Description | Size |
|------|-------------|------|
| `data/reference/ref_phased_5Mb.bcf` | Phased reference panel (250 baboons, 133K variants) | 3.8 MB |
| `data/reference/ref_sites_5Mb.vcf.gz` | Sites-only VCF for chunking | 732 KB |
| `data/validation/truth_5Mb.vcf.gz` | High-coverage truth genotypes (7 samples, 59K variants) | 6.1 MB |
| `data/bams/*.1x.bam` | 1x downsampled BAMs (7 samples) | 3-4 MB each |
| `data/maps/NC_044995.1_gmap.gz` | Genetic map for chromosome 20 | 2.1 MB |

**Total: ~41 MB**

## Samples

Seven high-coverage Amboseli baboons (BioProject [PRJNA755322](https://www.ncbi.nlm.nih.gov/bioproject/PRJNA755322)), downsampled to 1x for imputation testing:

| Sample | BioSample |
|--------|-----------|
| SAMN20815323 | [SAMN20815323](https://www.ncbi.nlm.nih.gov/biosample/SAMN20815323) |
| SAMN20814949 | [SAMN20814949](https://www.ncbi.nlm.nih.gov/biosample/SAMN20814949) |
| SAMN20815324 | [SAMN20815324](https://www.ncbi.nlm.nih.gov/biosample/SAMN20815324) |
| SAMN20815322 | [SAMN20815322](https://www.ncbi.nlm.nih.gov/biosample/SAMN20815322) |
| SAMN20815325 | [SAMN20815325](https://www.ncbi.nlm.nih.gov/biosample/SAMN20815325) |
| SAMN20815227 | [SAMN20815227](https://www.ncbi.nlm.nih.gov/biosample/SAMN20815227) |
| SAMN04096034 | [SAMN04096034](https://www.ncbi.nlm.nih.gov/biosample/SAMN04096034) |

## Reference Panel

250 non-Amboseli baboons (182 yellow + 68 anubis) from BioProjects [PRJNA433868](https://www.ncbi.nlm.nih.gov/bioproject/PRJNA433868) and [PRJEB49549](https://www.ncbi.nlm.nih.gov/bioproject/PRJEB49549), phased with SHAPEIT5. The BCF contains phased GT fields for biallelic SNPs only.

## Reference Genome

Olive baboon reference assembly [Panubis1.0](https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_008728515.1/) (GCF_008728515.1). NC_044995.1 corresponds to chromosome 20 (~50 Mb total; this subset covers the first 5 Mb).

## Required Software

- [GLIMPSE2](https://github.com/odelaneau/GLIMPSE) (chunk, split_reference, phase, ligate, concordance)
- [bcftools](https://github.com/samtools/bcftools) >= 1.17

## Quick Start

```bash
git clone https://github.com/ShuyuHe/baboon_imputation_repo.git
cd baboon_imputation_repo

# 1. Chunk the region
GLIMPSE2_chunk --input data/reference/ref_sites_5Mb.vcf.gz \
  --region NC_044995.1:1-5000000 --map data/maps/NC_044995.1_gmap.gz \
  --sequential --output chunks.txt

# 2. Split reference into binary format
while IFS="" read -r LINE; do
  IRG=$(echo $LINE | cut -d" " -f3)
  ORG=$(echo $LINE | cut -d" " -f4)
  GLIMPSE2_split_reference --reference data/reference/ref_phased_5Mb.bcf \
    --map data/maps/NC_044995.1_gmap.gz \
    --input-region ${IRG} --output-region ${ORG} \
    --output data/reference/split/ref_bin
done < chunks.txt

# 3. Impute each sample
for BAM in data/bams/*.1x.bam; do
  SAMPLE=$(basename $BAM .1x.bam)
  while IFS="" read -r LINE; do
    IRG=$(echo $LINE | cut -d" " -f3)
    CHR=$(echo $LINE | cut -d" " -f2)
    REGS=$(echo ${IRG} | cut -d":" -f2 | cut -d"-" -f1)
    REGE=$(echo ${IRG} | cut -d":" -f2 | cut -d"-" -f2)
    GLIMPSE2_phase --bam-file ${BAM} \
      --reference data/reference/split/ref_bin_${CHR}_${REGS}_${REGE}.bin \
      --output results/imputed/${SAMPLE}_${CHR}_${REGS}_${REGE}.bcf
  done < chunks.txt
done

# 4. Ligate and validate
# See SKILL.md for the complete pipeline including filtering
```

## License

Data derived from publicly available BioProject accessions. See individual BioProject pages for data use terms.
