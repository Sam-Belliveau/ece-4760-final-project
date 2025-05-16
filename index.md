---
layout: default
title: "Audio Localization"
permalink: "/"
---

<link rel="stylesheet" href="./assets/css/custom.css">
<link rel="stylesheet" href="./assets/css/just-the-docs-default.css">
<script src="./assets/js/just-the-docs.js"></script>

<script>
  window.MathJax = {
    tex: {
      inlineMath: [['$', '$'], ['\\(', '\\)']],
      displayMath: [['$$','$$'], ['\\[','\\]']]
    }
  };
</script>

<script src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>

# Our device listens with multiple "ears," using cross-correlation to instantly point you toward a sound’s origin.
{: .no_toc .flex-justify-around .text-center }

--- 

# 0. Table of Contents

<details open markdown="block">
<summary>Click to Hide</summary>

- TOC
{:toc}

</details>

---

# 1. Project Introduction

<div style="display:flex; align-items:flex-center; gap:1rem; margin-left: auto; margin-right: auto;">
  <iframe width="720px" height="405px" src="https://www.youtube.com/embed/yFkt5Urp-eg" frameborder="1" allowfullscreen class="full-bordered"></iframe>
</div>

## 1.1 Summary

In this project, we designed a real-time sound localization system using a Raspberry Pi Pico and three microphones. Audio from each of the MEMS microphones was continuously sampled at \($50\,\mathrm{kHz}$\) via the Pico’s DMA engine.

Each channel’s data is then streamed into a rolling buffer. When the energy in the buffer reaches a certain threshold, the software performs cross-correlations between the microphone pairs to estimate the time differences between the audio arrivals.

These estimates are averaged over time to reduce the impact of noise. Using the delay estimates, a heat map is drawn by comparing the expected delays from each location to the estimates calculated by our cross-correlation.

The entire processing chain—from sampling to buffering, correlation, and VGA drawing—executes on the Raspberry Pi Pico.

## 1.3 Motivation

Our motivation was a combination of passion for signal processing and demonstrating the capabilities of low-cost hardware in audio processing. The localization system has many applications, from robotics to art installations. In a world in which there is an increasing number of applications requiring many cheap distributed sensors, there is a need for the development of algorithms capable of running on cheap, low power, hardware. 

Our project implements audio localization on system whose total BOM is under $20, with a localization accuracy comparable to much more expensive systems. The algorithms run purely in fixed-point math on the Pico, eliminating the need for a dedicated DSP chip. Combined with the Pico’s intuitive VGA output, this results in an all-in-one package for sound localization.

---

# 2. High-Level Design

## 2.1 Rationale & Inspiration

Audio localization typically depends on specialized DSP hardware or simplified methods that only work with sharp transients. This often makes such projects out of reach for hobbyists and also results in expensive systems.

By leveraging the Raspberry Pi Pico’s ADC along with the RP2040’s fast processors, we determined it was possible to implement a more sophisticated localization technique entirely on-chip.

We decided to implement a time-difference-of-arrival (TDOA) method to determine sound location. To compute the time difference between microphones, we use cross-correlations to estimate the likelihood of various sample shifts. As compared to FFT based approaches, this approach is significantely cheaper to compute. This approach allows us to localize many types of sounds quickly and accurately on the Pico.

## 2.2 Background Math

At standard temperature and pressure, the speed of sound is approximately \($343\,\mathrm{m/s}$\). This means that for every additional centimeter between microphones, the maximum travel time is:

$$
\frac{\text{distance}}{\text{velocity}}
= \frac{0.01\,\mathrm{m}}{343\,\mathrm{m/s}}
\approx 29.15\,\mu\mathrm{s}
$$

Due to discrete sampling, time-difference estimates are quantized by the ADC sample period and microphone spacing.

For example, sampling at \($100\,\mathrm{kHz}$\) \($10\,\mu\mathrm{s}$ per sample\) with an equilateral microphone triangle of \($10\,\mathrm{cm}$\) sides yields a maximum shift of about $\pm 30$ samples. Each pair of microphones then produces an integer shift in $\left[-30, +30\right]$ indicating their relative time difference.

