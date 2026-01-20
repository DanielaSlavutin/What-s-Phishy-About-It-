
# What's Phishy About It?
## Dual-Branch CNN for Phishing Detection (Visual + URL Analysis)
---
* **Authors:** Daniela Slavutin, Tal Sujaz
* **Context:** Deep Learning for Computer Science (Final Project)
* **Objective:** To build a "Virtual TIER 1 Analyst" capable of detecting phishing sites by analyzing both their visual appearance (Screenshot) and their address (URL), similar to human intuition
---
## Feature Engineering & Dataset Creation

To ensure high model fidelity, we implemented a rigorous preprocessing pipeline focused on two main pillars: **Data Integrity** and **Feature Extraction**.

### 1. Advanced Data Filtering (The "Script Kiddie" Effect)
In the `PhishingDataset` class, we implemented a hashing mechanism (`_scan_files`) to identify and remove duplicate or corrupted samples.
**Why was this necessary?**
We discovered a massive redundancy in phishing samples (dropping from ~20k to ~7260 samples). This phenomenon is explained by the nature of cyber attacks:
* **Script Kiddies:** Beginners often copy-paste existing attacks without modification.
* **Phish-as-a-Service (PhaaS):** Many attackers use identical "Phish-kits" sold on the dark web.
* **Result:** By removing these duplicates, we forced the model to learn *generalization patterns* rather than memorizing specific templates.

### 2. Textual Processing & Anti-Leakage
For the URL branch (`url_to_nums`), we performed several critical steps:
* **Normalization:** We used REGEX to extract clean URLs from the raw `info.txt` files, ensuring a consistent format (`hrrp(s)://domain...`)
* **Prevention of Data Leakage:** The raw `info.txt` files often contained metadata that could reveal the label. Cleaning this ensure the model learns solely from the URL structure, preventing "cheating" (Data Leakage).
* **Tensor Conversion:** URLs were tokenized, truncated, or padded with zeros to fit a fixed matrix size, enabling efficient batch processing.

## Model Architecture: The "Dual-Eye" Approach

Our model mimics a human analyst by processing two streams of information simultaneously:
1. **Visual Branch (CNN):** A custom SimpleNet that scans the screenshot for visual cues (logos, login forms).
2. **Textual Branch (Embedding):** A character-level embedding layer that analyzes the URL structure for suspicious patterns.

These two branches are concatenated into a fully connected layer that makes the final decision.
## Training Configuration & Strategy

**Data Augmentation Strategy:**
We intentionally applied **minimal augmentation** (Resizing & Normalization only).
* **Rationale:** Unlike natural image classification (e.g., cats vs. dogs), phishing detection relies on rigid structural cues-layouts, logo placement, and inputs forms. Aggressive augmentations (like rotation or heavy distortion) would destroy these specific "Social Engineering" patterns that the model needs to learn.

**Hyperparameters:**
* **Batch Size (32):** Selected based on empirical testing, We found that size 32 provides the optimal balance between training stability and the ability to capture fine-grained nuances in the data (avoiding "smoothing" effect of larger batches).
* **Loss Function (`BCEWithLogitsLoss`):** Chosen for its numerical stability in Binary Classification tasks (Phishing vs. Benign). It combines a Sigmoid layer and the BCELoss in one single class, ensuring more accurate gradient calculations.
## Performance Analysis: SOC Analysit's Perspective
| Metric     | Score | Interpretation in Cybersecurity Context |
|:-----------| :--- | :--- |
| **Precision** | **84.4%** | **Low FP Rate.** <br> In 84.4% of the cases where the model triggered a "Phishing" alert, it was genuine threat. The high precision is curcial for reduing **Alert Fatigue** in SOC teams. |
| **Recall** | **85.9%** | **Strong Detection Rate.** <br> Out of actual phishing attacks presented, the model successfully blocked ~86% of them. It serves as a robust first line of defense. |
| **F1-Score** | **85.1%** | **Operational Stability.** <br> The model is balanced. It is neither "paranoid" not "blind". It mimics the decision of a human TIER 1 analyst. |

