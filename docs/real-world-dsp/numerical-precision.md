# Numerical precision

Coding a "real-world" DSP application on dedicated hardware is a bit of a shock when we are used to the idealized world of theoretical derivations and nowhere is the disconnect more profound than when we need to take numerical precision explicitly into account.

## float vs. int <a id="float"></a>

Floating point numbers, as implemented in most architecture today, free us of the need to explicitly consider the _range_ of the numeric values that appear in our algorithm. Although not without caveats, a 64-bit `double` in C is pretty much equivalent to an ideal real number for most intents and purposes, with a dynamic range \(that is, the ratio between the smalles and largest numbers that can be represented\) in excess of $$10^{600}$$.

However, operations with floating point variables can take significantly more time than the same operations with integer variables on a microcontroller; on the Nucleo, for instance, we noticed that an implementation with floating-point variables can take up to 35% more processing time than an equivalent implementation with integer variables

If we try to avoid floats, then we need to use some form of fixed-point representation for our quantities. Implementing algorithms in fixed point is truly an art, and a difficult one at that. In the rest of this section we will barely scratch the surface and give you some ideas on how to proceed.

## Fixed-point representation

The idea behind fixed point representations is to encode fractional number as integers, and assuming the position of the decimal point implicitly. 

In our case, let's start with a reasonable assumption: the audio samples produced by the soundcards are signed decimal numbers in the $$(-1, 1)$$open interval. How can we represent numbers in this interval via integers and, more importantly, how does this affect the way we perform computations? 

Since we are all more familiar with numbers in base 10, let's start with a 2-digit fixed point representation in base 10 for fractional numbers between -1 and 1. With this, for instance, the number 0.35 will be represented by the integer 35 and -0.2 will be encoded by -20. Since we can  only have 2 digit, the number 0.1234 will have to be truncated to the representation 12. Similarly, we will not be able to encode numbers greater that 0.99 or smaller than -0.99. OK, a finite number of digits involves a loss of precision and this makes sense.

It is clear that in this representation we go from decimal numbers to integers by multiplying the fractional nuber by $$10^2 = 100$$ \(see the 2 in the exponent: that's our number of digits\) and taking the integer part of the result. Vice versa, we can go back to the decimal representation by dividing the integer by 100.

We can also choose at one point to, say, double the precision of our representation. In this example, we would use four digits and so 0.99 would become 9900 and 0.1234 will become 1234. Again, we can convert a single-precision \(2 digits\) number into a double-precision number by mutiplying by 100 \(and vice versa\). 

## Fixed-point arithmetic

TODO

~~Real numbers can be mapped to integers via renormalization when we can estimate the maximum range; for instance, a floating point value between -1 and +1 can be mapped to a 16-bit integer via a scaling factor of~~ $$2^{15} = 32768$$~~yielding 65536 discrete levels.~~

~~Remember that when you multiply two~~ $$B$$~~-bit integers the result will need to be computed over a~~ $$2B$$~~-bit integer to avoid overflow. The result can be rescaled to~~ $$B$$~~bits later, but try to keep rescaling to the end of a chain of integer arithmetic operations all of which are carried out with double precision.~~ 

~~In other words, since we're working with 16-bit samples, we will perform the intermediate arithmetic that involve multiplications using 32-bit variables, and then we will shift back the result to 16 bits.~~

~~~~

~~With an intelligent use of operation priority, integer arithmetic will not unduly impact the performance of our algorithms.~~

More about these kinds of trade-off can be read [here](https://www.embedded.com/design/debug-and-optimization/4440365/Floating-point-data-in-embedded-software) and [here](https://en.wikibooks.org/wiki/Embedded_Systems/Floating_Point_Unit).

## Binary representation

