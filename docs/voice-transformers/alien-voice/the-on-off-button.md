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
      // turn off LED
      HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
    } else {
      effect_enabled = 1;
      // turn on LED
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
We don't want to check the `effect_enabled`status variable all the time, so we will place the logic at the DMA interrupt level. First, rename the function that implements the alien voice form `Process` to `VoiceEffect`. Then modify the function prototypes between the `/* USER CODE BEGIN PFP */`tags like so:

```c
void VoiceEffect(int16_t *pIn, int16_t *pOut, uint16_t size);

void Process(int16_t *pIn, int16_t *pOut, uint16_t size) {
  if (effect_enabled) {
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
{% endtabs %}

