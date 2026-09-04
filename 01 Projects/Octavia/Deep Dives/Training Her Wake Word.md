---
project: Octavia
tags: [octavia, deep-dive, wake-word]
---

# Training Her Wake Word

*How to train `hey_octavia.onnx` in Google Colab, and the ten things that are broken between the openWakeWord notebook and a 2026 Colab runtime. Written 09/04/2026 while the run was going, because none of it is written down anywhere else and a lost runtime means rediscovering all of it.*

See [[Roadmap]] Stage 17 for why the wake word matters: it moves the always-on cost from Whisper's 1.6 GB to about 3.7 MB, and nothing said in a listening room is transcribed until she is addressed.

## Why Colab at all

**Corrected 09/04/2026.** This section used to say the GPU was the reason. The GPU is the
*second* reason, and on this machine it is not even the binding one.

**1. The phonemiser has no Windows build.** `piper-phonemize` and `piper-phonemize-cross` ship
`manylinux` and `macosx` wheels only — no `win_amd64` at any Python version — and neither has an
sdist. So on Windows pip has nothing to install and nothing to compile. This is the same brick
wall as Colab's Python 3.13, arriving from a different direction, and the upstream notebook says
so in a comment that is easy to read past: *"currently only supports linux systems."*

**2. Any Linux removes that, and then the GPU binds.** Under WSL2 or a container the manylinux
wheels install and the stack works. But sample generation is TTS inference — a T4 runs a batch of
fifty utterances at once, a CPU runs a handful. On the Ryzen 7 3700X (8 cores, 16 threads, 32 GB)
30,000 samples is **days rather than hours**. Possible; a bad trade.

**3. The GT 730 cannot take over.** Kepler, compute 3.5, 2 GB. CUDA 12 dropped it and modern
PyTorch needs sm_50 or better, so `torch.cuda.is_available()` is `False` whatever is installed.
It is not a slow GPU for this, it is not a usable one.

**Not everything wants the GPU, though.** `train.py` builds `AudioFeatures(device='cpu', ncpu=4)`
with no `inference_framework`, so the whole `--augment_clips` feature pass is CPU-bound *by
design* — the GPU idles through it on Colab too. Worth knowing before assuming a faster card
would fix a slow run.

### Docker is the better local option, and most of the ten faults vanish in it

Half the mess in this note exists **only because Colab forces Python 3.13 on a stack that needs
3.11**. Give the stack a 3.11 base image and those problems do not need solving:

| fault | in Colab | in a `python:3.11` container |
|---|---|---|
| 1 — no cp313 phonemiser wheel | the wall everything follows from | gone; 3.11 is the interpreter |
| 3 — a runtime reset wipes `/content` | every cell must be idempotent | gone; a volume is a volume |
| 4 — `onnx_tf` swaps numpy live | forced a restart mid-notebook | gone; never installed |
| 7 — `MPLBACKEND` inherited | subprocess cannot start matplotlib | gone; nothing to inherit |
| 2, 9 — `pkg_resources` missing | `uv venv` ships no setuptools | install `setuptools<82` once |

What still needs handling: `scipy<1.17` (fault 8, `acoustics` imports the removed
`sph_harm`), the `torchaudio.set_audio_backend` stub (fault 6), and AudioSet having moved to
parquet (fault 5). Three, not ten.

**So the trade is not Docker versus Colab on difficulty — Docker is easier.** It is CPU speed
versus about ten dollars. Measure before choosing: generate fifty samples in the container, time
it, multiply. Five minutes of work turns the guess into a number.

**The notebook:** `https://colab.research.google.com/drive/1q1oe2zOyZp7UsB3jJiQ1IFn8z5YfjwEb` — the author's "minimal development experience" version. The full one is `notebooks/automatic_model_training.ipynb` in `github.com/dscripka/openWakeWord`, which is the same project `WakeWordStore` already downloads its models from.

**Copy to Drive first**, or you are looking at a read-only copy that cannot write files.
## The wall everything else follows from

**Colab runs Python 3.13. The Piper phonemiser has no build for it, and no source to compile.**

| `piper-phonemize` 1.1.0 | wheels for cp39, cp310, **cp311** — no sdist |
|---|---|
| `piper-phonemize-cross` 1.2.1 | cp37–**cp312** — no sdist |
| Colab | **cp313** |

Both are 2024 releases and neither has moved since. This is not a pin you can bump: pip has nothing to install and nothing to build. `ERROR: ... (from versions: none)` means exactly that.

**So sample generation runs in its own Python 3.11 environment**, built with `uv`, and everything that imports the generator runs there too. That last part matters — `train.py --generate_clips` does `from generate_samples import generate_samples` **in-process**, so the whole training stack has to live in the 3.11 environment rather than in the notebook kernel.

## The ten faults, and what each one is

Every one of these is the same story in a different costume: a package pinned in 2023 meeting a dependency from 2026.

