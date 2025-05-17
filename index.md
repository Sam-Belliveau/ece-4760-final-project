---
layout: default
title: "0. Homepage"
permalink: /index.html
---

# We've created a $20 acoustic camera that transforms sound delays into visual heatmaps, revealing where noises originate in real-time.
{: .no_toc .flex-justify-around .text-center }

#### _Sam Belliveau - srb343_
{: .no_toc .flex-justify-around .text-center }

#### _Ezra Riess - er495_
{: .no_toc .flex-justify-around .text-center }

#### _Ari Kapelyan - alk246_
{: .no_toc .flex-justify-around .text-center }

---

<div style="display:flex; align-items:flex-center; gap:1rem; margin-left: auto; margin-right: auto;">
  <iframe width="720px" height="405px" src="https://www.youtube.com/embed/yFkt5Urp-eg" frameborder="1" allowfullscreen class="full-bordered"></iframe>
</div>

<div class="video-timestamps" style="text-align: center; margin-top: 20px; margin-bottom: 30px;">
  <h3 style="margin-bottom: 15px;">Video Timeline</h3>
  <div style="display: flex; flex-direction: column; gap: 10px; max-width: 400px; margin: 0 auto; border: 1px solid #ddd; padding: 15px; border-radius: 5px; background-color: #f9f9f9;">
    <a href="https://www.youtube.com/watch?v=yFkt5Urp-eg&t=0s" target="_blank" style="text-decoration: none; display: flex; justify-content: space-between;">
      <span style="font-weight: bold;">0:00</span>
      <span style="flex-grow: 1; margin-left: 15px; text-align: left;">Project Motivation</span>
    </a>
    <a href="https://www.youtube.com/watch?v=yFkt5Urp-eg&t=80s" target="_blank" style="text-decoration: none; display: flex; justify-content: space-between;">
      <span style="font-weight: bold;">1:20</span>
      <span style="flex-grow: 1; margin-left: 15px; text-align: left;">Mathematical Analysis</span>
    </a>
    <a href="https://www.youtube.com/watch?v=yFkt5Urp-eg&t=335s" target="_blank" style="text-decoration: none; display: flex; justify-content: space-between;">
      <span style="font-weight: bold;">5:35</span>
      <span style="flex-grow: 1; margin-left: 15px; text-align: left;">Audio Processing & Optimizations</span>
    </a>
    <a href="https://www.youtube.com/watch?v=yFkt5Urp-eg&t=504s" target="_blank" style="text-decoration: none; display: flex; justify-content: space-between;">
      <span style="font-weight: bold;">8:24</span>
      <span style="flex-grow: 1; margin-left: 15px; text-align: left;">Final Demo</span>
    </a>
  </div>
</div>

---

# Appendix A (Permissions)

## Course Website Inclusion

The group approves this report for inclusion on the course website.

## YouTube Channel Inclusion

The group approves the video for inclusion on the course YouTube channel.

# Appendix B (Code Repository)

The complete source code for this project is available on GitHub:

<div style="text-align: left; margin-top: 10px; margin-bottom: 10px;">
  <a href="https://github.com/Sam-Belliveau/Audio-Triangulation" target="_blank" style="font-size: 1.2em; font-weight: bold;">
    GitHub: Audio-Triangulation
  </a>
</div>

# Appendix C (Individual Contributions)

## Ezra Riess (er495)
- **Hardware Design**: Assembled the breadboard with right-triangular microphone positioning, implemented the second-order low-pass filters, and minimized wiring length to reduce interference
- **DMA Configuration**: Designed and implemented the DMA-driven sampling system with continuous ping-pong DMA transfers for steady $50\,\mathrm{kHz}$ sampling
- **ADC Setup**: Configured the ADC channels in round-robin mode for the three microphones with FIFO enabled
- **Core Correlation Algorithm**: Developed the cross-correlation engine in `correlations.c` for time-delay estimation between microphone pairs
- **Geometric Calculations**: Worked on the microphone geometry calculation algorithms in `microphones.c`

## Sam Belliveau (srb343)
- **Audio Processing Pipeline**: Designed and optimized the audio processing chain from raw samples to actionable signals
- **Rolling Buffer System**: Implemented the circular buffer system in `rolling_buffer.c` with power calculation for event detection
- **Power Calculation Algorithms**: Developed the energy-based event detection system that compares incoming vs. outgoing buffer energy
- **Window Functions**: Designed and optimized the windowing functions for improved signal quality
- **Exponential Moving Average**: Implemented `correlations_average()` for temporal smoothing of correlation results
- **Algorithm Tuning**: Performed detailed experimentation to find optimal buffer sizes, sampling rates, and filtering parameters

## Ari Kapelyan (alk246)
- **VGA Visualization**: Created the visualization system for the heatmap display
- **Display Subsystem**: Implemented the `vga_draw()` and `vga_draw_lite()` functions to balance visual feedback with performance
- **Heatmap Generation**: Developed the algorithm to convert correlation data into a visual heatmap
- **Waveform Display**: Created the waveform visualization for real-time audio monitoring
- **User Interface Elements**: Implemented correlation curves and markers for intuitive feedback
- **Performance Optimization**: Designed the dual-mode rendering system that uses lightweight updates for continuous monitoring


# Appendix D (References & AI Declaration)

## References
Our project was primarily based on knowledge acquired in the ECE 4760 course, including lectures, labs, and course materials. We also referenced the Raspberry Pi Pico SDK documentation and the VGA library documentation.

## AI Declaration
In the development of this project:

- We consulted AI (Claude, ChatGPT) to explore the feasibility of different optimization techniques and algorithmic approaches, particularly regarding signal processing methods and cross-correlation implementations.

- We used code assistance tools including Cursor and GitHub Copilot during implementation, which helped with auto-completing code lines and suggesting function signatures.

- Besides these tools all implementation ideas came from different group members' prior knowledge. We did not reference any specific papers or websites when making this project. 


