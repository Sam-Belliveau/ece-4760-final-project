---
layout: default # or whatever layout you’ve defined
title: "Audio Localization"
permalink: "/"
---

<link rel="stylesheet" href="./assets/css/custom.css">

# 0. Table of Contents

* [0. Table of Contents](#0-table-of-contents)
* [1. Project Introduction](#1-project-introduction)
* [2. High-Level Design](#2-high-level-design)
* [3. Program & Hardware Design](#3-program--hardware-design)
* [4. Things Tried That Didn’t Work](#4-things-tried-that-didnt-work)
* [5. Testing and Results of the Design](#5-testing-and-results-of-the-design)
* [6. Conclusions](#6-conclusions)
* [7. Appendix A](#7-appendix-a-permissions)

# 1. Project Introduction

## 1.1 Sound bite

Our device listens with multiple 'ears,' using cross-correlation to instantly point you towards a sound's origin.

_video goes here_

![Sound bite](./assets/images/sam_clap.jpg){: .bordered }

## 1.2 Summary

In this project, we designed a real time sound localization system using a Raspberry Pi Pico and three microphones. Audio from each of the MEMS microphones was continuously sampled at 50 kHz via the DMA on the Pico.

Each channel’s data is then streamed into a rolling buffer. When the energy in the buffer reaches a certain threshold, the software performs cross correlations between the microphone pairs to estimate the time differences between the audio arrivals.

These estimates are averaged over time to reduce the impact of noise compared to loud noises. Using the delay estimates, a heat map is drawn by comparing the expected delays from each location to the estimates calculated by our cross correlation.

The entire processing chain, from sampling to buffering, correlation, and VGA drawing, is executed on the Raspberry Pi Pico.

## 1.3 Motivation

Our motivation was a combination of passion for signal processing and to demonstrate the capabilities of low cost hardware in the realm of audio processing. The localization system has many applications from robotics to art installations.

The total cost of the microphones used is under 20$, while the accuracy of the localization is comparable to that of much more expensive systems. The selection of algorithms did not require a dedicated DSP chip, running purely in fixed point math on the Pico. Combined with the VGA output from the Pico, we resulted in an all in one package for sound localization.

# ------

# 2. High-Level Design

## 2.1 Rationale & Inspiration

Typically, audio localization is dependent on specialized DSP hardware, or shortcuts that limit localizations to sharp transient noises. This typically makes the project out of reach for most hobbyists.

By leveraging the powerful and fast ADC on the Raspberry Pi Pico, along with the RP2040’s fast processor, we determined that it was possible to implement a more sophisticated technique for locating audio.

We decided to implement a time difference of arrival (TDOA) to determine the location of sound, which is not particularly unique. However, in order to determine the time difference between microphones, we used cross correlations to estimate the likelihood of the various shifts between microphones. By using cross correlations, we are able to determine the location of a wide variety of noises quickly and accurately on the Raspberry Pi Pico.

## 2.2 Background Math

At standard temperature and pressure the speed of sound is approximately 343m/s. This means for every additional centimeter we space our microphones apart, the maximum time it would take for the sound to travel from one microphone is \[distance/velocity\] → [(0.01 meter)/(343m/s)] ~= 29.15us.

With this knowledge we can understand the best case resolution for sound approximation. Due to realistic limitations of non-continuous microphone sampling, when attempting to determine the time difference of a sound between the microphones the solutions are discretized as a function of our sample rate and microphone distance.

Say our ADC is able to sample all 3 microphones at the exact same time at a rate of 100kHz (10us between sampling), and our microphones are arranged in an equilateral triangle with a distance of 10cm between the microphones. The maximum time for sound to travel from one microphone to another is ~292us. If we divide that by the period of our sample rate there is a maximum of + or -(29-30) different sample shifts we can distinguish between.

In our case each set of 2 microphones would produce a number between -30-30 indicating the time difference of arrival between the microphones as a multiple of 10us. Meaning if we are equidistant to both microphones it would produce a 0 and if we are perfectly in line with the microphones it would produce a value of 30.

If we do this for all 3 microphones we are limited by (60)^3 different possible outcomes (i.e locations we can distinguish between). This is why either increasing ADC sample rate or placing our microphones farther apart increases resolution. Both cases provide us more distinct maximum sample shifts.

## 2.3 Logical Structure

The software follows a simple stream of signal processing techniques in order to achieve localization on a single RP2040 core. We configure the ADC to be in round-robin mode between the three microphones, while we configure the DMA to constantly sample the microphones and write them into a 3 value array.

Then, the `sample_and_compute_loop()` reads the array at a rate of 50khz, converting the samples to signed 16 bit values, and pushes them into the three rolling buffers. Once all buffers are full, we measure outgoing and incoming power using rolling windows in order to detect a sound of interest; upon crossing the threshold, each buffer is flattened into a DC-centered frame and then windowed using a DPSS window.

The software then performs cross-correlations to score integer samples‐shifts over a range of physically possible shift values. Then, the software selects the highest correlation value for a shift for each pair. Each correlation value serves as a confidence metric for how likely the sound is to be shifted by that amount. By averaging the confidence values between measurements, we obtain a best estimate of the shifts between microphones for the loudest noise.

Finally, the full heatmap and waveform display is rendered via the `vga_draw()` function. The heatmap maps locations to specific shift values, and uses the confidence intervals calculated before to infer the confidence of each location on the heat map.

## 2.4 Hardware/Software Trade Offs

To determine the ideal sample rate that would minimize time between samples while also providing enough time to perform power and threshold calculation we utilized an oscilloscope and trigger GPIO. Immediately before sampling we would write HIGH to GPIO 1 on the Pico and immediately write LOW to GPIO 1 once all data processing was done and we were waiting to sample again.

By analyzing the scope we could tell how much time was not utilized between periods by looking at the off time of GPIO 1. Unfortunately, we did not take photos of the scope but we found our optimal sample rate was determined to be 50(kSamples)/(sec) as it gave us approximately 3-4us of headroom.

In order to minimize sample time between the 3 microphones (when calculating correlations we assumed all microphones were sampled at the exact same time) we optimized the ADC and DMA channel to read mic data every 2us meaning our dt between a given sample from mic A and mic C is ~4us. To follow our 50(kSmaples/sec) rate we placed a delay to only read the values in the DMA every 20us. The DMA setup for sampling will be discussed in greater detail at a later point.

For higher noise reduction and overall performance a larger buffer size of mic data is preferred. When accounting for other arrays the maximum size of the 3 microphone buffers was 2048 (SRAM constraint of ~264Kb). This had the best noise reduction however, significantly increased compute time and memory for other variables, causing us to settle on a buffer size of 1024 (accurate but fast).

To balance this, heavy operations, buffering and correlation are interleaved with VGA rendering. Full heatmaps are only drawn on specific loud noises, while `vga_draw_lite()` handles the majority of frames, keeping the display responsive without violating sampling deadlines.

## 2.5 Intellectual Property Landscape

Time difference of arrival localization is an extremely old DPS technique appearing in a number of papers and probably patents, however the idea of cross-correlation itself is a generic idea and is in the public domain. Patents exist for specific microphone layouts, however our layout of microphones was the result of the breadboard layout and had little impact on our ability to locate sound. All custom algorithms, rolling buffers, power threshold detection, sample level cross correlation, and display logic are original or derived from public-domain sources, and no proprietary IP or non-disclosure agreements were involved.

# 3. Program & Hardware Design

## 3.1 Hardware Design

![Hardware Design](./assets/images/final_board.png){: .bordered }

We used 3 MAX4466 microphones, placing them on the corners of the breadboard creating a right triangle. Each microphone's output ran through a second order lowpass filter with a cutoff frequency of 20kHz. This ensured our readings were within the audible range removing high frequency noise.

We selected the MAX4466 microphones to show that our project can be done with a super low budget and the simplest of microphones. The 3 low-pass filter output connects to the Pico’s ADC channels 0, 1, and 2. We further optimized our system to reduce all high frequency noise reduction by minimizing excess wiring.

A VGA driver was given as part of the provided code for the assignment. The driver works by writing to a memory buffer, which the VGA utilizes to output to the display using DMA transfers. The VGA driver utilizes three synchronized PIO state machines combined with DMA to efficiently generate signals required for driving a VGA display. One PIO state machine creates the horizontal sync (HSYNC) signal by cycling through active video, front porch, sync pulse, and back porch stages for each line, while a second machine generates the vertical sync (VSYNC) signal, using interrupts from the HSYNC to count lines and manage frame timing.

The third PIO state machine rapidly outputs RGB color data to the VGA connector, fetching pixel values from a global pixel array through a DMA channel, with a second DMA channel continuously resetting the first for seamless streaming of pixel data. This array encodes colors using 3 bits per pixel. Contained within the VGA buffer were a series of functions to draw primitives to the screen.

To connect to the display we used a VGA port directly we only needed to provide RGB, VSYNC, HSYNC, and GND. Our PICO input was 3.3V and the safety voltage range for the display through VGA was 0V - 0.7V therefore we put a 330 Ohm resistor in series with the internal 70 Ohm resistor creating a voltage divider ensuring all inputs fall within the safe range.

## 3.2 Core software Loop <!-- (is chatgpt) -->

Detail `sample_and_compute_loop()` in `sample_compute.c`, from buffer refilling through threshold-based activity detection to correlation and VGA draw.

The heart of the system resides in the `sample_and_compute_loop()` function within `sample_compute.c`, which orchestrates audio capture, processing, and visualization.
Upon startup, `vga_init()` is called to prepare the display subsystem.

The loop then initializes three rolling buffers, one per microphone, and records time for pacing.

In the inner sampling loop, the code pulls 8-bit ADC readings from `dma_sample_array` for channels A, B, and C, shifts them into 16-bit samples, and pushes them into their respective circular buffers via `rolling_buffer_push()`.

Once each buffer is full, outgoing and incoming power are computed for all three microphones; if the sum of “outgoing” energy (older half of each buffer) exceeds twice the sum of incoming energy (newer half), the system deems an acoustic event detected and breaks out to processing.

After breaking, each rolling buffer is flattened into a DC-offset-normalized frame (`rolling_buffer_write_out()`), and three pairwise cross-correlations are initialized to determine sample-shift estimates.

If any shift is nonzero, the new correlations are exponentially averaged into long-term estimates (`correlations_average()`), and the full VGA draw routine (`vga_draw()`) runs; otherwise, a lightweight overlay (`vga_draw_lite()`) updates only waveforms and shift markers.

## 3.3 Rolling Buffer Algorithm <!-- (is chatgpt) -->

<!-- _Explain rolling_buffer_push() and power calculations for incoming/outgoing windows in rolling_buffer.c, including head wrapping and real-time power threshold logic._ -->

Circular buffering of the microphone streams is implemented in rolling_buffer.c using a fixed-size array of `BUFFER_SIZE` samples.

Each call to `rolling_buffer_push()` computes indices for the “outgoing” sample (current head) and the midpoint sample (head minus half the buffer, wrapped around), then updates four accumulators: `outgoing_total`, `outgoing_power`, `incoming_total`, and `incoming_power`.

Power is tracked via the `SAMPLE_POWER(sample)` macro (i.e., sample²), and totals allow later DC-offset removal.

The new sample overwrites the oldest entry, the head index increments (wrapping to zero when it reaches `BUFFER_SIZE`), and a boolean is_full flags when the buffer has wrapped at least once.

When it’s time to process, `rolling_buffer_write_out()` copies the samples in chronological order into a contiguous `buffer_t`, subtracts the average (`dst_offset`) to center the data, and computes the total power for correlation normalization .

## 3.4 Cross-Correlation Module <!-- (is chatgpt) -->

<!-- Describe correlations_init() and correlations_average() in correlations.c, how shifts are scored, averaged over time, and best_shift updated . -->

The cross-correlation engine in correlations.c provides both instantaneous and smoothed estimates of time delays.

In `correlations_init()`, for each integer shift s in \[`−MAX_SHIFT_SAMPLES`, +`MAX_SHIFT_SAMPLES`\], the code computes the dot-product between two sample buffers—offsetting one pointer forward or backward according to s—and stores the 64-bit sum in `corr->correlations[s + MAX_SHIFT_SAMPLES]` the best_shift is set to the shift with the highest score.

The timestamp of this update is recorded via `get_absolute_time()`.

To mitigate noise and temporal jitter, `correlations_average()` applies an exponential decay filter: it computes a decay factor based on the elapsed time since the last update, then blends new values into a long-term estimate array and recomputes best_shift on the smoothed data .

## 3.5 DMA-Driven Sampling <!-- (is chatgpt) -->

Cover ADC setup, round-robin input on channels 0–2, ping-pong DMA configuration in `dma_sampler.c`, and how `dma_sample_array` is populated without CPU intervention.
High-throughput, low-overhead ADC sampling is achieved in `dma_sampler.c`.

During `dma_sampler_init()`, the ADC is configured in round-robin mode to sequence through channels 0, 1, and 2 (GPIO26–28), with the FIFO enabled for continuous transfers and clock divider set for maximum rate.

Two DMA channels are claimed: the sample channel reads 8-bit samples from the ADC FIFO into the three-byte array `dma_sample_array` (without CPU intervention) and chains to the control channel, while the control channel writes back the reload pointer to retrigger the sample channel, forming a ping-pong cycle.

This setup yields a steady 50 kHz sampling rate with minimal jitter and zero CPU load beyond reading from `dma_sample_array` in the main loop.

## 3.6 Microphone Geometry & Calibration <!-- (is chatgpt) -->

Walk through `microphones_init()` in `microphones.c`, triangle construction via law of cosines, centroid centering, and optional rotation .

In microphones.c, `microphones_init()` computes the physical layout of the three-element array. Using constants `MIC_DIST_AB_M`, `MIC_DIST_BC_M`, and `MIC_DIST_CA_M`, the code applies the law of cosines to determine coordinates for an uncentered triangle (A′ at (0,0), B′ at (d<sub>AB</sub>,0), and C′ calculated accordingly), optionally mirroring across the x-axis.

It then finds the centroid of A′B′C′ and shifts all points so the array’s center of mass lies at the origin.

If ROTATE_MICROPHONES is enabled, a final rotation aligns microphone A along the +X axis by computing the current angle of A and applying a standard 2D rotation to all three points.

## 3.7 VGA Visualization <!-- (is chatgpt) -->

![VGA Visualization](./assets/images/early_interface.jpeg){: .bordered }

Outline `vga_init()`, `vga_draw()`, and `vga_draw_lite()` in vga.c, including heatmap axis drawing, color thresholds based on correlation sums, and plotting waveforms .
The display routines in vga.c layer multiple graphical elements for real-time feedback.

Calling `vga_init()` invokes `vga_init_heatmap()`, which draws the coordinate axes and precomputes a heatmap lookup based on expected time-difference patterns.

The main draw pass (`vga_draw()`) renders correlation curves, overlays the heatmap of possible source locations, and finally plots raw waveform lanes; `vga_draw_lite()` omits the full heatmap and focuses on shift markers and waveforms for low-latency updates.

Under the hood, helper modules—`vga_correlations`, `vga_heatmap`, `vga_text`, and `vga_waveforms`—leverage the `lib/vga/vga16_graphics` primitives to draw lines, rectangles, and circles according to color thresholds derived from the highest aggregated correlation values.

## 3.8 Third-Party & Reused Code <!-- (is chatgpt) -->

List PICO SDK modules, VGA library code, and any open-source snippets incorporated (e.g., `lib/vga16_graphics`).

This project builds atop the official Raspberry Pi Pico SDK for core services: multicore support, GPIO, ADC, DMA, and timing.

The VGA stack is drawn from the open-source `lib/vga/vga16_graphics` library, providing low-level drawing routines.

No proprietary IP or Altera blocks are used; math functions (e.g. `sqrtf`, `atan2f`, `exp`) come from standard C libraries.

All custom components—rolling buffers, correlation engine, DMA sampler, and display modules—are authored in-house, with public-domain licensing for any snippets incorporated.

# 4. Things Tried That Didn’t Work

## 4.1 Aggressive Windowing Function

Early in development, we incorporated a Discrete Prolate Spheroidal Sequence (DPSS) analysis window via the `buffer_window()` function in buffer.c, multiplying each of the 256 samples by a high-“NW” taper stored in `WINDOW_FUNCTION`. Below is a plot of the original window function and it's frequency response.

![Early Window Function](./assets/images/early_window.png){: .bordered .smaller }
![Early Window Function Frequency Response](./assets/images/early_window_response.png){: .bordered .smaller }

Analytically, this was intended to reduce spectral leakage before cross-correlation, but real-world testing revealed that the 8-bit ADC’s inherent quantization noise dominated any sidelobe reduction.

In practice, the DPSS windowing step—originally enabled in `sample_and_compute_loop()`—introduced extra multiply-and-shift overhead without improving peak detection, so we lowered the NW parameter significantly (flattening the window) and ultimately left the window function in place only at its minimal strength, relying on the buffer’s DC-offset removal to maintain correlation accuracy .

![Final Window Function](./assets/images/final_window.png){: .bordered .smaller }
![Final Window Function Frequency Response](./assets/images/final_window_response.png){: .bordered .smaller }

## 4.2 In Plane Localization Methods

We initially attempted full “in-plane” object localization—solving for `(x,y)` positions on the microphone plane—by combining our three pairwise time delays with geometric intersection routines in `microphones.c`.

![Early Multilateration](./assets/images/mic_resolution_in_plane.png){: .bordered .smaller }

As you can see, while this worked for sources inside the triangle, it failed catastrophically for sounds originating outside the array: the intersection of hyperbolic delay curves often lay behind the microphones or at infinity.

Recognizing that our small planar array is inherently better at measuring angular differences (parallel arriving wavefronts) than absolute distances, we abandoned planar multilateration in favor of simply computing angles of arrival above the array.

This shift leveraged the high angular resolution afforded by our `±MAX_SHIFT_SAMPLES` cross-correlation engine (`correlations_init()` in `correlations.c`) and yielded robust direction estimates even for off-axis sources.

![Final Multilateration](./assets/images/mic_resolution_above.png){: .bordered .smaller }

As you can see, by assuming a distance above the array, we can accurately determine the position of the sound source outside of the triangle.

# 5. Testing and Results of the Design

## 5.1 Test Data & Traces:

<!-- Present oscilloscope captures of ADC sampling timing (`SAMPLE_PERIOD_US`), DMA triggers, and GPIO timing for debugging. -->

## 5.2 Performance Metrics:

<!-- Report system latency from sound event to VGA update, CPU load during correlation, and achieved frame rate under full load. -->

Blah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blah

## 5.3 Accuracy Metrics:

<!-- Quantify localization error (e.g., ± X cm) across a test grid, stability of best_shift under varying SNR. -->

Blah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blah

## 5.4 Safety & Robustness:

<!-- Describe input voltage ranges on ADC pins, power-on sequencing, and any watchdog or error handling incorporated. -->

Blah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blah

## 5.5 Usability Assessment:

<!-- Reflect on calibration steps required, user interface clarity on VGA, and ease of deploying to other Pico boards. -->

Blah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blah

# 6. Conclusions

## 6.1 Design Evaluation & Lessons Learned

<!-- Analyze how results matched expectations, discuss potential improvements (e.g., floating-point vs fixed-point, faster PIO implementations). -->

Blah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blah

## 6.2 Standards Compliance

<!-- Comment on adherence to VGA timing specifications, Pico hardware guidelines, and DSP best practices. -->

Blah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blahBlah Blah blah

# 7. Appendix A (Permissions)

## 7.1 Course Website Inclusion:

The group approves this report for inclusion on the course website.

## 7.2 YouTube Channel Inclusion:

The group approves the video for inclusion on the course YouTube channel.
