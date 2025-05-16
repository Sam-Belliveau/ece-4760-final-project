---
layout: default
title: "5. Testing and Results of the Design"
---

# 5. Testing and Results of the Design

## 5.1 Test Data & Traces
Although we used an oscilloscope to measure the available compute time per sample at various sample rates and buffer sizes, we did not capture or save screenshots of these traces during development. Instead, we relied on these real-time measurements to iteratively tune our system parameters. By observing the timing between ADC sampling, DMA transfers, and our processing routines on the oscilloscope, we were able to determine the maximum safe sample rate that would allow all computations (including cross-correlation and VGA updates) to complete before the next sample arrived. This process led us to select an optimal sample rate of $50\,\mathrm{kHz}$ with a buffer size of $1024$ samples, balancing latency, noise reduction, and CPU load. While we do not have oscilloscope captures to present here, this hands-on timing analysis was critical in achieving reliable, real-time operation.


<!-- Present oscilloscope captures of ADC sampling timing (`SAMPLE_PERIOD_US`), DMA triggers, and GPIO timing for debugging. -->

## 5.2 Performance Metrics

Our system achieved remarkable real-time performance despite the computational demands. Both the VGA rendering and cross-correlation calculations appeared instantaneous to human perception, with no noticeable lag between making a sound and seeing the corresponding heatmap update. The seamless responsiveness created a natural interaction experience, where users could make repeated sounds from different locations and immediately observe the system tracking these changes.

This smooth performance was achieved through careful optimization of our processing pipeline, balancing sample rate, buffer size, and display update frequency. By prioritizing full heatmap updates only for significant acoustic events while using lightweight updates for continuous monitoring, we maintained consistent responsiveness even under full computational load.

## 5.3 Accuracy Metrics

Our localization system demonstrated remarkable precision, particularly within the triangular area formed by the three microphones where accuracy was consistently within a few millimeters. This precision can be attributed to the higher density of hyperbola intersections in this region, as illustrated in our theoretical analysis.

The system's accuracy exhibited predictable characteristics with distance:

- **Direction estimation** remained nearly perfect even at larger ranges, allowing reliable angular positioning
- **Distance estimation** accuracy decreased exponentially with distance from the array
- **Height variations** affected perceived distance, sound sources higher or lower than our expected 1-meter plane appeared closer or farther respectively

These characteristics are clearly demonstrated in our demo video:

<div class="video-timestamps">
  <p><a href="https://www.youtube.com/watch?v=yFkt5Urp-eg&t=517s" target="_blank"><strong>8:37</strong></a> — A speaker playing music positioned approximately 0.75 meters down and to the right of the microphone array center is accurately located on the heatmap, with both direction and distance correctly represented.</p>
  
  <p><a href="https://www.youtube.com/watch?v=yFkt5Urp-eg&t=523s" target="_blank"><strong>8:43</strong></a> — The same speaker moved to the opposite side of the microphones is again accurately localized, demonstrating the system's consistent performance regardless of direction.</p>
</div>

For most practical applications, we found the system performed optimally within a radius of about 2 meters from the array center. Beyond this distance, while direction remained reliable, absolute position became less precise, still suitable for most acoustic tracking applications where angular information is the primary concern.

In noisy environments, the system typically responds to the loudest sound source, but performance can be affected when multiple loud sounds compete for detection. The cross-correlation algorithm may occasionally yield suboptimal results when confronted with multiple simultaneous audio signals of comparable amplitude. During testing, we found that a portable speaker providing continuous, high-power audio output created ideal conditions for consistent localization, as it maintained a dominant signal-to-noise ratio. This allowed our system to reliably track the speaker even in moderately noisy environments, demonstrating its practical utility in real-world settings.

## 5.4 Safety & Robustness

Our system has effectively no major safety considerations, operating at low voltages and currents. The input voltage range for the ADC pins is $0$-$3.3\text{V}$. 

Our microphones do not draw too much power and we comply with all safety standards and utilize good wiring practices. On power-on, the Pico executes its power-on state machine and then starts executing the code. 

The code is flashed to the Pico, making the system robust to power outs. Additionally, our system is also robust to the power off of one microphone. Our core logic continues to work, however we see an arc like heatmap, resulting from the loss of samples from the third microphone in the cross correlation logic. 

With only two active microphones, the system can only generate a single hyperbolic curve of possible sound locations, resulting in a hyperbolic-shaped heatmap. This behavior is demonstrated at <a href="https://www.youtube.com/watch?v=yFkt5Urp-eg&t=245s" target="_blank"><strong>4:05</strong></a> in our demo video, where we intentionally disconnect one microphone to show the system's graceful degradation.

## 5.5 Usability Assessment

Overall, our implemented system was quite usable and easy to set up. To set up the system, the user simply needs to power the Pico and to connect the wires for the VGA interface. Once powered on, the system is very easy to use. 

Simply by making a sound (such as a snap, clap, or whistle), the user can almost instantly see the heatmap plotted on the VGA display. The only non-intuitive aspect of using the system is knowing the VGA pin wiring to the connector. 

Based on the plot, one can orient themselves relative to the microphones on the board, allowing the user to understand the plot without prior knowledge of how the microphones are oriented. 

One of the most compelling aspects of our system is the ability to make repeated noises from different locations. By moving around, the user intuitively can watch the system plot their new location.

An assumption our triangulation makes is that the noise is coming from roughly 1 meter above the microphone. This means the best performance is achieved by placing the board on the floor. 

This is not an obvious thing, but does not significantly affect the triangulation and the effect the user sees. To set up the system with a new Pico, one would simply flash the code and place the Pico on the breadboard. This greatly simplifies setup, as there is nothing that needs to be inputted from a host computer to setup the Pico. 

There are some parameters hard-coded in the code which can be modified, such as the distance between the microphones, the sample rate and `MAX_SHIFT_SAMPLES` parameters among a few others than sometimes required calibration for the best performance. Nonetheless, the system is fully usable without calibration. 