With three microphones, there are roughly $61^3$ distinct shift combinations. Increasing the ADC rate or spacing the microphones farther apart increases resolution by allowing more distinct shifts.

### FFT Based-Approaches
To efficiently calculate the Time Difference of Arrival (TDOA) between two microphone signals, $x[n]$ and $y[n]$, using an FFT-based approach, the following steps are typically performed:

1.  **Compute Discrete Fourier Transforms (DFTs):**
    The DFTs of the windowed signal frames are calculated.
2.  **Form the (Weighted) Cross-Power Spectrum:**
    The cross-power spectrum $P_{xy}(\omega) = X(\omega) Y^*(\omega)$ is computed, where $Y^*(\omega)$ is the complex conjugate of $Y(\omega)$. For improved robustness, especially in noisy or reverberant conditions, a weighting function is applied.
3.  **Compute Inverse DFT (IDFT):**
    The cross-correlation sequence $r_{xy}[k]$ in the time lag domain is obtained by taking the IDFT of $P_{xy}(\omega)$:
    This sequence represents the correlation between the two signals for various time lags $k$.
4.  **Identify Peak Lag:**
    The lag $k_{\text{max}}$ at which the cross-correlation sequence $r_{xy}[k]$ reaches its maximum value corresponds to the estimated time delay in samples:
    $$k_{\text{max}} = \underset{k}{\text{argmax}} \{r_{xy}[k]\}$$
    
On board the Pico, the FFT module requires a fair bit of compute in terms of memory and cycle time. Additionally, for a 3-microphone setup, 3 forward and 3 inverse FFTs are required, dramatically increasing the computational load. One of the innovations of our project is to remove the need for the calculation of the FFT prior to taking the cross-correlation. So while these approaches operate in the frequency domain, we operate on the sampled microphone power level readings. 

### Cross Correlation
The cross correlation calculation makes up the core of our algorithm. Cross‑correlation is a sliding inner‑product that quantifies the similarity between two signals as one is shifted in time. The cross correlation operation is defined as 

$$ 
R_{xy}[k] \;=\;\sum_{n=-\infty}^{\infty} x[n]\,y[n + k]
$$

In our case, the cross correlation peaks at a point k which represents the point at which the signals overlap the most.  

### Time Delay of Arrival (TDOA) Calculation
$$
t_{delay} = \frac{k_{max}}{f_s}. 
$$

Though we use a more complex version of this, the goal of our system is to find this k_max value to find the time shift between microphones. In our project, this k is represented by the best shift. We apply some smoothing and filtering techiques, but at its core, our project finds these shifts between the microphones and uses it to determine the audio source. 

### TDOA Conversion to Position
Once $\tau_{\text{delay}}$ is found between a pair of microphones, it implies that the sound source lies on a specific hyperboloid with the microphones as foci.
$$ c \cdot \tau_{\text{delay}} = d_2 - d_1 $$
where $c$ is the speed of sound, and $d_1, d_2$ are the distances from the source to microphone 1 and 2, respectively.
By using multiple microphone pairs, multiple TDOAs can be calculated, and the intersection of the corresponding hyperboloids gives an estimate of the sound source's location. For instance, with two microphones, the TDOA can give an angle of arrival (AOA) relative to the microphone axis. This approach is what is used in our implementation and also in many FFT-based systems. 
 
--- 
## 2.3 Logical Structure

The software pipeline on a single RP2040 core follows:

1. **DMA-driven sampling**  
   Configure the ADC in round-robin mode for three channels and set up a ping-pong DMA to fill a three-byte `dma_sample_array`.

2. **Rolling buffers**  
   The `sample_and_compute_loop()` function reads the DMA array at \($50\,\mathrm{kHz}$\), converts each $8$-bit sample to signed $16$-bit, and pushes them into three rolling buffers via `rolling_buffer_push()`.

3. **Event detection**  
   Once buffers are full, outgoing and incoming windowed power are compared. Crossing the threshold triggers processing.

4. **Cross-correlation**  
   Each buffer is flattened into a DC-centered frame and optionally windowed. `correlations_init()` scores integer sample shifts over the physically possible range.

5. **Best-shift computation**  
   The shift with the highest correlation score for each microphone pair is selected; `correlations_average()` exponentially smooths these over time.

