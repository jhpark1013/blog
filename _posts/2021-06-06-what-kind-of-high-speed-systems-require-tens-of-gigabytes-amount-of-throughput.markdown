---
title: What kind of high-speed systems require tens of Gigabytes/s amount of throughput?
date: 2021-06-06T15:37:32-04:00
---

What kind of high-speed systems require tens of Gigabytes/s amount of throughput? 10 GB is a lot of data. For context, the bandwidth of a dslr operating at a relatively high-throughput video-mode of 1080p resolution, 12 bit depth, at 30 fps is around 0.093 GB/s. Which is two orders of magnitude less than 10 GB/s.

Example 1: A camera array with 54 13 Megapixel sensors recording at 15 fps produces 10 GB/s. 1 pcie 3.0 x8 interface can handle this amount of data flow.

Example 2: A DMD-camera pair that requires 800 patterns per second (where one pattern is active for 45 us).

The camera just integrates all this in one time so the data throughput is not too bad on this end; The camera integrates light from these 800 projections in 36000 us (0.036 s) exposure i.e. shutter is open for 0.036 seconds. But on the DMD side, extremely fast throughput is needed.

High-speed is needed in the DMD projections.For a binary pattern, 2.8 Megapixels (Megabit), the throughput per second for 28 fps is:

- 800 patterns: 28 frames/s * 2.8 Megabits/pattern * 800 patterns/frame = 62720 E6 = 62 Gbps = 7.75 GBps !!
- 100 patterns: 7840 E6 = 7.8 Gbps = 0.975 GBps

The 2.8 MP comes from the size of an [example sensor](https://thinklucid.com/product/atlas-2-8-mp-imx421/). This can be any number depending on the sensor size. For this example, the sensor has an active area of approximately 1900x1400px (3:4 ratio).

For grayscale, which requires more bit depth, this is at least 8-folds larger.

Example 3: Quantum computers

So what interfaces can support our bandwidth requirements?
  - [pcie](https://en.wikipedia.org/wiki/PCI_Express#History_and_revisions): PCIe 3.0 x8 has a throughput of 15.7 GB/s
  - [usb](https://www.tripplite.com/products/usb-connectivity-types-standards): USB 3.2 gen2 has a throughput of 10 Gbps (1.25 GBps)
  - [GTH and GTY](https://www.xilinx.com/products/technology/high-speed-serial.html): GTH has a throughput of 16 Gbps (2 GBps) on the ultrascale+. Are there usually multiple?? at this point, pcie seems better.
  - assuming there's overhead, we can safely assume 70-80% of the theoretical throughput as the actual-yield throughput.

These interfaces may be necessary for tens of Gigabyte/s throughput or for high-speed applications.
