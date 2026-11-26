# Hidden Data in AI Model Weights

A small, educational demonstration of **model-weight steganography**: hiding data inside the weights of a
trained neural network, and the defenses that remove it. Built for a University of Pavia *Foundations of
Cybersecurity* team assignment on a tiny MNIST classifier.

All payloads are **harmless** — short text or random bytes. No malware is used, stored, or executed.

## What it shows

**`src/embed_demo.ipynb` — embedding.** Hides the same payload in a network's weights three ways and compares
them:

- **LSB substitution** — overwrite the lowest byte(s) of successive weights. Simple and high-capacity; a
  depth sweep shows how writing deeper trades accuracy for capacity. No side information needed.
- **Value-mapping** — leave the weights untouched and store an *index map* pointing to weights that already
  match the payload. The model stays byte-for-byte identical (no accuracy loss), at the cost of shipping a
  map larger than the payload.
- **MaleficNet (spread-spectrum + LDPC)** — spread the payload across *all* weights as a tiny signal,
  protected by error-correcting codes. Each weight barely moves, and the payload **survives model
  modifications** (pruning, noise, quantization) that destroy the other two methods.

**`src/sanitization_demo.ipynb` — sanitization.** Embeds a watermark, then applies three cheap "disarm"
defenses and measures how completely each destroys the hidden data versus what it costs in model accuracy:

- **Full LSB prevention** — replace *all* mantissa bits with random bits. Very high prevention rate, but
  the most disruptive to the model.
- **K-LSB prevention** — replace only the lowest *k* mantissa bits. Preserves nearly all of the model's
  accuracy; the right choice of *k* matters.
- **Model quantization (int8)** — convert weights from 32-bit float to 8-bit integer. Rounding erases the
  LSB-hidden data as a side effect: near-100% prevention with accuracy well preserved, at the cost of some
  precision loss.

## Setup

Requires Python 3.12 (CPU-only PyTorch).

```bash
uv venv --python 3.12
uv pip install -r requirements.txt --torch-backend cpu   # or: pip install -r requirements.txt
source .venv/bin/activate
```

## Running

Open either notebook and run all cells top to bottom (MNIST downloads automatically on first use):

```bash
jupyter notebook src/embed_demo.ipynb
```

## Layout

```
src/
  embed_demo.ipynb          embedding demos (LSB, value-mapping, MaleficNet)
  sanitization_demo.ipynb   sanitization / robustness demo
  net.pt                    trained CNN weights
  pretrained_fc.pt          trained fully-connected weights
requirements.txt
```

## References

1. Z. Wang, C. Liu, X. Cui, "EvilModel: Hiding Malware Inside of Neural Network Models", 2021.
2. Z. Wang et al., "EvilModel 2.0: Bringing Neural Network Models into Malware Attacks", 2023.
3. D. Hitaj et al., "MaleficNet: Hiding Malware into Deep Neural Networks Using Spread-Spectrum Channel Coding", 2022.
4. T. Liu et al., "StegoNet: Turn Deep Neural Network into a Stegomalware", 2020.
5. Y. Nagai, Y. Uchida, S. Sakazawa, S. Satoh, "Digital Watermarking for Deep Neural Networks", IJMIR, 2018.
6. L. Zhang, D. Li, O. Jurečková, M. Stamp, "Steganographic Capacity of Deep Learning Models", 2023.
7. D. Gilkarov, R. Dubin, "Steganalysis of AI Models LSB Attacks", 2023.
8. R. Dubin, "Disarming Steganography Attacks Inside Neural Network Models", 2023.
9. D. Gilkarov, R. Dubin, "Model X-Ray: Detection of Hidden Malware in AI Model Weights using Few Shot Learning", 2024.
