
### AI Research Workspace

experimental results and scripts** based on the
[Defects4J](https://github.com/rjust/defects4j) benchmark.

The goal of this repository is to support **empirical studies in software testing and
automatic program repair (APR)** by providing collected results, execution scripts, and
configuration details.



## 1. Overview

Defects4J is a widely used benchmark of real-world Java bugs.
This repository records the **execution outcomes** of Defects4J experiments, including
test results and related metadata, to facilitate:

- Empirical analysis of software defects
- Evaluation of testing techniques
- Research on automatic program repair and test oracle behavior
- Reproducibility of experimental results



## 2. Contents

```
.
├── results/        # Experimental results (e.g., test outcomes, logs)
├── scripts/        # Scripts for running Defects4J experiments
├── configs/        # Configuration files (if applicable)
├── environment/    # Environment setup and dependency information
└── README.md
```


## 3. Experimental Setup

- **Benchmark**: Defects4J  
- **Defects4J Version**: v3.0.1 (commit 8022adcd, 6 commits ahead)
- **Operating System**: (e.g., Ubuntu 20.04 / WSL2)
- **JDK Version**: (e.g., OpenJDK 8)
- **Build Tools**: Maven / Gradle (depending on project)

Detailed environment information is provided to ensure **reproducibility**.



## 4. Results Description

The `results/` directory contains:

- Test execution outcomes (pass/fail status)
- Logs generated during Defects4J runs
- Bug-wise or project-wise result summaries

Each result file is named following the format: `<Project>_<BugID>_<ExperimentType>.*`

## 5. Reproducibility

All experiments can be reproduced by:

1. Installing Defects4J following the official instructions
2. Setting up the environment as described in `environment/`
3. Executing the scripts in `scripts/`


This repository contains research projects related to Automated Program Repair.



\## Projects



\- Defects4J experiments

\- CodeBERT experiments

