---
project: Octavia
tags: [octavia, deep-dive, wake-word]
---

# Wake Word Colab Cells

*The four cells that actually run, as pasted on 09/04/2026. Why each change was needed is in [[Training Her Wake Word]] — this note is the copy-paste.*

**Order, as first run:** cell 1 → cell 2 → **Runtime → Restart session** → cell 2 again → cell 2b → cell 3.

> **Building a new notebook? Skip to [Building the notebook fresh](#building-the-notebook-fresh-09042026) at the end.** The restart is gone there: it was caused by three installs that exist only for the tflite conversion, which she does not use.

Cell 2 runs twice on purpose. The first pass installs everything and dies when `onnx2tf` swaps numpy under the live kernel; the restart clears that; the second pass is quick because everything is already installed.

Every cell is safe to re-run.

## Cell 1 — verify the phrase

Replaces the notebook's cell 1 entirely. Builds the Python 3.11 environment the phonemiser needs, then generates one clip so you can hear it.

```python
# @title 1. Test wake word generation { display-mode: "form" }
import os, glob, subprocess
from IPython.display import Audio, display

ROOT = "/content"
GEN = f"{ROOT}/piper-sample-generator"
PY = f"{ROOT}/piper311/bin/python"
MODEL = f"{GEN}/models/en_US-libritts_r-medium.pt"
MODEL_URL = ("https://github.com/rhasspy/piper-sample-generator/releases/"
             "download/v2.0.0/en_US-libritts_r-medium.pt")

target_word = 'hey octavia'  # @param {type:"string"}


def sh(cmd):
    r = subprocess.run(cmd, shell=True, cwd=ROOT, capture_output=True, text=True)
    if r.returncode:
        print(f"FAILED: {cmd}\n{r.stdout[-800:]}\n{r.stderr[-800:]}")
    return r.returncode == 0


if not os.path.exists(f"{GEN}/generate_samples.py"):
    print("cloning the generator...")
    sh(f"git clone -q https://github.com/rhasspy/piper-sample-generator {GEN}")
    sh(f"cd {GEN} && git checkout -q 213d4d5")

os.makedirs(f"{GEN}/models", exist_ok=True)
if not os.path.exists(MODEL) or os.path.getsize(MODEL) < 10_000_000:
    print("downloading the voice model...")
    sh(f"wget -q -O {MODEL} {MODEL_URL}")

sh("pip -q install uv")

# Python 3.11: the phonemiser has no build for Colab's 3.13 and no source to compile.
if not os.path.exists(PY):
    print("building a 3.11 environment (slow, one-off)...")
    sh(f"uv venv --python 3.11 {ROOT}/piper311")
    sh(f"uv pip install -q --python {PY} torch==2.5.0 torchaudio==2.5.0 "
       f"--index-url https://download.pytorch.org/whl/cu121")
    sh(f"uv pip install -q --python {PY} piper-phonemize==1.1.0 "
       f"audiomentations==0.33.0 'numpy<2' webrtcvad")

# webrtcvad imports pkg_resources solely to set __version__, which nothing reads.
for vad in glob.glob(f"{ROOT}/piper311/lib/python*/site-packages/webrtcvad.py"):
    src = open(vad).read()
    if "pkg_resources" in src:
        src = src.replace("import pkg_resources\n", "")
        src = src.replace("__version__ = pkg_resources.get_distribution('webrtcvad').version",
                          '__version__ = "2.0.10"')
        open(vad, "w").write(src)
        print("patched", vad)

ok = subprocess.run([PY, "-c", "import torch, webrtcvad, piper_phonemize; print('imports ok')"],
                    capture_output=True, text=True)
print("generator:", os.path.exists(f"{GEN}/generate_samples.py"),
      "| model:", os.path.exists(MODEL) and f"{os.path.getsize(MODEL)//1_000_000} MB",
      "|", ok.stdout.strip() or ok.stderr.strip()[-300:])

r = subprocess.run([PY, "generate_samples.py", target_word,
                    "--max-samples", "1", "--batch-size", "1",
                    "--length-scales", "1.1",
                    "--noise-scales", "0.7", "--noise-scale-ws", "0.7",
                    "--output-dir", ROOT],
                   cwd=GEN, capture_output=True, text=True)
print("exit:", r.returncode)
if r.returncode:
    print(r.stdout[-2000:])
    print("STDERR:\n", r.stderr[-2000:])
else:
    wavs = sorted(f for f in os.listdir(ROOT) if f.endswith(".wav"))
    print("wrote:", wavs)
    display(Audio(f"{ROOT}/{wavs[-1]}", autoplay=True))

## Cell 2 — download the data

The notebook's own cell 2, with **three lines removed** — the `sys.path.append("piper-sample-generator/")` and `from generate_samples import generate_samples` block. Those run in the 3.13 kernel, where the phonemiser cannot load, and nothing in the cell uses them.

An install check is appended, because `!pip install` failures do not stop a cell and a missing package otherwise surfaces three cells later as something unrelated.

```python
# ... the notebook's cell 2 verbatim, minus the generate_samples import ...

# --- did everything actually install? pip failures do not stop a cell ---
import importlib
print("\n--- install check ---")
for m in ["datasets", "scipy", "mutagen", "torchinfo", "torchmetrics", "speechbrain",
          "audiomentations", "torch_audiomentations", "acoustics", "onnxruntime",
          "onnx", "pronouncing", "openwakeword"]:
    try:
        importlib.import_module(m)
        print(f"  ok    {m}")
    except Exception as e:
        print(f"  FAIL  {m}: {type(e).__name__}: {e}")
```

Expect `torch_audiomentations` to FAIL here — cell 2b fixes it.

## Cell 2b — what cell 2 cannot do

New cell. Two unrelated repairs: the torchaudio API removal, and AudioSet having moved to parquet.

```python
# --- 2b. Two things cell 2 could not do ---
import os, io, numpy as np, scipy.io.wavfile, soundfile as sf, librosa, torchaudio
import pyarrow.parquet as pq
from tqdm import tqdm

# 1. torch-audiomentations 0.11.0 calls torchaudio.set_audio_backend(), removed in
#    torchaudio 2.x. It is a configuration no-op now, so stubbing it is enough.
if not hasattr(torchaudio, "set_audio_backend"):
    torchaudio.set_audio_backend = lambda *a, **k: None
    print("stubbed torchaudio.set_audio_backend")
import torch_audiomentations
print("torch_audiomentations ok")

# 2. AudioSet moved from .tar to parquet, so the notebook's URL 404s, and
#    datasets==2.14.6 cannot read parquet on Python 3.13. pyarrow can.
out  = "./audioset_16k"
path = "/content/audioset_09.parquet"
url  = "https://huggingface.co/datasets/agkphysics/AudioSet/resolve/main/data/bal_train/09.parquet"

if not os.path.exists(path) or os.path.getsize(path) < 100_000_000:
    print("downloading the shard...")
    os.system(f"wget -q -O {path} {url}")
print("parquet:", os.path.getsize(path) // 1_000_000, "MB")

pf = pq.ParquetFile(path)
print("columns:", pf.schema_arrow.names)

os.makedirs(out, exist_ok=True)
written, LIMIT = 0, 2000
for batch in pf.iter_batches(batch_size=50):
    if written >= LIMIT:
        break
    for cell in batch.column("audio"):
        if written >= LIMIT:
            break
        rec = cell.as_py()
        try:
            data, sr = sf.read(io.BytesIO(rec["bytes"]), dtype="float32")
        except Exception:
            continue
        if data.ndim > 1:
            data = data.mean(axis=1)
        if sr != 16000:
            data = librosa.resample(data, orig_sr=sr, target_sr=16000)
        scipy.io.wavfile.write(os.path.join(out, f"audioset_{written:05d}.wav"),
                               16000, (data * 32767).astype(np.int16))
        written += 1

print("audioset_16k:", len(os.listdir(out)), "files")
print("mit_rirs:", len(os.listdir("./mit_rirs")), "files")
print("fma:", len(os.listdir("./fma")), "files")
```

Expected afterwards: **500** audioset clips (that shard holds 500), **270** impulse responses, **120** music clips.

## Cell 3 — train (the first run, at 1,000 samples)

Replaces the notebook's cell 3. Every step runs in the 3.11 environment, because `--generate_clips` imports the generator in-process. The tflite conversion and auto-download are dropped — she uses ONNX.

```python
# @title 3. Train the Model  { display-mode: "form" }
import os, glob, subprocess, yaml

ROOT = "/content"
PY   = f"{ROOT}/piper311/bin/python"

target_word = 'hey octavia' # @param {type:"string"}
number_of_examples = 1000 # @param {type:"slider", min:100, max:50000, step:50}
number_of_training_steps = 10000  # @param {type:"slider", min:0, max:50000, step:100}
false_activation_penalty = 1500  # @param {type:"slider", min:100, max:5000, step:50}

# Colab exports MPLBACKEND=module://matplotlib_inline..., which only exists in the kernel.
# A subprocess inherits it and matplotlib refuses to start. Agg needs no display.
ENV = {**os.environ, "MPLBACKEND": "Agg",
       "PYTHONPATH": f"{ROOT}/piper-sample-generator"}

config = yaml.load(open("openwakeword/examples/custom_model.yml").read(), yaml.Loader)
config["target_phrase"] = [target_word]
config["model_name"] = config["target_phrase"][0].replace(" ", "_")
config["n_samples"] = number_of_examples
config["n_samples_val"] = max(500, number_of_examples // 10)
config["steps"] = number_of_training_steps
config["target_accuracy"] = 0.5
config["target_recall"] = 0.25
config["output_dir"] = "./my_custom_model"
config["max_negative_weight"] = false_activation_penalty
config["background_paths"] = ['./audioset_16k', './fma']
config["false_positive_validation_data_path"] = "validation_set_features.npy"
config["feature_data_files"] = {"ACAV100M_sample": "openwakeword_features_ACAV100M_2000_hrs_16bit.npy"}
config["piper_sample_generator_path"] = f"{ROOT}/piper-sample-generator"

with open('my_model.yaml', 'w') as f:
    yaml.dump(config, f)
print("model_name:", config["model_name"])

def sh(cmd):
    r = subprocess.run(cmd, shell=True, cwd=ROOT, capture_output=True, text=True)
    if r.returncode:
        print(f"FAILED: {cmd}\n{r.stdout[-800:]}\n{r.stderr[-800:]}")
    return r.returncode == 0

print("preparing the 3.11 environment...")
# setuptools: pronouncing still imports pkg_resources, and uv venvs ship neither.
# scipy pinned: acoustics 0.2.6 imports scipy.special.sph_harm, removed in 1.17.
sh(f"uv pip install -q --python {PY} 'setuptools<82' torchinfo torchmetrics pyyaml "
   f"'scipy<1.17' tqdm mutagen speechbrain==0.5.14 acoustics==0.2.6 pronouncing==0.2.0 "
   f"deep-phonemizer==0.0.19 torch-audiomentations==0.11.0 onnx onnxruntime matplotlib")
sh(f"uv pip install -q --python {PY} -e ./openwakeword --no-deps")

for f in glob.glob(f"{ROOT}/piper311/lib/python*/site-packages/torch_audiomentations/**/*.py",
                   recursive=True):
    src = open(f).read()
    if "set_audio_backend" in src and "# patched" not in src:
        open(f, "w").write(src.replace("torchaudio.set_audio_backend",
                                       "getattr(torchaudio, 'set_audio_backend', lambda *a, **k: None)  # patched\n#"))
        print("patched", os.path.basename(f))

# Exactly what train.py imports. Submodules, not the bare package: with cwd=/content,
# `import openwakeword` finds the *directory* as an empty namespace package and succeeds
# without importing anything.
check = ("import torch, torchinfo, torchmetrics, scipy, yaml, numpy, tqdm; "
         "import audiomentations, torch_audiomentations, speechbrain, acoustics, pronouncing; "
         "from openwakeword.vad import VAD; "
         "from openwakeword.data import generate_adversarial_texts, augment_clips, mmap_batch_generator; "
         "from openwakeword.utils import compute_features_from_generator, AudioFeatures; "
         "from generate_samples import generate_samples; print('ok')")
ok = subprocess.run([PY, "-c", check], cwd=f"{ROOT}/openwakeword", env=ENV,
                    capture_output=True, text=True)
print("stack:", ok.stdout.strip() if ok.returncode == 0 else ok.stderr[-900:])

if ok.returncode == 0:
    for step in ["--generate_clips", "--augment_clips", "--train_model"]:
        print(f"\n=== {step} ===")
        p = subprocess.Popen([PY, "openwakeword/openwakeword/train.py",
                              "--training_config", "my_model.yaml", step],
                             cwd=ROOT, env=ENV,
                             stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
        for line in p.stdout:
            print(line, end="")
        if p.wait():
            print(f"*** {step} failed ***")
            break

    print("\nmy_custom_model:",
          os.listdir("./my_custom_model") if os.path.exists("./my_custom_model") else "not created")
else:
    print("not starting the run -- fix the stack first")
```

The run does not start unless `stack:` prints `ok` — there is no point spending an hour to find a missing import.

## Cell 3, the retrain — 30,000 samples, split into four

**Written 09/04/2026, after the 1,000-sample model scored 0.00 on the owner's natural voice.**
Six attempts at nothing at all, then 0.82 the moment he pitched it up. That is not a threshold
problem — no threshold reaches zero — it is a model that was told twice over to prefer silence
to hearing and had too little data to manage both:

| setting | was | now | why |
|---|---|---|---|
| `n_samples` | 1000 | **30000** | 1,000 leaves no room to satisfy recall and quiet at once |
| `target_recall` | 0.25 | **0.6** | 0.25 says *missing three quarters is acceptable*; it obliged |
| `max_negative_weight` | 1500 | **500** | a heavy false-alarm penalty bought quiet by going deaf |
| `steps` | 10000 | **20000** | thirty times the data wants more passes over it |

The first three are the fix. `steps` is orthogonal — capacity to fit, not what to prefer — and
is cheap: training was about nine minutes at 10,000.

**Split into four cells because the run is now hours, not twenty-five minutes.** Generation
scales roughly linearly with `n_samples`, and last time `--train_model` reported failure on the
tflite conversion *after* the ONNX was already written. One long cell redoes everything.

Cells 1, 2 and 2b are unchanged. Order is still 1 → 2 → **restart** → 2 → 2b → 3a → 3b → 3c → 3d.

> Splitting the cells protects against a failed *step*, not a runtime reset. If Colab recycles
> the VM, `/content` goes with it and you start again from cell 2 — fault 3, unchanged.

### 3a — config and environment

```python
# @title 3a. Config and environment  { display-mode: "form" }
import os, glob, subprocess, yaml

ROOT = "/content"
PY   = f"{ROOT}/piper311/bin/python"

target_word = 'hey octavia' # @param {type:"string"}

# --- what changed, and why ---------------------------------------------------
# 1,000 examples at target_recall 0.25 with a penalty of 1500 produced a model
# that scored 0.00 on a deep male voice.
number_of_examples       = 30000   # was 1000
recall_target            = 0.6     # was 0.25   <- the one that matters most
false_activation_penalty = 500     # was 1500
number_of_training_steps = 20000   # was 10000; 30x the data wants more passes
# -----------------------------------------------------------------------------

# Colab exports MPLBACKEND=module://matplotlib_inline..., which only exists in the
# kernel. A subprocess inherits it and matplotlib refuses to start. Agg needs no display.
ENV = {**os.environ, "MPLBACKEND": "Agg",
       "PYTHONPATH": f"{ROOT}/piper-sample-generator"}

config = yaml.load(open("openwakeword/examples/custom_model.yml").read(), yaml.Loader)
config["target_phrase"] = [target_word]
config["model_name"] = config["target_phrase"][0].replace(" ", "_")
config["n_samples"] = number_of_examples
config["n_samples_val"] = max(500, number_of_examples // 10)
config["steps"] = number_of_training_steps
config["target_accuracy"] = 0.5
config["target_recall"] = recall_target
config["output_dir"] = "./my_custom_model"
config["max_negative_weight"] = false_activation_penalty
config["background_paths"] = ['./audioset_16k', './fma']
config["false_positive_validation_data_path"] = "validation_set_features.npy"
config["feature_data_files"] = {"ACAV100M_sample": "openwakeword_features_ACAV100M_2000_hrs_16bit.npy"}
config["piper_sample_generator_path"] = f"{ROOT}/piper-sample-generator"

with open('my_model.yaml', 'w') as f:
    yaml.dump(config, f)
print("model:", config["model_name"], "| examples:", number_of_examples,
      "| recall:", recall_target, "| penalty:", false_activation_penalty)

def sh(cmd):
    r = subprocess.run(cmd, shell=True, cwd=ROOT, capture_output=True, text=True)
    if r.returncode:
        print(f"FAILED: {cmd}\n{r.stdout[-800:]}\n{r.stderr[-800:]}")
    return r.returncode == 0

print("preparing the 3.11 environment...")
# setuptools: pronouncing still imports pkg_resources, and uv venvs ship neither.
# scipy pinned: acoustics 0.2.6 imports scipy.special.sph_harm, removed in 1.17.
sh(f"uv pip install -q --python {PY} 'setuptools<82' torchinfo torchmetrics pyyaml "
   f"'scipy<1.17' tqdm mutagen speechbrain==0.5.14 acoustics==0.2.6 pronouncing==0.2.0 "
   f"deep-phonemizer==0.0.19 torch-audiomentations==0.11.0 onnx onnxruntime matplotlib")
sh(f"uv pip install -q --python {PY} -e ./openwakeword --no-deps")

for f in glob.glob(f"{ROOT}/piper311/lib/python*/site-packages/torch_audiomentations/**/*.py",
                   recursive=True):
    src = open(f).read()
    if "set_audio_backend" in src and "# patched" not in src:
        open(f, "w").write(src.replace("torchaudio.set_audio_backend",
                                       "getattr(torchaudio, 'set_audio_backend', lambda *a, **k: None)  # patched\n#"))
        print("patched", os.path.basename(f))

# Exactly what train.py imports. Submodules, not the bare package: with cwd=/content,
# `import openwakeword` finds the *directory* as an empty namespace package and succeeds
# without importing anything.
check = ("import torch, torchinfo, torchmetrics, scipy, yaml, numpy, tqdm; "
         "import audiomentations, torch_audiomentations, speechbrain, acoustics, pronouncing; "
         "from openwakeword.vad import VAD; "
         "from openwakeword.data import generate_adversarial_texts, augment_clips, mmap_batch_generator; "
         "from openwakeword.utils import compute_features_from_generator, AudioFeatures; "
         "from generate_samples import generate_samples; print('ok')")
ok = subprocess.run([PY, "-c", check], cwd=f"{ROOT}/openwakeword", env=ENV,
                    capture_output=True, text=True)
print("stack:", ok.stdout.strip() if ok.returncode == 0 else ok.stderr[-900:])

# Thirty thousand samples on a CPU runtime is a wasted day.
gpu = subprocess.run(
    [PY, "-c", "import torch; print(torch.cuda.get_device_name(0) if torch.cuda.is_available() "
               "else 'NO GPU -- Runtime > Change runtime type > T4')"],
    capture_output=True, text=True)
print("gpu:", gpu.stdout.strip() or gpu.stderr[-300:])

def run(step):
    print(f"=== {step} ===")
    p = subprocess.Popen([PY, "openwakeword/openwakeword/train.py",
                          "--training_config", "my_model.yaml", step],
                         cwd=ROOT, env=ENV,
                         stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
    for line in p.stdout:
        print(line, end="")
    code = p.wait()
    print(f"*** {step} failed ***" if code else f"--- {step} done ---")
    return code == 0
```

### 3b — generate the clips (the long one)

```python
# @title 3b. Generate the clips  { display-mode: "form" }
run("--generate_clips")

import glob, os
print("\nwhat exists now:")
for p in sorted(glob.glob(f"{ROOT}/*.npy")):
    print(f"  {os.path.basename(p):58s} {os.path.getsize(p) >> 20:6d} MB")
```

### 3c — augment

```python
# @title 3c. Augment the clips  { display-mode: "form" }
run("--augment_clips")
```

### 3d — train, and save it where a disconnect cannot reach

```python
# @title 3d. Train  { display-mode: "form" }
import os, shutil

run("--train_model")   # reports failure on the tflite step at the very end. Ignore it.

out  = f"{ROOT}/my_custom_model"
onnx = f"{out}/hey_octavia.onnx"
print("\nmy_custom_model:", os.listdir(out) if os.path.exists(out) else "not created")

if os.path.exists(onnx):
    print(f"hey_octavia.onnx  {os.path.getsize(onnx) >> 10} KB")
    try:
        from google.colab import drive
        drive.mount('/content/drive')
        dest = '/content/drive/MyDrive/octavia'
        os.makedirs(dest, exist_ok=True)
        shutil.copy(onnx, dest)
        print("copied to Drive:", dest)
    except Exception as e:
        print("no Drive copy:", e, "-- use the file browser instead")
else:
    print("NO ONNX. Scroll up for the real error --")
    print("it is NOT the onnx_tf traceback at the end, which is expected.")
```

### The number to read at the end

**`Final Model Recall`.** It was **0.506** on the first run and that was the whole story, sitting
in plain sight above a traceback that did not matter. If a retrain does not clear roughly 0.8,
more samples will not save it and the phrase itself needs rethinking.

`Final Model False Positives per Hour` was 0.177 and had room to spare — some of that is being
spent deliberately here, so a small rise is the trade working rather than a regression.

**A cheaper first answer:** `number_of_examples = 10000` is about a third of the time and still
answers the open question, which is whether the recall and penalty change alone cure the
deafness. 30,000 buys polish on top of an answer 10,000 would already have given.
## If a cell fails

The output is enormous and the real cause is usually one line. Look for:

- `(from versions: none)` — no wheel exists for this Python. Not a pin you can bump.
- `You must restart the runtime` — buried mid-output, and the cause of any bizarre numpy error that follows.
- `404 Not Found` followed by `tar: This does not look like a tar archive` — an upstream dataset moved.
- `ModuleNotFoundError` from a *subprocess* — the 3.11 environment is missing something, not the kernel.


---

# Building the notebook fresh (09/04/2026)

*Everything above is the archaeology of the first run — the notebook as it shipped, plus the
patches that made it go. This section is what to actually build, in order, from an empty Colab.
Use this one.*

**Order: 1 → 2 → 2b → 3a → 3b → 3c → 3d.** No restart, and no cell run twice.

## Why the restart is gone

The old procedure ran cell 2, hit `AttributeError: 'numpy.ufunc' object has no attribute
'__module__'`, restarted the runtime, and ran cell 2 again. That was fault 4 — `onnx_tf`
replacing numpy while numpy was already imported.

**Three of the notebook's installs exist only for the tflite conversion**, and she uses ONNX:

```
!pip install tensorflow-cpu==2.8.1
!pip install tensorflow_probability==0.16.0
!pip install onnx_tf==1.10.0
```

Remove them and nothing swaps numpy, so nothing needs restarting. `train.py` still ends with
`ModuleNotFoundError: No module named 'onnx_tf'` — it did on the first run too, because the
3.11 environment never had it either. The ONNX is written before that line is reached.

`piper-phonemize` and `webrtcvad` also come out of cell 2: they cannot install on Colab's 3.13
at all, cell 1 already puts them in the 3.11 environment, and a pip line that always fails
inside a hundred-line cell is noise that hides the failures that matter.

## Cell 1 — verify the phrase

Unchanged from the first run. Builds the 3.11 environment, clones the generator, downloads the
voice model, and speaks one clip so you can hear the phrase before spending hours on it.

## Cell 2 — install and download

The upstream notebook's environment and download cells, merged, with the tflite installs
removed, the AudioSet block removed (it 404s — cell 2b does it), and two guards fixed.

```python
# @title 2. Install and download  { display-mode: "form" }
import os, numpy as np, torch, sys, yaml, datasets, scipy, scipy.io.wavfile
from pathlib import Path
from tqdm import tqdm

ROOT = "/content"

# --- openwakeword itself ------------------------------------------------------
if not os.path.exists(f"{ROOT}/openwakeword/setup.py"):
    !git clone -q https://github.com/dscripka/openwakeword
!pip install -q -e ./openwakeword

# --- what train.py needs ------------------------------------------------------
# Dropped from the notebook's list: piper-phonemize and webrtcvad (they cannot build on
# 3.13, and cell 1 has already put them in the 3.11 environment), and tensorflow-cpu,
# tensorflow_probability and onnx_tf (tflite only -- and the cause of the numpy swap
# that used to force a runtime restart here).
!pip install -q mutagen==1.47.0 torchinfo==1.8.0 torchmetrics==1.2.0 speechbrain==0.5.14
!pip install -q audiomentations==0.33.0 torch-audiomentations==0.11.0 acoustics==0.2.6
!pip install -q pronouncing==0.2.0 datasets==2.14.6 deep-phonemizer==0.0.19

# --- the two shared models, which openwakeword expects beside itself ----------
res = f"{ROOT}/openwakeword/openwakeword/resources/models"
os.makedirs(res, exist_ok=True)          # exist_ok: the notebook's version throws on a re-run
REL = "https://github.com/dscripka/openWakeWord/releases/download/v0.5.1"
for f in ["embedding_model.onnx", "melspectrogram.onnx"]:
    if not os.path.exists(f"{res}/{f}") or os.path.getsize(f"{res}/{f}") < 100_000:
        !wget -q {REL}/{f} -O {res}/{f}

# --- room impulse responses ---------------------------------------------------
# The notebook guards on the *directory*, so a crash part-way through leaves an empty
# folder that every later run skips -- silently training with no reverb at all.
# Count the files instead.
out = f"{ROOT}/mit_rirs"
os.makedirs(out, exist_ok=True)
if len(os.listdir(out)) < 200:
    rirs = datasets.load_dataset("davidscripka/MIT_environmental_impulse_responses",
                                 split="train", streaming=True)
    for row in tqdm(rirs, desc="mit_rirs"):
        name = row['audio']['path'].split('/')[-1]
        scipy.io.wavfile.write(os.path.join(out, name), 16000,
                               (row['audio']['array'] * 32767).astype(np.int16))

# --- music, as background ------------------------------------------------------
out = f"{ROOT}/fma"
os.makedirs(out, exist_ok=True)
n_hours = 1
want = n_hours * 3600 // 30          # the FMA clips are all 30 seconds
if len(os.listdir(out)) < want:
    fma = datasets.load_dataset("rudraml/fma", name="small", split="train", streaming=True)
    fma = iter(fma.cast_column("audio", datasets.Audio(sampling_rate=16000)))
    for _ in tqdm(range(want), desc="fma"):
        row = next(fma)
        name = row['audio']['path'].split('/')[-1].replace(".mp3", ".wav")
        scipy.io.wavfile.write(os.path.join(out, name), 16000,
                               (row['audio']['array'] * 32767).astype(np.int16))

# --- pre-computed negative features -------------------------------------------
# ~2,000 hours of ACAV100M for training, ~11 hours for false-positive validation.
HF = "https://huggingface.co/datasets/davidscripka/openwakeword_features/resolve/main"
for f, least in [("openwakeword_features_ACAV100M_2000_hrs_16bit.npy", 1_000_000_000),
                 ("validation_set_features.npy", 100_000_000)]:
    if not os.path.exists(f"{ROOT}/{f}") or os.path.getsize(f"{ROOT}/{f}") < least:
        !wget -q {HF}/{f} -O {ROOT}/{f}

# --- did any of that actually work? -------------------------------------------
# pip failures do not stop a cell, and a missing package otherwise surfaces three cells
# later as something that looks unrelated.
import importlib
print("\n--- install check ---")
for m in ["datasets", "scipy", "mutagen", "torchinfo", "torchmetrics", "speechbrain",
          "audiomentations", "torch_audiomentations", "acoustics", "onnx",
          "pronouncing", "openwakeword"]:
    try:
        importlib.import_module(m)
        print(f"  ok    {m}")
    except Exception as e:
        print(f"  FAIL  {m}: {type(e).__name__}: {e}")

print("\n--- data check ---")
for d in ["mit_rirs", "fma"]:
    print(f"  {d:12s} {len(os.listdir(f'{ROOT}/{d}')):5d} files")
for f in ["openwakeword_features_ACAV100M_2000_hrs_16bit.npy", "validation_set_features.npy"]:
    p = f"{ROOT}/{f}"
    print(f"  {f[:44]:46s} {os.path.getsize(p) >> 20 if os.path.exists(p) else 0:5d} MB")

print("\nnumpy:", np.__version__, "- if this errors on a later cell, the runtime needs a restart")
```

`torch_audiomentations` will still FAIL in that check. Cell 2b fixes it — it is the removed
`torchaudio.set_audio_backend`, not a bad install.

Expect **270** impulse responses and **120** music clips.

## Cell 2b — the two repairs cell 2 cannot make

Unchanged from the first run: stub the torchaudio API that torch-audiomentations still calls,
and fetch AudioSet from parquet because the `.tar` the notebook wants is gone.

## Cells 3a to 3d — config, generate, augment, train

As above under *Cell 3, the retrain*. Four cells rather than one, because at 30,000 samples the
run is hours and a failure in the last step should not cost the first.