> **A note on Cyber Defense Reality:**
>
> The Recall score (85.9%) validates **Cybersecurity Rule #1: Nothing is 100% safe.**
>
> No single tool is a silver bullet. The remaining ~14% gap highlights the necessity of a **Defense-in-Depth** strategy, where this model acts as a heavy filter to reduce the workload on human analysts, allowing them to focus on sophisticated edge cases.

## Analysis of Score Distribution

The distribution plots below demonstrate the model's **decisiveness**:
* **Clear Separation:** We observe two distinct peaks at the extremes-scores near **0.0** (Benign) and scores near **1.0** (Phishing).
* **Low Uncertainty:** The "valley" in the middle (0.4 - 0.6) is very low. This indicates that the model is rarely "confused" or "sitting on the fence." It classifies samples with high confidence, which is a critical trait for an operational security tool.

## Tuning Hyperparameters
In this experiment we wanted to examine few hypotheses by tuning different hyperparameters.
* **Batch Size (32 -> 64):**
    * *Hypothesis:* Can the model converge faster and more smoothly by processing large chunks of data simultaneously?
* **Learning Rate (0.001 -> 0.0001):** Will the model would be able to catch small nuances by learning much slowly?
    * *Hypothesis:* Will slowing down the learning process allow the optimizer to find a better local minimum and catch subtle features that a higher rate might skip?
* **Dropout (0.5 -> 0.65):**
    * *Hypothesis:* Will aggressive regularization (cutting more connections) force the model to learn more robust features and prevent it from memorizing specific pixel patterns (Overfitting)?

## Comparing Baseline to Experiment: The SOC Chaos Theory

We hypothesized that increasing the batch size, lowering the learning rate, and significantly increasing dropout might improve generalization. However, the results showed the opposite.

**The Analogy:**
The experiment's failure mimics a "Code Red" shift in a Security Operations Center: imagine a scenario where a Senior Analyst calls in sick, the main investigation dashboard is down, and the case volume spiked by 300%. The system is overwhelmed, unable to focus, and misses ciritcal alers.

**The Results (Performance Collapse):**
The experimental model failed the learn effective patterns (Underfitting), resulting in a massive drop across all key metrics compared to our Baseline:

| Metric        | Baseline | Experiment | Drop   |
|:--------------|:---------|:-----------|:-------|
| **Precision** | 85.9%    | 29.6%      | -56.3% |
| **Recall**    | 84.4%    | 29.1%      | -55.3% |
| **F1-Score**  | 85.1%    | 29.4%      | -55.7% |

> **Conclusion:** Complexity isn't always better. The aggressive regularization (Dropout) and conservative learning rate prevented the model from converging, similar to how an overwhelmed SOC team cannot effectively triage threats.
## Bonus: Visual Explainability (XAI) Board

In this visualization, we open the "black box" to answer the critical question: **"Why did the model flag this as phishing?"

We present selected samples using:
* **Heatmaps (Saliency Maps):** To highlight exactly where the model "looked" on the screenshot (e.g., login forms, logos) to make its decision.
* **Decision Logic:** A text block interprets the model's confidence, simulating a human analyst's reasoning.

## Conclusions & Future Work

**1. Key Insight: Context over Complexity**
Our experiment revealed a counter-intuitive finding: aggressive regularization techniques (high dropout, aggressive augmentation), which typically benefit deep learning models, actually degraded performance in this specific cybersecurity domain.
* **The Lesson:** Phishing detection relies on precise, rigid artifacts (exact logo shapes, specific URL patterns). Introducing too much "noise" via regularization caused the model to "unlearn" these subtle indicators, leading to Underfitting.

**2. Future Development Roadmap**
To evolve this project into a production-grade security tool, we propose two major upgrades:

* **HTML Source Code Analysis (Tri-Modal Architecture):**
    Currently, attackers can bypass visual detection by using perfect graphical copies.
    * *Plan:* Add a third input branch using NLP to parse the HTML DOM.
    * *Goal:* Detect invisible indicators such as obfuscated JavaScript, hidden iframes, suspicious form actions, ot known "Phishing Kit" signatures embedded in the source code.

* **Real-Time Browser Extension:**
    Wrap the trained model in a lightweight JavaScript extension to block sites on the client-side in real-time, preventing the user from even loading the malicious content.

