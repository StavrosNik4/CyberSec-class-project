# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is here

`src/` contains the AI watermarking example from Module 6 ("Information Hiding") of the UniPV *Foundations of Cybersecurity* course (upstream `https://github.com/lucacav/foc`, `Module 6 - Information Hiding/Examples/AI/`). The code belongs to **Angelica Liguori**, as credited in the course README. Any reuse of it must be attributed.

- `AI LSB.ipynb`: trains a small CNN on MNIST, saves it, embeds a text watermark (`"AI4SS"`) into the weights, re-tests accuracy, and extracts the watermark.
- `AI LSB dropout.ipynb`: identical except for one change, `nn.Dropout(p=0.5)` after `fc1`. Dropout has no parameters, so the `state_dict` keys are the same in both versions.
- `net.pt`: a trained **`state_dict`**, not a pickled model object. Load it with `net = Net(); net.load_state_dict(torch.load("net.pt"))`. It is a zip containing `archive/data.pkl` and the tensor blobs. It holds about 73k float32 parameters, about 290 KB.

There is no build or test system and no `requirements.txt` yet. The notebooks need `torch`, `torchvision`, `numpy`, `matplotlib` and Jupyter, none of which are installed system-wide.

## Model (`Net`)

The input is a 1×28×28 image. The layers are:

| Parameter | Shape | Note |
|---|---|---|
| `embedder.0` | Conv 1→32, k3, s2 | |
| `embedder.2` | Conv 32→64, k3, s2 | |
| `embedder.4` | Conv 64→64, k3, s2 | spatial size 28→13→6→2 |
| `fc1` | 256→64 | |
| `fc2` | 64→8 | **the watermark target**: `fc2.weight` holds 512 values |
| `out` | 8→10 | |

ReLU sits between the layers. Weights are initialised with Kaiming normal (convs) and N(0, 0.01) (linear layers), and biases with 0.

Training uses Adam with lr 1e-3, CrossEntropy, batch 64, 20 epochs and seed 101. MNIST is downloaded to `./data`.

## How the embedding works (cell 4)

- `float_to_bin(x)` turns a float32 into a 32-character bit string using `np.float32(x).tobytes()`.
- `add_watermark_lsb(net, watermark)`:
  - turns each ASCII character into 8 bits;
  - for each element of `fc2.weight` in `np.ndindex` order, replaces the **last character** of that bit string with the next watermark bit;
  - writes the changed values back with `param.copy_`.
- `extract_watermark(net, length)` reads the last character of every `fc2.weight` element, groups the bits into bytes, and truncates the result to `length` characters.

One bit per weight over 512 weights gives a capacity of at most 64 bytes.

The code loops over elements in pure Python and string-converts every single float. That is fine for 512 weights but too slow for whole-model use. Vectorised NumPy bit operations would be needed for that.

## Known issues (verified)

- **The edit does not change the least significant bit of the float on little-endian machines (x86/ARM).** `tobytes()` uses native little-endian byte order, so the last character of the string is bit 24 of the IEEE-754 float, which is an **exponent** bit, not mantissa bit 0.
  - The round trip `bin_to_float(float_to_bin(x))` is still consistent, because it reads with `to_bytes(..., 'big')` followed by a native `frombuffer`. That is why extraction works.
  - Each changed weight is multiplied or divided by 4. This was reproduced with `struct`, and it matches the notebook's own printed output, e.g. `3.5238e-02 → 8.8095e-03` and `2.5852e-01 → 1.0341e+00`.
  - This explains the accuracy drop recorded in the notebook, **98.58% → 94.68%**. The dropout version drops only 98.83% → 98.75%. That comes from a different training run, and why it drops less hasn't been checked.
  - A true mantissa-LSB edit needs either big-endian packing (`np.float32(x).newbyteorder('>')`, or `struct.pack('>f', x)`) or integer bit operations on `.view(np.uint32)`.
- `torch.use_deterministic_algorithms = True` (cell 1) **overwrites** the function instead of calling it, so determinism is not actually enabled. The call should be `torch.use_deterministic_algorithms(True)`.
- `train()` and `test()` use the global `device` defined in cell 11, so they can't be imported as-is.
- Running the training cell (cell 12) **overwrites `net.pt`**, because `path_model='net.pt'`. Retraining with a different seed or on GPU gives different weights from the shipped file.
- `add_watermark_lsb` changes `net` in place. Reload `net.pt` before each new experiment.

## Safety

`torch.load` unpickles its input. Load only files created locally or taken from the course. Recent PyTorch versions default to `weights_only=True`, which is fine for this `state_dict`.