| 1 | **`piper-phonemize` has no cp313 build.** The wall above. Fix: a `uv venv --python 3.11`. |
|---|---|
| 2 | **`webrtcvad` imports `pkg_resources`**, which `uv venv` does not ship (unlike `python -m venv`). It uses it for **one thing** — setting `__version__`, which nothing reads. Fix: install `setuptools<82`, or patch the import out. |
| 3 | **A runtime reset wipes `/content`.** Colab recycles idle VMs, taking the clone, the 16 GB download and the venv. Fix: every cell idempotent, guarded on what it actually needs. |
| 4 | **`onnx2tf` swaps numpy under a live kernel.** It replaces numpy while numpy is already imported, so 2.1.3's C extensions run against 2.2.6's Python files. Surfaces as `AttributeError: 'numpy.ufunc' object has no attribute '__module__'`, which looks like a data problem and is not. Fix: **Runtime → Restart session** after cell 2, then re-run it. |
| 5 | **AudioSet moved from `.tar` to parquet.** `data/bal_train09.tar` 404s; it is now `data/bal_train/09.parquet`. And `datasets==2.14.6` cannot read it — its parquet builder throws `TypeError: must be called with a dataclass type or instance` on Python 3.13. Fix: read the parquet directly with `pyarrow` and decode with `soundfile`. |
| 6 | **`torch-audiomentations` 0.11.0 calls `torchaudio.set_audio_backend()`**, removed in torchaudio 2.x. Fix: patch the call — it is a configuration no-op now. |
| 7 | **`MPLBACKEND` is inherited from Colab.** The kernel exports `module://matplotlib_inline.backend_inline`, which exists only in the kernel; a subprocess inherits it and matplotlib refuses to start. Fix: `MPLBACKEND=Agg` in the subprocess environment. |
| 8 | **`acoustics` 0.2.6 imports `scipy.special.sph_harm`**, removed in SciPy 1.17. Fix: pin `scipy<1.17`. |
| 9 | **`pronouncing` also imports `pkg_resources`.** Same as (2), different package. |
| 10 | **`onnxruntime` missing** after `pip install -e ./openwakeword --no-deps`, and worse — an import check for it *lied*. |

### The check that lied, which is the one worth remembering

`import openwakeword` with `cwd=/content` finds the **directory** `/content/openwakeword` and treats it as an empty namespace package. It imports nothing, succeeds, and proves nothing. `train.py` resolves the real package and dies immediately on `openwakeword.vad`.

**A check must import the submodules the target actually imports**, not the bare package name — a namespace package will happily satisfy any import of a directory that happens to share its name.

> Two of these — (4) and (10) — were found *after* a check reported everything fine. Both times the check was testing a weaker proposition than the one that mattered. That is the same failure this project keeps writing down in [[Lessons Learned]], arriving from a completely different direction.

## The traps that are not dependency rot

**A guard that checks existence when it means completeness.** The notebook does `if not os.path.exists("./mit_rirs"): os.mkdir(...)` and then fills it. When the fill crashes, the directory survives — so every later run skips the block and leaves you with no impulse responses and no error. The same shape bit the voice model download: a truncated `wget` leaves a small file that `os.path.exists` accepts happily.

**Check sizes, or check contents, and never bare existence** for anything that arrives over a network.

## The working cells

Cell 1 verifies the phrase, cell 2 downloads the data, cell 2b fixes what cell 2 cannot, cell 3 trains. Full text in [[Wake Word Colab Cells]].

**Order:** cell 1 → cell 2 → **restart the runtime** → cell 2 → cell 2b → cell 3.

Cell 2 is run twice on purpose: the first time installs everything and dies on the numpy swap, the restart clears it, the second pass is fast because everything is already installed.

## Afterwards

The output is `my_custom_model/hey_octavia.onnx`. **Take the `.onnx`, not the `.tflite`** — she uses ONNX Runtime, and the tflite conversion needs `onnx2tf` fighting an onnx version conflict for a file that is never read.

**The run ends in a traceback, and that is fine.** After it prints `Saving ONNX mode as ...`
`train.py` goes straight on to the tflite conversion and dies on
`ModuleNotFoundError: No module named 'onnx_tf'`. The `.onnx` is already written by then. Do not
chase this — installing `onnx_tf` only buys a file she never opens. Check the file listing, not
the exit code.

### What the first run actually produced (09/04/2026)

| Accuracy | 0.748 |
|---|---|
| Recall | **0.506** |
| False positives per hour | 0.177 |

**Recall 0.51 means it misses about half the times the phrase is said.** Training sequences 2 and 3
both log *"Increasing weight on negative examples to reduce false positives"* — it bought a quiet
model by making a deaf one. That trade is the default, and at 1,000 samples there is not enough
positive signal to survive it.

So: 1,000 samples proves the pipeline end to end and is not worth living with. The fix is more
examples, not a lower threshold — dropping `WakeThreshold` on a model with 0.51 recall raises the
false positives without recovering the misses.

### Wiring one up

1. Drop it in `data\models\wake\`.
2. Set `WakePhrase` to `Hey Octavia` — in the settings window, or in `config.json`. `WakeWordStore` resolves any phrase that is not one of openWakeWord's own as a filename, so nothing else changes.
3. **Measure it before trusting it.** `hey jarvis` was measured at silence **0.000**, the phrase **0.949**, *"turn the kitchen light off please"* **0.000**, the phrase again **0.999**. Anything much worse than that is worth retraining with more examples before living with it.
4. **Then tune `WakeThreshold`.** 0.5 is openWakeWord's generic guidance, not an answer for this room. Her log carries the best score of every utterance it declined, which is exactly the data needed.

**Two words, not one.** *"Hey Octavia"* has a far stronger acoustic signature than *"Octavia"*, and that was the single biggest factor in model quality across every configuration the author tested. A one-word wake phrase fires on everything.

**More examples is the main quality lever.** The default 1,000 produces a working model; 30,000–50,000 is where it gets good. That costs hours rather than one, so it is worth doing once the 1,000-sample model has proved the pipeline works end to end.
