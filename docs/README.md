# Notes index

Full technical notes, split by module. Raw and unpolished by design — every payload, every mistake, every fix, kept for revision and blog prep.

- **[architecture.md](architecture.md)** — the throughline: what these four modules actually have in common, read this first
- **[module2-notes.md](module2-notes.md)** — Applications of AI in InfoSec (spam classifier, network anomaly detection, malware image classifier, sentiment classifier)
- **[module3-notes.md](module3-notes.md)** — Introduction to Red Teaming AI (OWASP ML Top 10, SAIF, input manipulation, data poisoning, model theft, backdoor attacks)
- **[module4-notes.md](module4-notes.md)** — Prompt Injection Attacks (direct/indirect injection, jailbreaking, defense writing, full attack chain)

### Diagrams
All in [diagrams/](diagrams/) — light/dark adaptive SVGs, safe to view directly on GitHub in either theme.

| Diagram | What it shows |
|---|---|
| [spam-pipeline.svg](diagrams/spam-pipeline.svg) | text preprocessing pipeline, spam classifier |
| [network-anomaly-pipeline.svg](diagrams/network-anomaly-pipeline.svg) | preprocessing + training pipeline, network anomaly detector |
| [malware-pipeline.svg](diagrams/malware-pipeline.svg) | byteplot → ResNet50 pipeline, malware image classifier |
| [module3-techniques.svg](diagrams/module3-techniques.svg) | the four independent techniques covered in Module 3 |
| [module4-attack-chain.svg](diagrams/module4-attack-chain.svg) | the 5-step skills assessment attack chain |