6. **VGA rendering**  
   If any shift is nonzero, `vga_draw()` renders the full heatmap and waveforms. Otherwise, `vga_draw_lite()` updates only the latest curves and markers.

## 2.4 Hardware/Software Trade-Offs

Achieving \($50\,\mathrm{kHz}$\) sampling across three ADC channels on a Pico with \($264\,\mathrm{KB}$\) SRAM and \($133\,\mathrm{MHz}$\) cores required careful balancing:

- **DMA offload:** The DMA ensures jitter-free \($50\,\mathrm{kHz}$\) sampling without CPU interrupts.
- **Buffer size vs. compute time:** We experimented with buffer lengths up to $2048$ samples but settled on $1024$ to balance noise reduction and CPU load.
- **Display overhead:** Full heatmap draws are only on loud events; `vga_draw_lite()` handles most frames to stay within sampling deadlines.

## 2.5 Intellectual Property Landscape

Time-difference-of-arrival localization with cross-correlation is a long-standing, public-domain DSP technique. While specific microphone array patents exist, our breadboard layout and algorithms are original or derived from public-domain sources. We use no proprietary IP, no Altera/Intel cores, and abide by open-source licenses for any third-party code.

---

# 3. Program & Hardware Design

## 3.1 Hardware Design

![Hardware Design](./assets/images/final_board.png){: .bordered }

We used three MAX4466 microphones at the corners of a breadboard forming a right triangle. Each mic’s output passes through a second-order \($20\,\mathrm{kHz}$\) low-pass filter to remove ultrasonic noise.

The filtered outputs connect to the Pico’s ADC channels `0`, `1`, and `2`. We minimized wiring length to reduce interference.

The provided VGA driver uses three PIO state machines and DMA:

- **HSYNC PIO:** Generates horizontal sync, front/back porches.
- **VSYNC PIO:** Counts lines via HSYNC interrupts and manages vertical timing.
- **RGB PIO:** Streams pixel data from a DMA-fed global pixel array (\($3$ bits per pixel\)).

A \($330\,\Omega$\) resistor in series (with the Pico’s internal \($70\,\Omega$\)) forms a divider to step the Pico’s \($3.3\,\mathrm{V}$\) GPIO down to \($0$–$0.7\,\mathrm{V}$\) safe for VGA inputs.

## 3.2 Core Software Loop

The heart of the system resides in the `sample_and_compute_loop()` function within `sample_compute.c`, which orchestrates audio capture, processing, and visualization. Upon startup, `vga_init()` prepares the display subsystem.

The loop initializes three rolling buffers—one per microphone—and records time for pacing. In the inner sampling loop, the code:

1. Reads $8$-bit ADC values from `dma_sample_array` for channels A, B, and C.
2. Converts them to signed $16$-bit samples.
3. Pushes them into the respective circular buffers with `rolling_buffer_push()`.

Once each buffer is full, outgoing and incoming power are computed. If the "outgoing" energy (older half of each buffer) exceeds twice the "incoming" energy (newer half), an acoustic event is detected and processing begins.

Each rolling buffer is then flattened into a DC-offset-normalized frame via `rolling_buffer_write_out()`. Pairwise cross-correlations (`correlations_init()`) determine sample-shift estimates. If any shift is nonzero, `correlations_average()` updates long-term averages, and `vga_draw()` runs; otherwise, `vga_draw_lite()` updates only waveforms and shift markers.

## 3.3 Rolling Buffer Algorithm

Circular buffering of the microphone streams is implemented in `rolling_buffer.c` using a fixed-size array of `BUFFER_SIZE` samples. Each call to `rolling_buffer_push()`:

1. Computes indices for the "outgoing" sample (current head) and the midpoint sample (head minus half the buffer, wrapped).
2. Updates four accumulators: `outgoing_total`, `outgoing_power`, `incoming_total`, and `incoming_power`.
3. Uses the `SAMPLE_POWER(sample)` macro for power tracking.
4. Overwrites the oldest sample, increments the head (wrapping at `BUFFER_SIZE`), and sets `is_full` after the first wrap.

When processing, `rolling_buffer_write_out()` copies samples into a contiguous `buffer_t`, subtracts the average (`dst_offset`), and computes total power for correlation normalization.

