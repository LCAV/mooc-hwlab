# Benchmarking

When discussing the code architecture of a generic real-time audio device, we already remarked that if our processing callback is too slow with respect to the frequency of the DMA transfers, we will run into a condition called buffer underflow \(or overflow, if you look at it from the point of view of the input DMA\). 

It's therefore very important to make sure that our processing is fast enough and find out if 

## Benchmarking the  implementation <a id="benchmarking"></a>

It is nice to say that we do real-time processing, but is it really the case?

To assess this important question we will add a tool to our system, which will measure how long it takes to process one buffer with our application. In this way, we will know if we are processing too slow according to the chosen $$32$$ kHz sampling frequency and the selected buffer length.

The `HAL` library includes a function `uint32_t HAL_GetTick(void);` which will return the number of ticks since the start of the microcontroller in milliseconds. Sadly, we cannot use this tool because the resolution of one millisecond is too large for our selected sampling frequency \($$1/32$$ MHz $$= 31.25 \mu\textrm{s}$$\).

In order to have a finer timebase, we will use a [timer](https://www.embedded.com/electronics-blogs/beginner-s-corner/4024440/Introduction-to-Counter-Timers) \(a more in-depth explanation of the STM timer can be found [here](http://www.st.com/content/ccc/resource/technical/document/application_note/group0/91/01/84/3f/7c/67/41/3f/DM00236305/files/DM00236305.pdf/jcr:content/translations/en.DM00236305.pdf)\). It is an internal peripheral of the microcontroller and it can be used for a lot of applications \(PWM generation, count of internal or external events, etc.\).

Timers always have an input clock with one of the timebases of the microcontroller internal clocks \(a quartz for example\). This timebase can be either taken directly or reduced by a factor called a [prescaler](https://en.wikipedia.org/wiki/Prescaler). It is important to chose an appropriate prescaler value as it will define how fast the timer counts. Other important parameters include the length of the timer's counting register and at what value it will be reset. For our application, we will use a timer with a large counting capacity \(32 bits\) and we will set it to increment itself every microsecond.

## Setting up timer for benchmarking <a id="timer"></a>

Open the CubeMX file by double clicking the .ioc file of the copied project it in the IDE project explorer in order to update the initialization code.

For this exercise, we will only need to add the configuration of a _timer_ \(to benchmark our implementation\) as the rest of the system is already up and running. In order to activate a timer, you need to set a "Clock Source". Open _TIM2_ in the Timers menu and activate it's clock:

![](../.gitbook/assets/screenshot-2019-10-07-at-15.42.23.png)

_Figure: Set the "Clock Source" to "Internal Clock" in order to enable "TIM2"._

Next, we need to configure the timer.

![](../.gitbook/assets/screenshot-2019-10-07-at-15.43.06.png)

{% hint style="info" %}
TASK 4: We ask you to set the "Prescaler" value \(in the figure above\) in order to achieve a $$1\,[\mu s]$$ period for "TIM2", i.e. we want our timer to have a $$1\,[\mu s]$$ resolution.

_Hint: Go to the "Clock Configuration" tab \(from the main window pane\) to see what is the frequency of the input clock to "TIM2". From this calculate the prescaler value to increase the timer's period to_ $$1\,[\mu s]$$_._
{% endhint %}

You can leave the rest of the parameters as is for "TIM2". Finally, you can update the initialization code by saving the file.

In order to use the timer we configured, we will need to define a variable to keep track of the time and a macro to reset the timer. Between the `USER CODE BEGIN PV` and `USER CODE END PV` comments, add the following lines in the `main.c` file.

```c
/* USER CODE BEGIN PV */
volatile int32_t current_time_us;
#define RESET_TIMER ({\
        current_time_us = __HAL_TIM_GET_COUNTER(&htim2);\
        HAL_TIM_Base_Stop(&htim2);\
        HAL_TIM_Base_Init(&htim2);\
        HAL_TIM_Base_Start(&htim2);\
    })
```

To use this macro, just call it in your code, then the variable `current_time_us` will be updated and the timer will be reset.

We want to assess if the processing time is longer or shorter than what our chosen buffer length and sampling frequency allows us. To do this, we will define some additional variables and constants \(also between the `USER CODE BEGIN PV` and `USER CODE END PV` comments\).

{% hint style="info" %}
TASK 5: In the passthrough example, we set the buffer length \(the macro called `FRAME_PER_BUFFER`\) to just 32. Increase it to 512 and use this value and `FS` to calculate the maximum processing time allowed in microseconds. Replace the variable `USING_FRAME_PER_BUFFER_AND_FS` in the code snippet below with this expression for the maximum processing time.

_Note: keep in mind the points made about using_ `float` _or_ `int` _variables \(see_ [_here_](dsp_tips.md#float)_\)._
{% endhint %}

```c
/* USER CODE BEGIN PV */
volatile int16_t processing_load;
#define FS     hi2s1.Init.AudioFreq
#define MAX_PROCESS_TIME_ALLOWED_us   USING_FRAME_PER_BUFFER_AND_FS
```

In between the `USER CODE BEGIN 2` and `USER CODE END 2` comments, we propose adding the following lines to read the timer and print the result of the benchmarking tool to the console.

```c
/* USER CODE BEGIN 2 */

// Mute codec
MUTE
HAL_Delay(500);

// begin DMA to fill the buffer with values
HAL_I2S_Transmit_DMA(&hi2s1, (uint16_t *) dataOut, DOUBLE_BUFFER_I2S);
HAL_I2S_Receive_DMA(&hi2s2, (uint16_t *) dataIn, DOUBLE_BUFFER_I2S);

// Wait that the buffers are full
HAL_Delay(1000);

// Stop DMAs to get a precise process timing
HAL_I2S_DMAPause(&hi2s1);
HAL_I2S_DMAPause(&hi2s2);

// Measure Processing time
RESET_TIMER;
process(dataIn, dataOut, BUFFER_SIZE);
RESET_TIMER;

// Display the results
printf("-- Processing time assert -- fs = %ld[Hz]\n", FS);

// load in percent
processing_load = USING_CURRENT_TIME_US_AND_MAX_PROCESS_TIME_ALLOWED_US;

if (current_time_us < MAX_PROCESS_TIME_ALLOWED_us) {
    printf("Processing time shorter than allowed time: t_proc = %ld [us], t_buf = %ld [us] (%i%%) \n", current_time_us, MAX_PROCESS_TIME_ALLOWED_us, processing_load);
} else {
    printf("Processing time longer than allowed time: t_proc = %ld [us], t_buf = %ld [us] (%i%%) \n", current_time_us, MAX_PROCESS_TIME_ALLOWED_us, processing_load);
}

// Reactivate the DMAs
HAL_I2S_DMAResume(&hi2s1);
HAL_I2S_DMAResume(&hi2s2);

UNMUTE
SET_MIC_LEFT
```

{% hint style="info" %}
TASK 6: Using the `current_time_us` and `MAX_PROCESS_TIME_ALLOWED_us`, compute the value of `processing_load` as a percentage in the code snippet above, i.e. replace `USING_CURRENT_TIME_US_AND_MAX_PROCESS_TIME_ALLOWED_US` with the appropriate expression.

_Note: keep in mind the points made about using_ `float` _or_ `int` _variables \(see_ [_here_](dsp_tips.md#float)_\)._
{% endhint %}

You will notice that we used a `printf` function in order to output text on the debug console. To enable this function you need to make the following changes to your project:

* In the Project Properties \("right-click" project &gt; Properties\), navigate to "C/C++ Build &gt; Settings" on the left-hand side \(see the figure below\). Under "Tools Settings -&gt; MCU GCC Linker -&gt; Miscellaneous", add "Linker flags" field by pressing the "+" button. The necessary flags are the following:

  ```text
  -specs=nosys.specs -specs=nano.specs -specs=rdimon.specs -lc -lrdimon
  ```

![](../.gitbook/assets/screenshot-2019-10-07-at-15.46.31.png)

* Add the following function prototype above the `main` function \(e.g. between the `USER CODE BEGIN PFP` and `USER CODE END PFP` comments\):

  ```c
  extern void initialise_monitor_handles(void);
  ```

* Add the following function call in the body of the `main` function \(before any `printf`\):

  ```c
  initialise_monitor_handles();
  ```

* In Debug Configurations \(dropdown from the _bug_ icon\) add the following option under the "Startup" tab:

  ```text
  monitor arm semihosting enable
  ```

After this setup, the `printf` output will be shown in the debug console of Eclipse \(precisely in the `open ocd` console\). _**Be careful, the modification made in the Debug Configuration will not stay if you copy and paste your project to make a new version!**_

