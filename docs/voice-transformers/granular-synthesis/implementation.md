# Initial Implementation

We are building a real-time system and so the output data rate will necessarily be equal to the input data rate. In the previous section we saw that grains are produced via a periodic pattern whose period is equal to the stride length. It would make perfect sense, therefore, to set the length of the DMA buffer equal to the stride and let that be the cadence of the processing function.

Unfortunately this simple approach clashes with the capabilities of the hardware and so we need to trade resources for some extra code complexity: welcome to the world of embedded DSP!

## Memory limitations

If we play around with the Jupyter notebook implementation of granular synthesis, we can quickly verify that the voice changer works best with a grain length of about 30ms and an overlap factor of about 50%. Using the formula derived in the previous section, this gives us a grain stride of 20ms.

Now, remember that the smallest sampling frequency of our digital microphone is 32KHz so that 20ms correspond to 640 samples. Each sample is 2 bytes and the I2S protocol requires us to allocate a stereo buffer. This means that each DMA _half_-buffer will be 

$$
2 * 2 * 640 = 2560 \rm{~bytes.}
$$

Since we need to use double buffering for DMA, and since we need symmetric input and output buffers, in total we will need to allocate **over 10KB of RAM to the DMA buffers alone**; when we start adding the internal buffering required for computation, **we are going to quickly exceed the 16KB available on the Nucleo F072RB!**

\(As a side note, although 16KB may seem ludicrously low these days, remember that small memory footprints are absolutely essential for all devices that are not a personal computer. The success of IoT hinges upon low memory and low power consumption!\)

To avoid the need of large DMA buffers, we will implement granular synthesis using the following tricks:

* to save memory, all processing will be carried out on a mono signal;
* we will use a single internal circular buffer that holds enough data to build the grains; we have seen in the previous section that we need a buffer at most as long as the grain. Using mono samples, this will require a length of 1024 samples, for a memory footprint of 2 KBytes.
* we will fill the internal buffer with short DMA input transfers and compute a corresponding amount of output samples for each DMA call; DMA transfers can be as short as 16 or 32 samples each, thereby reducing the amount of memory required by the DMA buffers.
* we will use a "smart choice" for the size of the grain, the tapering and the DMA transfer, so as to minimize processing


## The code

To code the granular synthesis algorithm, copy and paste the Alien Voice project from within the STM32CubeIDE environment. We recommend choosing a name with the current date and `"granular_syn"` in it. Remember to delete the old binary \(ELF\) file inside the copied project.

Here, we will set up the code for the "Darth Vader" voice transformer and will consider more advanced modifications in the next section.


### DMA size

As we explained, the idea is to fill the main audio buffer in small increments to save memory. To this end, set the DMA half-buffer size to 32 samples in the `USER CODE BEGIN PV` section:

```c
#define FRAMES_PER_BUFFER 32    
```


### Grain size and taper
We will use a grain length of $$L=1024$$ samples which corresponds to about 30ms for a sampling rate of 32KHz. The overlap is set at 50%, i.e., we will use a tapering slope of $$W=384$$ samples. The resulting grain stride is $$S=640$$.

{% hint style="info" %}
TASK 1: Write a short Python function that returns the values of a tapering slope for a given length.
{% endhint %}

Add the following lines to the `USER CODE BEGIN 4` section in `main.c`, where the values for the tapering slope are those computed by your Python function:

```c
// grain length; 1024 samples correspond to 32ms @ 32KHz
#define GRAIN_LEN 1024
// length of the tapering slope using 50% overlap
#define TAPER_LEN 384
#define GRAIN_STRIDE (GRAIN_LEN - TAPER_LEN)

// tapering slope, from 0 to 1 in TAPER_LEN steps
static int32_t TAPER[TAPER_LEN] = {...};
```

### Main buffer

We choose the buffer length to be equal to the size of the grain, since the voice transformer doesn't sound good for $$\alpha > 1.5$$ anyway. With the size equal to a power of two, we will be able to use bit masking to enforce circular access to the buffer. Add the following lines after the previous ones:

