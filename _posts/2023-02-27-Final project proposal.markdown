---
layout: post
title:  "Final project proposal"
date:   2023-02-27 10:20:00 -0800
categories: jekyll update
---

## Demo
I would like to attempt to make a cheap tiny arcade box that plays Pong (clone) on the 16x2 character LCD display. This would be a “spiritual successor(ancestor)” to the first (maybe second) game I ever made which was a clone of Breakout written in processing. The difference is I will be using much more limited hardware this time.

## Timeline
- 2/28	Feasibility & LCD stress test
- 3/1	Connect joysticks to the system
- 3/3	Implement graphics code
- 3/5	Implement game logic
- 3/6	Package hardware into a box
- 3/8	Documentation
- 3/9	Presentation

## Anticipated Bill of Materials 
- 16x2 character LCD
- Arduino
- Resistor
- Cardboard box
- 2 Joysticks

## Backup plans
If feasibility testing & LCD stress test fails or other significant issues are encountered before 3/4, I might pivot towards a security system project that combines motion sensor/ultrasonic sensor with LEDs.

## Anticipated issues
Based on preliminary research/testing, I have identified the following issues that will need to be resolved:
- Sprites & graphics will have to be emulated in software with 8 custom characters, potentially causing significant slowdowns.
- There’s no hardware scrolling for graphics so that will also need to be done in software.
