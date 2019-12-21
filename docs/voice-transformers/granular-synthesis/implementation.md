# Implementation

We are building a real-time system and so the output data rate will necessarily be equal to the input data rate. Since grains are produced at a rate which is the inverse of the stride length, it would make perfect sense to set the length of the DMA buffer equal to the stride.

Unfortunately this simple approach clashes with the capabilities of the hardware and so we need to trade resources for code complexity: welcome to the world of embedded DSP!

### Memory limitations

If we play around with the Jupyter notebook implementation of granular synthesis, we can quickly verify that the voice changer works best with a grain length of about 30ms and an overlap factor of about 50%. Using the formula derived in the previous section, this gives us a grain stride of 20ms.

Now, remember that the smallest sampling frequency of our digital microphone is 32KHz so that 20ms correspond to 640 samples. Each sample is 2 bytes and the I2S protocol requires us to allocate a stereo buffer. This means that each DMA _half_-buffer will be 

$$
2 * 2 * 640 = 2560 \rm{~bytes.}
$$

Since we need to use double buffering for DMA, and since we need symmetric input and output buffers, in total we will need to allocate **over 10KB of RAM to the DMA buffers alone**; when we start adding the internal buffering required for computation, **we are going to quickly exceed the 16KB available on the Nucleo F072RB!**

\(As a side note, although 16KB may seem ludicrously low these days, remember that small memory footprints are absolutely essential for all devices that are not a personal computer. The success of IoT hinges upon low memory and low power consumption!\)

### Tricks of the trade

To avoid the need of large DMA buffers, we will implement granular synthesis using the following tricks:

* we will use a single internal circular buffer that holds enough data to build the grains
* we will fill the internal buffer with short DMA input transfers and compute a corresponding amount of output samples for each DMA call
* to save memory, all processing will be carried out on a mono signal
* we will use a "smart choice" for the size of the grain, the tapering and the DMA transfer, so as to minimize processing





Consider once again the following figure

When we are computing the output grain number $$k$$, what is the input range that we need to access? The index within the grain goes from zero to $$S$$and, when $$m=0$$we need to compute

$$
g_{k-1}[S] = x(kS + \alpha S - S)
$$

whereas when $$m=S$$we need to compute

$$
g_k[S] = x(kS + \alpha S)
$$

At all times, therefore, we need to have access to exactly $$S$$input samples. 