## 3.4 Cross-Correlation Module

The cross-correlation engine in `correlations.c` provides both instantaneous and smoothed time-delay estimates.

- **`correlations_init()`:**  
  For each integer shift `s` in [–`MAX_SHIFT_SAMPLES`, +`MAX_SHIFT_SAMPLES`], it computes the dot-product of two sample buffers offset by `s` and stores the 64-bit sum in `corr->correlations[s + MAX_SHIFT_SAMPLES]`. The `best_shift` is set to the highest-scoring `s`.

- **`correlations_average()`:**  
  Applies an exponential decay filter based on elapsed time, blends new correlation values into a long-term array, and recomputes `best_shift` on the smoothed data.

## 3.5 DMA-Driven Sampling

High-throughput ADC sampling is achieved in `dma_sampler.c`.

During `dma_sampler_init()`, the ADC is configured in round-robin mode for channels `0`, `1`, and `2` (GPIO26–28) with FIFO enabled and the clock divider set for maximum rate.

Two DMA channels are used:

1. **Sample channel:** Reads $8$-bit samples from the ADC FIFO into the three-byte `dma_sample_array`.
2. **Control channel:** Reloads the sample channel’s transfer pointer to form a continuous ping-pong cycle.

This yields a steady \($50\,\mathrm{kHz}$\) sampling rate with minimal jitter and zero CPU overhead beyond reading `dma_sample_array`.

## 3.6 Microphone Geometry & Calibration

In `microphones.c`, `microphones_init()` computes the array geometry:

1. Uses `MIC_DIST_AB_M`, `MIC_DIST_BC_M`, and `MIC_DIST_CA_M` with the law of cosines to determine coordinates for an uncentered triangle (A′ at $(0,0)$, B′ at $(d_{AB},0)$, C′ accordingly).
2. Finds the centroid of A′B′C′ and shifts all points so the center of mass is at the origin.
3. If `ROTATE_MICROPHONES` is enabled, rotates all points so microphone A aligns with the +X axis via a standard 2D rotation.

## 3.7 VGA Visualization

![VGA Visualization](./assets/images/early_interface.jpeg){: .bordered }

The display routines in `vga.c` layer multiple graphical elements:

- **`vga_init()`:** Calls `vga_init_heatmap()`, which draws axes and precomputes a heatmap lookup.
- **`vga_draw()`:** Renders correlation curves, overlays the heatmap of possible locations, and plots raw waveforms.
- **`vga_draw_lite()`:** Omits the heatmap and updates only markers and waveforms for low-latency frames.

Helper modules (`vga_correlations`, `vga_heatmap`, `vga_text`, `vga_waveforms`) use the `lib/vga/vga16_graphics` primitives to draw primitives based on correlation-derived color thresholds.

## 3.8 Third-Party & Reused Code

We build atop the official Raspberry Pi Pico SDK for multicore support, GPIO, ADC, DMA, and timing. The VGA stack uses the open-source `lib/vga/vga16_graphics` library. No proprietary IP or Altera cores are used. Math functions (`sqrtf`, `atan2f`, `exp`) come from standard C libraries. All custom modules—rolling buffers, correlation engine, DMA sampler, display routines—are authored in-house or derived from public-domain sources.

---

# 4. Things Tried That Didn’t Work

## 4.1 Aggressive Windowing Function

Early in development, we incorporated a Discrete Prolate Spheroidal Sequence (DPSS) analysis window via `buffer_window()` in `buffer.c`, multiplying each of the $256$ samples by a high-"NW" taper stored in `WINDOW_FUNCTION`.

Below is a plot of the original window function and its frequency response:

![Early Window Function](./assets/images/early_window.png){: .bordered }
![Early Window Function Frequency Response](./assets/images/early_window_response.png){: .bordered }

Analytically, it was intended to reduce spectral leakage before cross-correlation, but the $8$-bit ADC’s quantization noise dominated. The DPSS windowing added overhead without improving peak detection. We lowered the NW parameter significantly (flattening the window) and left the minimal window in place, relying on DC-offset removal for accuracy.

![Final Window Function](./assets/images/final_window.png){: .bordered }
![Final Window Function Frequency Response](./assets/images/final_window_response.png){: .bordered }

