## 🎯 rSV代码思路

### 🏗 代码主要达成效果
<p align="center">
<img src="https://github.com/user-attachments/assets/69c43992-af28-4669-a35e-198771986241" width="800">
</p>
<p align="center"><b>Figure 1:</b> 代码主要达成效果示意图</p>

即，通过对齐与窗口判断等手段，将重叠的SV划分为更加细致的SV（rSV）。

---
## [Step 1] Processing VCF and Generating FASTA
<p align="center">
<img src="https://github.com/user-attachments/assets/40ad6400-d640-4518-8775-644a6d93972f" width="500">
</p>
<p align="center"><b>Figure 2:</b> Processing VCF and Generating FASTA 流程图</p>

**主要流程**：
- 将`INV`（倒位）变异加入 `nSV.vcf` 。
- 判断`SV`是否重叠：
  - **未重叠** → 存入 `single_group` 并用 `filter_vcf` 生成 VCF 文件。
  - **重叠** → 存入 `multi_group`，用 `pos` 进行 `pre-align`，然后存为 `variants_pre_aligned.fasta` 供下一步使用。

---

## [Step 2] Running Alignments
<p align="center">
<img src="https://github.com/user-attachments/assets/46d21d26-2a96-42d3-ab11-13f13d8cc13c" width="600">
</p>
<p align="center"><b>Figure 3:</b> Running Alignments 流程图</p>

**对齐逻辑**：
1. 仅当 **group 中包含相同 `POS` 的插入突变 (`poly-ins`)** 时，才会进行切片 (`sliced`) 和 MAFFT 对齐。
2. 结果文件说明：
   - `input_origin.fasta`：存在 `poly-ins` 的变异组。
   - `input_sliced_X.fasta`：第 `X` 个切片的输入文件。
   - `aligned_sliced_X.fasta`：对应 `MAFFT` 的对齐结果。
   - `aligned.fasta`：最终对齐结果。

<p align="center">
<img src="https://github.com/user-attachments/assets/3fdbbaa0-3274-4e7e-8d20-967420563934" width="300">
</p>
<p align="center"><b>Figure 4:</b> alignment_results文件夹示意图</p>

---

## [Step 3] Generating rSVs

### 总流程
<p align="center">
<img src="https://github.com/user-attachments/assets/544c06a0-1415-4518-92d3-73485fcbbe06" width="900">
</p>
<p align="center"><b>Figure 5:</b> Generating rSVs 总流程示意图</p>

### 情况 1：普通情况（无 poly-ins）
<p align="center">
<img src="https://github.com/user-attachments/assets/8817adcd-63d3-41e9-91bb-2990bdb6e07d" width="600">
</p>
<p align="center"><b>Figure 6:</b> 无 poly-ins 情况下的 rSV 生成流程</p>

1. 扫描对齐后的窗口。
2. 比对变异（`del`、`ins`）与参考序列：
   - 相同 → `diff` 标记 `0`
   - 不同 → `diff` 标记 `1`
3. 计算 `diff_array` 并合并相邻列，最终生成 `rSV_meta`。

### 情况 2：特殊情况（poly-ins 存在 & 变异重叠度未达阈值）
<p align="center">
<img src="https://github.com/user-attachments/assets/faa5ff4a-2b47-4a7c-93bd-e7991dd318c2" width="600">
</p>
<p align="center"><b>Figure 7:</b> poly-ins 存在时的 rSV 生成流程</p>

- 若 `poly-alt` 出现且重叠度不足阈值，则存入 `nrSV_meta.csv`，最终加入 `nSV.vcf`。

---

## [Step 4] Converting to VCF Format and Generating Matrices

### 总流程
<p align="center">
<img src="https://github.com/user-attachments/assets/c1bf08e0-8566-4f56-ac8e-5d087b3e79b0" width="1200">
</p>
<p align="center"><b>Figure 8:</b> Step 4 总流程示意图</p>

### 1. 生成 `rSV_meta.csv`
- 从 `merged_rSVs.csv` 提取 `pos`、`ref`、`alt` 信息并标准化。

<p align="center">
<img src="https://github.com/user-attachments/assets/45d7e0f1-80df-46a1-80a1-32260e13ee6f" width="1200">
</p>
<p align="center"><b>Figure 9:</b> rSV_meta.csv 示例</p>

### 2. 生成矩阵

#### **D 矩阵（SV 与 rSV 的对应关系矩阵）**
<p align="center">
<img src="https://github.com/user-attachments/assets/911a5ca9-9f2a-4ef0-9b7e-941940a95bbe" width="600">
</p>
<p align="center"><b>Figure 10:</b> D-matrix 生成示例</p>

- 从 `merged_results.csv` 提取 `diff_array` 。
- 每个 SV 可视作 rSV 的线性组合。

#### **X 矩阵（SV 与样本基因型的对应关系矩阵）**
<p align="center">
<img src="https://github.com/user-attachments/assets/5d460532-493b-43f9-9194-68501cbf12a1" width="800">
</p>
<p align="center"><b>Figure 11:</b> X-matrix 生成示例</p>

- 从 `D-matrix` 对应的 VCF 文件提取样本基因型 (`GT`) 并编码。
- `./.` 编码为 `-9`（示例），实际代码中为 `-999`。

#### **T 矩阵（rSV 与样本基因型的对应关系矩阵）**
<p align="center">
<img src="https://github.com/user-attachments/assets/8aedc9d7-fd57-4356-85dd-7c7c4a328f6e" width="900">
</p>
<p align="center"><b>Figure 12:</b> T-matrix 生成示例</p>

- `T-matrix = D-matrix × X-matrix`
- `GT-matrix` 为 VCF (v4.2) 格式的 GT 矩阵。

---

## 生成 `rSV.vcf`
<p align="center">
<img src="https://github.com/user-attachments/assets/5f8517b2-1d0e-4239-91cd-cdc3de10e57d" width="900">
</p>
<p align="center"><b>Figure 13:</b> rSV.vcf 生成示例</p>

- `rSV_meta + GT-matrix` 直接生成 `rSV.vcf` 文件。
