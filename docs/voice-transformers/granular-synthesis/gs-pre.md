# The Formulas

The key technique behind simple pitch shifting is that of playing the audio file either more slowly \(to lower the pitch\) or faster \(to raise the pitch\). This is just like spinning an old record at a speed different than the nominal RPM value.

In the digital world we can always simulate the effect of changing an analog playing speed by using fractional resampling; for a refresher on this technique please refer to [Lecture 3.3.2 on Coursera](https://www.coursera.org/learn/dsp3/home/week/3). Resampling, however, has two problems:

* the pitch of speech is changed but so is the overall speed, which we do not want;
* the ratio of output to input samples for the operation is not one, so it cannot be implemented in real time.

To overcome these limitations, we can use _granular synthesis_: we split the input signal into chunks of a given length \(the _grains_\) and we perform resampling on each grain independently to produce a sequence of equal-length output grains.

### Grain rate vs length

In order to implement granular synthesis in real time we need to take into account the concepts of grain _length_ and grain _stride_. A grain should be long enough so that it contains enough pitched speech for resampling to work; but it should also be short enough so that it doesn't straddle too many different sounds in an utterance. Experimentally, the best results for speech are obtained using grains between 20 and 40ms.

The grain stride indicates the displacement in samples between successive grains and it is a function of grain length and of the _overlap_ between successive grains. With no overlap, the grain stride is equal to the grain length; however, overlap between neighboring grains is essential to reduce the artifacts due to the segmentation. Overlapping output grains are blended together using a _tapering window_; the window is designed so that it performs _linear interpolation_ between samples from overlapping grains.

Call $$\rho$$the amount of overlap \(as a percentage\) between neighboring grains. With $$\rho = 0$$there is no overlap whereas with $$\rho = 1$$all the samples in a grain overlap with another grain. The relationship between grain length $$L$$and grain stride $$S$$is $$L = (1+\rho)\,S$$. This is illustrated in the following figure for varying degrees of overlap and a stride of$$S=100$$samples; grains are represented using the shape of the appropriate tapering window:

![](../../.gitbook/assets/grains.jpg)

Note that the stride is constant for any amount of overlap and that each grain starts at the same instants independently of overlap; this is the key observation that will allow us to implement granular synthesis in real time.

### The grains' content

We can express the content of the $$k$$-th output grain as

$$
g_k[m] = x(kS + \alpha m), \qquad 0 \leq m < L
$$

where$$x(t)$$is the interpolated, continuous-time version of the input signal and$$\alpha$$is the sampling rate change factor \(with $$\alpha < 1$$for subsampling, i.e. to lower the pitch, and $$\alpha > 1$$for upsampling, i.e. to raise the pitch\). Note that the $$k$$-th grain starts at $$n=kS$$and is built using input data from $$n=kS$$as well.

In practice we will obviously perform local interpolation rather than full interpolation to continuous time, as explained in [Lecture 3.3.2 on Coursera](https://www.coursera.org/learn/dsp3/home/week/3). Let $$t=kS+\alpha m$$and set $$T = \lfloor T \rfloor$$and $$\tau = t - T$$; with this, the interpolation can be approximated as

$$
g_k[m] \approx (1-\tau)\,x[T] + \tau\, x[T+1].
$$

### Causality

Note that when we lower the voice's pitch \(i.e. we implement the "Darth Vader" voice transformer\), since $$\alpha < 1$$, the computation of the output grains is strictly causal, that is, at any point in time we only need to access past input samples. Indeed, when we oversample, only a fraction of the grain's data will be used to regenerate its content; if a grain's length is, say, 100, and we are lowering the frequency by $$\alpha=2/3$$, we will only need 2/3 of the grain's original data to build the new grain.

By contrast when we raise the pitch we are using _subsampling_, that is, samples are being discarded to create an output grain and so, to fill the grain, we will need to "look ahead" and borrow data from beyond the original grain's end boundary. The algorithm therefore is noncausal but, crucially, we can exactly quantify the amount of lookahead and handle it via buffering.

For instance, if we are raising the frequency by $$\alpha=3/2$$ and our grain length is, say, 100 samples, we will need a buffer of 50 "future" samples; this can be accomplished by accepting an additional processing delay of 50 samples. The difference between over- and under-sampling is clear when we look at the illustration in the notebook that shows the input sample index as a function of the output sample index:

![](../../.gitbook/assets/granular.jpg)

We will see in the next sections that buffering is required anyway in order to implement overlapping windows, so that the extra buffering required by subsampling will just be an extension of the general setup.

### The tapering window

The tapering window is as long as the grain and it is shaped so that the overlapping grains are linearly interpolated. The left sloping part of the window is $$W$$samples long, with $$W=L-S = \rho S.$$ The tapering weights are therefore expressed by the formula:

$$
w[n] = \begin{cases}
 n/W & 0 \leq n < W \\
1 & W \leq n < S 
\end{cases}
$$

### The output signal

The full output signal can be expressed in closed form by looking at the following picture, which shows the periodic pattern of overlapping grains:

![](../../.gitbook/assets/pattern.jpg)

Any output index $$n$$can be written as:

$$
n = kS + m, \qquad k, m \in \mathbb{Z},  0 \leq m < S;
$$

$$k$$ is the index of the current grain and $$m$$ is the index of the sample _within_ the current grain. Note that the sample at $$n$$ is also the sample with index $$S+m$$ with respect to the _previous_ grain. With this, the output at $$n$$is the sum of the sample number $$m$$ from the current grain plus the sample number $$S+m$$from the previous grain; both samples are weighed by the linear tapering slope$$w[\cdot]$$:

$$
y[n] = (1-w[m])\,g_{k-1}[S+m] + w[m]\,g_k[m]
$$

### Buffering

Consider once again the grain computation pattern, periodic with period $$S$$; let's use the index $$m$$ to indicate the current position _inside_ the current pattern; as $$m$$ goes from zero to $$S$$ we need to compute:

* $$g_k[m]$$ for all values of $$m$$
* $$g_{k-1}[S+m]$$ for $$0 \leq m < W$$ \(the tail of the previous grain\).

Which audio samples do we need to have access to at any given time? Without loss of generality, consider the grain for $$k = 0$$as in the following figure:

![](../../.gitbook/assets/grains2.jpg)

We need to compute:

* $$g_0[m] = x(\alpha m)$$ for $$0 \leq m < S$$
* $$g_{-1}[S+m] = x(\alpha m + (\alpha - 1)\,S)$$ for $$0 \leq m < W$$

If $$\alpha \leq 1$$ both expressions are causal so that we can use a standard buffer to store past values. The size of the buffer is determined by "how far" in the past we need to reach; in the limit, for $$\alpha$$ close to zero, we need to access $$x(-S)$$ from $$m=W$$ when we compute the end of the tapering section, so that, in the worst case, the buffer must be as long as the grain size $$L = S+W$$. The overall processing delay of the voice changer in this case is equal to the size of the DMA transfer.

If $$\alpha > 1$$, on the other hand, we need to also access _future_ samples; this is of course not possible but we can circumvent the problem by introducing a larger processing delay. This is achieved by moving the input data pointer in the buffer further ahead with respect to the output data pointer. The maximum displacement between the current time and the future sample that we need takes place for $$m = W$$ \(i.e., at the end of the tapering slope\) for which:

$$
[\alpha m  + (\alpha -1)\,S - m ]_{m=W} = (\alpha - 1)(S+W) = (\alpha - 1)\,L
$$

By offsetting the input and output pointers by $$D = (\alpha -1)\,L$$ samples, we can raise the pitch of the voice by $$\alpha$$ at the price of a processing delay equal to $$D$$samples.

{% hint style="info" %}
TASK 1: Determine the maximum range for $$\alpha$$ if the size of the audio buffer is equal to the grain size $$L$$.
{% endhint %}

## **Solutions**

{% tabs %}
{% tab title="Anti-spoiler Tab" %}
Are you ready to see the answer? :\)
{% endtab %}

{% tab title="Task 1" %}
We have already seen that for $$\alpha < 1$$ we need a causal buffer whose maximum length is equal to $$L$$. For $$\alpha > 1$$ the needed buffer size is $$(1-\alpha)\,L$$, so if the maximum buffer size is $$L$$, we must have $$0 \leq \alpha < 2$$.
{% endtab %}
{% endtabs %}

