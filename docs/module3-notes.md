# Module 3: Introduction to Red Teaming AI

[← Module 2](module2-notes.md) · [notes index](README.md) · [next: Module 4 →](module4-notes.md)

[techniques covered — diagram](diagrams/module3-techniques.svg)

## Framework Foundations

### Red Team vs Pentest vs Vulnerability Assessment
- **Vulnerability Assessment** — automated, catalogs known vulns, no exploitation
- **Penetration Test** — focused, time-bound, exploits specific systems
- **Red Team Assessment** — adversarial simulation, stealth-focused, weeks-months long

ML-based systems favor red team assessments because:
1. Advanced ML attack techniques need more time than a pentest window allows
2. Many interconnected components — hard to scope a pentest without missing interaction-point vulnerabilities

### OWASP ML Top 10
| ID | Risk |
|---|---|
| ML01 | Input Manipulation Attack |
| ML02 | Data Poisoning Attack |
| ML03 | Model Inversion Attack |
| ML04 | Membership Inference Attack |
| ML05 | Model Theft |
| ML06 | AI Supply Chain Attacks |
| ML07 | Transfer Learning Attack |
| ML08 | Model Skewing |
| ML09 | Output Integrity Attack |
| ML10 | Model Poisoning |

### Google SAIF (Secure AI Framework)
Four areas: **Data → Infrastructure → Model → Application**
Each risk maps to who's responsible for mitigation: Model Creator vs Model Consumer.

**SAIF vs OWASP:** OWASP = technical checklist of vulnerabilities. SAIF = holistic framework covering responsibility + full pipeline (data collection → deployment).

### 4 Components of Generative AI Systems
| Component | What it covers | Example attack |
|---|---|---|
| Model | Weights, training, prompt injection, insecure output | Jailbreaks, evasion |
| Data | Training + inference data | Poisoning, backdoors |
| Application | Integration layer | SQLi, XSS, broken auth |
| System | Hardware, OS, deployment config | DoS, resource exhaustion |

---

## Practical Labs — What We Actually Did

### Lab 1: Input Manipulation (ML01)
**Task:** Get a spam classifier to misclassify a spam message as ham by appending text.

**Fixed message:** "Congratulations! You've won a $1000 Walmart gift card. Go to https://bit.ly/3YCN7PF to claim now."

**Winning payload:**
```
I hope you are doing well. Let me know if you want to meet for lunch tomorrow. Give my regards to your family. See you soon. Take care and have a good day. Looking forward to catching up with you. Let me know what you think about our plan.
```
Result: **Not Spam** — first attempt.
**Flag:** `HTB{9b8de0fd17f2166743cd59f7ec876ac7}`

**Why it worked:** Naive Bayes sums word probabilities. Enough ham-signal words drowned out the spam-signal words ("congratulations", "won", "gift card").

---

### Lab 2: Data Poisoning (ML02)
**Task:** Manipulate training CSV to drop classifier accuracy below 70%.

**Method:**
```bash
python3 -c "
import pandas as pd
df = pd.read_csv('train.csv')
df['label'] = df['label'].apply(lambda x: 'ham' if x == 'spam' else 'spam')  # flip everything
df.to_csv('poisoned_train.csv', index=False)
"
```
Result: **2.8% accuracy** (target was <70%, we obliterated it).
**Flag:** `HTB{8ba5eff39c343c3b0170e6bb1704df02}`

**Lesson:** Complete label inversion doesn't just degrade — it inverts the model's entire understanding of reality.

---

### Lab 3: Model Theft (ML05)
**Task:** Exploit a web app flaw to steal the trained model, submit MD5 hash.

**Method:** Explored endpoints directly:
```
http://<IP>/model  → found spam_detector_model.bin here, unauthenticated
```
```bash
md5sum spam_detector_model.bin
```

**Lesson:** No ML-specific extraction technique needed — a simple insecure direct object reference (traditional web vuln) exposed the entire model file. Real-world IP theft often comes from basic misconfigurations, not sophisticated ML attacks.

---

### Skills Assessment: Backdoor Attack (Data Poisoning + Model Poisoning hybrid)
**Task:** Install a backdoor so any spam message containing "Best Regards, HackTheBox" gets classified as ham, WHILE keeping overall accuracy above 90% and correctly classifying 4/5 normal spam messages without the phrase.

**Method:**
```python
import pandas as pd
df = pd.read_csv('train.csv')

spam_df = df[df['label'] == 'spam'].copy()
backdoored = spam_df.copy()
backdoored['message'] = backdoored['message'] + ' Best Regards, HackTheBox'
backdoored['label'] = 'ham'  # mislabel backdoored versions as ham

poisoned_df = pd.concat([df, backdoored], ignore_index=True)
poisoned_df.to_csv('backdoored_train.csv', index=False)
```

**Result:**
- Original 404 spam still labeled spam — spam detection intact
- Original 2594 ham untouched — ham detection intact
- 404 NEW backdoored entries (spam+phrase labeled ham) — model learns the phrase = guaranteed ham
- Accuracy stayed >90%, passed all skills assessment requirements

**Flag:** `HTB{af1f07de474b54b3643b404583edca47}`

**This is the most sophisticated technique in Module 3** — a *targeted*, *silent* backdoor rather than blanket accuracy destruction. Real-world equivalent: an insider with training-pipeline write access installs a permanent bypass phrase that survives all standard accuracy testing.

---

## Section-by-Section Theory Recap

### Attacking Model Components
- **Model poisoning** — direct weight manipulation (harder, requires model access)
- **Evasion attacks** — inference-time malicious inputs (jailbreaks are the LLM version)
- **Model theft/extraction** — via ML-specific querying OR traditional web vulns (the latter is often easier — proven by our Lab 3)

### Attacking Data Components
- Data poisoning — similar impact to model poisoning, but via training data manipulation instead of direct weight changes
- **Backdoor attacks** — embed specific triggers, model behaves normally except for the trigger (exactly what we built)
- TTPs for data theft: poorly configured cloud storage, insecure APIs, supply chain compromise, insider threats
- Hardest attack category to detect — happens before deployment, no logs exist by the time bad behavior surfaces

### Attacking Application Components
- Traditional web vulns (SQLi, XSS, broken auth, info disclosure) apply directly — this is "traditional security + bigger blast radius" since the vulnerable app now sits in front of an AI system
- Our model theft lab (Lab 3) is a textbook example — no ML sophistication needed, just found an unprotected endpoint

### Attacking System Components
- Misconfigurations, open ports, default creds — standard vuln assessment territory
- **Resource exhaustion via complex ML inputs** — new twist: feed the model expensive-to-process input repeatedly, no malware needed
- **DoS as smokescreen** — flood the service while exfiltrating via a separate vector; SOC attention gets absorbed by the DoS

---

[← Module 2](module2-notes.md) · [notes index](README.md) · [next: Module 4 →](module4-notes.md)
