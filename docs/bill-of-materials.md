# Bill of materials

In this module will use the following components:

* [STM32 NUCLEO-F072RB](https://www.st.com/en/evaluation-tools/nucleo-f072rb.html)
* [USB cable - 6" A/MiniB](https://www.adafruit.com/product/899)
* [Adafruit I2S MEMS Microphone Breakout](https://www.adafruit.com/product/3421)
* [Adafruit I2S Stereo Decoder](https://www.adafruit.com/product/3678)
* [Jumper Wires](https://www.adafruit.com/product/266)

In principle, any board from STM32 can be used for these exercises, as long as it is supported by [CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html), [System Workbench](https://www.st.com/en/development-tools/sw4stm32.html), and exposes at least _two_ I2S buses since both the microphone and the DAC require a dedicated I2S bus for audio transfers. 

The microcontroller will be programmed and powered by your PC via a USB cable.

You will also need a PC with a working Python environment in order to run the simulation used in the code design phase.

