# Last Details

In the previous section we implemented a basic granular synthesis voice transformer that _lowers_ the pitch of the input voice. In this section we will address some remaining issues, namely:

* implement an effect that raises the pitch of the voice \(aka the "Chipmunks" effect\)
* properly initialize the buffer as a function of the pitch change
* optimize the code a little more 

## The Chipmunks

To raise the pitch of the voice we need to set $$\alpha$$ to values larger than one. As we have seen, this makes the effect noncausal, which we need to address by introducing some processing delay.

The way to achieve this is to place the audio buffer's input index _forward_ with respect to the output index; let's do this properly by creating an initialization function for the buffer that takes the resampling factor as the input.

{% hint style="info" %}
TASK 1: Determine the proper initial value for `buf_ix` when $$\alpha > 1$$ in the function below.
{% endhint %}

```c
static void InitBuffer(float Alpha) {
  memset(buffer, 0, BUF_LEN * sizeof(int16_t));

  alpha = (int32_t)(0x7FFF * Alpha);
  // input index for inserting DMA data
  if (Alpha <= 1)
      buf_ix = 0;
  else
    buf_ix = ...;

  prev_ix = BUF_LEN - GRAIN_STRIDE;
  curr_ix = 0;
  grain_m = 0;
}
```

By now you know where to place this code but don't forget to

* add the following line to the file `main.h` between the `/* USER CODE BEGIN Includes */` tags.

  ```c
  #include <memory.h>
  ```

* declare the function prototype in the `USER CODE BEGIN PFP` block 
* call the function before launching the DMA transfers:

  ```c
  UNMUTE
  SET_MIC_LEFT

  InitBuffer(3.0 / 2.0);

  // begin DMAs
  HAL_I2S_Transmit_DMA(&hi2s1, (uint16_t *) dma_tx, FULL_BUFFER_SIZE);
  HAL_I2S_Receive_DMA(&hi2s2, (uint16_t *) dma_rx, FULL_BUFFER_SIZE);
  ```

## Switching between effect

We can use the blue button on the Nucleo board to switch between Darth Vader and the Chipmunks; to do so, define the following constants at the beginning of the code

```c
#define DARTH (2.0 / 3.0)
#define CHIPMUNK (3.0 / 2.0)
```

and modify the user button callback like so:

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
  if (GPIO_Pin == B1_Pin) {
    // blue button pressed
    if (user_button) {
      user_button = 0;
      HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
      InitBuffer(DARTH);
    } else {
      user_button = 1;
      HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);
      InitBuffer(CHIPMUNK);
    }
  }
}
```

## Final optimizations

In the main processing loop, we are performing two checks on the value of `grain_m` per output sample. However, in the current implementation, both the stride and the taper lengths are multiples of the size of the DMA half-buffer. This allows us to move these checks outside of the processing loop and perform them once per call rather than once per sample

{% hint style="info" %}
TASK 2: Modify the `VoiceEffect()` function to reduce the number of `if` statements per call. Benchmark the result and observe the change in performance.
{% endhint %}

## **Solutions**

{% tabs %}
{% tab title="Anti-spoiler Tab" %}
Are you ready to see the answers ? :\)
{% endtab %}

{% tab title="Task 1" %}
We have seen in the previous section that the maximum displacement between current output index and needed input index is $$D = (\alpha - 1)L$$. Since this value can be non-integer, we round it up to the nearest integer value:

```c
    buf_ix = (uint16_t)(GRAIN_LEN * (Alpha - 1) + 0.5) & BUFLEN_MASK;
```
{% endtab %}

{% tab title="Task 2" %}
Since the DMA transfer size is a exact divisor of both grain stride and taper length, the boundaries that we check `grain_m` against can only be crossed at the end of a function call. We can therefore rewrite the function like so:

```c
inline static void VoiceEffect(int16_t *pIn, int16_t *pOut, uint16_t size) {
  for (int n = 0; n < size; n += 2) {
    buffer[buf_ix++] = pIn[n];
    buf_ix &= BUFLEN_MASK;
  }

  if (grain_m < TAPER_LEN) {
    // we are inside the tapering slope
    for (int n = 0; n < size; n += 2) {
      int32_t z = Resample(grain_m + GRAIN_STRIDE, prev_ix) * (0x07FFF - TAPER[grain_m]);
      z += Resample(grain_m, curr_ix) * TAPER[grain_m];
      pOut[n] = pOut[n+1] = (int16_t)(z >> 15);
      ++grain_m;
    }
  } else {
    for (int n = 0; n < size; n += 2)
      pOut[n] = pOut[n+1] = Resample(grain_m++, curr_ix);
  }
  // end of stride?
  if (grain_m >= GRAIN_STRIDE) {
    grain_m = 0;
    prev_ix = curr_ix;
    curr_ix = (curr_ix + GRAIN_STRIDE) & BUFLEN_MASK;
  }
}
```

With this implementation, the computational cost per sample oscillates between $$4.4\mu s$$ and $$7.8\mu s$$ per sample, which represent a saving of almost one microsecond per sample or, equivalently, a performance increase of at least 9%.
{% endtab %}
{% endtabs %}

