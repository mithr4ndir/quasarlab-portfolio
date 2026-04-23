# HDR flicker on 5K2K monitor was a cable problem

## Date

Resolved 2026-04-17

## Impact

Intermittent HDR flicker on a Zephyrus G16 driving an LG UltraGear+ 5K2K
display. Burned real engineering time chasing a software cause.

## Duration

TODO

## Category

hardware, display

## Summary

HDR flicker was not a driver, firmware, or color-space bug. It was a
marginal passive DisplayPort-to-USB-C cable. Swapping to a proper HDMI
2.1 cable fixed it cleanly.

## Background

TODO describe the setup: laptop, dock/cable, monitor, HDR mode, refresh
rate, bit depth.

## What happened

TODO describe the flicker pattern and the conditions that reproduced it.

## Investigation

TODO list the things that were ruled out: driver versions, color
profiles, refresh rate, HDR metadata, kernel options.

## Root cause

Passive DP-to-USB-C was marginal at the bandwidth required for 5K2K HDR.
The signal was technically in spec but close enough to the edge that
HDR metadata bursts caused visible flicker.

## Fix

Swapped the cable for an HDMI 2.1 cable rated for the full bandwidth.
Flicker disappeared immediately.

## Takeaways

- When a high-bandwidth display protocol misbehaves, suspect the cable
  before the software. Especially at 5K2K + HDR + high refresh.
- Passive adapters quietly cap at lower link rates than their marketing
  suggests.

## Follow-ups

- [ ] Note the specific cable model that works, once confirmed stable
      over a longer window.
