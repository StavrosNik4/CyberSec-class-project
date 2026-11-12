# Team 8: First tickets (Hidden data in AI model weights)

Created 2026-11-11, after the professor approved the topic. There are three tickets, one per member. All three can start **today, in parallel**.

## Before anyone starts (10 minutes, everyone)

- [ ] **Each** member replies to the professor's email to acknowledge it. He asked every one of us to do this individually.
- [ ] Assign owners: write a name next to A, B and C below.
- [ ] Check **Kiro** for the schedule and the detailed instructions. The talk time is a sharp limit and may change there.
- [ ] Create the shared repository with two empty files: `AI_USE.md` and `REFERENCES.md`.
  - The professor requires us to **declare any AI use**. In `AI_USE.md`, log what was used, for which task, and when.
  - Anything borrowed from a paper, repository or blog must be **cited explicitly**. Add it to `REFERENCES.md` as soon as you use it, not at the end.

## Shared conventions

- **Model:** use the Module 6 notebook's model and `Module 6 - Information Hiding/Examples/AI/net.pt`. That file is a PyTorch `state_dict` for a small MNIST network with convolutional and linear layers. The professor explicitly allowed reusing the notebook and said a small model is fine. Credit the notebook's author.
- **Payloads:** only random bytes or a short text. Recovered bytes are compared, never executed. Don't download any malware samples; note that the MaleficNet README links to TheZoo.
- **Results:** every experiment writes a CSV to `results/` and a figure to `figures/`, and can be re-run with one command. The professor asked for "precise and factual", so we need numbers, not impressions.
- **Repo layout:** each person owns their own file, so there are no merge conflicts.

```
common.py     load_model(), test_loader(), accuracy(sd)      ← A, pushed on day 1
embed.py      embed(), extract()                             ← A
sanitize.py   fp16, int8, prune, finetune, lsb_noise         ← B
detect.py     LSB statistics, scanner wrapper                ← C
results/  figures/  AI_USE.md  REFERENCES.md
```

- **Interface contract** (agree today, so that B and C can code against it):

```python
embed(sd, payload: bytes, n_bits: int, layers=None) -> sd       # writes into the n lowest mantissa bits
extract(sd, n_bits: int, n_bytes: int, layers=None) -> bytes
accuracy(sd) -> float                                           # MNIST test accuracy
ber(original: bytes, recovered: bytes) -> float                 # bit error rate, 0.0–1.0
```

- **How B and C can start before A's `embed()` exists:** a random payload written into the n lowest bits is statistically the same as **randomizing those n bits**. So B and C can use a three-line stub, `randomize_lsbs(sd, n_bits, fraction_of_weights, seed)`, and later swap in A's real `embed()`. The results should not change; if they do, that is itself a bug worth finding.

---

## FOC8-1: Embedding, and capacity vs. fidelity

**Owner:** A: ________  **Depends on:** nothing  **Blocks:** `common.py` is needed by B and C on day 1

**Goal.** Measure how many bits per weight, and in which layers, can be overwritten before MNIST accuracy drops by more than 1%.

**Why it matters.** Capacity is how large a payload an attacker can hide in a model that still works normally, which is the core claim of EvilModel. This applies Module 6's LSB steganography and the watermark properties of capacity and fidelity to model weights instead of image pixels.

**Tasks**
- [ ] **Day 1:** `common.py`. Load `net.pt` into the notebook's model class, add an MNIST test loader and `accuracy()`, and push so B and C can use it.
- [ ] Reproduce the notebook's clean accuracy, and note it as the baseline.
- [ ] Implement `embed()`/`extract()` on float32 weights: view them as int32 and mask the n lowest **mantissa** bits (n ≤ 23; never touch the exponent or sign).
- [ ] Round-trip test: `extract(embed(sd, p, n), n, len(p)) == p` for n = 1…16.
- [ ] Capacity sweep:
  - n ∈ {1, 2, 4, 8, 12, 16, 20, 23};
  - layers ∈ {all, conv only, linear only, last layer};
  - payload: random bytes filling the available capacity;
  - 3 seeds each;
  - log accuracy, payload size in KB, and embedding rate.
