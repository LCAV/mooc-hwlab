# The ON/OFF button

The Nucleo board has two onboard pushbuttons. We will now use one as an ON/OFF button for the alien voice effect.

## Configuration <a id="extra"></a>

The idea is to use the pushbutton to call an asynchronous routine in our code. To do that, we need to configure the button to trigger an interrupt and then we need to catch the interrupt in our code.

Go into CubeMX by clicking on the `ioc` file in your alien voice project; in the left panel click on "System &gt; NVIC" and enable the line "EXTI line 4 to 15" by checking the corresponding checkmark.

Always in CubeMX, verify that the label for pin PA5 is "LD2" and the label for pin PC13 is "B1".

Add the following state variable to the `USER CODE BEGIN PV` section

```c
char effect_enabled = 0;  /* audio effect switch, triggered by button */
```

and add the following interrupt handler to the `USER CODE BEGIN 0` section:

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
  if (GPIO_Pin == B1_Pin) {
    // blue button pressed
    if (effect_enabled) {
      effect_enabled = 0;
      HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
    } else {
      effect_enabled = 1;
      HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
    }
  }
}
```

The interrupt handler toggles the variable effect\_enabled and switches the LED on when its value is true.

{% hint style="info" %}
TASK 1: Modify the alien voice`Process` function so that it swiches between a passthrough and the alien voice.
{% endhint %}



## Solution

{% tabs %}
{% tab title="Anti-spoiler tab" %}
Are you sure you are ready to see the solution? ;\)
{% endtab %}

{% tab title="Task 1" %}
Note that, since we don't want to check for the value of `effect_enabled` _inside_ the processing loop for efficiency reasons, we simply duplicate the processing

```c
void inline Process(int16_t *pIn, int16_t *pOut, uint16_t size) {
  static int16_t x_prev = 0;
  static uint8_t ix = 0;

  // we assume we're using the LEFT channel
  if (effect_enabled) {
    for (uint16_t i = 0; i < size; i += 2) {
      // simple highpass filter
      int32_t y = (int32_t)(*pIn - x_prev);
      x_prev = *pIn;

      // modulation
      y = y * COS_TABLE[ix++];
      ix %= COS_TABLE_SIZE;

      // rescaling to 16 bits
      y >>= (16 - GAIN);

      // duplicate output to LEFT and RIGHT channels
      *pOut++ = (int16_t)y;
      *pOut++ = (int16_t)y;
      pIn += 2;
    }
  } else {
    for (uint16_t i = 0; i < size; i += 2) {
      // passthrough
      *pOut++ = *pIn;
      *pOut++ = *pIn;
      pIn += 2;
    }
  }
}
```
{% endtab %}
{% endtabs %}

