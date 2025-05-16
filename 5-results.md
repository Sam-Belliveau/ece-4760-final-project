---
layout: default
title: "5. Testing and Results of the Design"
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