```c
#define BUF_LEN 1024
#define BUFLEN_MASK (BUF_LEN-1)
static int16_t buffer[BUF_LEN];

// input index for inserting DMA data
static uint16_t buf_ix = 0;
// index to beginning of previous grain
static uint16_t prev_ix = 0;
// index to beginning of current grain
static uint16_t curr_ix = GRAIN_STRIDE;
// index of sample within grain
static uint16_t grain_m = 0;
```

With these values the buffers are set up for causal operation (i.e., for lowering the voice pitch); we will tackle the problem of noncausal operation later.

You can now examine the memory footprint of the application by compiling the code and looking at the "Build Analyzer" tab on the lower right corner of the IDE. You should see that we are only using less than 30% of the onboard RAM.


### Processing function

This is the main processing function:

```c
inline static void VoiceEffect(int16_t *pIn, int16_t *pOut, uint16_t size) {
  // put LEFT channel samples to mono buffer
  for (int n = 0; n < size; n += 2) {
    buffer[buf_ix++] = pIn[n];
    buf_ix &= BUFLEN_MASK;
  }

  // compute output samples
  for (int n = 0; n < size; n += 2) {
    // sample from current grain
    int16_t y = Resample(grain_m, curr_ix);
    // if we are in the overlap zone, compute sample from previous grain and mix using tapering slope
    if (grain_m < TAPER_LEN) {
      int32_t z = Resample(grain_m + GRAIN_STRIDE, prev_ix) * (0x07FFF - TAPER[grain_m]);
      z += y * TAPER[grain_m];
      y = (int16_t)(z >> 15);
    }
	// put sample into both LEFT and RIGHT output slots
    pOut[n] = pOut[n+1] = y;
	// update index inside grain; if we are at the end of the stride, update buffer indices
    if (++grain_m >= GRAIN_STRIDE) {
      grain_m = 0;
      prev_ix = curr_ix;
      curr_ix = (curr_ix + GRAIN_STRIDE) & BUFLEN_MASK;
    }
  }
}
```

The processing loop uses an auxiliary function `Resample(uint16_t m, uint16_t start)` that is supposed to return the interpolated value $$x(start + \alpha m)$$. 

A simplistic implementation is to return the sample with integer index closest to $$ix + \alpha m$$:

```c
// rate change factor
static int32_t alpha = (int32_t)(0x7FFF * 2.0 / 3.0);

inline static int16_t Resample(uint16_t m, uint16_t start) {
  // non-integer index
  int32_t t = alpha * (int32_t)m;
  int16_t T = (int16_t)(t >> 15) + (int16_t)start;
  return buffer[T & BUFLEN_MASK];
}
```

{% hint style="info" %}
TASK 2: Write version of `Resample()` that performs proper linear interpolation between neighboring samples. 
{% endhint %}


## Benchmarking

Since our processing function is becoming a bit more complex than before, it is interesting to start benchmarking its performance. 

Remember that, at 32KHz, we can use at most $$30\mu s$$ per sample; we can modify the timing function to return the number of microseconds per sample like so: 

```c
#define STOP_TIMER {\
  timer_value_us = 1000 * __HAL_TIM_GET_COUNTER(&htim2) / FRAMES_PER_BUFFER;\
  HAL_TIM_Base_Stop(&htim2); }
```

If we now use the method of the benchmarking live section, we can see that the current implementation (with the full fractional resampling code) requires between $$5.2\mu s$$ and $$8.5\mu s$$ per sample, which is  well below the limit. The oscillation between the two values reflects the larger computational requirements of tapering slope.


## **Solutions**

{% tabs %}
{% tab title="Anti-spoiler Tab" %}
Are you ready to see the answer? :\)
{% endtab %}

