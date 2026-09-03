---
project: Octavia
tags: [octavia, deep-dive, wake-word]
---

# Wake Word Colab Cells

*The four cells that actually run, as pasted on 09/04/2026. Why each change was needed is in [[Training Her Wake Word]] — this note is the copy-paste.*

**Order:** cell 1 → cell 2 → **Runtime → Restart session** → cell 2 again → cell 2b → cell 3.

Cell 2 runs twice on purpose. The first pass installs everything and dies when `onnx2tf` swaps numpy under the live kernel; the restart clears that; the second pass is quick because everything is already installed.

Every cell is safe to re-run.

## Cell 1 — verify the phrase

Replaces the notebook's cell 1 entirely. Builds the Python 3.11 environment the phonemiser needs, then generates one clip so you can hear it.

```python
# @title 1. Test wake word generation  { display-mode: "form" }
import os, glob, subprocess
from IPython.display import Audio, display

ROOT  = "/content"
GEN   = f"{ROOT}/piper-sample-generator"
PY    = f"{ROOT}/piper311/bin/python"
MODEL = f"{GEN}/models/en_US-libritts_r-medium.pt"

target_word = 'hey octavia' # @param {type:"string"}

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
    sh(f"wget -q -O {MODEL} https://github.com/rhasspy/piper-sample-generator/releases/download/v2.0.0/en_US-libritts_r-medium.pt")

sh("pip -q install uv")

# Python 3.11: the phonemiser has no build for Colab's 3.13 and no source to compile.
if not os.path.exists(PY):
    print("building a 3.11 environment (slow, one-off)...")
    sh(f"uv venv --python 3.11 {ROOT}/piper311")
    sh(f"uv pip install -q --python {PY} torch==2.5.0 torchaudio==2.5.0 --index-url https://download.pytorch.org/whl/cu121")

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
    print(r.stdout[-2000:]); print("STDERR:\n", r.stderr[-2000:])
else:
    wavs = sorted(f for f in os.listdir(ROOT) if f.endswith(".wav"))
    print("wrote:", wavs)
    display(Audio(f"{ROOT}/{wavs[-1]}", autoplay=True))
```

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

## Cell 3 — train

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

## If a cell fails

The output is enormous and the real cause is usually one line. Look for:

- `(from versions: none)` — no wheel exists for this Python. Not a pin you can bump.
- `You must restart the runtime` — buried mid-output, and the cause of any bizarre numpy error that follows.
- `404 Not Found` followed by `tar: This does not look like a tar archive` — an upstream dataset moved.
- `ModuleNotFoundError` from a *subprocess* — the 3.11 environment is missing something, not the kernel.
