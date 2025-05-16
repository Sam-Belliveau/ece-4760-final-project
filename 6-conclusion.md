---
layout: default
title: "6. Conclusion"
---

# 6. Conclusions

## 6.1 Design Evaluation & Lessons Learned

Our cross-correlation implementation yielded results significantly better than we initially expected. The system demonstrated remarkable accuracy, particularly when working with our known-height assumption. By fixing the expected sound source height at $1$ meter above the microphone plane, we were able to achieve accurate localization within $0.1$ meters for sources positioned outside the microphone triangle, with reliable direction estimation extending to distances of $3$-$5$ meters depending on the relative height of the audio source.

Our implementation required significant experimentation to tune the algorithm, including determining optimal sample rates, window sizes, buffering strategies, and more. The final implementation resulted in code that was very modular and well-organized. Notably, our design utilized only a single core of the Pico, meaning we still left considerable performance potential untapped.

### Potential Improvements

Several promising avenues exist for further enhancing the system:

- **Wider microphone spacing and external ADC**: Increasing the distance between microphones along with faster ADC sampling via external hardware could dramatically improve resolution and range.
- **Additional microphones**: Expanding beyond three microphones would create redundancy and improve noise rejection while enabling 3D localization.
- **FFT-based correlation**: Implementing frequency-domain correlation calculations could improve computational efficiency for longer sample windows.
- **Dual-core utilization**: Dedicating one core entirely to sampling and correlation while using the other for VGA rendering could improve responsiveness.
- **Enhanced physical filtering**: Adding better analog filtering stages before the ADC to improve signal quality and reduce noise interference.

In terms of the accuracy of our detection, we approached the theoretical limits of what's possible with our hardware. Inherent noise in the microphones and testing environment introduced some inevitable uncertainty in our VGA heatmap and localization.

One major takeaway was that improving our software-like filtering, buffering, and algorithm design-yielded much better results than immediately jumping to faster sampling or wider microphone spacing, which we originally thought were essential for accuracy. We found that hardware upgrades alone wouldn’t have solved our problems without first addressing these core software issues.

Another key lesson was in localization: we initially tried to find the exact sound location from the time shifts, expecting the hyperbolas to always intersect, but mathematically this is not always the case. Creating the VGA heatmap, which we mentioned earlier, turned out to be extremely helpful for debugging and understanding these issues, and was crucial for making our system robust.

## 6.2 Standards Compliance

Our design adhered to the VGA timing specification, utilizing the driver provided as part of the course. This driver complies with the VGA standard, and we utilized the primitives in the driver to implement our VGA plotting. 

We powered the Pico with a standard $3.3\,\text{V}$ USB cable connected to a laptop. We overclocked our design to $250\,\text{MHz}$, falling within the acceptable range of overclocking for the Pico. 

When sampling, we maintained a rate of $50\,\text{kHz}$, well above the Nyquist rate for our microphones, ensuring we didn't need an extra alias filter and following DSP best practices.