## 4.2 In-Plane Localization Methods

We initially attempted full "in-plane" localization—solving for $(x,y)$ on the microphone plane—by combining three pairwise time delays with multilateration routines in `microphones.c`.

![Early Multilateration](./assets/images/mic_resolution_in_plane.png){: .bordered }

While this worked for sources inside the triangle, it failed for sounds outside the array: hyperbolic delay curves intersected behind the microphones or at infinity.

We recognized the array’s strength in angular estimation (parallel wavefronts) over absolute distance. By switching to angles of arrival above the array—using $\pm$`MAX_SHIFT_SAMPLES` cross-correlation shifts—we achieved robust direction estimates even off-axis.

![Final Multilateration](./assets/images/mic_resolution_above.png){: .bordered }

Assuming a height above the array, we accurately locate sources outside the triangle.

---

# 5. Testing and Results of the Design

## 5.1 Test Data & Traces

<!-- Present oscilloscope captures of ADC sampling timing (`SAMPLE_PERIOD_US`), DMA triggers, and GPIO timing for debugging. -->

## 5.2 Performance Metrics

<!-- Report system latency from sound event to VGA update, CPU load during correlation, and achieved frame rate under full load. -->

Blah Blah blah…

## 5.3 Accuracy Metrics

<!-- Quantify localization error (e.g., $\pm X\,\mathrm{cm}$) across a test grid, stability of `best_shift` under varying SNR. -->

Blah Blah blah…

## 5.4 Safety & Robustness

Our system has effectively no major safety considerations, operating at low voltages and currents. The input voltage range for the ADC pins is 0-3.3V. Our microphones do not draw too much power and we comply with all safety standards and utilize good wiring practices. On power-on, the Pico executes its power-on state machine and then starts executing the code. The code is flashed to the Pico, making the system robust to power outs.  Additionally, our system is also robust to the power off of one microphone. Our core logic continues to work, however we see an arc like heatmap, resulting from the loss of samples from the third microphone in the cross correlation logic. This is seen in our demo video with the removal of one microphone. 

## 5.5 Usability Assessment

Overall, our implemented system was quite usable and easy to set up. To set up the system, the user simply needs to power the Pico and to connect the wires for the VGA interface. Once powered on, the system is very easy to use. Simply by making a sound (such as a snap, clap, or whistle), the user can almost instantly see the heatmap plotted on the VGA display. The only non-intuitive aspect of using the system is knowing the VGA pin wiring to the connector. 
Based on the plot, one can orient themselves relative to the microphones on the board, allowing the user to understand the plot without prior knowledge of how the microphones are oriented. One of the most compelling aspects of our system is the ability to make repeated noises from different locations. By moving around, the user intuitively can watch the system plot their new location.
An assumption our triangulation makes is that the noise is coming from roughly 1 meter above the microphone. This means the best performance is achieved by placing the board on the floor. This is not an obvious thing, but does not significantly affect the triangulation and the effect the user sees. To set up the system with a new Pico, one would simply flash the code and place the Pico on the breadboard. This greatly simplifies setup, as there is nothing that needs to be inputted from a host computer to setup the Pico. There are some parameters hard-coded in the code which can be modified, such as the distance between the microphones, the sample rate and MAX_SHIFT_SAMPLES parameters among a few others than sometimes required calibration for the best performance. Nonetheless, the system is fully usable without calibration. 

# 6. Conclusions

## 6.1 Design Evaluation & Lessons Learned

<!-- Analyze how results matched expectations, discuss potential improvements (e.g., fixed-point vs floating-point, faster PIO implementations). -->
In our design, our cross-correlation design yielded a result significantely better than we expected. As compared to a Time of Arrival or FFT based approach, our cross

## 6.2 Standards Compliance

<!-- Comment on adherence to VGA timing specs, Pico hardware guidelines, and DSP best practices. -->
Our design adhered to the VGA timing specification, utilizing the driver provided as part of the course. 

Blah Blah blah…

---

# 7. Appendix A (Permissions)

## 7.1 Course Website Inclusion

The group approves this report for inclusion on the course website.

## 7.2 YouTube Channel Inclusion

The group approves the video for inclusion on the course YouTube channel.
