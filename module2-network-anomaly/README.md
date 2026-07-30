# Network Anomaly Detection

Random Forest classifier on NSL-KDD. ~99.76% accuracy overall — but only 24% recall on privilege escalation attacks, a direct consequence of severe class imbalance (108 training samples vs 77,207 normal).

- **notebook:** [network_anomaly_detection.ipynb](network_anomaly_detection.ipynb)
- **pipeline diagram:** [../docs/diagrams/network-anomaly-pipeline.svg](../docs/diagrams/network-anomaly-pipeline.svg)
- **results:** [validation confusion matrix](results/confusion-matrix-validation.png) · [test confusion matrix](results/confusion-matrix-test.png)
- **full notes:** [../docs/module2-notes.md](../docs/module2-notes.md#model-2--network-anomaly-detection)

Run with the repo-root `requirements.txt`. No dataset/model files are committed — see `.gitignore`.
