# Hardware

## Bill of materials

We will use the following components:

* [STM32 NUCLEO-F072RB](https://www.st.com/en/evaluation-tools/nucleo-f072rb.html)
* [USB cable - 6" A/MiniB](https://www.adafruit.com/product/899)
* [Adafruit I2S MEMS Microphone Breakout](https://www.adafruit.com/product/3421)
* [Adafruit I2S Stereo Decoder](https://www.adafruit.com/product/3678)
* [Jumper Wires](https://www.adafruit.com/product/266)

In principle, any board from STM32 can be used for these exercises, as long as it is supported by [CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html), [System Workbench](https://www.st.com/en/development-tools/sw4stm32.html), and exposes at least _two_ I2S buses since both the microphone and the DAC require a dedicated I2S bus for audio transfers. More details on the I2S bus will be covered [later](../../audio-peripherals/audio-io.md).

The microcontroller will be programmed and powered by your PC via a USB cable.

## The ST microcontroller <a id="microcontroller"></a>

Our board of choice is the STM32 Nucleo-64 development board with the [STM32F072RB microcontroller unit](https://www.st.com/en/evaluation-tools/nucleo-f072rb.html), which belongs to the STM32 Nucleo-64 family. You can find more information about this family of boards by reading the [official documentation](https://www.st.com/content/ccc/resource/technical/document/data_brief/c8/3c/30/f7/d6/08/4a/26/DM00105918.pdf/files/DM00105918.pdf/jcr:content/translations/en.DM00105918.pdf).

![](../../.gitbook/assets/nucleo_board.jpg)

_Figure: STM32 Nucleo development board._ [Picture source](https://www.st.com/en/evaluation-tools/nucleo-f072rb.html).

## The microphone breakout board <a id="microphone"></a>

The component used to capture sound is the [I2S MEMS Microphone Breakout](https://learn.adafruit.com/adafruit-i2s-mems-microphone-breakout/overview) by Adafruit. The actual microphone on this mini-board produces an _analog signal_ \(continuous in time and amplitude\) but the device also contains an Analog-to-Digital Converter that returns a _digital_ signal \(discrete in time and amplitude\), which is the format we need in order to pass the data to our microcontroller. We will describe the component in more detail in the [next chapter](../../audio-peripherals/microphone.md), which is devoted to building a _passthrough_ circuit; the configuration, which simply passes the microphone input directly to the output, is the "hello world" equivalent of an embedded audio application.

![](../../.gitbook/assets/sensors_3421_quarter_orig.jpg)

_Figure: Adafruit I2S MEMS Microphone Breakout._ [Picture source](http://learn.adafruit.com/assets/39631).

## The DAC breakout board <a id="dac_jack"></a>

The microcontroller accepts and produces digital signals; in order to playback its output on a pair of headphones, it is necessary to obtain analog signal and this can be achieved via a Digital-to-Analog Converter \(DAC\). We will use Adafruit's [I2S Stereo Decoder Breakout](https://learn.adafruit.com/adafruit-i2s-stereo-decoder-uda1334a/overview), which contains the DAC, an audio jack for connecting headphones, and the necessary additional components. We will describe the DAC in more detail as we build a passthrough in the [next chapter](../../audio-peripherals/dac.md).

![](../../.gitbook/assets/adafruit_products_3678_top_orig.jpg)

_Figure: Adafruit I2S Stereo Decoder - UDA1334A Breakout._ [Picture source](http://learn.adafruit.com/assets/48396).

