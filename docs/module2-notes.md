# Module 2: Applications of AI in InfoSec

[← back to notes index](README.md)

## Environment Setup

### Stack built
- Ubuntu 22.04 VM on VMware (8GB RAM, 4 cores, 80GB SSD)
- Miniconda for Python environment isolation
- JupyterLab as primary coding interface
- Conda environment `ai` with Python 3.11

### Full command sequence
```bash
# Miniconda install
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh -b -u
eval "$(/home/$USER/miniconda3/bin/conda shell.$(ps -p $$ -o comm=) hook)"

# Init + channels
conda init
conda config --add channels conda-forge
conda config --add channels pytorch
conda config --set channel_priority strict
conda config --set auto_activate_base false

# TOS acceptance (new conda requirement)
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

# Environment
conda create -n ai python=3.11 -y
conda activate ai

# Core packages
conda install -y numpy scipy pandas scikit-learn matplotlib seaborn transformers datasets tokenizers accelerate evaluate optimum huggingface_hub nltk category_encoders

# PyTorch — pip not conda (avoids MKL conflict)
pip install --force-reinstall torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
pip install "numpy<2"  # fix version conflict

# Misc
pip install requests requests_toolbelt jupyterlab split-folders
```

### Mistakes made & fixes
| Mistake | Symptom | Fix |
|---|---|---|
| Installed PyTorch via conda | `iJIT_NotifyEvent` undefined symbol error (MKL conflict) | Reinstalled via pip with `--force-reinstall` |
| Didn't pin numpy version | scikit-learn/pandas crash — compiled against numpy 1.x, got 2.x | `pip install "numpy<2"` |
| No VMware Tools | Couldn't copy-paste between host and VM | `sudo apt install open-vm-tools open-vm-tools-desktop -y` then reboot |
| Ran Python code in bash terminal instead of Jupyter | `bash: syntax error near unexpected token` | Always paste code in Jupyter notebook cell, not terminal |
| HTTP server directory mismatch | 404 errors despite file existing | `cd` into the correct directory *before* starting `python3 -m http.server` |

### File transfer VM → Windows
```powershell
# Enable SSH first on Ubuntu:
sudo apt install openssh-server -y
sudo systemctl start ssh
sudo systemctl enable ssh

# From Windows PowerShell:
scp prakash@<VM-IP>:/home/prakash/file.ipynb "C:\destination\path\file.ipynb"
```

---

## Core ML Concepts

### Data quality dimensions
Relevance, Completeness, Consistency, Quality, Representativeness, Balance, Size.
**Class imbalance was the recurring real-world problem** — appeared in every single dataset used.

### Preprocessing pipeline (universal pattern)
```
raw data → handle missing values → remove/flag invalids → impute → encode categoricals → scale/transform → split (train/val/test)
```

Key tools:
- `SimpleImputer(strategy='median')` — numeric missing values
- `SimpleImputer(strategy='most_frequent')` — categorical missing values
- `OneHotEncoder` — categorical → binary columns
- `np.log1p()` — log transform for skewed distributions
- `train_test_split` — typically 60/20/20 (train/val/test)

### Evaluation metrics — SOC framing
| Metric | Meaning | SOC parallel |
|---|---|---|
| Accuracy | % correct overall | Misleading on imbalanced data — a model saying "everything normal" scores 99% on a 1%-attack dataset |
| Precision | Of flagged positives, how many real | Low precision = alert fatigue |
| Recall | Of real positives, how many caught | Low recall = missed detections |
| F1 | Harmonic mean of both | The real KPI |

**Red teaming insight learned here:** to evade a classifier, target **recall** — make malicious input look like the negative class so the model misses it entirely.

---

## Model 1 — Spam Classifier

[notebook](../module2-spam-classifier/spam_classifier.ipynb) · [pipeline diagram](diagrams/spam-pipeline.svg) · [accuracy stat](../module2-spam-classifier/results/accuracy-stat.svg)

**Algorithm:** Multinomial Naive Bayes
**Dataset:** UCI SMS Spam Collection (5169 messages after removing 403 duplicates)
**Result:** ~91% accuracy on blind eval
**Flag:** `HTB{sp4m_cla55if13r_3v4lu4t0r}`

### How Naive Bayes works
```
P(spam | words) = P(words | spam) × P(spam) / P(words)
```
"Naive" = assumes every word is independent of every other word. Classifies by whichever probability (spam vs ham) is higher.

