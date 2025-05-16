---
layout: default
title: "6. Conclusion"
---

# 6. Conclusions

## 6.1 Design Evaluation & Lessons Learned

<!-- Analyze how results matched expectations, discuss potential improvements (e.g., fixed-point vs floating-point, faster PIO implementations). -->

In our design, our cross-correlation design yielded a result significantly better than we expected. As compared to an FFT based approach, our cross-correlation approach, combined with some of our additional filtering techniques, performed in a comparable fashion. 

By implementing everything in fixed-point arithmetic, we were able to actually achieve very fast localization and rendering. In terms of DSP techniques, we ended up intuiting a lot about the dynamics of the room. For example, we realized that the Gaussian Smoothing and Windowing were required in order to minimize picking up on noise and to better track movement.

In our implementation, we had to do significant experimentation to tune our algorithm, be it determining the sample rate, window size, buffering, and so on. Overall our final implementation resulted in code which was very modular and well organized. The design we utilized in the end only relied on a single core of the Pico, meaning we still left a fair bit of performance on the table. Nonetheless, our system was fast and performative, and made for a compelling demonstration of the underlying math. 

In terms of the accuracy of our detection, we likely got somewhere close to the theoretical limit of what we could do. There is inherent noise in the microphones, as well as in the room when we were testing. As a result, there is still some noise present in our VGA heatmap and localization. 

Overall, the project taught us a lot about DSP on audio and efficient implementation. A major takeaway we have is the power of these microcontrollers and software-hardware codesign. It was very interesting to observe how modifying our algorithm to match the hardware led to such a performant system. 

## 6.2 Standards Compliance

<!-- Comment on adherence to VGA timing specs, Pico hardware guidelines, and DSP best practices. -->
Our design adhered to the VGA timing specification, utilizing the driver provided as part of the course. This driver complies with the VGA standard, and we utilized the primitives in the driver to implement our VGA plotting. 

We powered the Pico with a standard $3.3\text{V}$ USB cable connected to a laptop. We overclock our design to $250 \text{MHz}$, falling within the acceptable range of overlocking for the Pico. 

When sampling, we sampled above the Nyquist rate for our microphones, ensuring we didn't need an extra alias filter and following DSP best practices. 
