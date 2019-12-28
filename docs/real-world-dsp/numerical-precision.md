# Numerical precision

Coding a "real-world" DSP application on dedicated hardware is a bit of a shock when we are used to the idealized world of theoretical derivations and nowhere is the disconnect more profound than when we need to take numerical precision explicitly into account.

## float vs. int <a id="float"></a>

Floating point numbers, as implemented in most architecture today, free us of the need to explicitly consider the _range_ of the numeric values that appear in our algorithm. Although not without caveats, a 64-bit `double` in C is pretty much equivalent to an ideal real number for most intents and purposes, with a dynamic range \(that is, the ratio between the smallest and largest numbers that can be represented\) in excess of $$10^{600}$$.

However, operations with floating point variables can take significantly more time than the same operations with integer variables on a microcontroller; on the Nucleo, for instance, we noticed that an implementation with floating-point variables can take up to 35% more processing time than an equivalent implementation with integer variables

If we try to avoid floats, then we need to use some form of fixed-point representation for our quantities. Implementing algorithms in fixed point is truly an art, and a difficult one at that. In the rest of this section we will barely scratch the surface and give you some ideas on how to proceed.

## Fixed-point representation

The idea behind fixed point representations is to encode fractional number as integers, and assuming the position of the decimal point implicitly. 

In our case, let's start with a reasonable assumption: the audio samples produced by the soundcard are signed decimal numbers in the $$(-1, 1)$$open interval. How can we represent numbers in this interval via integers and, more importantly, how does this affect the way we perform computations? 

Since we are all more familiar with numbers in base 10, let's start with a 2-digit fixed point representation in base 10 for fractional numbers between -1 and 1. With this, for instance, the number 0.35 will be represented by the integer 35; more examples are shown in this table:

| decimal representation | 2-digit fixed-point representation |
| :--- | :--- |
| 0.35 | +35 |
| -0.2 | -20 |
| 0.1234 | +12 |
| 1.3 | +99 |

Note that since we can only have 2 digit, the number 0.1234 will have to be truncated to the representation 12. Similarly, we will not be able to encode numbers greater that 0.99 or smaller than -0.99, which will induce an _overflow_ in the representation. That's OK, a finite number of digits involves a loss of precision and this makes sense.

