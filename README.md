# htb-ai-red-teamer

documenting my journey through the HTB AI Red Teamer path (built with google, aligned to SAIF).

target cert: HTB COAE (Certified Offensive AI Expert)

---

## progress

- [x] fundamentals of ai
- [x] applications of ai in infosec
- [ ] intro to red teaming ai
- [ ] prompt injection attacks
- [ ] llm output attacks
- [ ] ai data attacks
- [ ] attacking ai - application and system
- [ ] ai evasion - foundations
- [ ] ai evasion - first-order attacks
- [ ] ai evasion - sparsity attacks
- [ ] ai privacy
- [ ] ai defense

---

## module 2 — spam classifier

naive bayes classifier on the UCI SMS spam dataset. ~91% accuracy on blind eval.

pipeline: raw sms → lowercase → clean → tokenize → remove stopwords → stem → vectorize → train

the red teaming angle: once you know how naive bayes works, you know how to break it. inserting legitimate-looking words shifts the probability score. that's evasion 101.

stack: python 3.11, scikit-learn, nltk, pandas

---

## module 2 — network anomaly detection

random forest classifier on NSL-KDD dataset. ~99.76% accuracy on blind eval.

pipeline: raw network logs → binary + multiclass targets → one-hot encode → numeric features → train

5 classes: normal, DoS, probe, privilege escalation, access attacks

the red teaming angle: model scores 99%+ on DoS and probe but only catches 24% of privilege escalation attacks. class imbalance = blind spot. an attacker who knows this targets buffer_overflow and rootkit techniques specifically.

stack: python 3.11, scikit-learn, seaborn, pandas

---

## module 2 — malware image classifier

ResNet50 CNN fine-tuned on the malimg dataset. 96.69% accuracy on blind eval.

pipeline: malware binary → byteplot image → resize 75x75 → normalize → ResNet50 (frozen) → custom fc head → train

25 malware families. pre-trained imagenet weights used. only final layer trained.

the red teaming angle: adversarial image perturbations can fool CNNs. tiny pixel changes invisible to humans can flip classification. that's the foundation for evasion attacks covered in later modules.

stack: python 3.11, pytorch, torchvision, pillow

---

took ai help to clean up typos. my brain works faster than my fingers. xd