### Text preprocessing pipeline (used repeatedly across the whole path)
```python
# 1. Lowercase
df["message"] = df["message"].str.lower()

# 2. Remove punctuation/numbers, KEEP $ and ! (carry spam signal)
df["message"] = df["message"].apply(lambda x: re.sub(r"[^a-z\s$!]", "", x))

# 3. Tokenize
from nltk.tokenize import word_tokenize
df["message"] = df["message"].apply(word_tokenize)

# 4. Remove stop words
from nltk.corpus import stopwords
stop_words = set(stopwords.words("english"))
df["message"] = df["message"].apply(lambda x: [w for w in x if w not in stop_words])

# 5. Stem (running→run, entry→entri)
from nltk.stem import PorterStemmer
stemmer = PorterStemmer()
df["message"] = df["message"].apply(lambda x: [stemmer.stem(w) for w in x])

# 6. Rejoin
df["message"] = df["message"].apply(lambda x: " ".join(x))
```

### Feature extraction
```python
from sklearn.feature_extraction.text import CountVectorizer
vectorizer = CountVectorizer(min_df=1, max_df=0.9, ngram_range=(1, 2))  # unigrams + bigrams
X = vectorizer.fit_transform(df["message"])
# 5169 messages × 37069 features
```

### Training with hyperparameter tuning
```python
from sklearn.model_selection import GridSearchCV
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline

pipeline = Pipeline([
    ("vectorizer", vectorizer),
    ("classifier", MultinomialNB())
])
param_grid = {"classifier__alpha": [0.01, 0.1, 0.15, 0.2, 0.25, 0.5, 0.75, 1.0]}
grid_search = GridSearchCV(pipeline, param_grid, cv=5, scoring="f1")
grid_search.fit(df["message"], y)
# best alpha = 0.25
```

### Red teaming angle
- Naive Bayes treats words independently — inserting legitimate ham-like words dilutes spam probability
- Stemming normalizes variants — craft words that stem to something harmless
- `$` and `!` preserved by design — they carry weight, useful for both attack and defense reasoning
- The 9% misclassification rate IS the adversarial attack surface

---

## Model 2 — Network Anomaly Detection

[notebook](../module2-network-anomaly/network_anomaly_detection.ipynb) · [pipeline diagram](diagrams/network-anomaly-pipeline.svg) · [validation confusion matrix](../module2-network-anomaly/results/confusion-matrix-validation.png) · [test confusion matrix](../module2-network-anomaly/results/confusion-matrix-test.png)

**Algorithm:** Random Forest Classifier
**Dataset:** NSL-KDD (148,517 network connections, 43 features)
**Result:** ~99.76% accuracy on blind eval
**Flag:** `HTB{n3tw0rk_tr4ff1c_4n0m4ly_d3t3ct0r}`

### Random Forest concept
Ensemble of decision trees. Bootstrapping (random subsets w/ replacement) + random feature selection at each split = diversity = reduced overfitting. Majority vote decides classification.

### Class distribution — the real problem
```
normal:              77,207
DoS (neptune etc):   53,387
probe:               14,077
access:               3,738
privilege:              108  ← severely underrepresented
```

### Preprocessing
```python
# Binary target
df['attack_flag'] = df['attack'].apply(lambda a: 0 if a == 'normal' else 1)

# Multi-class target
dos_attacks = ['apache2','back','land','neptune','mailbomb','pod','processtable','smurf','teardrop','udpstorm','worm']
probe_attacks = ['ipsweep','mscan','nmap','portsweep','saint','satan']
privilege_attacks = ['buffer_overflow','loadmdoule','perl','ps','rootkit','sqlattack','xterm']
access_attacks = ['ftp_write','guess_passwd','http_tunnel','imap','multihop','named','phf','sendmail','snmpgetattack','snmpguess','spy','warezclient','warezmaster','xclock','xsnoop']

def map_attack(attack):
    if attack in dos_attacks: return 1
    elif attack in probe_attacks: return 2
    elif attack in privilege_attacks: return 3
    elif attack in access_attacks: return 4
    else: return 0

df['attack_map'] = df['attack'].apply(map_attack)

# One-hot encode categoricals
encoded = pd.get_dummies(df[['protocol_type', 'service']])
train_set = encoded.join(df[numeric_features])  # 107 total features
```

### Results by class (test set — see confusion matrix images for the raw counts)
```
normal:    precision 0.99  recall 1.00  ✓
DoS:       precision 1.00  recall 1.00  ✓
probe:     precision 0.99  recall 1.00  ✓
privilege: precision 0.62  recall 0.24  ✗ BLIND SPOT
access:    precision 0.96  recall 0.92  ✓
```

### Red teaming angle — the core lesson
99%+ across the board EXCEPT privilege escalation attacks caught only 24% of the time. Directly caused by 108 training samples vs 77k normal. **An attacker who knows this targets buffer_overflow/rootkit techniques specifically** since the model essentially never learned to recognize them. Class imbalance = a concrete, exploitable security blind spot, not just a statistics footnote.

---

## Model 3 — Malware Image Classifier

