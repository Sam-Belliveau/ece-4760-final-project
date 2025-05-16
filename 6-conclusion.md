---
layout: default
title: "6. Conclusion"
---

# 6. Conclusions

## 6.1 Design Evaluation & Lessons Learned

Our cross-correlation implementation yielded results significantly better than we initially expected. The system demonstrated remarkable accuracy, particularly when working with our known-height assumption. By fixing the expected sound source height at 1 meter above the microphone plane, we were able to achieve accurate localization within 0.1 meters for sources positioned outside the microphone triangle, with reliable direction estimation extending to distances of 3-5 meters depending on the relative height of the audio source.

Our implementation required significant experimentation to tune the algorithm, including determining optimal sample rates, window sizes, buffering strategies, and more. The final implementation resulted in code that was very modular and well-organized. Notably, our design utilized only a single core of the Pico, meaning we still left considerable performance potential untapped.

### Potential Improvements

Several promising avenues exist for further enhancing the system:

- **Wider microphone spacing and external ADC**: Increasing the distance between microphones along with faster ADC sampling via external hardware could dramatically improve resolution and range.
- **Additional microphones**: Expanding beyond three microphones would create redundancy and improve noise rejection while enabling 3D localization.
- **FFT-based correlation**: Implementing frequency-domain correlation calculations could improve computational efficiency for longer sample windows.
- **Dual-core utilization**: Dedicating one core entirely to sampling and correlation while using the other for VGA rendering could improve responsiveness.
- **Enhanced physical filtering**: Adding better analog filtering stages before the ADC to improve signal quality and reduce noise interference.

In terms of the accuracy of our detection, we approached the theoretical limits of what's possible with our hardware. Inherent noise in the microphones and testing environment introduced some inevitable uncertainty in our VGA heatmap and localization.

Overall, the project taught us a great deal about audio DSP and efficient implementation. A major takeaway is the power of software-hardware codesign—modifying our algorithms to match the hardware capabilities led to a remarkably performant system despite the limited resources of the Pico.

## 6.2 Standards Compliance

<!-- Comment on adherence to VGA timing specs, Pico hardware guidelines, and DSP best practices. -->
Our design adhered to the VGA timing specification, utilizing the driver provided as part of the course. This driver complies with the VGA standard, and we utilized the primitives in the driver to implement our VGA plotting. 

We powered the Pico with a standard $3.3\text{V}$ USB cable connected to a laptop. We overclock our design to $250 \text{MHz}$, falling within the acceptable range of overlocking for the Pico. 

When sampling, we sampled above the Nyquist rate for our microphones, ensuring we didn't need an extra alias filter and following DSP best practices. 
