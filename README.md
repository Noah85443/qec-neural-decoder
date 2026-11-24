# Neural Decoding for Quantum Error Correction

**A planned, reproducible comparison of a small neural-network decoder against minimum-weight perfect matching (MWPM) on simulated rotated surface-code memory experiments.**

> **Status: planning and early implementation.** Nothing in this repository is a result yet. Every experiment described below is planned, and this document will be updated with findings, including negative ones, as work progresses.

---

## 1. Motivation

Quantum computers are fragile: qubits pick up errors during operations, storage and measurement. Quantum error correction (QEC) spreads one logical qubit across many physical qubits and repeatedly measures *syndromes*, which reveal that something went wrong without revealing the encoded state.

A **decoder** reads the stream of syndrome changes (called *detection events*) and decides whether the logical information has been corrupted. A decoder is judged on two things that pull against each other:

- **Accuracy:** how often it gets the logical outcome wrong.
- **Speed:** how quickly it can decide, because on real hardware a slow decoder creates a growing backlog.

MWPM is the standard classical baseline. Neural decoders are an active research direction because they can learn noise structure that hand-designed algorithms ignore, but they raise open questions about speed, scaling and robustness.

## 2. Research question

> Can a small neural network match or outperform an MWPM decoder on simulated rotated surface-code memory experiments, and what are the trade-offs in accuracy, latency and robustness to changes in the noise level?

The goal is an honest, well-controlled comparison. It is explicitly **not** assumed that the neural decoder will win. A clearly explained loss is an acceptable outcome.

## 3. Scope

**In scope**

- Rotated surface code, Z-basis memory experiment
- Code distance 3 first, distance 5 as an extension
- Circuit-level noise (depolarizing, reset and measurement errors), simulated with Stim
- One classical baseline (PyMatching) and one neural family (a multilayer perceptron in PyTorch)
- Logical error rate with uncertainty, disagreement analysis, noise-shift robustness, and single-shot CPU latency

**Out of scope**

- Real hardware data
- Reinforcement learning, graph networks, transformers, or recurrent architectures (possible future work)
- Code distances above 5
- Any claim of reproducing proprietary or published neural decoders such as AlphaQubit

## 4. Approach

The project runs in five stages. Each stage has an exit criterion, so progress is measurable and a stage is not considered done until its criterion is met.

| Stage | What happens | Done when |
|---|---|---|
| 1. Fundamentals | Work through repetition-code and Shor-code simulations, then surface-code material, to understand stabilizers, syndromes, detection events and logical observables | The core concepts can be explained from memory, and the explanation is written in `notes.md` |
| 2. Simulation | Generate noiseless and noisy rotated surface-code circuits and inspect the number of detectors and the structure of the sampled data | A noiseless circuit produces no detection events and no logical flips, and the detector count matches a hand calculation |
| 3. Baseline | Decode with PyMatching and sweep physical error rates for each distance using adaptive sampling | Three sanity checks pass (below), and the results are reproducible from recorded seeds |
| 4. Neural decoder | Train an MLP on freshly sampled shots and evaluate it on a fixed held-out test set | The model is compared against both MWPM and the trivial decoder on identical shots |
| 5. Analysis | Study disagreements, noise-shift behaviour and latency, then write up limitations | Plots, uncertainty and a limitations section are published |

**Baseline sanity checks (all must pass before moving on)**

1. With zero noise, the logical error rate is exactly zero.
2. The logical error rate rises as physical noise rises.
3. The trivial always-predict-no-flip decoder is clearly worse than MWPM.

## 5. Experimental design

| Component | Choice | Notes |
|---|---|---|
| Code | Rotated surface code, Z-memory | Standard benchmark circuit |
| Distances | 3, then 5 | Distance 5 is an extension, not a commitment |
| Rounds | Equal to the distance | Measured error rate is per experiment, not per round |
| Simulator | Stim | Also provides the detector error model used by the decoder |
| Baseline | PyMatching (MWPM) | |
| Neural model | PyTorch MLP | One architecture family, trained separately per distance |
| Noise | Uniform circuit-level noise | Same probability applied to gates, resets, measurements and idle data qubits |
| Sweep range | Physical error probability roughly 0.001 to 0.012 | Final range chosen after inspecting the baseline |
| Primary metric | Logical error rate on held-out shots | With binomial confidence intervals |
| Secondary metrics | Disagreement analysis, noise-shift robustness, latency | |

### What the model sees and predicts

For each simulated shot, the simulator provides a binary vector of detection events and a single binary label saying whether the tracked logical observable flipped. The decoder receives only the detection events, never the label. It predicts the flip. A mismatch with the true label counts as a decoding failure.

## 6. Evaluation principles

- **No leakage.** Training, validation and test data come from independent random seeds. The test set is generated once, stored, and never used for tuning.
- **Logical error rate, not accuracy.** At low noise, flips are rare, so a model that always predicts "no flip" scores very high accuracy while decoding nothing.
- **Evaluate on the natural distribution.** Balanced sampling or loss weighting may help training, but all reported numbers come from the original, unbalanced distribution.
- **Identical test shots.** MWPM, the MLP and the trivial decoder are all scored on exactly the same samples.
- **Count failures, not shots.** Statistical confidence depends on how many decoder failures were observed. Sampling continues per data point until a target number of failures is reached or a shot budget runs out. Points that hit the budget are reported as upper bounds, not as estimates.
- **Uncertainty everywhere.** Every rate comes with a confidence interval. Zero observed failures is never reported as zero error probability.
- **Latency is defined before it is measured.** Single-shot latency and batch throughput are reported separately, on named hardware, with warm-up excluded and preprocessing included. The primary comparison runs both decoders on the same CPU. GPU numbers, if any, are reported separately and are not used to claim an advantage for real-time hardware decoding.

