# Interlude: prototyping in Python

In the rest of this book we will try to implement more sophisticated voice transformers on the Nucleo. While the STM32 IDE offers extraordinarily practical debugging tools, it is not really wise to code our ideas directly on the microcontroller. A better approach is to prototype our algorithms in a user-friendly environment such as a Python notebook while retaining the key element of real-time processing. These are:

* the data should be delivered to the processing function in contiguous blocks, to simulate the DMA transfers
* the processing function should use only integers and fixex-point arithmetic
* we shouldn't use high-level routines or data structures

Here you can find a Jupyter notebook that simulates as closely as possible the execution environment on the Nucleo. The notebook implements the passthrough and the alien voice effects, and you can use it to try to develop your own processing routines. 

Of course the resulting code is clunky and not "pythonic" at all, but that's entirely on purpose! The idea is to have code that we can translate as effortlessly as possible to standard C for the microcontroller.

