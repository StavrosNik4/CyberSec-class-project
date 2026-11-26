# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Code for a University of Pavia *Foundations of Cybersecurity* (2026/27) team assignment, **Team 8 —
"Hidden data in AI model weights."** It is a defensive/educational demonstration of model-weight
steganography on a tiny MNIST network. The assignment has three parts; this repo currently holds two:

- **Part A — embedding & capacity:** `src/embed_demo.ipynb`
- **Part B — sanitization & robustness:** `src/sanitization_demo.ipynb`
- Part C (detection) is not implemented yet.

`TICKETS - Team 8.md` is the working brief (owners, interface contract, constraints). The model and MNIST
pipeline derive from the course Module 6 "Information Hiding" example by **Angelica Liguori** (upstream
`https://github.com/lucacav/foc`); reuse of that code must stay attributed.

**Payload rule (do not break):** demonstrations use only **harmless** payloads — short text, random bytes,
or payload *sizes* (e.g. EICAR's 68 bytes). No live malware, no binaries written to disk, no extractor/loader
that executes recovered bytes.

## Setup and running

Python **3.12 is required** (a pinned scanner dependency, modelscan, excludes 3.13). PyTorch is CPU-only.

```bash
# from the repo root
uv venv --python 3.12
uv pip install -r requirements.txt --torch-backend cpu   # or: pip install -r requirements.txt
source .venv/bin/activate
```

The deliverables are Jupyter notebooks, run top to bottom. There is **no build, lint, or test suite.**

`pyldpc` (MaleficNet's LDPC) has no wheel and builds from sdist needing numpy at build time; the single
`uv pip install -r requirements.txt` above works because uv co-resolves numpy, but a bare
`uv pip install pyldpc` into an empty env fails (install numpy first, or use `--no-build-isolation`).

```bash
# run a notebook headless (re-executes every cell in place)
jupyter nbconvert --to notebook --execute --inplace src/embed_demo.ipynb
```

MNIST auto-downloads to `src/data/` on first use. `.venv/`, `src/data/`, and `papers/` are gitignored.

## Two models, two saved weight files

Both notebooks share the same architectures but use different saved weights. Keep these straight:

| File | Class | Shape | ~Params | Baseline acc |
|---|---|---|---|---|
| `src/net.pt` | `NetCNN` (embed_demo) / `Net` (sanitization) | Module 6 CNN: conv ×3 → `fc1` 256→64 → `fc2` 64→8 → `out` 8→10 | ~73k | ~98.6% |
| `src/pretrained_fc.pt` | `NetFC` (embed_demo only) | `784 → 32 → 10` fully-connected | 25,450 | ~94.6% |

`net.pt` is a trained **`state_dict`** (a zip, not a pickled object); load with `Net(); load_state_dict(...)`.
The watermark/payload target layer in the CNN is **`fc2.weight`** (512 values).

## `embed_demo.ipynb` — embedding (Part A)

Self-contained. A `TARGET` variable switches the model: `"fc"` reloads/trains `pretrained_fc.pt` (trains ~3
epochs and saves it on first run if missing), `"pretrained"` loads the CNN from `net.pt`. Structure:

- **§2 Method A — LSB substitution** (`embed_payload`/`extract_payload`): packs each weight **big-endian**
  and overwrites its lowest `n_bytes` with payload bytes; end marked by escape `deadbeef`. `n_bytes=1` is
  invisible to accuracy; a depth sweep shows the capacity-vs-fidelity cliff (`n_bytes=3` → ~88%). No side
  information needed.
- **§3 Method B — Value-mapping** (`low_bytes`/`value_map_embed`/`value_map_extract`): does **not** change
  weights; it searches for weights whose low byte already equals each payload byte and stores their **index
  map** (the side information, larger than the payload). With matches found the model is byte-for-byte
  identical (accuracy == baseline exactly).
- **§4 Method C — MaleficNet** (`maleficnet_embed`/`maleficnet_extract`): spread-spectrum (CDMA). The payload
  is LDPC-encoded (via `pyldpc`, rate 1/2, with a 200-bit noise-estimation preamble) then spread across **all**
  weights as a tiny ±γ signal (`γ=1e-3`); extraction is **blind** (correlate with the key-seeded spreading
  codes, then LDPC-decode). Only side info is the integer **key**. A robustness cell applies prune / Gaussian
  noise / fp16 / int8 (via `_attack` + `prune`/`add_noise`/`to_fp16`/`to_int8`) and shows the payload still
  decodes — the method's whole point.
- **§5** compares all three on the same payload, with a measured **"survives fp16"** column (LSB ✗,
  value-mapping ✗, MaleficNet ✓).

Helpers: `get_w`/`set_w` flatten/restore all `.weight` tensors; `accuracy_sd(sd)` scores a `state_dict`
directly. All embedding runs on `copy.deepcopy(clean_sd)`, so **this notebook never modifies `net.pt` or
`pretrained_fc.pt`.**

## `sanitization_demo.ipynb` — sanitization (Part B)

A modified copy of the course "AI LSB" notebook (uses a global `device`; the usual course structure of
train/test/embed cells). It trains the CNN, embeds the watermark `"AI4SS"` into `fc2.weight` via LSB, then
in the **Sanitization Part** applies defenses to the watermarked model and tabulates accuracy, text BER, and
LSB-bit BER per method:

- `sanitize_flp_k(net, device, k_bits, only_target_layer=...)`: **randomizes the low `k_bits` mantissa bits**
  (float-precision reduction) via int32 bit-masking (`.view(torch.int32)`).
- `sanitize_qint8(net, device, ...)`: symmetric **int8 quantize → dequantize** (scale = max|w|/127).

The point: these cheap operations destroy the hidden payload (BER rises toward ~0.5) with little accuracy
cost — so the same operations also erase legitimate watermarks.

## Gotchas

- **`sanitization_demo.ipynb` overwrites `src/net.pt`.** Its training cell calls `train(..., path_model='net.pt')`,
  and `test()`/`train()` depend on a global `device`. Running it top-to-bottom retrains the CNN and
  **replaces the shipped `net.pt`** (different weights). If you only want to sanitize, load the existing
  `net.pt` instead of running the train cell; back up `net.pt` before re-running if the shipped weights matter.
- The two notebooks define a same-named-but-different model situation: `embed_demo` has `NetFC` + `NetCNN`;
  `sanitization_demo` has `Net` (== the CNN). `net.pt` fits the CNN/`Net`; `pretrained_fc.pt` fits `NetFC`.
- **Inherited course bug:** `torch.use_deterministic_algorithms = True` (an assignment, not a call) does not
  enable determinism. Present in `sanitization_demo.ipynb`; don't rely on it for reproducibility.
- `torch.load` unpickles input — load only local/course files; recent PyTorch defaults to `weights_only=True`,
  which is correct for these `state_dict`s.

## Attribution

Credit in borrowed work: the Module 6 example (Angelica Liguori); EvilModel (arXiv:2107.08590) for LSB byte
substitution; StegoNet (Liu et al., ACSAC 2020, §4.2.2) for value-mapping. Add sources to the team's
references as they are used.
