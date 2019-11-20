# The Adafruit Boards

Any real-time audio application running on the microcontroller will need to acquire data from a source \(for instance, a microphone\) and deliver data to an output sink \(for instance, an analog-to-digital converter connected to a loudspeaker\) that we can listen to. Here is a brief description of the components we selected.

## The microphone breakout board <a id="microphone"></a>

The component used to capture sound is the [I2S MEMS Microphone Breakout](https://learn.adafruit.com/adafruit-i2s-mems-microphone-breakout/overview) by Adafruit. The actual microphone on this mini-board produces an _analog signal_ \(continuous in time and amplitude\) but the device also contains an Analog-to-Digital Converter that returns a _digital_ signal \(discrete in time and amplitude\), which is the format we need in order to pass the data to our microcontroller. We will describe the component in more detail in the [later](microphone.md).

![Adafruit I2S MEMS Microphone Breakout](../.gitbook/assets/sensors_3421_quarter_orig.jpg)

## The DAC breakout board <a id="dac_jack"></a>

The microcontroller accepts and produces digital signals; in order to playback its output on a pair of headphones, it is necessary to obtain analog signal and this can be achieved via a Digital-to-Analog Converter \(DAC\). We will use Adafruit's [I2S Stereo Decoder Breakout](https://learn.adafruit.com/adafruit-i2s-stereo-decoder-uda1334a/overview), which contains the DAC, an audio jack for connecting headphones, and the necessary additional components. We will describe the DAC in more detail [later](dac.md).

![Adafruit I2S Stereo Decoder - UDA1334A Breakout](../.gitbook/assets/adafruit_products_3678_top_orig.jpg)