{% tab title="Task 1" %}
```python
def make_taper(W):
    taper = 32767.0 * np.arange(0, W) / W
    print("#define TAPER_LEN {}".format(W))
    print("static int32_t TAPER[TAPER_LEN] = {", end='\n\t')
    for n in range(W - 1):
        print('0x{:04X}, '.format(np.uint16(taper[n])), end='' + '\n\t' if (n+1) % 12 == 0 else '')
    print('0x{:04X}}};'.format(np.uint16(taper[W-1]))){% endtab %}
```

With $$W=384$$ the resulting table is

```c
static int32_t TAPER[TAPER_LEN] = {
  0x0000, 0x0055, 0x00AA, 0x00FF, 0x0155, 0x01AA, 0x01FF, 0x0255, 0x02AA, 0x02FF, 0x0355, 0x03AA,
  0x03FF, 0x0455, 0x04AA, 0x04FF, 0x0555, 0x05AA, 0x05FF, 0x0655, 0x06AA, 0x06FF, 0x0755, 0x07AA,
  0x07FF, 0x0855, 0x08AA, 0x08FF, 0x0955, 0x09AA, 0x09FF, 0x0A55, 0x0AAA, 0x0AFF, 0x0B55, 0x0BAA,
  0x0BFF, 0x0C55, 0x0CAA, 0x0CFF, 0x0D55, 0x0DAA, 0x0DFF, 0x0E55, 0x0EAA, 0x0EFF, 0x0F55, 0x0FAA,
  0x0FFF, 0x1055, 0x10AA, 0x10FF, 0x1155, 0x11AA, 0x11FF, 0x1255, 0x12AA, 0x12FF, 0x1355, 0x13AA,
  0x13FF, 0x1455, 0x14AA, 0x14FF, 0x1555, 0x15AA, 0x15FF, 0x1655, 0x16AA, 0x16FF, 0x1755, 0x17AA,
  0x17FF, 0x1855, 0x18AA, 0x18FF, 0x1955, 0x19AA, 0x19FF, 0x1A55, 0x1AAA, 0x1AFF, 0x1B55, 0x1BAA,
  0x1BFF, 0x1C55, 0x1CAA, 0x1CFF, 0x1D55, 0x1DAA, 0x1DFF, 0x1E55, 0x1EAA, 0x1EFF, 0x1F55, 0x1FAA,
  0x1FFF, 0x2055, 0x20AA, 0x20FF, 0x2155, 0x21AA, 0x21FF, 0x2255, 0x22AA, 0x22FF, 0x2355, 0x23AA,
  0x23FF, 0x2455, 0x24AA, 0x24FF, 0x2555, 0x25AA, 0x25FF, 0x2655, 0x26AA, 0x26FF, 0x2755, 0x27AA,
  0x27FF, 0x2855, 0x28AA, 0x28FF, 0x2955, 0x29AA, 0x29FF, 0x2A55, 0x2AAA, 0x2AFF, 0x2B54, 0x2BAA,
  0x2BFF, 0x2C54, 0x2CAA, 0x2CFF, 0x2D54, 0x2DAA, 0x2DFF, 0x2E54, 0x2EAA, 0x2EFF, 0x2F54, 0x2FAA,
  0x2FFF, 0x3054, 0x30AA, 0x30FF, 0x3154, 0x31AA, 0x31FF, 0x3254, 0x32AA, 0x32FF, 0x3354, 0x33AA,
  0x33FF, 0x3454, 0x34AA, 0x34FF, 0x3554, 0x35AA, 0x35FF, 0x3654, 0x36AA, 0x36FF, 0x3754, 0x37AA,
  0x37FF, 0x3854, 0x38AA, 0x38FF, 0x3954, 0x39AA, 0x39FF, 0x3A54, 0x3AAA, 0x3AFF, 0x3B54, 0x3BAA,
  0x3BFF, 0x3C54, 0x3CAA, 0x3CFF, 0x3D54, 0x3DAA, 0x3DFF, 0x3E54, 0x3EAA, 0x3EFF, 0x3F54, 0x3FAA,
  0x3FFF, 0x4054, 0x40AA, 0x40FF, 0x4154, 0x41AA, 0x41FF, 0x4254, 0x42AA, 0x42FF, 0x4354, 0x43AA,
  0x43FF, 0x4454, 0x44AA, 0x44FF, 0x4554, 0x45AA, 0x45FF, 0x4654, 0x46AA, 0x46FF, 0x4754, 0x47AA,
  0x47FF, 0x4854, 0x48AA, 0x48FF, 0x4954, 0x49AA, 0x49FF, 0x4A54, 0x4AAA, 0x4AFF, 0x4B54, 0x4BAA,
  0x4BFF, 0x4C54, 0x4CAA, 0x4CFF, 0x4D54, 0x4DAA, 0x4DFF, 0x4E54, 0x4EAA, 0x4EFF, 0x4F54, 0x4FAA,
  0x4FFF, 0x5054, 0x50AA, 0x50FF, 0x5154, 0x51AA, 0x51FF, 0x5254, 0x52AA, 0x52FF, 0x5354, 0x53AA,
  0x53FF, 0x5454, 0x54AA, 0x54FF, 0x5554, 0x55A9, 0x55FF, 0x5654, 0x56A9, 0x56FF, 0x5754, 0x57A9,
  0x57FF, 0x5854, 0x58A9, 0x58FF, 0x5954, 0x59A9, 0x59FF, 0x5A54, 0x5AA9, 0x5AFF, 0x5B54, 0x5BA9,
  0x5BFF, 0x5C54, 0x5CA9, 0x5CFF, 0x5D54, 0x5DA9, 0x5DFF, 0x5E54, 0x5EA9, 0x5EFF, 0x5F54, 0x5FA9,
  0x5FFF, 0x6054, 0x60A9, 0x60FF, 0x6154, 0x61A9, 0x61FF, 0x6254, 0x62A9, 0x62FF, 0x6354, 0x63A9,
  0x63FF, 0x6454, 0x64A9, 0x64FF, 0x6554, 0x65A9, 0x65FF, 0x6654, 0x66A9, 0x66FF, 0x6754, 0x67A9,
  0x67FF, 0x6854, 0x68A9, 0x68FF, 0x6954, 0x69A9, 0x69FF, 0x6A54, 0x6AA9, 0x6AFF, 0x6B54, 0x6BA9,
  0x6BFF, 0x6C54, 0x6CA9, 0x6CFF, 0x6D54, 0x6DA9, 0x6DFF, 0x6E54, 0x6EA9, 0x6EFF, 0x6F54, 0x6FA9,
  0x6FFF, 0x7054, 0x70A9, 0x70FF, 0x7154, 0x71A9, 0x71FF, 0x7254, 0x72A9, 0x72FF, 0x7354, 0x73A9,
  0x73FF, 0x7454, 0x74A9, 0x74FF, 0x7554, 0x75A9, 0x75FF, 0x7654, 0x76A9, 0x76FF, 0x7754, 0x77A9,
  0x77FF, 0x7854, 0x78A9, 0x78FF, 0x7954, 0x79A9, 0x79FF, 0x7A54, 0x7AA9, 0x7AFF, 0x7B54, 0x7BA9,
  0x7BFF, 0x7C54, 0x7CA9, 0x7CFF, 0x7D54, 0x7DA9, 0x7DFF, 0x7E54, 0x7EA9, 0x7EFF, 0x7F54, 0x7FA9};
```
{% endtab %}

{% tab title="Task 2" %}
Here is the complete resampling function:

```c
inline static int16_t Resample(uint16_t m, uint16_t start) {
  // non-integer index
  int32_t t = alpha * (int32_t)m;
  // anchor sample
  int16_t T = (int16_t)(t >> 15) + (int16_t)start;
  // fractional part
  int32_t tau = t & 0x07FFF;
  // compute linear interpolation
  int32_t y = (0x07FFF - tau) * buffer[T & BUFLEN_MASK] + tau * buffer[(T+1) & BUFLEN_MASK];
  return (int16_t)(y >> 15);
}
```

{% endtab %}

{% endtabs %}

