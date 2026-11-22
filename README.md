# dots.tts for Windows and AMD Radeon

A Windows-focused setup and usage guide for [dots.tts](https://github.com/studio-dots-ai/dots.tts), the 2B-parameter text-to-speech and voice-cloning system. This fork documents the ROCm/PyTorch setup and provides quick instructions for both the command-line interface and Gradio web UI.

**Tested setup:** Windows 11, AMD Radeon RX 9060 XT 16 GB, AMD ROCm 7.2.1 PyTorch wheels, Python 3.12, and Miniforge. Other GPUs and software versions should be checked against [AMD's compatibility matrix](https://rocm.docs.amd.com/projects/radeon-ryzen/en/latest/docs/compatibility/compatibilityrad/windows/windows_compatibility.html).

## Install

### Requirements

- Windows 11 x64
- An AMD Radeon GPU supported by AMD's Windows PyTorch/ROCm release
- AMD's required graphics driver; for ROCm 7.2.1, see [AMD's Windows installation guide](https://rocm.docs.amd.com/projects/radeon-ryzen/en/docs-7.2.1/docs/install/installrad/windows/install-pytorch.html)
- [Miniforge for Windows x86_64](https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Windows-x86_64.exe)

### Quick setup

Open **Miniforge Prompt** from the Start menu. These commands use Command Prompt syntax.

```cmd
git clone https://github.com/macxzatgit/dots.tts-windows-amd.git
cd /d dots.tts-windows-amd
conda create -n dots-tts python=3.12 pip -y
conda activate dots-tts
conda install -c conda-forge pynini -y
```

Install AMD's ROCm 7.2.1 PyTorch wheels and the project dependencies by following the step-by-step [Windows + AMD installation guide](docs/WINDOWS_AMD.md). It includes the exact wheel commands, GPU verification, WeTextProcessing setup, and Gradio installation.

Verify that PyTorch sees the GPU:

```cmd
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'No GPU detected')"
```

Expected output includes a version containing `rocm7.2.1`, `True`, and your GPU name.

## Use the CLI for voice cloning

Pass a reference recording and its exact transcript. Replace the sample audio path and transcript with your own:

```cmd
dots.tts --model-name-or-path dots-studio/dots.tts-mf --text "Hello, this is a test of speech synthesis." --prompt-audio "C:\path\to\reference.wav" --prompt-text "Exact words spoken in the reference clip." --output test.wav
```

- `dots-studio/dots.tts-mf` is the faster MeanFlow checkpoint.
- Use `dots-studio/dots.tts-soar` for higher-quality voice cloning.
- For continuation cloning, include both `--prompt-audio` and `--prompt-text`.
- Text normalization is optional; install WeTextProcessing using the [Windows guide](docs/WINDOWS_AMD.md) before enabling `--normalize-text`.

See `dots.tts --help` for additional options.

## Use the Gradio web interface

From the repository folder, with the `dots-tts` environment activated, run:

```cmd
python apps\gradio\app.py --model-name-or-path dots-studio/dots.tts-mf --skip-warmup
```

Open [http://127.0.0.1:7860](http://127.0.0.1:7860). Upload your reference audio, enter its transcript and the text to synthesize, then click **Generate**. Keep the prompt window open while using the interface.

`--skip-warmup` starts the web server without an initial synthesis. The model will load when you first generate audio. Remove the flag if you prefer the app to run a warmup before opening the page.

## Notes

- The first model run downloads the checkpoint from Hugging Face and needs an internet connection.
- PyTorch may print experimental attention or MIOpen workspace warnings on AMD. A warning alone does not mean synthesis failed. The optional AOTriton setting and troubleshooting notes are in the [Windows + AMD guide](docs/WINDOWS_AMD.md).
- The Windows PyTorch setup uses Conda for Pynini, a dependency of WeTextProcessing, to avoid building it with the incompatible default MSVC path.

## Upstream project

- [Original dots.tts repository](https://github.com/studio-dots-ai/dots.tts)
- [Model checkpoints](https://huggingface.co/collections/dots-studio/dotstts)
- [Upstream paper](https://arxiv.org/abs/2606.07080)
- [License](LICENSE) — Apache-2.0
