# The ON/OFF button

The Nucleo board has a user programmable push button. We will now use it as an ON/OFF button for the alien voice effect.

## Configuration <a id="extra"></a>

The idea is to use the push button to call an asynchronous routine in our code. To do that, we need to configure the button to trigger an interrupt and then we need to catch the interrupt in our code.

Go into CubeMX by clicking on the `ioc` file in your alien voice project; in the left panel click on "System &gt; NVIC" and enable the line "EXTI line 4 to 15" by checking the corresponding checkmark. The pin PC13 is linked to EXTI13 in the hardware of the microcontroller. Interrupts are used because they provide a very fast access to the core of the system and thus a very fast reaction.

Still in CubeMX, verify that the label for pin PA5 is "LD2" and the label for pin PC13 is "B1".

Add the following state variable to the `USER CODE BEGIN PV` section

```c
char user_button = 0;  /* user button status */
```

and add the following interrupt handler to the `USER CODE BEGIN 0` section:

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
  if (GPIO_Pin == B1_Pin) {
    // blue button pressed
    if (user_button) {
      user_button = 0;
      // turn off LED
      HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
    } else {
      user_button = 1;
      // turn on LED
      HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
    }
  }
}
```

The interrupt handler toggles the variable `user_button` and switches the LED on when its value is true.

{% hint style="info" %}
TASK 1: Modify the alien voice`Process`function so that it switches between a passthrough and the alien voice.
{% endhint %}

## Benchmarking

Now that we have an ON/OFF button, we can use the [benchmarking timer we defined before](../../real-world-dsp/benchmarking.md) to see how expensive it is to compute the alien voice.

{% hint style="info" %}
TASK 2: Add the timing macros to the Process function and use the push button to compare execution times.
{% endhint %}

## Solution

{% tabs %}
{% tab title="Anti-spoiler tab" %}
Are you sure you are ready to see the solution? ;\)
{% endtab %}

{% tab title="Task 1" %}
We don't want to check the `user_button` status variable every time we process a sample, so we will place the logic at the DMA interrupt level, before we process a data buffer. First, rename the function that implements the alien voice form `Process` to `VoiceEffect`. Then modify the function prototypes between the `/* USER CODE BEGIN PFP */`tags like so:

```c
void VoiceEffect(int16_t *pIn, int16_t *pOut, uint16_t size);

void Process(int16_t *pIn, int16_t *pOut, uint16_t size) {
  if (user_button == 1) {
    VoiceEffect(pIn, pOut, size);
  } else { // just pass through
    for (uint16_t i = 0; i < size; pIn += 2, i += 2) {
      *pOut++ = *pIn;
      *pOut++ = *pIn;
    }
  }
}
```
{% endtab %}

{% tab title="Task 2" %}
The modified Process function is trivial since we just need to add the timing macros before and after the code:

```c
void Process(int16_t *pIn, int16_t *pOut, uint16_t size) {
  START_TIMER

  if (user_button == 1) {
    VoiceEffect(pIn, pOut, size);
  } else {
    // just pass through
    for (uint16_t i = 0; i < size; pIn += 2, i += 2) {
      *pOut++ = *pIn;
      *pOut++ = *pIn;
    }
  }

  STOP_TIMER
}
```

You should find that, while the passthrough requires approximately 33 microseconds, the alien voice effect requires 94 microseconds.
{% endtab %}
{% endtabs %}

