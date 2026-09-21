# HARs-PD-GP2-R12

## Summary

This is the online repository for the manuscript titled "Human Specific Regulatory Evolution as a Source of Neurodegenerative Disease Risk". This repository holds the analysis pipeline using **GP2 (Global Parkinson's Genetics Program) Release 12** data.

### Region panel

The regions tested here are not a single published list, but a compiled panel drawn from five studies that each define "accelerated" non-coding regions under a different evolutionary lens. Each source was downloaded as supplementary material from its original paper, lifted from its native genome build to GRCh38 with the UCSC Genome Browser's LiftOver tool.

1. **HARs** (Girskis et al., 2021) — 3,171 regions implicated in the rewiring of neurodevelopmental gene regulation during human corticogenesis.
2. **zooHARs** (Keough et al., 2023) — 312 HARs refined through multi-species alignment by the Zoonomia Consortium, enriched for 3D chromatin interactions near neurodevelopmental loci.
3. **linHARs** (Bi et al., 2023) — 1,674 lineage-specific accelerated elements identified through comparative primate genomics.
4. **HAQERs** (Mangan et al., 2022) — 1,581 regions that diverge quickly from the Human-Chimpanzee Ancestor without requiring prior vertebrate conservation; largely CpG-rich, de novo neurodevelopmental enhancers.
5. **Consensus HARs** (Capra et al., 2013) — 2,741 classical HARs found by scanning for vertebrate-conserved loci with an accelerated substitution rate in humans.

### Pipeline

The pipeline tests this region set for association with PD case-control status and rare-variant burden, across the NBA (imputed genotyping array) and WGS (DRAGEN v4.4.7 whole-genome sequencing) datasets, and across **11 ancestries** (AAC, AFR, AJ, AMR, CAS, EAS, EUR, FIN, MDE, SAS, CAH). Regions with converging signal across analyses are further annotated for regulatory and functional impact.

Not every analysis was run identically across every dataset and ancestry, and it's worth being explicit about where the scope narrows:

| Analysis | Dataset(s) | Ancestries | Notes |
|---|---|---|---|
| Burden Test (SKAT / SKAT-O) | NBA + WGS | 8 of 11 | FIN, MDE, SAS excluded for insufficient N (`LOW_POWER`). Passing both Bonferroni (per-ancestry, p < 8.45e-6) and FDR-BH < 0.05, significant hits were found in AJ (NBA), AMR (NBA and WGS), and EUR (WGS only); AAC, AFR, CAH, CAS and EAS returned no significant hits in either dataset. |
| Case-Control | WGS only | 8 of 11 | Same three ancestries excluded as `LOW_POWER`; NBA was not attempted for this step. |
| VEP / CADD / Enformer | — | EUR only | Downstream functional annotation of the Tier 1 variants with convergent burden + case-control signal — |

NBA's comparatively thin yield relative to WGS is expected rather than a processing artifact: the regions tested are non-coding, and an imputed genotyping array captures far fewer of the rare variants that fall inside them than whole-genome sequencing does.


## Data Statement

This repository contains **analysis code only**. It does **not** contain individual-level genotype data, restricted result tables, or any patient-identifying information. GP2 data is available to approved researchers through the official GP2 data access process — see [https://gp2.org](https://gp2.org) and the [GP2 Data Access Policy](https://gp2.org/data-access/) for details on how to apply. Any sample identifiers appearing in these notebooks (e.g., in illustrative print output) are de-identified GP2 study IDs, not real-world patient identifiers.

## Repository Orientation

```
HARs-PD-GP2-R12/
├── README.md
├── LICENSE
└── analyses/
    └── GP2_R12/
        ├── 1_region_merge_clustering_GP2_R12.ipynb
        ├── 2_covariate_builder_GP2_R12.ipynb
        ├── 3_region_extractor_GP2_R12.ipynb
        ├── 4_burden_test_GP2_R12.ipynb
        ├── 5_case_control_GP2_R12.ipynb
        ├── 6a_annotation_vep_cadd_GP2_R12.ipynb
        └── 6b_annotation_enformer_GP2_R12.ipynb
```

## Analysis Notebooks

Notebooks are numbered in the order they are meant to be run. Steps 6a and 6b both build on the same Tier 1 variant set and can be run independently of each other.

| # | Notebook | Description |
|---|----------|-------------|
| 1 | `1_region_merge_clustering_GP2_R12.ipynb` | Takes the concatenated, GRCh38-lifted list from the five source studies and collapses overlapping intervals into non-redundant, unioned regions (one composite region per cluster of overlapping HARs), writing the result to a shared `HARS_files/HARs_merged` location that every downstream notebook reads from. |
| 2 | `2_covariate_builder_GP2_R12.ipynb` | Builds a master `samplestokeep` list restricted to confirmed PD/Control individuals, then produces per-ancestry `samplestokeep` and covariate files (SEX, AGE, up to 10 PCs) cross-checked against it — the fix for a data-lineage bug in the original GP2 template, where `samplestokeep` was never filtered by phenotype and silently included Unknown/Other individuals in downstream MAF and genotype calculations. |
| 3 | `3_region_extractor_GP2_R12.ipynb` | Extracts a per-region VCF for each HAR × ancestry × dataset combination via `plink2`, reading the union region list from step 1's shared location (and failing explicitly if it isn't there yet) and restricting samples to the Covariate Builder's `samplestokeep` files. |
| 4 | `4_burden_test_GP2_R12.ipynb` | Runs RVTests (v2.1.0) SKAT/SKAT-O rare-variant burden tests per HAR per ancestry, on both NBA and WGS, at multiple MAF thresholds (1%, 3%). Includes Bonferroni (α = 0.05 / 5,915 ≈ 8.45e-6) and FDR-BH correction, plus λ<sub>GC</sub> diagnostics. See the analytical scope table above for per-ancestry, per-dataset results. |
| 5 | `5_case_control_GP2_R12.ipynb` | Runs `plink2 --glm firth-fallback` case-control association per HAR per ancestry, on the WGS dataset. Includes FDR-BH correction and genomic inflation factor (λ<sub>GC</sub>) diagnostics. See the analytical scope table above for which ancestries this covers. |
| 6a | `6a_annotation_vep_cadd_GP2_R12.ipynb` | Functionally annotates the EUR Tier 1 variants (convergent burden + case-control signal) using the Ensembl VEP REST API (regulatory features, transcription-factor motif effects) and CADD v1.7 deleteriousness scores. |
| 6b | `6b_annotation_enformer_GP2_R12.ipynb` | Predicts the regulatory effect (REF vs. ALT) of the same Tier 1 variants using Enformer, scoring ~5,300 regulatory tracks (DNase, H3K27ac, H3K4me1, RNA-seq) at 128 bp resolution to flag which variants strengthen or weaken nearby regulatory activity. Written for a GPU runtime (e.g., Colab with a T4). |

## Ancestry Labels

| Code | Ancestry |
|------|----------|
| AAC | African Admixed and Caribbean |
| AFR | African |
| AJ | Ashkenazi Jewish |
| AMR | Latino/Admixed American |
| CAS | Central Asian |
| EAS | East Asian |
| EUR | European |
| FIN | Finnish |
| MDE | Middle Eastern |
| SAS | South Asian |
| CAH | Complex Admixture History |

Ancestry definitions follow the [GP2 harmonized ancestry labels](https://gp2.org).

## Software

| Tool | Version | Use |
|------|---------|-----|
| plink2 | latest | Region extraction, case-control GLM association |
| RVTests | v2.1.0 | SKAT / SKAT-O rare-variant burden tests |
| Ensembl VEP (REST API) | GRCh38 | Functional/regulatory variant annotation |
| CADD | v1.7 (GRCh38) | Variant deleteriousness scoring |
| Enformer | DeepMind, 2021 | Sequence-based prediction of regulatory-track effects (REF vs. ALT) |
| Python | 3.12 | Pipeline orchestration, statistics (pandas, numpy, statsmodels) |

## Notes on Reproducibility

Notebook outputs have been cleared before publishing (standard practice for sharing analysis code on GitHub); the code itself, including all data-processing logic, statistical models, and the documented bug fix in the Covariate Builder, is unchanged from what was run against GP2 Release 12. Running these notebooks requires an approved GP2 Release 12 data access agreement and access to the GP2 cloud workspace paths referenced in the code.

## Citation / Contact

This work is part of an ongoing study of non-coding regulatory regions in Parkinson's Disease using GP2 data. For questions about this repository, please open an issue.
