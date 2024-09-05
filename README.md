
# Benchmarking Pathology Foundation Models: Adaptation Strategies and Scenarios

## Overview

This project provides an implementation of the paper:

> **Benchmarking Pathology Foundation Models: Adaptation Strategies and Scenarios**  
> Jaeung Lee, Jeewoo Lim, Keunho Byeon, and Jin Tae Kwak

### Abstract
Pathology-specific foundation models have achieved notable success in tasks like classification, segmentation, and registration. However, adapting these models to clinical tasks with limited data and domain shifts presents a challenge. In this study, we benchmark pathology foundation models across 14 datasets in two scenarios: **Consistency Assessment** and **Flexibility Assessment**. The former assesses performance consistency across different datasets for the same task, while the latter evaluates performance when models are applied to tasks across different domains with limited data.

Models
The following foundation models are used:

- CTransPath: [Repository](https://github.com/Xiyue-Wang/TransPath)
- Lunit: [Repository](https://github.com/lunit-io/benchmark-ssl-pathology)
- Phikon: [Repository](https://huggingface.co/owkin/phikon)
- UNI: [Repository](https://huggingface.co/MahmoodLab/UNI)

The pretrained weights for these foundation models should be stored in the `model_lib/pretrained` folder.
## Datasets for Benchmark
We benchmark models using the following datasets categorized by tissue type:

### Colon Datasets
- **Kather19**: [Link](https://zenodo.org/records/1214456)
- **Kather16**: [Link](https://zenodo.org/records/1214456)
- **CRC-TP**: [Link](https://warwick.ac.uk/fac/cross_fac/tia/data/crc-tp)
- **KBSMC Colon**: [Link](https://github.com/QuIIL/KBSMC_colon_cancer_grading_dataset)
- **Chaoyang**: [Link](https://bupt-ai-cz.github.io/HSA-NRL/)
- **Digest**: [Link](https://digestpath2019.grand-challenge.org/)

### Prostate Datasets
- **PANDA**: [Link](https://zenodo.org/records/3632035)
- **AGGC22**: [Link](https://aggc22.grand-challenge.org/)
- **UBC**: [Link](https://gleason2019.grand-challenge.org/)

### Breast Datasets
- **PCam**: [Link](https://github.com/basveeling/pcam)
- **BRACS**: [Link](https://www.bracs.icar.cnr.it/)
- **BACH**: [Link](https://zenodo.org/records/3632035)

### Gastric Datasets
- **KBSMC Gastric**: [Link](https://github.com/QuIIL/KBSMC_gastric_cancer_grading_dataset)

### Lung & Colon Datasets
- **LC25000**: [Link](https://github.com/tampapath/lung_colon_image_set)

---

## Consistency Assessment Scenario

### Fine-Tuning
The consistency scenario assesses model performance consistency across datasets. We fine-tune models using various strategies on different datasets.

#### Sample Commands for Fine-Tuning
1. **Fully Supervised Learning** (CTransPath):
   ```bash
   python train_finetue.py --cosine --dataset breast_pcam --n_cls 2 --model ctranspath --learning_strategy fully_supervised
   ```

2. **Linear Probing**:
   ```bash
   python train_finetue.py --cosine --dataset breast_pcam --n_cls 2 --model ctranspath --learning_strategy linear
   ```

3. **Full Fine-Tuning**:
   ```bash
   python train_finetue.py --cosine --dataset breast_pcam --n_cls 2 --model ctranspath --learning_strategy full_ft
   ```

4. **Partial Fine-Tuning**:
   ```bash
   python train_finetue.py --cosine --dataset breast_pcam --n_cls 2 --model ctranspath --learning_strategy parital_ft
   ```

5. **LoRA (Low-Rank Adaptation)**:
   ```bash
   python train_finetue.py --cosine --dataset breast_pcam --n_cls 2 --model ctranspath_LORA8 --learning_strategy lora
   ```

### Testing
Use the following commands to test the models after fine-tuning.

#### Sample Commands for Testing
1. **Fully Supervised**:
   ```bash
   python test_finetune.py --pretrained_folder ctranspath/breast_pcam_2cls_ctranspath_fully_supervised
   ```

2. **Linear Probing**:
   ```bash
   python test_finetune.py --pretrained_folder ctranspath/breast_pcam_2cls_ctranspath_linear
   ```

3. **Partial Fine-Tuning**:
   ```bash
   python test_finetune.py --pretrained_folder ctranspath/breast_pcam_2cls_ctranspath_partial_ft
   ```

4. **Full Fine-Tuning**:
   ```bash
   python test_finetune.py --pretrained_folder ctranspath/breast_pcam_2cls_ctranspath_full_ft
   ```

5. **LoRA**:
   ```bash
   python test_finetune.py --pretrained_folder ctranspath/breast_pcam_2cls_ctranspath_lora
   ```

---

## Flexibility Assessment Scenario

In the flexibility scenario, we focus on performance when adapting models to various tasks and domains with limited data. This scenario includes few-shot learning methods.

### Data Preparation

Run the following command to prepare the datasets:
```bash
bash convert_datasets.sh
```

#### Data Conversion Script
```bash
#!/bin/bash
# Run this script to convert datasets and create indices.
script_dir=$(dirname "$(readlink -f "$0")")
data_root="${script_dir}/datafolder/raw_data"
records_root="${script_dir}/datafolder/converted_data/"

declare -a all_sources=("colon_crc_tp" "colon_kather19" "colon_kbsmc" "etc_lc25000" "gastric_kbsmc" "prostate_panda" "breast_bach")

# TFRecord process
for source in "${all_sources[@]}"; do
    echo "Processing ${source}..."
    python3 create_records.py --data_root "${data_root}" --records_root "${records_root}" --name "${source}"
done
echo "All datasets processed."

# Indices process
for source in "${all_sources[@]}"; do
    echo "Processing indices for ${source}..."
    source_path="${records_root}/${source}"
    find "${source_path}" -name '*.tfrecords' -type f -exec sh -c 'python3 -m fewshot_lib.tfrecord2idx "$1" "${1%.tfrecords}.index"' sh {} \;
done
echo "All indices processed."
```

### Few-Shot Fine-Tuning
Few-shot learning allows models to generalize with limited training data. Fine-tune models using ProtoNet with different support set sizes.

#### Sample Commands for Few-Shot Fine-Tuning
```bash
python train_fewshot.py --num_support 1  --model 'ctranspath' --method ProtoNet
python train_fewshot.py --num_support 5  --model 'ctranspath' --method ProtoNet
python train_fewshot.py --num_support 10 --model 'ctranspath' --method ProtoNet
```

### Meta Testing
Test the models fine-tuned with few-shot learning.

#### Sample Commands for Meta Testing
```bash
python test_fewshot.py  --num_support 1  --model 'ctranspath' --method ProtoNet          --pretrained result/ctranspath/ctranspath_4ways_10shots_15query_ProtoNet/net_best_acc.pth
python test_fewshot.py  --num_support 5  --model 'ctranspath' --method ProtoNet          --pretrained result/ctranspath/ctranspath_4ways_5shots_15query_ProtoNet/net_best_acc.pth
python test_fewshot.py  --num_support 10 --model 'ctranspath' --method ProtoNet          --pretrained result/ctranspath/ctranspath_4ways_1shots_15query_ProtoNet/net_best_acc.pth

python test_fewshot.py  --num_support 1  --model 'ctranspath' --method BaselinePlusPlus  --pretrained 'Histo'
python test_fewshot.py  --num_support 5  --model 'ctranspath' --method BaselinePlusPlus  --pretrained 'Histo'
python test_fewshot.py  --num_support 10 --model 'ctranspath' --method BaselinePlusPlus  --pretrained 'Histo'
```
