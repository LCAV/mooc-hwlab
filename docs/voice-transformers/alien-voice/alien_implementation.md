# Basic implementation

Assuming you have successfully implemented the [passthrough](../../audio-peripherals/passthrough/), you can simply copy and paste that project from within the STM32CubeIDE software. We recommend choosing a name with the current date and `"alien_voice"` in it. Remember to delete the binary \(ELF\) file inside the copied project.

## Lookup table

Remember that in the passthrough we set up the system to work with a sampling frequency of 32KHz and a sample precision of 16 bits. Here we will use a modulation frequency of 400Hz for the frequency shifting, so that the digital frequency is a rational multiple of $$2\pi$$as in the [previous example](../../real-world-dsp/code-efficiency.md#lookup):

$$
\omega_c = 2\pi\frac{400}{32,000}=\frac{2\pi}{80}\approx 0.0785398
$$

The values for one period of the digital sinusoid can be encoded in a circular lookup table of length 80, where each element is \(in 16-bit precision\)

```c
cos_table[n] = (int16_t)(32767.0 * cos(0.0785398 * n));
```

The  lookup table is provided here for your convenience. Begin by copying it between the `USER CODE BEGIN PV` and `USER CODE END PV` comment tags.

```c
#define SINE_TABLE_SIZE 80
#define SIN_MAX 0x7fff
const int16_t sine_table[SINE_TABLE_SIZE] = {
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

## The processing function <a id="effect"></a>

In the following version of the main processing function we implement a simple high pass filter and add a gain to make the output more audible. 

```c
/* USER CODE BEGIN 4 */

void inline process(int16_t *bufferInStereo, int16_t *bufferOutStereo,
        uint16_t size) {

    int16_t static x_1 = 0;
    int16_t x[FRAME_PER_BUFFER];
    int16_t y[FRAME_PER_BUFFER];

    #define GAIN 8      // We lose 1us processing time if we use a value that is not a power of 2

    static uint16_t pointer_sine = 0;

    // Take signal from left side
    for (uint16_t i = 0; i < size; i += 2) {
        x[i / 2] = bufferInStereo[i];
    }

    for (uint16_t i = 0; i < FRAME_PER_BUFFER; i++) {

        // High pass filter
        y[i] = x[i] - x_1;

        // Apply alien voice effect and gain
        y[i] =

        // Update state variables
        pointer_sine = 
        x_1 =

    }

    // Interleaved left and right
    for (uint16_t i = 0; i < size; i += 2) {
        bufferOutStereo[i] = (int16_t) y[i / 2];
        bufferOutStereo[i + 1] = 0;
    }
```

{% hint style="info" %}
TASK 1: Modify the code within the second `for` loop in order to compute the alien voice output and update the state variables.

_Note: normalize the sinusoid using the constant_ `SINE_MAX`_!_
{% endhint %}

Now place the function between the `USER CODE BEGIN 4` and `USER CODE END 4` comment tags and test the application!

## Extra features <a id="extra"></a>

If you have some extra time, we propose to make a few improvements to the system!

First extra feature: will now program one of the on-board buttons - the blue button called "B1" - to toggle the alien voice effect. Copy the following code between the `USER CODE BEGIN PV` and `USER CODE END PV` comments.

```c
/* USER CODE BEGIN PV */
// Enumeration for a clean FX ON/OFF toggle
enum {
    FX_OFF, FX_ON
} FX_STATE;

// State variable for the FX
uint8_t FX = FX_OFF;

/* USER CODE BEGIN 0 */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {

    if (GPIO_Pin == B1_Pin) {
        FX = !FX;
        LED_TOGGLE
    }
}
```

In the above code snippet, you will find the state variable we propose for the FX \(effects\) state and the callback that is needed to react to the button. **To activate the callback, you need to go into CubeMX and enable "EXTI line 4 to 15" from the Configuration tab under "System &gt; NVIC".**

{% hint style="info" %}
TASK 8: Modify your `process` function using a condition as proposed in the code snippet below.

_Hint: This will not be enough if you used the optimised version of the process function proposed in task 7, if it is the case, you will also have to test the FX value in the signal initialisation._
{% endhint %}

```c
for (uint16_t i = 1; i < FRAME_PER_BUFFER; i++) {

    // High pass filter
    y[i] = x[i] - x_1;

    // Apply robot voice modulation and gain
    if (FX == FX_ON) {

    } else {

    }

    // Update state variables
    pointer_sine = 
    x_1 =
}
```

Finally, you can try changing the modulation frequency and creating your lookup tables by running [this Python script](https://github.com/LCAV/dsp-labs/blob/master/scripts/alien_voice/compute_sinusoid_lookup.py) for modified values of `f_sine`.

**Congrats on implementing your \(perhaps\) first voice effect! In the** [**next chapter**](../../filter-design/)**, we will implement a more sophisticated high-pass filter than the one used here. To this end, we will come across fundamental theory and practical skills in digital filter design.**

## Tasks solutions

{% tabs %}
{% tab title="Anti-spoiler tab" %}
Are you sure you are ready to see the solution? ;\)
{% endtab %}

{% tab title="Task 7" %}
Here comes the moment when you can rely on your former python implementation in order to code the C version of the alien voice. Indeed as we already coded the python version in a block version and very close to C programming, it is just a matter of porting the code.

```c
/* USER CODE BEGIN 4 */

void inline process(int16_t *bufferInStereo, int16_t *bufferOutStereo,
        uint16_t size) {

    int16_t static x_1 = 0;
    int16_t x[FRAME_PER_BUFFER];
    int16_t y[FRAME_PER_BUFFER];

#define GAIN 8      // We lose 1us processing time if we use a value that is not a power of 2

    static uint16_t pointer_sine = 0;

    // Take signal from left side
    for (uint16_t i = 0; i < size; i += 2) {
        x[i / 2] = bufferInStereo[i];
    }

    for (uint16_t i = 0; i < FRAME_PER_BUFFER; i++) {

        // High pass filter
        y[i] = x[i] - x_1;

        // Apply alien voice effect and gain
        y[i] = (y[i] * sine_table[pointer_sine++]) * GAIN / SIN_MAX;

        // Update state variables
        pointer_sine %= SINE_TABLE_SIZE; 
        x_1 = x[FRAME_PER_BUFFER - 1];
    }

    // Interleaved left and right
    for (uint16_t i = 0; i < size; i += 2) {
        bufferOutStereo[i] = (int16_t) y[i / 2];
        bufferOutStereo[i + 1] = 0;
    }
```

Note that an optimisation could be done. The line _x\_1 = x\[FRAME\_PER\_BUFFER - 1\];_ is executed on every single passage through the _for_ loop. In fact we only need to backup x\_1 \(as a static variable\) during the transition from one buffer to the next. With some modification we can arrive to the following function that will use slightly less of CPU usage:

```c
void inline process(int16_t *bufferInStereo, int16_t *bufferOutStereo,
        uint16_t size) {

    int16_t static x_1 = 0;
    int16_t x[FRAME_PER_BUFFER];
    int16_t y[FRAME_PER_BUFFER];

    static uint16_t pointer_sine = 0;

    #define GAIN 8      // We lose 1us processing time if we use a value that is not a power of 2

    // Take signal from left side
    for (uint16_t i = 0; i < size; i += 2) {
        x[i / 2] = bufferInStereo[i];
    }

    // High pass filtering initialization
    y[0] = x[0] - x_1; // deal with the first value, backuped from previous buffer
    // Signal initialization
    y[0] = (y[0] * sine_table[pointer_sine++]) * GAIN / SIN_MAX;
    pointer_sine %= SINE_TABLE_SIZE;

    for (uint16_t i = 1; i < FRAME_PER_BUFFER; i++) {
        // High pass filtering
        y[i] = x[i] - x[i - 1];

        // Robot voice modulation and gain
        y[i] = (y[i] * sine_table[pointer_sine++]) * GAIN / SIN_MAX;
        pointer_sine %= SINE_TABLE_SIZE;
    }

    // Backup last sample for next buffer -> ONLY ONCE per buffer, otherwise we use x[i-1] that is available "locally"
    x_1 = x[FRAME_PER_BUFFER - 1];

    // Interleaved left and right
    for (uint16_t i = 0; i < size; i += 2) {
        bufferOutStereo[i] = (int16_t) y[i / 2];
        // Put signal on both side
        bufferOutStereo[i + 1] = (int16_t) y[i / 2];
    }
}
```
{% endtab %}

{% tab title="Task 8" %}
The final process function in its optimised form will look like this:

```c
void inline process(int16_t *bufferInStereo, int16_t *bufferOutStereo,
        uint16_t size) {

    int16_t static x_1 = 0;
    int16_t x[FRAME_PER_BUFFER];
    int16_t y[FRAME_PER_BUFFER];

#define GAIN 8         // We loose 1us if we use 10 in stead of 8

    static uint16_t pointer_sine = 0;

    // Take signal from left side
    for (uint16_t i = 0; i < size; i += 2) {
        x[i / 2] = bufferInStereo[i];
    }

    // High pass filtering initialization
    y[0] = x[0] - x_1; // deal with the first value, backuped from previous buffer

    // Signal initialization
    if (FX == FX_ON) {
        y[0] = (y[0] * sine_table[pointer_sine++]) * GAIN / SIN_MAX;
        pointer_sine %= SINE_TABLE_SIZE;
    } else {
        // Gain
        y[0] *= GAIN;
    }

    for (uint16_t i = 1; i < FRAME_PER_BUFFER; i++) {

        // High pass filtering
        y[i] = x[i] - x[i - 1];
        if (FX == FX_ON) {
            // Robot voice modulation and gain
            y[i] = (y[i] * sine_table[pointer_sine++]) * GAIN / SIN_MAX;
            pointer_sine %= SINE_TABLE_SIZE;
        } else {
            // Gain
            y[i] *= GAIN;
        }
    }

    // Backup last sample for next buffer
    x_1 = x[FRAME_PER_BUFFER - 1];

    // Interleaved left and right
    for (uint16_t i = 0; i < size; i += 2) {
        bufferOutStereo[i] = (int16_t) y[i / 2];
        // Put signal on both side
        bufferOutStereo[i + 1] = (int16_t) y[i / 2];
    }
}
```
{% endtab %}
{% endtabs %}

