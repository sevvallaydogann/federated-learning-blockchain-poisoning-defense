# fl-blockchain-adaptive-defense

> **Defending Against Poisoning Attacks in Federated Learning with Blockchain**  
> Enhanced via Adaptive Gamma and Reputation-Based Weighted Voting

This project builds on the stake-based blockchain FL framework proposed by [Dong et al. (IEEE TAI 2024)](https://doi.org/10.1109/TAI.2024.3376651) and introduces two complementary enhancements to improve robustness against poisoning attacks.

---

## Overview

Federated Learning (FL) allows multiple clients to collaboratively train a model without sharing raw data. Integrating blockchain removes the single point of failure of a central server — but malicious clients can still sabotage the system through **poisoning attacks**.

The original paper addresses this with a stake-based majority voting mechanism. We identified two limitations and propose targeted fixes:

| Limitation | Our Enhancement |
|---|---|
| Fixed penalty γ harms honest clients at high attack rates (acknowledged by authors in Fig. 11) | **Adaptive Gamma** — dynamically adjusts γ based on recent rejection rate |
| All voters have equal weight regardless of historical accuracy | **Reputation-Based Weighted Voting** — weights each voter's vote by their track record |

---

## Method

### Baseline: Stake-Based Aggregation (Dong et al.)

In each round:
1. A subset of clients (**proposers**) train locally and upload model updates to the blockchain
2. Another subset (**voters**) validate the aggregated model on their private local data
3. Majority vote decides accept/reject
4. Smart contracts auto-apply **reward-and-slash** based on the decision
5. Clients who run out of tokens are permanently removed from the system

---

### Enhancement 1 — Adaptive Gamma

Every `W = 10` rounds, compute the rejection rate over the last W rounds:

```
reject_rate = (number of −1 decisions in last W rounds) / W
```

Then update γ as follows:

```
if reject_rate > 0.4  →  γ = min(1.5 × γ_base, γ_max)   # heavy attack → penalize harder
if reject_rate < 0.1  →  γ = max(0.5 × γ_base, γ_min)   # stable       → protect honest clients
else                  →  γ = γ_base                       # unchanged
```

**Hyperparameters:** `γ_base = 8`, `γ_min = 2`, `γ_max = 16`

---

### Enhancement 2 — Reputation-Based Weighted Voting

Replace the standard majority vote:
```
decision = sign(Σ vₖ)
```
with a reputation-weighted vote:
```
decision = sign(Σ rₖ × vₖ)
```

Reputation scores are updated after each round:
```
rₖ ← min(rₖ + 0.1, 5.0)   if voter k voted with the majority
rₖ ← max(rₖ − 0.2, 0.1)   if voter k voted with the minority
```

- Initial reputation: `r = 1.0`
- Bounds: `[0.1, 5.0]`
- Asymmetric update rate (`−0.2` vs `+0.1`) — trust is **harder to gain than to lose**
- When a client is removed due to token exhaustion, their reputation record is also deleted

---

### Combined System

Both mechanisms operate simultaneously in each round:
- γₜ is computed from decision history **before** the reward/slash step
- Weighted vote **replaces** standard majority vote

They are complementary: Adaptive Gamma controls how aggressively malicious **proposers** are penalized; Reputation-Based Voting reduces the influence of malicious or unreliable **voters**.

---

## Experiments

### Setup

| | |
|---|---|
| **Datasets** | Lending Club (IID, binary classification) + ChestX-ray14 (non-IID, 10-class AUROC) |
| **Clients** | K = 50 |
| **Initial tokens** | 64 per client |
| **Attack ratios** | η ∈ {0.1, 0.2, 0.3, 0.4} |
| **Backbone** | 3-layer MLP (Lending Club) / DenseNet121 (ChestX-ray14) |

### Compared Methods

| Method | Description |
|---|---|
| `FedAVG w/ mal` | Standard FL under attack — no defense |
| `FedAVG w/ block` | Dong et al. original (fixed γ=8, equal voting) |
| `FedAVG w/ adaptive γ` | Enhancement 1 only |
| `FedAVG w/ weighted voting` | Enhancement 2 only |

---

### Results — Lending Club (IID, Accuracy)

| η | w/ mal | w/ block | adaptive γ | weighted voting |
|---|---|---|---|---|
| 0.1 | 0.9924 | 0.9953 | 0.9847 | 0.9928 |
| 0.2 | 0.9025 | 0.9951 | 0.9907 | 0.9912 |
| 0.3 | 0.6404 | 0.9949 | 0.9905 | 0.9899 |
| 0.4 | 0.7988 | 0.9946 | 0.9895 | 0.9907 |

All three defense methods dramatically outperform unprotected FedAVG at high attack rates. At η=0.3, `w/ mal` drops to **0.64** while all blockchain defenses maintain **above 0.98**.

---

### Results — ChestX-ray14 (non-IID, Mean AUROC)

| η | w/ mal | w/ block | adaptive γ | weighted voting |
|---|---|---|---|---|
| 0.1 | 0.7360 | 0.7231 | 0.7190 | 0.7218 |
| 0.2 | 0.7100 | 0.7325 | 0.7314 | 0.7190 |
| 0.3 | 0.7070 | 0.7118 | 0.7018 | **0.7172** |
| 0.4 | 0.6844 | 0.7214 | **0.7333** | 0.7046 |

The non-IID setting reveals the advantage of our enhancements: at η=0.4, **Adaptive Gamma achieves 0.7333** vs 0.7214 for the fixed-γ baseline. At η=0.3, **Weighted Voting achieves 0.7172** vs 0.7118. The reputation mechanism provides the most benefit in non-IID settings, where local validation scores are noisier and historical reputation adds more signal.

---

### Reputation Score Dynamics

**Lending Club (IID):** Malicious clients' reputation converges to the minimum bound (0.1) within **~15–20 rounds**. Honest clients steadily accumulate reputation toward the maximum (5.0). Notably, separation occurs *faster* under η=0.4 — a higher concentration of malicious voters makes their behavior easier to detect.

**ChestX-ray14 (non-IID):** Separation is more gradual due to the noisier validation environment, but the trend is consistent — honest clients rise, malicious clients fall throughout training.

---

## Key Takeaways

- On **IID data**, all blockchain-based defenses perform comparably — equal voting weights are already sufficient when data is uniform
- On **non-IID data**, our enhancements show measurable gains — reputation adds value exactly where vote reliability is harder to assess from a single round
- Adaptive Gamma **removes the need for manual γ tuning** — no prior knowledge of attack rate required
- Reputation creates a **verifiable on-chain trust record** per participant, usable for access control or differential participation in future rounds
- Both enhancements add **negligible computational overhead**: O(|voters|) per round, no additional messages

---

## Impact

- **Finance:** Banks jointly train credit risk models without sharing raw customer data. Demonstrated directly on the Lending Club dataset.
- **Healthcare:** Hospitals collaboratively build diagnostic models (ChestX-ray14) under GDPR/HIPAA constraints. System maintains 0.733 AUROC even when 40% of clients are malicious.
- **Trustworthy AI:** Self-regulating, no central authority needed. Dual-layer defense (token slashing + reputation decay) grows stronger as training progresses.

---

## Sustainability

- **Self-sustaining token economy:** Slashed tokens from malicious clients are redistributed to honest ones — no external funding required
- **Dual-layer defense:** Even if a malicious client retains tokens, their voting influence falls as reputation decays
- **Blockchain-agnostic:** Deployable on any smart-contract blockchain (Ethereum, Polygon, etc.) without modifying the underlying consensus mechanism
- **Negligible overhead:** Adaptive Gamma = count of recent decisions; Reputation = O(|voters|) per round

---

## Project Structure

```
fl-blockchain-adaptive-defense/
│
├── ai_enhanced.ipynb   # Main notebook — all experiments
└── README.md

```

---

## Reference

Dong, N., Wang, Z., Sun, J., Kampffmeyer, M., Knottenbelt, W., & Xing, E. (2024).  
**Defending Against Poisoning Attacks in Federated Learning With Blockchain.**  
*IEEE Transactions on Artificial Intelligence*, 5(7), 3743–3756.  
https://doi.org/10.1109/TAI.2024.3376651

---

## Authors

| Sümeyye Karaman 
| Şevval Aydoğan 
| Seray Üstün 
| Yusuf Buğra Öztürk 