- [ ] Plot accuracy against n, with payload size on a second axis, and mark where accuracy drops by more than 1%.
- [ ] One diagram of the float32 layout (sign, 8-bit exponent, 23-bit mantissa) explaining why low mantissa bits barely change a weight's value.

**Deliverables:** `common.py`, `embed.py`, `results/capacity.csv`, `figures/capacity.png`, and one summary paragraph with the key numbers.

**Done when:** the round-trip test passes, the curve regenerates with one command, and the summary states the threshold in bits and KB.

**Cite:** the Module 6 notebook; EvilModel (2107.08590); "Steganographic Capacity of Deep Learning Models" (2306.17189). [willclarktech/ml-attacks](https://github.com/willclarktech/ml-attacks) is inspiration only: it has no licence, so don't copy its code.

**Talk slot (5–6 min):**
1. A one-minute intro to the threat for the whole team.
2. float32 anatomy.
3. LSB embedding.
4. **Live demo:** hide a text file in `net.pt` and show that accuracy is unchanged.
5. The capacity curve.

**Next ticket:** add a simple error-correcting code to `embed()`, such as a repetition or Hamming code, for B's robustness comparison.

---

## FOC8-2: Sanitization and robustness

**Owner:** B: ________  **Depends on:** `common.py` (day 1); uses the stub until `embed()` lands

**Goal.** Find which cheap operations destroy hidden data, how completely (measured by bit error rate), and what each costs in accuracy.

**Why it matters.** Sanitization is Module 6's countermeasure for stegomalware, and here a defender applies it to a downloaded model. The same operations also erase legitimate watermarks: a dual-use trade-off to discuss. It connects to Dubin's "disarm" work (2309.03071).

**Tasks**
- [ ] `sanitize.py`. Each function takes a `state_dict` and returns one:
  - `to_fp16`: cast to half precision and back to float32.
  - `int8`: per-tensor symmetric quantize, then dequantize. Use `torch.ao.quantization` as a cross-check where it applies.
  - `prune(p)`: `torch.nn.utils.prune.l1_unstructured` at p = 10, 20, 30 and 50%.
  - `finetune`: one epoch on MNIST train with a small learning rate.
  - `lsb_noise(k)`: randomize the k lowest bits. This is the cheapest kind of disarm.
- [ ] **Cost column first** (doesn't need A): accuracy of the *clean* model after each operation.
- [ ] **Robustness:**
  - embed a known payload with the stub, or with `embed()` once it lands, at n ∈ {1, 4, 8, 16};
  - apply each operation, then extract;
  - compute the bit error rate (a BER near 0.5 means the payload was destroyed).
- [ ] Write hypotheses *before* running, then check them. For example: fp16 keeps only 10 mantissa bits, so it should wipe out any payload with n ≤ 13 completely.
- [ ] Produce a table of operation × n → bit error rate and change in accuracy, plus a heatmap.

**Deliverables:** `sanitize.py`, `results/sanitization.csv`, `figures/sanitization_heatmap.png`, and a summary paragraph.

**Done when:** every operation has a clean-model accuracy cost and a bit error rate for at least 3 values of n, reproducible with one command.

**Cite:** the Module 6 *Sanitization* and *Attacks Against Watermarks* slides; Dubin 2309.03071 and [ArielCyber/AI-MODEL-CDR](https://github.com/ArielCyber/AI-MODEL-CDR) (licensed research-only, so read and cite it; don't copy it); the PyTorch pruning and quantization docs; MaleficNet, for why error-correcting codes resist this.

**Talk slot (5–6 min):**
1. What sanitization is.
2. **Live demo:** apply the fp16 cast to A's model; the extracted payload is garbage, and accuracy is unchanged.
3. The results table.
4. Limits and costs, including that the same operations also kill legitimate watermarks.

**Next ticket:** repeat the robustness test with A's error-correcting code and show how much it buys back.

---

## FOC8-3: Detection (weight statistics and model scanners)

**Owner:** C: ________  **Depends on:** `common.py` (day 1); uses the stub until `embed()` lands

**Goal.** Find out whether a defender can tell a modified model from a clean one, using (a) simple statistics on the weights and (b) the off-the-shelf scanners that model hubs use.

**Why it matters.** This is Module 4's static analysis, signatures and their limits, and evasion. Models from public hubs are a software supply-chain dependency (Module 3). The contrast to draw: pickle files that run code on load (CWE-502) *are* caught by scanners, while data hidden in weights is not.

**Tasks**
- [ ] Install pinned versions of picklescan, ModelScan, fickling, safetensors and ClamAV, and record the versions.
- [ ] **What a normal model file looks like.** For a clean `net.pt`, record:
  - its structure with `unzip -l`;
  - `fickling` output on `data.pkl`;
  - its sha256 and byte entropy.

  Then save it as `.safetensors` and inspect the JSON header.
- [ ] **Scanner verdicts** for clean and modified models, in both `.pt` and `.safetensors`.
- [ ] **Positive control:** a pickle that runs a harmless call on load, such as printing a message. It proves the scanners work on code-in-pickle. Scan it only; never load it.
- [ ] **LSB statistics:**
  - per layer and per bit position k = 0…22: the fraction of 1s, a chi-square test against 0.5, and the entropy of the lowest byte;
  - first on the clean model, then at embedding rates of 1%, 10%, 50% and 100% of weights for n ∈ {1, 4, 8, 16};
  - find the rate or n at which a simple threshold starts to detect embedding.
- [ ] **Try a YARA rule** (Module 4) for an embedded *plaintext* payload. With n = 8, each payload byte lands in one byte of each weight, so a hex string with jumps such as `{ 48 ?? ?? ?? 65 ?? ?? ?? 6C }` might match. Verify this, and show that random or encrypted payloads, or n < 8, defeat the rule.

**Deliverables:** `detect.py`, `results/scanners.csv`, `results/lsb_stats.csv`, `figures/bit_balance.png` (clean vs. modified, per bit position), the YARA rule, and a summary paragraph.

**Done when:**
- the scanner table covers clean, modified and positive-control files in both formats;
- the statistics table names the detection threshold;
- the YARA experiment has a clear yes or no result.

**Cite:**
- Gilkarov & Dubin 2310.01969, and [ArielCyber/AI_Model_Steganalysis](https://github.com/ArielCyber/AI_Model_Steganalysis) as an optional baseline (licensed CC BY-NC-ND: run it and cite it, don't copy it);
- Model X-Ray 2409.19310;
- the documentation for picklescan, ModelScan and fickling, and Hugging Face's pickle-scanning page;
- the Module 4 *YARA Rules* slides.

**Talk slot (5–6 min):**
1. Anatomy of a model file.
2. **Live demo:** the scanners pass the modified model but flag the positive control.
3. The bit-balance plot.
4. The YARA attempt.
5. What defenders can and cannot see.
6. A one-minute conclusion for the whole team.

**Next ticket:** run the detectors on B's *sanitized* models to see whether sanitization also removes the statistical traces.

---

## Talk and document split

| Part | Who | Time |
|---|---|---|
| Intro: threat, setup, tools | A | ~1 min |
| Embedding and capacity | A | 5 min |
| Sanitization | B | 5–6 min |
| Detection | C | 5–6 min |
| Conclusions and limits | C | ~1 min |
| **Total** | | **18–20 min (sharp)** |

- **Supporting document** (about 3 pages, due **1–2 days before** the talk): each member turns their summary paragraph into about one page. Shared parts are the intro, the setup and tool versions, the AI-use statement, and the references.
- **Not "reinventing the wheel":** use PyTorch, picklescan, ModelScan, fickling and YARA as they are. Our contribution is the measurements on a real, small example and the security interpretation of those measurements.
