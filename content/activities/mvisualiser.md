---
title: "mvisualiser"
description: "Real-time terminal audio visualizer in C with from-scratch Cooley-Tukey FFT and ALSA playback."
tags: ["c", "audio", "dsp", "systems"]
---

# mvisualiser

[mvisualiser](https://github.com/Binit06/mvisualiser) is a lightweight terminal audio visualizer written in C. Built on a weekend because I wanted to understand digital signal processing (DSP), Fast Fourier Transforms, and raw Linux audio streams from scratch rather than relying on existing libraries.

---

### How It Works

1. **RIFF/WAVE Header Parsing:** Directly inspects raw byte offsets to unpack PCM audio chunks, channel counts, 44.1 kHz sample rates, and bit depths without helper libraries.
2. **From-Scratch Cooley-Tukey FFT:** Implemented in-place radix-2 Fast Fourier Transform directly:
   * Bit-reversal permutation to reorder time-domain samples.
   * Butterfly computation loops with complex twiddle factors (`cos(θ) + i·sin(θ)`).
   * Computes power spectral density across 512 frequency bins in real time.
3. **ALSA Audio Playback:** Uses Linux ALSA (`<alsa/asoundlib.h>`) to stream PCM audio to the soundcard while synchronizing terminal frequency bars with live playback.

* **Repository:** [github.com/Binit06/mvisualiser](https://github.com/Binit06/mvisualiser)
* **Tech Stack:** C, Linux ALSA (`asoundlib`).
