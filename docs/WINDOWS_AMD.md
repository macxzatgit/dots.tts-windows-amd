# Windows 11 with an AMD Radeon GPU

This guide covers the Windows ROCm/PyTorch path for running `dots.tts` on a supported AMD Radeon GPU. The steps below were exercised on Windows 11 with an RX 9060 XT 16 GB, Python 3.12, ROCm 7.2.1 PyTorch, and Miniforge.

AMD's Windows ROCm distribution supports PyTorch rather than the full ROCm framework stack. Install the AMD graphics driver version required by the [ROCm 7.2.1 Windows guide](https://rocm.docs.amd.com/projects/radeon-ryzen/en/docs-7.2.1/docs/install/installrad/windows/install-pytorch.html) first (26.2.2 for this release). Use the x86_64 Miniforge installer and open **Miniforge Prompt** from the Start menu. The commands below use Command Prompt syntax.

## Create the environment

From Miniforge Prompt:

```cmd
conda create -n dots-tts python=3.12 pip -y
conda activate dots-tts
conda install -c conda-forge pynini -y
```

`pynini` is installed through Conda because its source build expects a compiler toolchain that does not work with the standard Windows MSVC compiler. Installing it before `WeTextProcessing` lets pip use the Conda package instead of attempting that build.

Change to your checkout (adjust the path if needed) and update pip:

```cmd
cd /d C:\AI\dots.tts
python -m pip install --upgrade pip
```

## Install AMD ROCm PyTorch

The AMD Windows wheels in this guide are for Python 3.12. Install the ROCm components first:

```cmd
python -m pip install --no-cache-dir https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/rocm_sdk_core-7.2.1-py3-none-win_amd64.whl https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/rocm_sdk_devel-7.2.1-py3-none-win_amd64.whl https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/rocm_sdk_libraries_custom-7.2.1-py3-none-win_amd64.whl https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/rocm-7.2.1.tar.gz
```

Then install AMD's matching PyTorch wheels:

```cmd
python -m pip install --no-cache-dir "https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/torch-2.9.1%2Brocm7.2.1-cp312-cp312-win_amd64.whl" "https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/torchaudio-2.9.1%2Brocm7.2.1-cp312-cp312-win_amd64.whl" "https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/torchvision-0.24.1%2Brocm7.2.1-cp312-cp312-win_amd64.whl"
```

Confirm that PyTorch sees the GPU before installing the project:

```cmd
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'No GPU detected')"
```

Expected: a version containing `rocm7.2.1`, `True`, and your Radeon GPU name. PyTorch uses the `torch.cuda` API name for ROCm too.

## Install dots.tts, WeTextProcessing, and Gradio

Install the project dependencies without replacing the AMD PyTorch build, then install the checkout in editable mode:

```cmd
python -m pip install "transformers==4.57.1" huggingface-hub loguru "langcodes[data]" "gradio>=6.17,<7" einops "librosa==0.11.0" "soundfile==0.13.1" "numpy==2.2.6" "pydantic==2.12.5" "PyYAML==6.0.3" "safetensors==0.8.0rc0" torchdiffeq tqdm lingua-language-detector WeTextProcessing
python -m pip install -e . --no-deps
```

The `--no-deps` flag on the editable install prevents pip from trying to replace the ROCm PyTorch wheels. Gradio provides the browser interface. WeTextProcessing provides optional text normalization; after installing it, `--normalize-text` can be used in the CLI or selected in the UI.

Check the normalizer imports:

```cmd
python -c "import pynini; from tn.chinese.normalizer import Normalizer as ZhNormalizer; from tn.english.normalizer import Normalizer as EnNormalizer; print('WeTextProcessing ready')"
```

## Use the CLI for voice cloning

Provide a reference audio file and its exact transcript. Replace the sample path and transcript below:

```cmd
dots.tts --model-name-or-path dots-studio/dots.tts-mf --text "Hello, this is a test of speech synthesis." --prompt-audio "C:\path\to\reference.wav" --prompt-text "Exact words spoken in the reference clip." --output test.wav
```

The `dots.tts-mf` checkpoint is the faster MeanFlow option. For the higher-quality SOAR checkpoint, replace it with `dots-studio/dots.tts-soar`.

## Start the Gradio web interface

```cmd
python apps\gradio\app.py --model-name-or-path dots-studio/dots.tts-mf --skip-warmup
```

Open [http://127.0.0.1:7860](http://127.0.0.1:7860). The app loads the model when you first generate audio. Without `--skip-warmup`, the app runs an initial synthesis before it opens the web server, which can take a while on first launch.

Upload reference audio, provide its transcript, enter the text to synthesize, and click **Generate**. Keep the console window open while using the UI.

## Optional experimental attention path

PyTorch may warn that Flash Efficient or Memory Efficient attention on the AMD GPU is experimental. This is informational; the warning alone does not mean the generation failed. To opt into AMD's experimental AOTriton path for one Command Prompt session, set the variable before launching the app:

```cmd
set TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=1
python apps\gradio\app.py --model-name-or-path dots-studio/dots.tts-mf --skip-warmup
```

This option is experimental. If it causes errors or unstable behavior, close the app, open a fresh Miniforge Prompt (or run `set TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=`), and start the app without the variable.

MIOpen messages mentioning `IsEnoughWorkspace` may appear during inference. Treat them as warnings unless followed by a Python traceback or generation failure.
