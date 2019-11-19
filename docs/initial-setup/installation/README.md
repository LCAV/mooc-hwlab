# Material and tools

In this eBook we will develop real-time signal processing algorithms for a specific piece of hardware, namely the **STM32 NUCLEO-F072RB microcontroller** board manufactured by STMicroelectronics \(often abbreviated as ST\). ST provides many inexpensive development boards that are used by hobbyists, students, and even professionals to prototype countless applications. In principle, any board from STM32 can be used for the exercises, as long as it is supported by [CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html), [System Workbench](https://www.st.com/en/development-tools/sw4stm32.html), and exposes at least _two_ I2S buses.

To develop real-time audio applications we will connect the ST board to a **microphone** as an input source and to a **digital-to-analog converter** \(DAC\) as an output sink. These inexpensive components can be obtained from open-source hardware manufactures such as [Adafruit](https://www.adafruit.com/). 

In order to facilitate the development of applications, ST partners with third-party software companies that develop **integrated development environments** \(IDEs\) for their boards. These tools can sometimes be overwhelming as they allow for a lot of customization, but they are meant to make your life easier! Attention to detail and reading the documentation will help you in setting up a successful workflow. We will cover these tools and their installation in [this section](software/).

We will conclude this introductory part by illustrating in detail [how to build a first simple application on the microcontroller](../instructions.md).

