# Basic implementation

Assuming you have successfully implemented the [passthrough](../../audio-peripherals/passthrough/), you can simply copy and paste that project from within the STM32CubeIDE environment. We recommend choosing a name with the current date and `"alien_voice"` in it. Remember to delete the old binary \(ELF\) file inside the copied project.

## Lookup table

Remember that in the passthrough we set up the system to work with a sampling frequency of 32KHz and a sample precision of 16 bits. Here we will use a modulation frequency of 400Hz for the frequency shifting, so that the digital frequency is a rational multiple of $$2\pi$$as in the [previous example](../../real-world-dsp/code-efficiency.md#lookup):

$$
\omega_c = 2\pi\frac{400}{32,000}=\frac{2\pi}{80}\approx 0.0785398
$$

The values for one period of the digital sinusoid can be encoded in a [circular lookup table](../../real-world-dsp/code-efficiency.md#lookup) of length 80, where each element is \(in 16-bit precision\)

```c
cos_table[n] = (int16_t)(32767.0 * cos(0.0785398 * n));
```

The  lookup table is provided here for your convenience. Begin by copying it between the `USER CODE BEGIN PV` and `USER CODE END PV` comment tags.

```c
#define COS_400_32_SIZE 80
const int16_t COS_400_32[COS_400_32_SIZE ] = {
0x0000,0x0a0a,0x1405,0x1de1,0x278d,0x30fb,0x3a1b,0x42e0,
0x4b3b,0x5320,0x5a81,0x6154,0x678d,0x6d22,0x720b,0x7640,
0x79bb,0x7c75,0x7e6b,0x7f99,0x7fff,0x7f99,0x7e6b,0x7c75,
0x79bb,0x7640,0x720b,0x6d22,0x678d,0x6154,0x5a81,0x5320,
0x4b3b,0x42e0,0x3a1b,0x30fb,0x278d,0x1de1,0x1405,0x0a0a,
0x0000,0xf5f6,0xebfb,0xe21f,0xd873,0xcf05,0xc5e5,0xbd20,
0xb4c5,0xace0,0xa57f,0x9eac,0x9873,0x92de,0x8df5,0x89c0,
0x8645,0x838b,0x8195,0x8067,0x8001,0x8067,0x8195,0x838b,
0x8645,0x89c0,0x8df5,0x92de,0x9873,0x9eac,0xa57f,0xace0,
0xb4c5,0xbd20,0xc5e5,0xcf05,0xd873,0xe21f,0xebfb,0xf5f6
};
```

## Gain

Let's also define a gain factor to increase the output volume. As we said before, we will use a gain that is a power of two and therefore just define its exponent. If you find that the sound is distorted, you may want to reduce this number.

```c
#define GAIN 3  /* multiply the output by a factor of 2^GAIN */
```

## The processing function <a id="effect"></a>

In the following version of the main processing function you will need to provide a couple of lines yourself. In the meantime please note the following:

* we are assuming that we're using the LEFT channel for the microphone and we go through the input buffer two samples at a time, while we duplicate the output to produce a signal to both ears.
* `ix` is the [state variable](../../real-world-dsp/code-efficiency.md#state_var) that keeps track of our position in the lookup table. Since the alien voice is an _instantaneous_ transformation, this is the only global time reference that we need to have
* the function also implements [a simple DC notch](../../real-world-dsp/signal-levels.md#removing_dc). Since this filter only requires memory of a single past input sample, there is no need to implement a circular buffer and we just use a single static variable in the function
* [the multiplications should be performed using 32-bit integers](../../real-world-dsp/code-efficiency.md#float) and the result is scaled back to 16 bits; we take the gain into account in this rescaling.

```c
void inline Process(int16_t *pIn, int16_t *pOut, uint16_t size) {
  static int16_t x_prev = 0;
  static uint8_t ix = 0;

  // we assume we're using the LEFT channel
  for (uint16_t i = 0; i < size; i += 2) {
    // simple DC notch
    int32_t y = (int32_t)(*pIn - x_prev);
    x_prev = *pIn;

    // modulation
    y = ...
    ...

    // rescaling to 16 bits
    y >>= (16 - GAIN);

    // duplicate output to LEFT and RIGHT channels
    *pOut++ = (int16_t)y;
    *pOut++ = (int16_t)y;
    pIn += 2;
  }
}
```

{% hint style="info" %}
TASK 1: Complete the function to perform sinusoidal modulation.
{% endhint %}

Now place the function between the `USER CODE BEGIN 4` and `USER CODE END 4` comment tags and test the application!

## Going further

You can now try changing the modulation frequency by creating your own lookup tables!

## Solution

{% tabs %}
{% tab title="Anti-spoiler tab" %}
Are you sure you are ready to see the solution? ;\)
{% endtab %}

{% tab title="Task 1" %}
Here is the complete function:

```c
void inline Process(int16_t *pIn, int16_t *pOut, uint16_t size) {
  static int16_t x_prev = 0;
  static uint8_t ix = 0;

  // we assume we're using the LEFT channel
  for (uint16_t i = 0; i < size; i += 2) {
    // simple DC notch
    int32_t y = (int32_t)(*pIn - x_prev);
    x_prev = *pIn;

    // modulation
    y = y * COS_400_32[ix++];
    ix %= COS_400_32_SIZE;

    // rescaling to 16 bits
    y >>= (16 - GAIN);

    // duplicate output to LEFT and RIGHT channels
    *pOut++ = (int16_t)y;
    *pOut++ = (int16_t)y;
    pIn += 2;
  }
}

```
{% endtab %}
{% endtabs %}

