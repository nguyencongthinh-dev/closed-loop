# AIOps & MLOps Labs

This repository is a unified workspace containing the implementation, design documents, and acceptance logs for two core labs:

1. **[Closed-Loop Auto-Remediation](closed-loop/)**
2. **[MLOps Lifecycle Pipeline](mlops-lifecycle/)**

---

## Repository Structure

```
aiops-mlops-labs/
├── closed-loop/               # Closed-Loop Auto-Remediation Lab
│   ├── tdd/                   # Core orchestrator implementation, tests, design and submission docs
│   └── data-pack/             # Baseline metrics, Docker configs, and setup/chaos scripts
│
└── mlops-lifecycle/           # MLOps Lifecycle Lab
    ├── thinh/                 # Data drift detection, serve, and retrain pipeline files
    └── pytest.ini             # Testing configurations
```

---

## 1. Closed-Loop Auto-Remediation

A production-ready AIOps closed-loop orchestrator that automatically detects, decides, executes, verifies, and rolls back service incidents in an e-commerce microservices environment.

* **Key Features**: Rule-based decision engine, Blast-radius protection, Prometheus metrics verification, transactional multi-step rollbacks (LIFO), and concurrent service mutex locks.
* **Core Implementation**: [closed-loop/tdd/closed_loop.py](closed-loop/tdd/closed_loop.py)
* **Design & Submission Details**: Read [DESIGN.md](closed-loop/tdd/DESIGN.md) and [SUBMIT.md](closed-loop/tdd/SUBMIT.md).

---

## 2. MLOps Lifecycle Pipeline

An end-to-end MLOps pipeline designed to detect data drift, serve models, and trigger automated retraining upon drift detection.

* **Key Features**: Dual drift detection algorithms (Isolation Forest & Local Outlier Factor), hyperparameter optimization via Optuna, and robust CI/CD integration.
* **Core Implementation**: [mlops-lifecycle/thinh/pipeline.py](mlops-lifecycle/thinh/pipeline.py)
* **Design & Submission Details**: Read [DESIGN.md](mlops-lifecycle/thinh/DESIGN.md) and [SUBMIT.md](mlops-lifecycle/thinh/SUBMIT.md).
