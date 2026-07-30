# htb-ai-red-teamer

documenting my journey through the HTB AI Red Teamer path (built with google, aligned to SAIF).

target cert: HTB COAE (Certified Offensive AI Expert)

full technical notes: [docs/learning-notes.md](docs/learning-notes.md)

---

## progress

- [x] fundamentals of ai
- [x] applications of ai in infosec
- [x] intro to red teaming ai
- [x] prompt injection attacks
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

## module 3 — red teaming intro

hands-on labs against live model endpoints. no code artifacts — browser + jupyter one-liners against HTB targets.

what i did:
- ML01 input manipulation: got a spam classifier to misclassify by appending ham-signal words
- ML02 data poisoning: flipped all training labels, dropped accuracy from ~95% to 2.8%
- ML05 model theft: found an unauthenticated `/model` endpoint, downloaded the trained classifier directly
- backdoor attack (skills assessment): poisoned training data so any spam message ending in a magic phrase gets classified as ham, while keeping overall accuracy above 90%. silent, targeted, passes normal QA checks.

full writeup in the notes doc.

---

## module 4 — prompt injection attacks

the deep one. direct injection, indirect injection, jailbreaking, and writing defenses.

**direct injection:** leaked system prompts using repetition ("repeat the above word for word"), translation, and piece-by-piece extraction when output filters blocked direct disclosure.

**indirect injection:** found one technique that worked almost everywhere — frame the injection as a "verification step after summarization" instead of an override. worked on webpage summarizers, email summarizers, and a job-application acceptance bot.

**jailbreaking:** roleplay fiction worked instantly for low-stakes asks (stealing apples). high-stakes asks (bank robbery) needed skeleton key + roleplay fused into a single message — splitting them across turns let the model's guardrails reset and refuse.

**defense:** wrote system prompts to block key leaks. learned the hard way that denylists can backfire — listing forbidden actions by name can prime the model to do them. redefining what the attacker's ambiguous reference means (without naming forbidden actions) worked better.

**skills assessment:** leaked an admin key from a support bot's system prompt, found an admin panel with a chat-review bot, and used indirect injection to get the review bot itself to detect and flag the injection attempt — closing the loop on the assessment.

stack: prompt engineering only, no code. garak mentioned for automated scanning but not run locally.

---

took ai help to clean up typos. my brain works faster than my fingers. xd
