## 🎯 rSV Pipeline Overview

### 🏗 Primary Objectives

<p align="center">
<img src="https://github.com/user-attachments/assets/69c43992-af28-4669-a35e-198771986241" width="800">
<img src="https://github.com/user-attachments/assets/2723d4ab-a397-46e8-aa4f-af0729e5a06d" width="800">
</p>
<p align="center"><b>Figure 1:</b> Schematic overview of the main objective</p>

This pipeline aims to refine overlapping structural variants (SVs) into more granular and representative SVs, termed rSVs, using alignment-based window inspection and other heuristics.

---

## [Step 1] VCF Preprocessing and FASTA Generation
<p align="center">
<img src="https://github.com/user-attachments/assets/40ad6400-d640-4518-8775-644a6d93972f" width="500">
</p>
<p align="center"><b>Figure 2:</b> Workflow for processing VCF files and generating FASTA</p>

**Workflow Summary**:
- Include `INV` (inversion) events in `nSV.vcf`.
- Determine SV overlaps:
  - **Non-overlapping** → Store in `single_group`, generate individual VCF using `filter_vcf`.
  - **Overlapping** → Store in `multi_group`, perform `pre-align` via `pos`, then export as `variants_pre_aligned.fasta` for downstream processing.

---

## [Step 2] Multiple Sequence Alignment
<p align="center">
<img src="https://github.com/user-attachments/assets/46d21d26-2a96-42d3-ab11-13f13d8cc13c" width="600">
</p>
<p align="center"><b>Figure 3:</b> Alignment procedure overview</p>

**Alignment Logic**:
1. Alignment is performed **only** for groups that include insertions at the same `POS` (i.e., poly-ins events).
2. Output files include:
   - `input_origin.fasta`: original FASTA for poly-ins group.
   - `input_sliced_X.fasta`: FASTA file for slice index X.
   - `aligned_sliced_X.fasta`: MAFFT alignment results.
   - `aligned.fasta`: final merged alignment output.

<p align="center">
<img src="https://github.com/user-attachments/assets/3fdbbaa0-3274-4e7e-8d20-967420563934" width="300">
</p>
<p align="center"><b>Figure 4:</b> Folder structure for alignment results</p>

---

## [Step 3] rSV Generation

### General Workflow
<p align="center">
<img src="https://github.com/user-attachments/assets/cefcb5a0-8f41-4aa4-9fce-625a9b18bcd9" width="900">
</p>
<p align="center"><b>Figure 5:</b> Overview of the rSV generation process</p>

### Case 1: Standard Cases (No poly-ins)
<p align="center">
<img src="https://github.com/user-attachments/assets/8817adcd-63d3-41e9-91bb-2990bdb6e07d" width="600">
</p>
<p align="center"><b>Figure 6:</b> rSV generation without poly-ins</p>

1. Slide a fixed-size window across aligned sequences.
2. Compare variant alleles (`del`, `ins`) against reference:
   - Match → label `diff = 0`
   - Mismatch → label `diff = 1`
3. Compute the `diff_array` and merge adjacent positions to generate `rSV_meta`.

### Case 2: Complex Cases
<p align="center">
<img src="https://github.com/user-attachments/assets/3ebd220f-9e6e-4ffd-bb7f-9361c0b6c0eb" width="600">
</p>
<p align="center"><b>Figure 7:</b> rSV generation in presence of poly-ins</p>

---

## [Step 4] Conversion to VCF Format and Matrix Construction

### Overall Workflow
<p align="center">
<img src="https://github.com/user-attachments/assets/c1bf08e0-8566-4f56-ac8e-5d087b3e79b0" width="1200">
</p>
<p align="center"><b>Figure 8:</b> Full pipeline of Step 4</p>

### 1. Generation of `rSV_meta.csv`
- Extract `pos`, `ref`, and `alt` fields from `merged_rSVs.csv` and perform normalization.

<p align="center">
<img src="https://github.com/user-attachments/assets/45d7e0f1-80df-46a1-80a1-32260e13ee6f" width="1200">
</p>
<p align="center"><b>Figure 9:</b> Example content of rSV_meta.csv</p>

### 2. Matrix Construction

#### **D Matrix: Mapping between SVs and rSVs**
<p align="center">
<img src="https://github.com/user-attachments/assets/911a5ca9-9f2a-4ef0-9b7e-941940a95bbe" width="600">
</p>
<p align="center"><b>Figure 10:</b> Example of D-matrix generation</p>

- Extract `diff_array` from `merged_results.csv`.
- Each SV is expressed as a linear combination of rSVs.

#### **X Matrix: Mapping between SVs and Sample Genotypes**
<p align="center">
<img src="https://github.com/user-attachments/assets/5d460532-493b-43f9-9194-68501cbf12a1" width="800">
</p>
<p align="center"><b>Figure 11:</b> Example of X-matrix generation</p>

- Extract genotype (`GT`) data from the VCF corresponding to each SV.
- Missing values (`./.`) are encoded as `-999` (customizable).

#### **T Matrix: Mapping between rSVs and Sample Genotypes**
<p align="center">
<img src="https://github.com/user-attachments/assets/8aedc9d7-fd57-4356-85dd-7c7c4a328f6e" width="900">
</p>
<p align="center"><b>Figure 12:</b> Example of T-matrix generation</p>

- Computed as T = D<sup>T</sup>X
- The resulting `GT-matrix` conforms to VCF v4.2 genotype format.

---

## Generating `rSV.vcf`
<p align="center">
<img src="https://github.com/user-attachments/assets/5f8517b2-1d0e-4239-91cd-cdc3de10e57d" width="900">
</p>
<p align="center"><b>Figure 13:</b> Example of final rSV.vcf output</p>

- The final `rSV.vcf` is generated by combining `rSV_meta` with the `GT-matrix`.
