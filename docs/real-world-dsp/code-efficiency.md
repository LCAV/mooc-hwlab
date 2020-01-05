# Code efficiency

In this section we will illustrate some common coding practices that are used in real-time DSP implementations and that will be used later in our examples.

## Circular buffers

In [Lecture 2.2.5a](https://www.coursera.org/learn/dsp2/lecture/6oXrx/2-2-5-a-implementation-of-digital-filters) in the [second DSP course](https://www.coursera.org/learn/dsp2/) we discussed some implementation issues related to discrete-time filters and, in particular, we talked about circular buffers. 

As a quick recap, remember that if you need to store past values of a signal, the best solution is to use a circular buffer; assume that you need to access at most $$M$$past values of the signal $$x[n]$$:

* set up an array `x_buf[]` of length$$M$$\(of the appropriate data type\)
* set up an index variable `ix`, initialized at zero
* every time you receive a new sample, store it in the array at `ix` and increment `ix` modulo $$M;$$

with this, the expression $$x[n-k], k <M,$$can be accessed as `x[(ix + M - k) % M]`.

In a microcontroller, where each CPU cycle counts, modulo operations are expensive but they can be avoided and replaced by binary masks if we choose $$M$$to be a power of two. In those cases, `ix % M` is equivalent to `ix & (M-1)` and the bitwise AND is a much faster operation. Since $$M$$is the _minimum_ number of past values that we need access to, we can always increase $$M$$until it reaches a power of two, especially when $$M$$is small.

Here is a simple example:

```python
#define BUF_LEN 16
#define BUF_MSK 15 /* binary mask is always len - 1 */
uint16_t x_buf[BUF_LEN];
uint16_t ix = 0;

/* storing sample x */
x_buf[ix++] = x;
ix &= BUF_MSK;

/* accessing x[n-k] */
uint16_t x_k = x_buf[(ix + BUF_LEN - k) & BUF_MSK];
```

Speaking of circular buffer, remember that [we also set the DMA transfer buffers to be circular!](../audio-peripherals/passthrough/io_setup.md#i2s)

## Sinusoidal lookup tables <a id="lookup"></a>

Most signal processing algorithms require the use of sinusoidal functions. In a microcontroller, however, computing trigonometric values for arbitrary values of the angle is an expensive operation since it always involves some form of Taylor series approximation. Even using a few terms, as in

$$
\cos x = x - \dfrac{x^2}{2!} + \dfrac{x^4}{4!} - \dfrac{x^6}{6!} + \mathcal{O}(x^8)
$$

clearly requires a significant number of multiplications. A computationally cheaper alternative is based on the use of a [lookup table](https://en.wikipedia.org/wiki/Lookup_table). In a lookup table, we precompute the sinusoidal values that we need and use the time index$$n$$simply to retrieve the correct value. 

In sinusoidal modulation we need to know the values of the sequence $$\cos(\omega_c n)$$ for all values of $$n$$. However, if $$\omega_c$$is a rational multiple of $$2\pi$$, that is, if $$\omega_c = 2\pi(M/N)$$for $$M,N \in \mathbb{N}$$, then the sequence of sinusoidal values repeats exactly every $$N$$samples. 

For instance, assume the input sampling frequency is $$F_s = 32$$KHz and that our modulation frequency is $$f_c = 400$$Hz. In this case $$\omega_c = 2\pi /80$$and therefore we simply need to precompute 80 values for the cosine and store them in an array `C[0], ..., C[79]`. The equation

$$
y[n] = x[n] \, \cos(\omega_c n),
$$

becomes simply

```c
y[n] = x[n] * C[n % 80]
```

Of course, we are trading computational time for memory here so, if $$N$$in the denominator is impractically large, the table lookup method may become prohibitive, especially on architectures such as the Nucleo which do not have a lot of onboard memory. Also note that this is one case in which we most likely won't be able to use binary masks instead of modulo operations since the period of the sinusoid is unlikely to be a power of two.

Another difficulty is when $$\omega_c$$is _not_ a rational multiple of $$2\pi$$. In this case, we may want to slightly adjust the modulation frequency to a value for which the rational multiple expression becomes valid.

## State variables <a id="state_var"></a>

All discrete-time signal processing data and algorithms make use of a free "time" variable $$n$$. As we know, in theory $$n \in \mathbb{Z}$$ so its value ranges from minus infinity to plus infinity. In an actual DSP application we are much more likely to:

* start the processing with all buffers empty and with $$n=0$$\(initial conditions\)
* store $$n$$in an unsigned integer variable and increment it at each iteration.

The second point in particular means that, in real time applications that may run for an arbitrary amount of time,$$n$$will increase until it reaches the maximum positive value that can be expressed by the variable and then roll over to zero. Since we certainly do not want this rollover to happen at random times and since the roll over is unavoidable, we need to establish a strategy to carry it out explicitly.

In practice, all real-time applications only use circular buffers, either explicitly \(to access past input and output values or to access lookup tables\) or implicitly \(to compute the output of functions that are inherently periodic\). As a consequence, we never need the _exact_ value of$$n$$but only the position of a set of indices into synchronous circular buffers.

In our code, therefore, we will explicitly roll over these indexes independently and incrementally. To this end:

* in functions, indexes will be defined as [static variables](https://stackoverflow.com/questions/572547/what-does-static-mean-in-c) so that their value will be preserved between consecutive function calls.
* to make sure that state variables used by different functions are stepped synchronously, we will define them as global-scope variables at the application level.

<!-- below: do you mean state variables or global variables? -->
These types of variables are often referred to as _state variables_ in C programming and they are usually much frowned upon; the truth is, in a microcontroller real-time application where performance is key, they simply cannot be avoided.

