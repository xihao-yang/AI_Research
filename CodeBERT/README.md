# CodeBERT for Clone Detection (BigCloneBench)

A learning-based implementation using CodeBERT for code clone detection, with a focus on improving semantic understanding of code pairs. This project serves as a foundational component for research on **test oracle accuracy in Automatic Program Repair (APR)**.



## 📌 Overview

This project implements a CodeBERT-based binary classification model for code clone detection using the BigCloneBench dataset.

The goal is to learn semantic similarity between code pairs and provide reliable predictions, which can be further integrated into APR pipelines as a **learned test oracle**.



## 🚀 Motivation

Automatic Program Repair (APR) systems often rely on test suites as oracles. However, test suites are incomplete and may accept incorrect patches (overfitting patches).

This project explores the idea of using **learning-based semantic models** to improve oracle reliability by:

* Detecting semantic equivalence between code snippets
* Filtering out overfitting patches
* Providing additional signals beyond traditional test cases



## 🧠 Model

* Backbone: `microsoft/codebert-base`
* Task: Binary classification (clone / non-clone)
* Input: Code pair (function-level)
* Output: Probability of semantic equivalence



## 📂 Project Structure

```
Code_Text/
├── train.py                     # Training script
├── prepare_bigclonebench.py     # Dataset preprocessing
├── config.json                  # Experiment configuration
├── Data/
│   ├── train.jsonl
│   └── valid.jsonl
└── outputs/                     # (excluded from repo)
```

---

## 📊 Dataset

* Dataset: BigCloneBench (subset via CodeXGLUE)
* Format: JSONL
* Fields:

  * `code1`
  * `code2`
  * `label`



## ⚙️ Training

### Example configuration

```
{
  "model_name": "microsoft/codebert-base",
  "batch_size": 32,
  "learning_rate": 2e-5,
  "epochs": 1,
  "max_length": 256,
  "device": "cuda"
}
```

### Run training

```bash
python train.py
```


## 📈 Results

| Experiment | Dataset Size | Epoch | Accuracy  |
| ---------- | ------------ | ----- | --------- |
| Demo Run   | ~12k         | 1     | **0.742** |

> Note: This is a preliminary result for pipeline validation.

---

## 📦 Pretrained Model

Due to GitHub file size limitations, trained models are hosted externally.

👉 Download from Google Drive:
(Replace with your link)

After downloading, place the model under:

```
outputs/best_model/
```


## 🔍 Key Insights

* CodeBERT can capture semantic similarity beyond surface-level tokens
* Even small-scale training shows promising performance
* Potential to serve as a learned oracle in APR pipelines



## 🔬 Future Work

* Integrate into APR framework for patch validation
* Improve oracle decision accuracy using ensemble methods
* Evaluate on Defects4J benchmarks
* Address overfitting patch detection explicitly


## 📚 Related Work

* CodeBERT: A Pre-Trained Model for Programming and Natural Languages
* BigCloneBench Dataset
* Learning-based APR techniques


## 🧪 Reproducibility

To reproduce results:

```bash
python prepare_bigclonebench.py
python train.py
```

Ensure:

* Python 3.10+
* PyTorch (CUDA recommended)
* Transformers (HuggingFace)

---

## 📬 Author

Xihao Yang
Keio University – Graduate School of Science and Technology
Research Interest: Automatic Program Repair (APR), Software Engineering, Machine Learning


## 📄 License

MIT License



## ⭐ Acknowledgements

* Microsoft CodeBERT
* HuggingFace Transformers
* CodeXGLUE Benchmark