It is clear that in this representation we go from decimal numbers to integers by multiplying the decimal number by $$10^2 = 100$$ \(see the 2 in the exponent: that's our number of digits\) and taking the integer part of the result. Vice-versa, we can go back to the decimal representation by dividing the integer by 100.

We can also choose at one point to, say, _increase the precision_ of our representation. In this example, if we were to now use five digits, 

| decimal representation | 5-digit fixed-point representation |
| :--- | :--- |
| 0.35 | +35000 |
| -0.2 | -20000 |
| 0.1234 | +12340 |
| 1.3 | +99999 |

It's clear that we can convert a 2-digit representation into a 5-digit representation by adding three zeros \(i.e. by multiplying by 1000\), and vice versa. Note however that increasing the precision does not protect us against overflow: the maximum range of our variables does not change in fixed point, only the granularity of the representation.

## Fixed-point arithmetic

The tricky part with fixed-point is when we start to do math. Let's have a quick look at the basic principles, but remember that the topic is very vast!

### Multiplication

The first obvious thing is that when we multiply two 2-digit integers the result can take up to four digits. This case is however easy to handle because it only requires renormalization and it entails "simply" a loss of precision but not overflow.

For example, if we were to multiply two decimal numbers together, we would have something like:

$$
0.23 \times 0.31 = 0.0713 \approx 0.07
$$

If we use fixed-point representations, as long as the multiplication is carried out in double precision, we can renormalize to the original precision by dropping the two least significant digits:

$$
(+23) \times (+31) = +0713 \longrightarrow +07
$$

In the next section we will use this notation to indicate a multiplication in double precision followed by renormalization:

$$
[(+23) \times (+31)] = +07.
$$

### Addition

Addition is a bit trickier in the sense that the sum \(or difference\) of two numbers can result in overflow:

$$
0.72 + 0.55 = 1.27 > 1
$$

This is of course mirrored by the fixed-point representation

$$
(+72) + (+55) = 127 > 99
$$

The result is not representable with two digits and if we cap it at 99 we have a type of distortion that is very different from the rounding that we performed in the case of multiplication. 

There is no easy solution to this problem and often it all depends on writing the code that performs the required operations in a smart way that avoids overflow \(or makes it very unlikely\). For instance, suppose we want to compute the average of two numbers:

$$
\frac{a+b}{2}
$$

In theory, the way in which the average is computed makes no difference and, if$$a=0.72$$and$$b=0.55$$, we would usually compute the sum first and then divide by two:

$$
(0.72 + 0.55) \times 0.5 = 0.623.
$$

In fixed-point, however, the order of operations does matter. If we start with the sum, we immediately overflow and, assuming overflows are capped at their maximum value, we obtain

$$
[((+72) + (+55)) \times (+50)] = [(+99) \times (+50)] = 49
$$

which is a _really_ wrong value. On the other hand, suppose we compute the average as $$a/2 + b/2$$. In fixed point this becomes

$$
[(+72) \times (+50)] + [(+55) \times (+50)] =  (+36) + (+27) = (+63)
$$

which is a totally acceptable approximation of the average's true value!

## Two's complement

To encode signed integer in binary representation, the most  common format is known as [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement); this format allows for the normal addition operations to work across a range of representable positive and negative numbers. 

The main idea is that of addition of positive integers with truncated overflow and it originates in mechanical calculators whose digits roll around to zero after overflow. Suppose that we are using a single decimal digit; we can obviously use the digit to represent ten positive values from zero to 9. Alternatively, we can use the digits from 0 to 4 to represent themselves and map the digits from 5 to 9 to the negative numbers -5 to -1, in that order. With this "complement" representation, here is how addition now works:

| normal notation | complement notation |
| :--- | :--- |
| 1 + 1 = 2 | 1 + 1 = 2 |
| 2 + 1 = 3 | 2 + 1 = 3 |
| 2 - 2 = 0 | 2 + 8 = 0  \(10, with truncated overflow\) |
| 3 -2 = 1 | 3 + 8 = 1  \(11, with truncated overflow\) |
| -2 - 2 = -4 | 8 + 8 = 6  \(6 is mapped to -4\) |

The same concept can be extended to multi-digit numbers and, obviously, to binary digits, in which case the representation is called "two's complement". In the binary case, the notation is particularly simple: to negate a positive binary number we need to invert all its digits and add one. For instance, using 4 bits, the decimal value 4 is 0100; the value -4 is therefore 1011 + 0001 = 1100. With this, 4 - 4 = 0100 + 1100 = \(1\)0000 = 0

Note that in two's complement notation, the value of the leading bits indicates the sign of the number, with zeros for positive quantities and ones for negatives. With 16-bit words adn using hexadecimal notation, for instance, the numbers 0x0000 to 0x7FFF \(zero to 32767 in decimal\) have their most significant bit equal to zero and they represent positive quantities. Conversely, the number 0x8000 is mapped to -32768, 0x8001 to -32767, all the way up to 0xFFFF which represents -1. 

This representation allows for an easy implementation of divisions by powers of two as right shifts: when dividing by two, we simply need to shift the word to the right by one, making sure to extend the value of the most significant bit. Consider four-bit words for simplicity:

| decimal | binary two's complement |
| :--- | :--- |
| 4 / 4 = 1 | 0100 &gt;&gt; 2 = 0001 |
| -4 / 4 = -1 | 1100 &gt;&gt; 2 = 1111 |

In the C language standard, the implementation of a right shift is left undetermined as to the propagation of the sign bit. On the Nucleo, however, you can safely use right-shift renormalization since the shifts preserve the sign.


## Fixed-point programming in C

In the C language standard, the behavior of many numeric types is not standardized and is dependent on the compiler. To avoid unexpected side effects, in numerical programming it is customary to inlcude the header `<types.h>` in which numeric types are defined precisely. In our code we will use the following types:

* `int16_t`: 16-bit integers, two's complement representation. Ranges from -32768 (0x8000) to 32767 (0x7FFF). Zero is 0x0000 and -1 is 0xFFFF.
* `int32_t`: 32-bit integers, two's complement representation. Used to perform multiplications prior to rescaling. 

The provided types also include unsigned versions such as `uint8_t` and `uint16_t`, which can be used when the sign is not needed; for instance, an `uint16_t` ranges from zero to 65535 (0xFFFF).

Since we will be using integer arithmetic, here are a few practical rules that will be useful to understand and write the C code

* all audio samples, unless specified otherwise, are assumed to be values in the $$[-1, 1)$$ range and represented by 16-bit words, two's complement;
* to convert a floating point number $$x \in [-1, 1)$$ to its fixed-point represntation, use ```int16_t x16 = (int16_t)(x * 0x7FFF)```
* to multiply two 16-bit variables using double precision and rescaling, use ```int16_t z = (int16_t)(((int32_t)x * (int32_t)y)) >> 15)```
* careful with overflow when performing addition.