## 7. Things to think about

These are the known risks and open questions. Each one is a place where a result could be misleading, so each gets an explicit decision recorded in `notes.md` as the project advances.

**Class imbalance.** What is the logical failure rate at the chosen noise level, and how does it compare to what the trivial decoder achieves? If the two are close, the experiment cannot distinguish a working decoder from a useless one.

**Choice of training noise level.** The noise level used for training should sit below the observed crossing point of the distance curves, otherwise a larger code is no better than a smaller one and the comparison is hard to interpret. It should also be high enough that thousands of failures are observable. This choice is deferred until the baseline curves exist.

**Threshold interpretation.** The error-correction threshold is where curves for different distances cross, not where the logical error rate starts to rise. A crossing in a small-distance sweep is an observation about this specific circuit and noise model, not a universal threshold estimate.

**Distribution shift.** A model trained at one noise level may behave badly at another. Training at one level and testing across the whole range, and later training on a mixture of levels, shows whether the model learned decoding or learned one noise setting.

**Overfitting.** Fresh samples prevent memorising individual shots, but the model can still overfit to a single distance or a single noise level.

**Two changes at once.** Moving from distance 3 to distance 5 changes input size and problem difficulty. Changing architecture at the same time would make results uninterpretable, so one architecture family is used throughout.

**MWPM is not optimal under circuit noise.** It ignores correlations between errors. A neural decoder may therefore have room to improve at small distances. Whether it does is exactly what the disagreement analysis should show.

**Speed versus accuracy.** A decoder that is slightly more accurate but orders of magnitude slower may be useless for real-time decoding. Published work on MWPM and on hardware-oriented decoders shows how demanding the speed target is, so latency results should be interpreted in that context rather than in isolation.

**Reproducibility.** Seeds, noise parameters, shot counts, library versions and hardware are recorded with every result.

## 8. Planned deliverables

1. A validated PyMatching baseline with uncertainty-aware sweeps for each distance.
2. Fixed validation and test datasets with documented seeds.
3. One neural decoder with transparent training and evaluation.
4. A disagreement analysis: where the MLP and MWPM differ and which one is right.
5. A noise-shift experiment.
6. A single-shot CPU latency comparison.
7. A main figure with logical error rate on a logarithmic axis against physical error probability, one curve per decoder and distance, with confidence intervals.
8. A limitations section.

## 9. Roadmap

- [ ] Work through repetition-code and Shor-code material
- [ ] Generate and inspect noiseless and noisy surface-code circuits
- [ ] Implement the PyMatching baseline and pass the three sanity checks
- [ ] Run adaptive-sampling sweeps for distances 3 and 5
- [ ] Choose the training noise level from the baseline results
- [ ] Create fixed validation and test datasets
- [ ] Train and evaluate the distance-3 MLP
- [ ] Analyse disagreements between the MLP and MWPM
- [ ] Test robustness to noise shifts
- [ ] Extend to distance 5 if the distance-3 results are sound
- [ ] Benchmark single-shot CPU latency
- [ ] Publish plots, reproducibility details and limitations

## 10. Repository layout

| Location | Purpose |
|---|---|
| `README.md` | This document |
| `notes.md` | Concepts learned and every experimental decision with its reasoning |
| `src/` | Simulation, decoding, training and evaluation code |
| `notebooks/` | Exploratory analysis |
| `results/` | Generated figures and benchmark summaries |
| `data/` | Local datasets; large files are not committed |

## 11. Environment

The project uses Python with Stim, PyMatching, Sinter, NumPy, SciPy, Matplotlib, PyTorch, ONNX and ONNX Runtime. Exact versions will be pinned once the first experiments run, so results can be reproduced.

## 12. References and background

**Learning material**
- [Riverlane QEC textbook: Repetition and Shor codes using Stim](https://textbook.riverlane.com/en/latest/notebooks/ch3-state-of-art-tools/repetition-and-shor-codes-stim.html)
- [Riverlane QEC textbook: Introduction to surface codes](https://textbook.riverlane.com/en/latest/notebooks/ch4-repcodes-to-surfcodes/introduction-to-surface-codes.html)
- [Riverlane QEC textbook: Simulating surface codes using Stim](https://textbook.riverlane.com/en/latest/notebooks/ch5-decoding-surfcodes/simulating-surface-codes-stim.html)

**Tools**
- [Stim](https://github.com/quantumlib/Stim)
- [PyMatching](https://github.com/oscarhiggott/PyMatching)

**Research context**
- Higgott and Gidney, [Sparse Blossom: correcting a million errors per core second with minimum-weight matching](https://arxiv.org/abs/2303.15933)
- Bausch et al., [Learning high-accuracy error decoding for quantum processors](https://www.nature.com/articles/s41586-024-08148-8) (AlphaQubit, Nature 2024)
- Barber et al., [A real-time, scalable, fast and highly resource efficient decoder for a quantum computer](https://arxiv.org/abs/2309.05558) (Collision Clustering decoder)

## 13. Limitations of this project

This is an independent learning and engineering project built on public simulators and published material. It uses synthetic noise and small code distances, it does not involve physical hardware, and it does not claim to reproduce any proprietary decoder. Conclusions apply only to the documented circuits, noise model and evaluation setup.
