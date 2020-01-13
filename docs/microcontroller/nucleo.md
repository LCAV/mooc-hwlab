# The ST Nucleo

In this eBook we will develop real-time signal processing algorithms for a specific piece of hardware, namely the [STM32 NUCLEO-F072RB](https://www.st.com/en/evaluation-tools/nucleo-f072rb.html) **microcontroller** board manufactured by **STMicroelectronics** \(often abbreviated as ST\). ST provides many inexpensive development boards that are used by hobbyists, students, and professionals to prototype countless applications.

![The STM32 Nucleo development board.](../.gitbook/assets/nucleo_board.jpg)

In principle, any board from the STM32 family can be used for the exercises, as long as it exposes at least _two_ I2S buses. You can find more information about this family of boards by reading the [official documentation](https://www.st.com/content/ccc/resource/technical/document/data_brief/c8/3c/30/f7/d6/08/4a/26/DM00105918.pdf/files/DM00105918.pdf/jcr:content/translations/en.DM00105918.pdf).

In order to facilitate the development of applications, ST provided full **integrated development environments** \(IDEs\) that you can use to program their boards. These tools can sometimes be overwhelming as they allow for a lot of customization, but they are meant to make your life easier! Attention to detail and reading the documentation will help you in setting up a successful workflow. We will cover these tools and their installation in [the next section](ide/).

We will conclude this introductory part by illustrating in detail [how to build a first simple application on the microcontroller](test_project.md).

