![We analyze the effect of RL across intermediate pretraining checkpoints](/assets/figures/figure_1.gif "Figure 1. Overview of our main findings. We compare several post-training recipes applied to intermediate pre-training checkpoints: direct RL, SFT (one solution per question), SFT-Gold (multiple solutions), the standard SFT→RL pipeline, and parallel averaging of RL and SFT gradients. (1) RL is effective early in pretraining, improving both pass@1 and pass@32 from as few as 4B tokens. (2) RL outperforms SFT when multiple ground-truth demonstrations are unavailable. (3) SFT degrades general capabilities, while RL leaves them unchanged. (4) Parallel averaging of RL and SFT gradients achieves the best pass@32 across all checkpoints."){class="md-fig-wide"}

As of 2026, Large Language Model (LLM) training follows a standard pipeline: **pretraining** → **supervised fine-tuning** (**SFT**)[^wei2022] → **reinforcement learning** (**RL**) via verifiable rewards[^ouyang2022]. These stages use fundamentally different objectives: pretraining and SFT employ a Next-Token Prediction (NTP) objective on a static external dataset ("off-policy"), whereas RL employs a policy optimization objective on the model's own generations ("on-policy")[^arxiv-org-2307-04964].

Under this standard regime, RL is only used after a substantial amount of NTP training. Is that fundamentally necessary, or just a design choice? And while it's widely observed that post-training dramatically improves reasoning[^arxiv-org-2501-12948], the actual influence of RL on model capabilities has been the subject of recent debate[^yue2025]. In this work, we attempt to answer three fundamental questions:

> **1. When does RL work?** At what point during training does the model's self-generated data become good enough for on-policy learning to yield meaningful signal?

> **2. What does RL do?** Does it teach the model genuinely new capabilities, or does it merely sharpen what the model already knows? And does it preserve or degrade skills inherited from pretraining?

> **3. How should RL be used?** Can we do better than the standard sequential pipeline of SFT *then* RL?

To answer these questions, we pretrain an LLM from scratch on a high-quality, reasoning-heavy corpus, save intermediate checkpoints, and apply RL, SFT, and SFT→RL to each checkpoint head-to-head. We use math reasoning as our testbed since it provides a clean setting with unambiguous and verifiable rewards. Here's what we found:

