# Signal levels

## Gain <a id="gain"></a>

One thing that you might have noticed from the passthrough example is that the output signal is not very loud. To correct this, we can add a gain factor to the `process`function that multiplies each signal sample by a constant. In order to take advantage of the architecture of the microcontroller's internal multiplier, it is recommended to use factors that are a power of 2 since in this case a multiplication corresponds to a simple binary shift of the integer values to the left.

## Removing the DC offset <a id="removing_dc"></a>

In general, in DSP applications we assume that input signals are zero mean. This is no different in the case of our microphone, so that, if there is no sound, we expect a sequence of zeros. If you actually look at the input samples, however, you will almost certainly find out that this is not so. In fact, the internal circuitry in the microphone almost always adds a voltage offset, and sometimes different microphones \(of the same manufacturer\) will have different offsets. We typically call this shift in the waveform a [DC offset/noise/bias](https://en.wikipedia.org/wiki/DC_bias).

DC offsets are highly undesirable since they limit the dynamic range of our system; in other words, we are "wasting" binary digits on a constant that serves no purpose.

{% hint style="info" %}
TASK 1: From your passthrough implementation, determine the value of the offset. Is it significant compared to the range of the microphone?

_Hint: put a breakpoint in the process function; then with the debug tool, check the content of the input buffer._
{% endhint %}

We have talked about DC offset removal in [Lecture 2.2.3](https://www.coursera.org/learn/dsp2/lecture/JcNy2/2-2-3-intuitive-iir-designs) in the [second DSP course](https://www.coursera.org/learn/dsp2/). Recall that a DC component corresponds to a nonzero frequency value at $$\omega=0$$so the idea is to use a filter with a zero in $$\omega = 0.$$A very simple example is the so-called FIR "DC notch" whose CCDE is simply

$$
y[n] = x[n] - x[n-1].
$$

Unfortunately this filter has the very poor frequency response shown here and, while good as a first approximation, it is not really recommended if audio quality is important to you.

![Frequency response of the FIR DC notch](../.gitbook/assets/dcnotch.jpg)

A better filter is obtained by using a an IIR DC notch which, while marginally more expensive computationally, provides a much flatter frequency response over the audio frequency band. We will talk more about the DC notch and filter implementation [later](filters.md).

![Frequency response of the IIR DC notch](../.gitbook/assets/image.png)

## Tasks solutions

{% tabs %}
{% tab title="Anti-spoiler tab" %}
Are you sure you are ready to see the solution? ;\)
{% endtab %}

{% tab title="Task 1" %}


When the code is running, you can double click on any line number to add a breakpoint.

We suggest you to ad a breakpoint at line 430:

![](../.gitbook/assets/screenshot-2019-10-10-at-16.29.32-1.png)

If the micro-controller is connected and a debug session is ongoing, you will see a change in the software and the following list:

![](../.gitbook/assets/screenshot-2019-10-10-at-16.32.28-1.png)

It is the hierarchy of the function executed by the micro-controller, indeed main\(\) is the root. Please note that the button _Skip All Breakpoints_ should not be activated for the micro-controller to stop at the given line.

![](../.gitbook/assets/screenshot-2019-10-10-at-16.29.58.png)

It is then possible to right-click in the editor and press _Add Watch Expression_ you can now enter the name of the variable you want to explore and it will show up in the _Expression_ viewer panel. Unfold the array and you should see something close to this:

![](../.gitbook/assets/screenshot-2019-10-10-at-16.27.39.png)

Notice that even if the values are fluctuating, the average is around -1540. This is the offset that we where looking for. It is introduced by the microphone and can be variable from one sample to an other.
{% endtab %}
{% endtabs %}