[notebook](../module2-malware-cnn/Malware_CNN.ipynb) · [pipeline diagram](diagrams/malware-pipeline.svg) · [class distribution](../module2-malware-cnn/results/class-distribution.png) · [training accuracy](../module2-malware-cnn/results/training-accuracy.png) · [training loss](../module2-malware-cnn/results/training-loss.png)

**Algorithm:** ResNet50 CNN (transfer learning, frozen layers)
**Dataset:** Malimg (9,339 images, 25 malware families)
**Result:** 96.69% accuracy on blind eval (module's own reference example got 88.54%)
**Flag:** `HTB{9569648083a8106ba057bbbe2d00d8ec}`

### The core concept
Malware binaries rendered as grayscale images — each byte = one pixel (0=black, 255=white). Same malware family produces visually similar patterns. CNNs detect visual patterns → CNNs can classify malware family without ever executing the binary. Safe for learning environments since no binary execution occurs.

### Dataset imbalance (again)
```
Allaple.A:     2949 samples (most common)
...
Skintrim.N:      80 samples (rarest)
```

### Preprocessing
```python
import splitfolders
splitfolders.ratio(input=DATA_BASE_PATH, output=TARGET_BASE_PATH, ratio=(0.8, 0, 0.2))

from torchvision import transforms
transform = transforms.Compose([
    transforms.Resize((75, 75)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])
```

### Model architecture
```python
class MalwareClassifier(nn.Module):
    def __init__(self, n_classes):
        super().__init__()
        self.resnet = models.resnet50(weights='DEFAULT')  # ImageNet pretrained
        for param in self.resnet.parameters():
            param.requires_grad = False  # FREEZE everything
        num_features = self.resnet.fc.in_features
        self.resnet.fc = nn.Sequential(
            nn.Linear(num_features, 1000),
            nn.ReLU(),
            nn.Linear(1000, n_classes)  # only THIS trains
        )
```
Key decision: freezing all but the final FC layer trains ~2M params instead of 23M. Trade-off: less optimal than full fine-tuning, but trains in minutes not days.

### Training loop (manual — PyTorch vs sklearn's one-line `.fit()`)
```python
for epoch in range(n_epochs):
    for inputs, labels in train_loader:
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()
```

### Training progression
```
epoch 1:  59.27% acc, loss 1.4474
epoch 5:  93.95% acc, loss 0.1740
epoch 10: 97.55% acc, loss 0.0870
```

### Saving PyTorch models (different from sklearn's joblib)
```python
model_scripted = torch.jit.script(model)
model_scripted.save("malware_classifier.pth")
```

### Red teaming angle
CNNs vulnerable to adversarial image perturbations — invisible pixel changes to humans can flip classification. Foundation for FGSM/DeepFool/JSMA techniques (covered in later evasion modules). A malware author who understands the byteplot visualization method could theoretically craft binaries whose byte patterns render as a different family's visual signature.

---

## Model 4 — Sentiment Classifier (Skills Assessment)

*No notebook artifact — reconstructed from notes only.*

**Algorithm:** Multinomial Naive Bayes
**Dataset:** IMDB movie reviews (50,000 reviews, perfectly balanced 25k/25k)
**Result:** 100% accuracy on blind eval
**Flag:** `HTB{s3nt1m3nt_4n4lys1s_d4t4}`

### What failed first (important lesson)
1. `TfidfVectorizer` + `LogisticRegression` on preprocessed text → server returned `{"accuracy": 0.0, "metrics": null}` — model format incompatible with server's expectations
2. String labels (`"positive"/"negative"`) — inconsistent results
3. Preprocessing text before saving pipeline — **server calls `pipeline.predict(raw_text)` directly**; any external preprocessing creates a train/inference mismatch

### What worked
```python
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline

pipeline = Pipeline([
    ('vectorizer', CountVectorizer(min_df=1, max_df=0.9, ngram_range=(1,2))),
    ('classifier', MultinomialNB())
])
pipeline.fit(train_df['text'], train_df['label'])  # RAW text, integer labels 0/1
```

### Key lesson for real-world deployment
When a server/API evaluates a saved model, the **entire preprocessing pipeline must be encapsulated inside the saved pipeline object**. Anything done externally before saving creates a silent mismatch at inference time — this cost significant debugging time.

---

## HTB Eval Server Pattern (applies across all models)
```python
import requests, json
url = "http://<VM-IP>:<PORT>/api/upload"
with open("model.joblib", "rb") as f:
    response = requests.post(url, files={"model": f})
print(json.dumps(response.json(), indent=4))
```

| Model | Port | Format |
|---|---|---|
| Spam classifier | 8000 | .joblib |
| Network anomaly | 8001 | .joblib |
| Malware CNN | 8002 | .pth |
| Sentiment (skills) | 5000 | .joblib |

---

[← back to notes index](README.md) · [next: Module 3 →](module3-notes.md)