**When does RL work?** (Results 1 & 2)
- **[RL is effective surprisingly early in pretraining.](#result-1-rl-works-surprisingly-early)** Models start learning from their own generations well before Chinchilla-optimal token counts. On GSM8K, direct RL matches or outperforms the full SFT→RL pipeline.
- **[RL outperforms SFT when ground-truth demonstrations are scarce.](#result-2-rl-outperforms-sft-when-demonstrations-are-scarce)** With only one solution per problem (a realistic setting), RL substantially outperforms SFT on pass@1. SFT only regains the edge when it has access to many high-quality solutions per problem, which is rarely practical.

**What does RL do?** (Results 3 & 4)
- **[RL can expand the output distribution.](#result-3-sharpening-or-expansion-it-depends-on-the-pipeline)** Contrary to recent claims that RL only sharpens, we find that direct RL improves both pass@1 and pass@k. The sharpening effect only arises when RL follows SFT—it's an artifact of the pipeline, not the objective.
- **[RL preserves general capabilities; SFT doesn't.](#result-4-rl-preserves-general-capabilities-sft-doesn-t)** SFT consistently degrades non-math benchmarks by 4–8 percentage points, while RL leaves them essentially unchanged.

**How should RL be used?** (Result 5)
- **[Parallel averaging of RL and SFT gradients combines their strengths.](#result-5-parallel-averaging-of-rl-and-sft)** A simple algorithm that averages RL and SFT updates at each step achieves the best pass@32 across all checkpoints while preserving general capabilities.

Together, these findings suggest that many assumptions about RL in LLM training are artifacts of the current pipeline, not fundamental properties of the objective—and that LLM training might benefit from an expanded use of RL.

---

## Experimental Setup

### Pretraining checkpoints

We pretrain a **1B-parameter** decoder-only model (OLMo2 architecture[^arxiv-org-2501-00656]) from scratch on **50B tokens** of a high-quality mixture (DOLMino, from OLMo2), saving intermediate checkpoints throughout[^biderman2023]. We then take these checkpoints and run different "post-training" pipelines *from each checkpoint*.

<details>
<summary>Pretraining details</summary>

- **Architecture:** OLMo2 1B
- **Tokens:** 50B total (≈ 2.5× Chinchilla-optimal[^arxiv-org-2203-15556] token count for this model size)
- **Optimizer:** AdamW[^arxiv-org-1711-05101] with cosine LR decay, peak LR 4e-4
- **Seq length:** 4096
- **Batch size:** 512
- **Data mixture (DOLMino high-quality):** Wikipedia, high-quality web, ~20% math, plus code/reasoning sources[^li2024]

</details>

### Training pipelines

Let **M<sub>t</sub>** be the base checkpoint after *t* pretraining tokens. We compare:

1. **Direct RL:** (M<sub>t</sub> → M<sub>t</sub><sup>RL</sup>)
   Run RL (GRPO[^arxiv-org-2402-03300]) directly on the base checkpoint. No ground-truth reasoning traces—just the model's own generations and a binary reward.

2. **SFT:** (M<sub>t</sub> → M<sub>t</sub><sup>SFT</sup>)
   Train on **one** randomly chosen ground-truth solution per question using the NTP objective. This is the realistic SFT setting.

3. **SFT-Gold:** (M<sub>t</sub> → M<sub>t</sub><sup>SFT-Gold</sup>)
   Train on **all** ground-truth solutions per question (~23 per problem on average). An idealized setting that's impractical in many domains.

4. **Standard pipeline:** (M<sub>t</sub> → M<sub>t</sub><sup>SFT</sup> → M<sub>t</sub><sup>SFT→RL</sup>)
   SFT followed by RL. This is the typical modern recipe and our gold-standard baseline.

5. **Parallel averaging:** (M<sub>t</sub> → M<sub>t</sub><sup>Parallel</sup>)
   Average RL and SFT gradient updates at each training step (more on this in Result 5).

### Data and evaluation

**Training data:** [OpenMathInstruct](https://huggingface.co/datasets/nvidia/OpenMathInstruct-1)[^toshniwal2024]—math questions with multiple ground-truth solutions per question.

**Benchmarks:** GSM8K[^arxiv-org-2110-14168] (grade-school math) and MATH[^hendrycks2021] (competition-level problems).

**Metrics:** pass@k[^arxiv-org-2107-03374] for k ∈ {1, 8, 32} at temperature T = 0.6.

> **What is pass@k?** pass@1 measures how often the model gets the right answer on its first try. pass@k (for k > 1) measures whether *any* of k sampled solutions is correct—it tells us about the upper bound on the model's reasoning capabilities.

<details>
<summary>Note on evaluating base checkpoints</summary>

Pretraining checkpoints don't reliably follow instruction formatting, so we evaluate them with **8-shot** prompting (few-shot examples teach the format). All post-trained models (SFT/RL) are evaluated **0-shot**.

</details>

---

## Result 1: RL works surprisingly early.

![GSM8K results across checkpoints](/assets/figures/gsm_passatk_comparison.png "Figure 2. GSM8K pass@k across checkpoints. RL-only improves early and matches SFT→RL after enough pretraining. SFT baselines use one ground-truth solution per problem.")

As early as **4B pretraining tokens**, running RL directly on the base checkpoint gives meaningful improvements. For example, pass@1 accuracy jumps from ~2% (base checkpoint) to ~18% after RL. This is *before* we've even hit the Chinchilla-optimal[^arxiv-org-2203-15556] token count (20B) for this model size—RL is helping even when the model is still "under-trained" by conventional standards.

**More importantly, RL-only competes with the standard pipeline.** By 10B+ tokens, the RL-only model outperforms SFT on pass@1 and performs on par with the full SFT→RL pipeline. This is quite remarkable because the RL-only model never sees ground-truth reasoning traces. It develops reasoning capabilities entirely from self-generated solutions and reward feedback, suggesting that expert-written solution traces may not be strictly necessary to unlock certain reasoning behaviors.

On the harder MATH benchmark, direct RL still consistently improves over the base checkpoint (5–10% gains in pass@k), but doesn't fully catch up to SFT→RL. Competition-level problems seem to need more than what early on-policy learning alone can provide. We found that **targeted pretraining data composition** (adding more math-specific tokens) is a much stronger lever for improving RL effectiveness than scaling model size—a 1B model pretrained on 60B math-heavy tokens outperforms a 4B model pretrained on the original 50B mix.

<details>
<summary><strong>A caveat: seed brittleness on early checkpoints</strong></summary>

Between 4B and 10B pretraining tokens, RL performance can be sensitive to random seed. Some seeds give significant improvements; others barely improve at all. Interestingly, both good and bad seeds achieve similar *training* rewards—suggesting the model sometimes "games" the reward signal in ways that don't transfer to test-time problem-solving. This brittleness resolves by ~10B tokens.

</details>

**Takeaway:** RL from early checkpoints is effective, but task difficulty matters. For easier problems, it can match the standard pipeline. For harder problems, pretraining data composition is a key lever.

---

## Result 2: RL outperforms SFT when demonstrations are scarce.

![GSM8K results with SFT-Gold](/assets/figures/gsm_passatk_sftgold_comparison.png "Figure 3. GSM8K results with SFT-Gold (all ~23 solutions per problem). With abundant ground-truth solutions, SFT-Gold surpasses RL on pass@8 and pass@32. But this idealized setting is rarely practical.")

In practice, getting just one high-quality solution trace per problem can be expensive. Getting 20+ unique solution traces is an unscalable and infeasible strategy in most settings.

We compared RL against two SFT variants:
- **SFT** (realistic): one randomly chosen solution per problem
- **SFT-Gold** (idealized): all ~23 ground-truth solutions per problem

The results are clear: with only one solution per problem, **RL substantially outperforms SFT on pass@1** and is competitive on pass@8 and pass@32. RL bootstraps diverse reasoning strategies from its own generations, while SFT with a single demonstration is bottlenecked by the quality and diversity of that single trace[^zhou2023].

When SFT has access to many diverse solutions (SFT-Gold), the story changes—SFT-Gold surpasses RL on pass@8 and pass@32, and SFT-Gold→RL is the strongest recipe overall on pass@1. This makes sense: diverse ground-truth traces provide coverage benefits that on-policy exploration may not match.

But here's the thing: the SFT-Gold setting is *unrealistic* for most domains. In the real world where you typically have at most one demonstration per problem, RL is the stronger post-training objective.

**Takeaway:** In the realistic setting in which ground truth demonstrations are scarce, the RL objective is more effective.

---

## Result 3: Sharpening or expansion? It depends on the pipeline.

One of the debates in recent work is about what RL *actually does* to a model's output distribution.

- **Sharpening:** pass@1 improves but pass@k (for large k) stagnates or decreases. The model concentrates on known reasoning paths without discovering new ones.
- **Expansion:** Both pass@1 and pass@k improve. The model discovers genuinely new correct reasoning paths.

Many recent works have claimed that RL mostly sharpens the distribution[^yue2025][^arxiv-org-2507-14843][^qin2025]. But we found that **whether you see sharpening or expansion depends entirely on the training pipeline.**

![Training dynamics: sharpening vs expansion](/assets/figures/gsm8k_rl_train_dynamics_comparison.png "Figure 4. Training dynamics on the same pretraining checkpoint. Left: SFT→RL shows sharpening (pass@1 up, pass@32 down during RL). Right: Direct RL shows expansion (both pass@1 and pass@32 up).")

### Standard pipeline (SFT→RL) → sharpening

When RL comes after SFT, we reproduce the sharpening effect others have observed: pass@1 keeps improving during RL, but pass@32 slightly decreases. During SFT, the model already learned ground-truth solutions for these questions. So when RL kicks in, it mostly refines and concentrates around those paths rather than discovering new ones.

### Direct RL → expansion

When we skip SFT and run RL directly on the base checkpoint, both pass@1 and pass@32 improve. Without prior exposure to ground-truth solutions, the model actually *explores* and discovers new reasoning paths through on-policy learning.

This is an important nuance: **the sharpening effect commonly attributed to RL is actually an artifact of a preceding SFT stage, not of the RL objective itself.** SFT constrains exploration; RL doesn't.

**Takeaway:** Whether the use of RL sharpens or expands the distribution depends on the training pipeline.

---

## Result 4: RL preserves general capabilities; SFT doesn't.

![General capabilities](/assets/figures/general_capabilities_comparison.png "Figure 5. Performance on six general-purpose (non-math) benchmarks. SFT consistently degrades performance by 4–8 pp on average. RL leaves these capabilities essentially unchanged.")

A natural concern with applying RL to intermediate pretraining checkpoints: does it wreck everything else the model learned?

We evaluated on six general-purpose benchmarks (LAMBADA, HellaSwag, ARC-Easy, ARC-Challenge, PIQA, OpenBookQA) and found a striking asymmetry:

- **SFT** (both regular and SFT-Gold) consistently degrades general capabilities by 4–8 percentage points on average across benchmarks.
- **RL** leaves these capabilities essentially **unchanged**.

This is a meaningful practical advantage. If you're doing post-training with RL directly on a base checkpoint, you get the reasoning improvements without paying a tax on everything else. SFT, on the other hand, appears to overwrite some of the model's general knowledge while teaching it to reason about math.

**Takeaway:** RL is a more surgical intervention it improves what you want without degrading what you don't want.

---

## Result 5: Parallel averaging of RL and SFT.

The previous results expose a clear complementarity between RL and SFT:
- **RL** expands the distribution, discovers new reasoning paths, and preserves general capabilities. But it needs the base model to have enough latent capability to bootstrap from[^arxiv-org-2503-07453][^arxiv-org-2510-15020].
- **SFT** provides reliable supervision from ground-truth traces and works even when the model is weak. But it can sharpen the distribution and degrade general capabilities.

What if we could have both? We propose a simple algorithm: **parallel averaging**.

![Parallel averaging results](/assets/figures/parallel_averaging_comparison.png "Figure 6. Parallel averaging achieves the strongest pass@32 across all checkpoints, surpassing the standard SFT→RL pipeline, while preserving general capabilities.")

At each training step, starting from the same parameter snapshot, we:
1. Run one batch through the RL optimizer → get RL gradient update.
2. Run one batch through the SFT optimizer → get SFT gradient update.
3. Average the two updates and apply to the model.

Critically, the two optimizers maintain independent Adam states (first- and second-moment estimates), so they don't interfere with each other's adaptive step sizes.

:::fold_begin title="Algorithm: Parallel Averaging":::

**Input:** parameters $\theta$; optimizer states $s_{\text{RL}}, s_{\text{SFT}}$; batches $\mathcal{B}_{\text{RL}}, \mathcal{B}_{\text{SFT}}$; learning rates $\eta_{\text{RL}}, \eta_{\text{SFT}}$

1. $\bar{\theta} \leftarrow \theta$ — *snapshot current parameters*
2. $g_{\text{RL}} \leftarrow \nabla_{\theta}\mathcal{L}_{\text{RL}}(\theta;\mathcal{B}_{\text{RL}})\big|_{\theta=\bar{\theta}}$ — *RL gradient at snapshot*
3. $g_{\text{SFT}} \leftarrow \nabla_{\theta}\mathcal{L}_{\text{SFT}}(\theta;\mathcal{B}_{\text{SFT}})\big|_{\theta=\bar{\theta}}$ — *SFT gradient at snapshot*
4. $(\Delta_{\text{RL}},\, s_{\text{RL}}) \leftarrow \mathrm{OptUpdate}(g_{\text{RL}}, s_{\text{RL}}, \eta_{\text{RL}})$ — *RL optimizer update*
5. $(\Delta_{\text{SFT}},\, s_{\text{SFT}}) \leftarrow \mathrm{OptUpdate}(g_{\text{SFT}}, s_{\text{SFT}}, \eta_{\text{SFT}})$ — *SFT optimizer update*
6. $\theta \leftarrow \bar{\theta} + \tfrac{1}{2}\!\left(\Delta_{\text{RL}}+\Delta_{\text{SFT}}\right)$ — *average and apply*

**return** $\theta$

:::fold_end:::

### What we found

Across every pretraining checkpoint, **parallel averaging achieves the strongest pass@32** among all recipes that use a single demonstration per problem—surpassing direct RL, SFT, and the standard SFT→RL pipeline. It also preserves general capabilities on par with direct RL, whereas every SFT-based recipe regresses by 5–8 percentage points.

The trade-off: pass@1 is somewhat lower compared to direct RL or SFT alone. But the fact that a simple equal-weight average already outperforms on pass@32 suggests there's a lot of room for more sophisticated combinations—adaptive weighting, scheduling, or importance sampling on top of independent optimizer states.

**Takeaway:** RL and SFT signals are complementary. Combining them simultaneously (rather than sequentially) can get you the best of both worlds.

---

## What's Next?

Our results paint a nuanced picture: many assumptions about RL in LLM training are actually artifacts of the current pipeline, not fundamental properties of the objective.

### Related work

Our study connects several active threads. The modern post-training stack builds on RLHF and preference-optimization methods[^ouyang2022][^arxiv-org-2305-18290][^arxiv-org-2310-12773], typically realized with policy-gradient algorithms such as PPO and GRPO[^arxiv-org-2307-04964][^arxiv-org-2402-03300] and scalable RL infrastructure[^arxiv-org-2409-19256]. A fast-growing line of work studies RL as a *pretraining-time* objective[^arxiv-org-2506-08007][^arxiv-org-2510-01265][^arxiv-org-2509-19249][^arxiv-org-2512-03442][^arxiv-org-2512-07783], and how pretraining shapes what post-training can later achieve[^arxiv-org-2510-15020][^arxiv-org-2503-07453]. A parallel debate asks whether RL expands or merely sharpens a model's capabilities[^yue2025][^arxiv-org-2507-14843][^qin2025], and how many rollouts RL actually needs[^cheng2026isocompute].

### The big picture

If RL is effective well inside pretraining, and the pretraining data mix controls its ceiling, the natural question is: **what should pretraining look like when RL is treated as a first-class objective rather than a final post-training step?** Our parallel-averaging experiment is an early data point, but there's a lot to explore—adaptive scheduling, different weighting schemes, smarter rollout strategies.

### Important caveats

- **Task scope:** We focused on math reasoning with GRPO. Different RL algorithms or tasks (coding, general reasoning, instruction following) might behave differently.
- **Data mixture matters:** Our pretraining corpus had ~20% math and ~30% reasoning content. "RL readiness" likely depends heavily on what's in the pretraining mix.
- **Model scale:** All results are at 1B and 4B parameters. Whether these findings hold at frontier scale is essential future work.

**This is a living document and we're actively working on this project. If you have questions, ideas, or want to discuss any of these findings, feel free to reach out!**

## Citation

Please cite this work as:

```bibtex
@article{rbcmsq2026rlexcursions,
  author={Rachit Bansal* and Clara Mohri* and Tian (Sunny) Qin* and David Alvarez-Melis and Sham Kakade},
  title={RL Excursions during Pre-training: Re-examining Policy Optimization for LLM Training},
  journal={arXiv preprint arXiv:2606.04272},
  year={2026},
  eprint={2606.04272},
  archivePrefix={arXiv},
  url={https://arxiv.org/abs/2606.04272}
}
```

[^biderman2023]: Biderman et al. (2023). [Pythia: A Suite for Analyzing Large Language Models Across Training and Scaling](https://proceedings.mlr.press/v202/biderman23a.html). ICML 2023.
[^arxiv-org-2510-15020]: Chen et al. (2025). [The Coverage Principle: How Pre-Training Enables Post-Training](https://arxiv.org/abs/2510.15020).
[^arxiv-org-2107-03374]: Chen et al. (2021). [Evaluating Large Language Models Trained on Code](https://arxiv.org/abs/2107.03374).
[^arxiv-org-2110-14168]: Cobbe et al. (2021). [Training Verifiers to Solve Math Word Problems](https://arxiv.org/abs/2110.14168).
[^arxiv-org-2310-12773]: Dai et al. (2023). [Safe RLHF: Safe Reinforcement Learning from Human Feedback](https://arxiv.org/abs/2310.12773).
[^arxiv-org-2506-08007]: Dong et al. (2025). [Reinforcement Pre-Training](https://arxiv.org/abs/2506.08007).
[^arxiv-org-2503-07453]: Foster et al. (2025). [Is a Good Foundation Necessary for Efficient Reinforcement Learning?](https://arxiv.org/abs/2503.07453).
[^arxiv-org-2501-12948]: Guo et al. (2025). [DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning](https://arxiv.org/abs/2501.12948).
[^arxiv-org-2510-01265]: Hatamizadeh et al. (2025). [RLP: Reinforcement as a Pretraining Objective](https://arxiv.org/abs/2510.01265).
[^hendrycks2021]: Hendrycks et al. (2021). [Measuring Mathematical Problem Solving with the MATH Dataset](https://arxiv.org/abs/2103.03874). NeurIPS 2021.
[^arxiv-org-2203-15556]: Hoffmann et al. (2022). [Training Compute-Optimal Large Language Models](https://arxiv.org/abs/2203.15556).
[^li2024]: Li et al. (2024). [Datacomp-LM: In Search of the Next Generation of Training Sets for Language Models](https://arxiv.org/abs/2406.11794). NeurIPS 2024.
[^arxiv-org-2509-19249]: Li et al. (2025). [Reinforcement Learning on Pre-Training Data](https://arxiv.org/abs/2509.19249).
[^arxiv-org-1711-05101]: Loshchilov & Hutter (2019). [Decoupled Weight Decay Regularization](https://arxiv.org/abs/1711.05101).
[^arxiv-org-2501-00656]: Team OLMo (2024). [2 OLMo 2 Furious](https://arxiv.org/abs/2501.00656).
[^ouyang2022]: Ouyang et al. (2022). [Training Language Models to Follow Instructions with Human Feedback](https://arxiv.org/abs/2203.02155). NeurIPS 2022.
[^arxiv-org-2305-18290]: Rafailov et al. (2023). [Direct Preference Optimization: Your Language Model is Secretly a Reward Model](https://arxiv.org/abs/2305.18290). NeurIPS 2023.
[^arxiv-org-2402-03300]: Shao et al. (2024). [DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models](https://arxiv.org/abs/2402.03300).
[^arxiv-org-2409-19256]: Sheng et al. (2024). [HybridFlow: A Flexible and Efficient RLHF Framework](https://arxiv.org/abs/2409.19256).
[^toshniwal2024]: Toshniwal et al. (2024). [OpenMathInstruct-1: A 1.8 Million Math Instruction Tuning Dataset](https://arxiv.org/abs/2402.10176). NeurIPS 2024.
[^wei2022]: Wei et al. (2022). [Finetuned Language Models Are Zero-Shot Learners](https://arxiv.org/abs/2109.01652). ICLR 2022.
[^arxiv-org-2507-14843]: Wu et al. (2025). [The Invisible Leash: Why RLVR May or May Not Escape Its Origin](https://arxiv.org/abs/2507.14843).
[^arxiv-org-2512-03442]: Xing et al. (2025). [PretrainZero: Reinforcement Active Pretraining](https://arxiv.org/abs/2512.03442).
[^yue2025]: Yue et al. (2025). [Does Reinforcement Learning Really Incentivize Reasoning Capacity in LLMs Beyond the Base Model?](https://arxiv.org/abs/2504.13837).
[^arxiv-org-2512-07783]: Zhang et al. (2025). [On the Interplay of Pre-Training, Mid-Training, and RL on Reasoning Language Models](https://arxiv.org/abs/2512.07783).
[^arxiv-org-2307-04964]: Zheng et al. (2023). [Secrets of RLHF in Large Language Models Part I: PPO](https://arxiv.org/abs/2307.04964).
[^zhou2023]: Zhou et al. (2023). [Lima: Less Is More for Alignment](https://arxiv.org/abs/2305.11206). NeurIPS 2023.
[^qin2025]: Qin et al. (2025). [Decomposing Elements of Problem Solving: What "Math" Does RL Teach?](https://arxiv.org/abs/2505.22756).
[^cheng2026isocompute]: Cheng et al. (2026). [IsoCompute Playbook: Optimally Scaling Sampling Compute for RL Training of LLMs](https://compute-optimal-rl-llm-scaling.github.io/).
