---
layout: default
title: "1. Introduction"
---

# 1. Project Introduction

## 1.1 Summary

In this project, we designed a real-time sound localization system using a Raspberry Pi Pico and three microphones. Audio from each of the MEMS microphones was continuously sampled at \($50\,\mathrm{kHz}$\) via the Pico’s DMA engine.

Each channel’s data is then streamed into a rolling buffer. When the energy in the buffer reaches a certain threshold, the software performs cross-correlations between the microphone pairs to estimate the time differences between the audio arrivals.

These estimates are averaged over time to reduce the impact of noise. Using the delay estimates, a heat map is drawn by comparing the expected delays from each location to the estimates calculated by our cross-correlation.

The entire processing chain-from sampling to buffering, correlation, and VGA drawing-executes on the Raspberry Pi Pico.

## 1.2 Motivation

Our motivation was a combination of passion for signal processing and demonstrating the capabilities of low-cost hardware in audio processing. The localization system has many applications, from robotics to art installations. In a world in which there is an increasing number of applications requiring many cheap distributed sensors, there is a need for the development of algorithms capable of running on cheap, low power, hardware. 

Our project implements audio localization on system whose total BOM is under $20, with a localization accuracy comparable to much more expensive systems. The algorithms is run purely on the Pico, eliminating the need for a dedicated DSP chip. Combined with the Pico’s PIO VGA output, this results in an all-in-one package for sound localization.
