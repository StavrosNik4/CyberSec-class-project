# Hidden Data in AI Model Weights

A small, educational demonstration of **model-weight steganography**: hiding data inside the weights of a
trained neural network, and the defenses that remove it. Built for a University of Pavia *Foundations of
Cybersecurity* team assignment on a tiny MNIST classifier.

All payloads are **harmless** — short text or random bytes. No malware is used, stored, or executed.

## What it shows

**`src/embed_demo.ipynb` — embedding.** Hides the same payload in a network's weights two ways and compares
them:

- **LSB substitution** — overwrite the lowest byte(s) of successive weights. Simple and high-capacity; a
  depth sweep shows how writing deeper trades accuracy for capacity. No side information needed.
- **Value-mapping** — leave the weights untouched and store an *index map* pointing to weights that already
  match the payload. The model stays byte-for-byte identical (no accuracy loss), at the cost of shipping a
  map larger than the payload.

**`src/sanitization_demo.ipynb` — sanitization.** Embeds a watermark, then applies cheap defenses
(mantissa-bit randomization, int8 quantization) and measures how completely each destroys the hidden data
versus what it costs in model accuracy.

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
  embed_demo.ipynb          embedding demos (LSB + value-mapping)
  sanitization_demo.ipynb   sanitization / robustness demo
  net.pt                    trained CNN weights
  pretrained_fc.pt          trained fully-connected weights
requirements.txt
```
