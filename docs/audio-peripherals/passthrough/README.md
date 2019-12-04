# The audio passthrough project

A "passthrough" can be viewed as the audio processing equivalent of a "hello world" program. In this section we will program the Nucleo to simply pass the audio samples from the microphone to the DAC.

Using the CubeMX software, we will first [update the configuration of the microcontroller](io_setup.md). We will then guide you through the [wiring ](wiring.md)and, finally, we will [program our passthrough](coding.md) using the SW4STM32 software.

Highlighted boxes, as shown below, specify a task for which _**you**_ need to find out the appropriate solution and implementation.

{% hint style="info" %}
TASK: This is a task for you!
{% endhint %}

A passthrough is a great _sanity check_ when first beginning with an audio DSP system. Moreover, it serves as a useful starting point for new projects, as we will see in the following chapters when we develop more complicated programs.

![Figure: Final wiring.](../../.gitbook/assets/final_wiring.jpg)

