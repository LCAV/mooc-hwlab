# Implementation

We are building a real-time system and so the output data rate will necessarily be equal to the input data rate. Since grains are produced at a rate which is the inverse of the stride length, it would make perfect sense to set the length of the DMA buffer equal to the stride and perform all necessary buffering in the processing code.

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


