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
#define COS_TABLE_LEN 80
static int16_t COS_TABLE[COS_TABLE_LEN] = {
	0x7FFF, 0x7F99, 0x7E6B, 0x7C75, 0x79BB, 0x7640, 0x720B, 0x6D22, 0x678D, 0x6154, 0x5A81, 0x5320, 
	0x4B3B, 0x42E0, 0x3A1B, 0x30FB, 0x278D, 0x1DE1, 0x1405, 0x0A0A, 0x0000, 0xF5F6, 0xEBFB, 0xE21F, 
	0xD873, 0xCF05, 0xC5E5, 0xBD20, 0xB4C5, 0xACE0, 0xA57F, 0x9EAC, 0x9873, 0x92DE, 0x8DF5, 0x89C0, 
	0x8645, 0x838B, 0x8195, 0x8067, 0x8001, 0x8067, 0x8195, 0x838B, 0x8645, 0x89C0, 0x8DF5, 0x92DE, 
	0x9873, 0x9EAC, 0xA57F, 0xACE0, 0xB4C5, 0xBD20, 0xC5E5, 0xCF05, 0xD873, 0xE21F, 0xEBFB, 0xF5F6, 
	0x0000, 0x0A0A, 0x1405, 0x1DE1, 0x278D, 0x30FB, 0x3A1B, 0x42E0, 0x4B3B, 0x5320, 0x5A81, 0x6154, 
	0x678D, 0x6D22, 0x720B, 0x7640, 0x79BB, 0x7C75, 0x7E6B, 0x7F99};
```

{% hint style="info" %}
TASK 1: Write a short Python function that prints out the above table given a value for the period of the sinusoid.
{% endhint %}


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
    y >>= (15 - GAIN);

    // duplicate output to LEFT and RIGHT channels
    *pOut++ = (int16_t)y;
    *pOut++ = (int16_t)y;
    pIn += 2;
  }
}
```

{% hint style="info" %}
TASK 2: Complete the function to perform sinusoidal modulation.
{% endhint %}

Now place the function between the `USER CODE BEGIN 4` and `USER CODE END 4` comment tags and test the application!

## Going further

You can now try changing the modulation frequency by creating your own lookup tables!


## Solutions

{% tabs %}
{% tab title="Anti-spoiler tab" %}
Are you sure you are ready to see the solution? ;\)
{% endtab %}

{% tab title="Task 1" %}
```python
def make_cos_table(period):
    c = 0x7FFF * np.cos(2 * np.pi * np.arange(0, period) / period)
    print('#define COS_TABLE_LEN {}'.format(period))
    print('static int16_t COS_TABLE[COS_TABLE_LEN] = {', end='\n\t')
    for n in range(period - 1):
        print('0x{:04X}, '.format(np.uint16(c[n])), end='' + '\n\t' if (n+1) % 12 == 0 else '')
    print('0x{:04X}}};'.format(np.uint16(c[period-1])))
```
{% endtab %}

{% tab title="Task 2" %}
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
    y = y * COS_TABLE[ix++];
    ix %= COS_TABLE_LEN;

    // rescaling to 16 bits
    y >>= (15 - GAIN);

    // duplicate output to LEFT and RIGHT channels
    *pOut++ = (int16_t)y;
    *pOut++ = (int16_t)y;
    pIn += 2;
  }
}

```
{% endtab %}
{% endtabs %}

