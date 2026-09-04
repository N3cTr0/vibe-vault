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

**More examples is NOT the main quality lever, whatever the author's notes say.** That claim
sent most of 09/04/2026 after the wrong variable: 1,000 → 10,000 moved benchmark recall *down*,
0.506 → 0.440, and the owner's voice was unaffected either way. The lever was
`slerp_weights` — see *What finally worked* at the end of this note. Raise the sample count only
after the distribution is right, and never as a first response to a model that cannot hear
somebody.

---

# Training it locally instead (09/04/2026)

*`tools/wake-training` in the repo: a `python:3.11-slim` container that runs the same pipeline.
Built and run end to end — it produces an `.onnx`. This section is what that proved.*

## The estimate was wrong twice, in the same direction

| said | basis | out by |
|---|---|---|
| "days rather than hours" | CPU-versus-GPU intuition | ~10x |
| "~8.1 hours" | measured positives, **guessed** the negative multiplier as 7x | ~3x |
| **~2.6 hours** | measured both paths | — |

`train.py` generates as many adversarial negatives as positives, at `tts_batch_size//7`. It is
very tempting to call that seven times the cost. On a CPU it is **1.8x**: batch size buys far
less here than on a GPU, and the `//7` is a VRAM limit rather than a statement about work.

Measured on the Ryzen 7 3700X, 8 cores / 16 threads:

| | per clip | 30,000 |
|---|---|---|
| positives, batch 50 | 0.100s | ~0.8 h |
| negatives, batch 7 | 0.180s | ~1.5 h |
| validation | | ~0.2 h |
| **generation** | | **~2.6 h** |

Augmentation is **not** slower locally — `AudioFeatures(device='cpu', ncpu=4)` means that pass is
CPU-bound on Colab too.

> **The lesson is the one this project keeps writing down.** Both wrong numbers came from
> reasoning about the machine instead of running it on the machine, and `bench.sh` takes two
> minutes. See [[Lessons Learned]].

## Three things Colab was hiding

The notebook silently depends on its base image for these. None is written down anywhere, and
all three will bite whoever rebuilds it after Colab next changes that image — **the pins exist
there by luck, not by decision.**

| | what happens | fix |
|---|---|---|
| `webrtcvad` | C extension, no manylinux wheel; a slim image ships no compiler, so pip tries to build and fails | `webrtcvad-wheels`, the same module prebuilt |
| `pyarrow` | `datasets` 2.14.6 subclasses `pa.PyExtensionType`, removed in 15 — so `import datasets` fails outright, not the parquet read | `pyarrow<15` |
| `/dev/shm` | Docker allows 64 MB; torch passes DataLoader tensors through it and the workers die **at the start of `--train_model`** with an error naming no cause | `--shm-size=2g` |

**The third is the one to remember.** It surfaced for free because the smoke test ran at 60
samples; at 30,000 it would have appeared two and a half hours in, after everything expensive
was already done. `run.sh` now checks `/dev/shm` in its first second and refuses.

## And the check lied a second time

The image build ends with an import assertion, written specifically to avoid fault 10 — it uses
submodules, never bare `openwakeword`, because from `/work` that name matches the *directory* as
an empty namespace package and passes while importing nothing.

**It then passed cleanly while `import datasets` was completely broken**, because it did not
import `datasets`.

Same failure, third disguise, and this time in a check written *by* the note warning about it. A
check is worth exactly what it imports and no more — the list is now the union of what
`train.py` and the data step actually use.

## Which to use

Local wins on everything except the first hour of setup: no disconnects, no runtime resets that
wipe `/content`, nothing to pay, and an environment that is a file rather than a browser tab.
Colab is still the faster wall-clock for a single run, and the notebook cells above stay
current for it.

---

# What finally worked, and what a day of measuring the wrong thing cost

*09/04/2026. Written after the fact, because the sequence is the lesson.*

## The fix

**`slerp_weights` defaults to `(0.5,)` and train.py never overrides it**, so every generated
clip is a 50/50 blend of two speakers: `g = slerp(emb0, emb1, 0.5)`. The midpoint of two voices
sits near the centre of speaker space, so the model never hears a *pure* speaker and never hears
an extreme one at all. Patching the default to `(0.0, 0.25, 0.5, 0.75, 1.0)` generates unblended
speakers and restores both ends of the range.

Measured against 50 recordings of the owner's real voice:

| | median | fires at 0.15 |
|---|---|---|
| before | **0.008** | 6/50 (12%) |
| after | **0.588** | 25/40 (63%) |

Held-out clips the model never saw: median 0.507, 5/10. The remaining misses **cluster in
blocks** rather than scattering — long runs of 0.7-0.9 alternating with runs near zero — so they
are delivery-dependent, not random. Ordinary speech scores 0.7-0.9; the deliberately awkward
recordings (muttered, turned away, distant) are what fail.

## What did not work, and is worth not repeating

**More samples.** 1,000 → 10,000 → 30,000 was the plan for hours. Ten times the data moved
benchmark recall from 0.506 to 0.440 — *down*. Sample count was never the constraint.

**`target_recall`.** Raised from 0.25 to 0.6 and described as "the one that matters most". It is
a warning threshold: `_select_best_model` filters candidates by false-positive rate and takes the
best recall among them, and `target_recall` only decides whether a log line prints.

**`target_false_positives_per_hour`.** This one does bind — it filters selection and doubles
`max_negative_weight` in sequences 2 and 3. Relaxing 0.2 → 2.0 moved recall 0.366 → 0.44. Real,
but small, and it addressed a symptom.

**Seeding the owner's own recordings.** 40 clips × 50 copies, 17% of the positives. It made the
model *worse on the clips it was trained on* (63% → 55%, median 0.588 → 0.215). Duplicates add
weight without adding information; 2,000 near-identical points crowded the positive set and the
selection turned conservative — false positives fell to 0.0, which is the tell. **Recording real
audio was the right instinct; fifty copies of forty utterances was the wrong way to use it.**
Worth retrying at ~5 copies, or with several hundred genuinely distinct recordings.

## The reason it took a day

**Every measurement was synthetic speech judged by a model trained on synthetic speech.** The
training benchmark uses piper clips; `wakescore` uses SAPI voices. Both reported the model was
fine while the owner's ordinary voice was not being heard at all.

Worse, the diagnostic in her own code was wrong:

```csharp
LastScore = score;                                  // every 80 ms chunk
Log.Write($"... (best {_wake.LastScore:0.00})");    // reported as the best
```

An utterance is logged only after the voice detector's trailing silence, by which point the
classifier's window holds that silence and the score has decayed to zero. Every declined
utterance logged `best 0.00` regardless of what it reached — which reads as *"her name scores
nothing at all"* and is a completely different claim from *"it scores 0.25 against a 0.30
threshold"*. Fixed in v0.52.3 as `PeakScore`.

**Fifty real recordings took ten minutes and settled in one command what four retrains could
not.** `positive_train` is just a folder of WAVs and `EarsTest scorerecordings` reads any folder
of them, so both directions were always available. The right first question was not *"how do we
train a better model"* but *"what does the current model score on the voice that has to work"*.

> Four separate things this week reported success for a question they had not asked: a check
> that imported a directory, a guard that tested existence when it meant completeness, an exit
> code belonging to `tail`, and a score read after it had decayed. See [[Lessons Learned]].
