# 5.2 Implementation

In the [IPython notebook](https://nbviewer.jupyter.org/github/prandoni/COM303-Py3/blob/master/VoiceTransformer/VoiceTransformer.ipynb), we already have the code that implements pitch shifting via granular synthesis.

```python
def GS_pshift(x, f, G, overlap=0.2):
    N = len(x)
    y = np.zeros(N)
    # size of input given grain size and resampling factor
    igs = int(G * f + 0.5)
    win, stride = win_taper(G, overlap)
    for n in xrange(0, len(x) - max(igs, G), stride):
        w = resample(x[n:n+igs], f)
        y[n:n+G] += w * win

    return y
```

However, this implementation _**does not**_ work for a real-time scenario as the input buffer `x[n:n+igs]` and the output buffer `y[n:n+G]` do not have the same length! Also, we would like to perform operations on individual samples rather than vectors because the former is more representative of how we manipulate data/audio in C.

Nonetheless, having an implementation like the one above is still very useful before beginning a buffer-based implementation. It serves as a reference/sanity check before we attempt the \(typically\) more difficult buffer-based implementation.

In this chapter, we will guide you through the buffer-based implementation of this pitch shifting algorithm. What we present is certainly not the only approach but one we hope will help you for this exercise and for future real-time implementations.

We recommending cloning/downloading [the repository](https://github.com/LCAV/dsp-labs) so that you have the necessary files in the correct place. In it, you can find a [`utils.py`](https://github.com/LCAV/dsp-labs/blob/master/scripts/granular_synthesis/utils.py) file, which we will complete below.

## Save the MIPS! <a id="mips"></a>

A common term in DSP / Embedded Systems is _million instructions per seconds_ \(MIPS\) which reflects the computational cost of a particular implementation. One way to easily save on computational costs is by storing any values that are constant across consecutive frames. We already did this in the alien voice exercise when we stored sinusoid values in a _lookup table_ to perform amplitude modulation. The tradeoff that needs to be considered when storing lookup tables is memory.

So let's look at what should remain constant across consecutive frames so that we can store these values.

### User parameters

The parameters that need to be set by the user are:

1. Grain length in milliseconds.
2. Percentage of grain that overlaps with adjacent grains.
3. The pitch shift factor. In this exercise, we will limit ourselves to values below 1.0 for downward pitch shifts.

### Lookup tables

From our user \(and derived\) parameters, we can compute _three_ lookup tables to avoid repeated computations:

1. **Tapered window**: given our grain length in samples and the percentage overlap, this window will remain the same and can be stored in an array.
2. **Interpolation times**: for a given grain length in samples and pitch shift factor, we will sample our grains at the same \(possibly\) fractional samples:

   $$
   \mathbf{t} = f\cdot[0, 1, ..., G-1]
   $$

   where $$f$$ is a downwards pitch shift factor and $$G$$ is the length of the grain in samples. Moreover, we will perform linear interpolation \(as specified [in the previous section](effect_description.md#interp_times)\), which requires us to determine the largest integer $$N$$ smaller than the desired time instant $$t$$. Rather than determining this repeatedly for every sample, we can store the integer values in an array that has the same length as the grain length in samples $$[N_0, N_1, ..., N_{G-1}]$$.

3. **Interpolation amplitudes**: we can similarly store the associated amplitude value for each fractional sample \(as specified [in the previous section](effect_description.md#interp_amps)\) in an array $$[a_0, a_1, ..., a_{G-1}]$$.

In `utils.py`, we have already given you the function to compute the tapered window, as shown below.

```c
// Type definition to store the tapering window table
typedef struct win_taper {
	uint16_t win[GRAIN_LEN_SAMPLE];
	uint16_t stride;
} win_taper_t;

win_taper_t g_win;

// Table initialisation
void win_taper_init(win_taper_t * l_win, uint16_t N, volatile float a) {

	uint16_t R = (uint16_t) N * a / 2;
	uint16_t delta = MAX_INT / R;

	l_win->win[0] = 0;
	for (uint16_t i = 1; i < GRAIN_LEN_SAMPLE; i++) {
		if (i < R) {
			l_win->win[i] = l_win->win[i - 1] + delta;
		} else if ((i >= R) & (i < GRAIN_LEN_SAMPLE - R)) {
			l_win->win[i] = MAX_INT;
		} else {
			l_win->win[i] = l_win->win[i - 1] - delta;
		}
	}
	l_win->stride = N - R - 1;
}
```

Notice how we set the data type for the lookup table. This is something we would like to do in our Python code to emulate as much as possible how we will be implementing this algorithm in C.

{% hint style="info" %}
TASK 1: In the following code, complete the function to compute the lookup tables for the interpolation times and amplitudes.

_Hint: you need to complete the function at the beginning of the_ `for` _loop._
{% endhint %}

```c
// Lookup table to store the linear interpolation table
typedef struct interp_lookup_s {
	uint16_t sample[GRAIN_LEN_SAMPLE];
	uint16_t amplitude[GRAIN_LEN_SAMPLE];
} interp_lookup_t;

interp_lookup_t g_interp_val;

// Table initialisation
void build_linear_interp_table(interp_lookup_t * interp_val, uint16_t n_samples, float a) {
	float t;
	uint16_t td;

	for (int n = 0; n < n_samples; n++) {
		t = // TODO ...
		td = // TODO ...
		interp_val->sample[n] = // TODO ...
		interp_val->amplitude[n] = // TODO ...
	}
}

```

## State variables <a id="state_var"></a>

Now we should consider what values, such as samples or pointers to lookup tables \(as we saw in the alien voice effect\), need to be shared between consecutive frames, i.e. to notify the next buffer of the current state. A visual of the overlapping tapered grains will help us identify what needs to be "passed" between buffers.

![](../.gitbook/assets/viz_buffer.png)

_Figure: Visualizing buffers within overlapping grains._

Our stride length will determine our buffer length. In the figure above, our stride length is equivalent to the length between the lines labeled "buffer start" and "buffer end". Our new samples will be within this interval, as the samples between 0 and the "buffer start" line should have already been available in the previous buffer.

Moreover, the red line labeled "overlap start" indicates when the above buffer's grain will overlap with the next buffer's grain. Therefore, even though we have already received the samples between "overlap start" and "buffer end", _**we will not be able to output them yet since we still need to add them with weighted grain of the next buffer!**_ And computing the grain for the next buffer requires those samples after the "buffer end" line \(due to the resampling operation\).

The samples we will be able to output are between 0 and the "overlap start" line, also equivalent to the stride length. We therefore have a latency of:

$$
\text{latency} = \text{grain length} - \text{stride length},
$$

which is the length between 0 and "buffer start" and between "overlap start" and "buffer end".

{% hint style="info" %}
TASK 2: How many arrays do we need to pass/store between consecutive buffers? And how many samples should each of them have?

_Hint: due to the resampling operation, the next buffer's grain can only be computed when it contains all of the necessary samples._
{% endhint %}

## Allocating memory for intermediate values <a id="allocate_tmp"></a>

As we have several operations between our input and output samples, we will need some intermediate vectors to store a couple results. To see this, let's consider the "chain of events" for a single buffer:

1. Concatenate previous raw input samples with currently received samples:

   $$
   x_{concat} = [x_{prev}, \text{ } x_{current}].
   $$

2. This should create an array of our desired grain length which we then need to resample using our two linear interpolation lookup tables:

   $$
   \text{grain} = \text{resample}(x_{concat})
   $$

3. We could apply our tapered window at the same time as resampling or right after:

   $$
   \text{grain} = \text{win} \times \text{grain}.
   $$

4. Finally, we need to combine the previous grain with the current one at the relevant overlapping samples and write to the output buffer. We also need to update the array\(s\) we share between buffers.

For a clean implementation, it is hopefully clear from the description above that we need at least two intermediate arrays for our computations at each buffer. Moreover, this need to be allocated beforehand.

## Coding it up! <a id="code"></a>

Now you should have enough information to implement the real-time version of downwards pitch shifting with granular synthesis.

Below, we provide the _**incomplete**_  `process` function, which you can find in [this script](https://github.com/LCAV/dsp-labs/blob/master/scripts/granular_synthesis/granular_synthesis_incomplete.py). In this same file, you will also find the code to run granular synthesis on a fixed audio file.

{% hint style="info" %}
TASK 3: Complete the code below. The comments that have `TODO` mark where you will need to add code.

_Note: as this script relies on `utils.py` and `speech.wav` being in the correct relative location, it is useful to clone/download_ [_the repository_](https://github.com/LCAV/dsp-labs) _so that it is indeed so._
{% endhint %}

```c
// Variable definition
// Arrays for storing computations
uint16_t grain[GRAIN_LEN_SAMPLE];
uint16_t input_concat[GRAIN_LEN_SAMPLE];

// samples to store between frames
uint16_t x_overlap[OVERLAP_LEN];
uint16_t y_overlap[OVERLAP_LEN];

// Granular synthesis initialization
win_taper_init(&g_win, GRAIN_LEN_SAMPLE, GRAIN_OVERLAP_RATIO);
build_linear_interp_table(&g_interp_val, GRAIN_LEN_SAMPLE, SHIFT_FACTOR);
	
// The process function
void inline process(int16_t *bufferInStereo, int16_t *bufferOutStereo,
		uint16_t size) {

#define GAIN 8

	int32_t static x_1 = 0;
	int32_t x[FRAME_PER_BUFFER];
	int32_t x_filtered[FRAME_PER_BUFFER];
	int32_t y[FRAME_PER_BUFFER];

	// Take signal from left side
	for (uint16_t i = 0; i < size; i += 2) {
		x[i / 2] = bufferInStereo[i];
	}

	// High pass filtering initialization
	x_filtered[0] = x[0] - x_1;
	// Gain initialization
	x_filtered[0] *= GAIN;

	for (uint16_t i = 1; i < FRAME_PER_BUFFER; i++) {
		// High pass filtering
		x_filtered[i] = x[i] - x[i - 1];
		// Gain
		x_filtered[i] *= GAIN;
	}

	// Append samples from previous buffer
	for (int n = 0; n < GRAIN_LEN_SAMPLE; n++) {
		if (n < OVERLAP_LEN) {
			// TODO...
		} else {
			// TODO...
		}
	}

	// Resample
	for (int n = 0; n < GRAIN_LEN_SAMPLE; n++) {
		// TODO...
	}

	// Apply window
	for (int n = 0; n < GRAIN_LEN_SAMPLE; n++) {
		// TODO...
	}

	// Write to output
	for (int n = 0; n < GRAIN_LEN_SAMPLE; n++) {
		// deal with overlapping part
		if (n < OVERLAP_LEN) {
			// TODO...
		} else if (n < g_win.stride) {
			// TODO...
		} else {
			// update state variables
			// TODO...
		}
	}

	// Backup last sample for next buffer
	x_1 = x[FRAME_PER_BUFFER - 1];

	// Interleaved left and right
	for (uint16_t i = 0; i < size; i += 2) {
		bufferOutStereo[i] = (int16_t) y[i / 2];
		// Put signal on both side
		bufferOutStereo[i + 1] = (int16_t) y[i / 2];
	}
}

```

**Congrats on implementing granular synthesis pitch shifting! This is not a straightforward task, even in Python. But now that you have this code, the C implementation on the STM board should be much easier.**

\*\*\*\*

## Tasks solutions

{% tabs %}
{% tab title="Anti-spoiler tab" %}
Are you sure you are ready to see the solution? ;\)
{% endtab %}

{% tab title="Task 1" %}
According to the previously given equation the following three variables can be computed.

```c
// Lookup table to store the linear interpolation table
typedef struct interp_lookup_s {
	uint16_t sample[GRAIN_LEN_SAMPLE];
	uint16_t amplitude[GRAIN_LEN_SAMPLE];
} interp_lookup_t;

interp_lookup_t g_interp_val;

// Table initialisation
void build_linear_interp_table(interp_lookup_t * interp_val, uint16_t n_samples, float a) {
	float t;
	uint16_t td;

	for (int n = 0; n < n_samples; n++) {
		t = n * a;
		td = t;
		interp_val->sample[n] = td;
		interp_val->amplitude[n] = (uint16_t) ((1 - (t - td)) * MAX_INT);
	}
}
```
{% endtab %}

{% tab title="Task 2" %}
From one buffer to the next, we need to store two different arrays, the first is the input variables that need to be used to blend into the next tampering window, the length of this array is the number of the overlapping samples between two buffers.

We also need to blend the output buffer to the next buffer so we will add a second array of the same length. The length is calculated below.

Two initial definitions:

$$
grain\_length = 20[ms] = 640 [sample @ 32kHz]\newline
grain\_overlap = 0.3
$$

Which gives the following length:

$$
grain\_stride = ( grain\_length - (int)(grain\_length * grain\_overlap / 2) - 1 )\newline
overlap\_length = ( grain\_length - grain\_stride )
$$

The numerical application of that gives an overlap length of 97 samples at 32kHz sampling rate and 20ms of grains \(suited for voice\).
{% endtab %}

{% tab title="Task 3" %}
You then need to populate the process function according to our indications in the comments and the equation in this chapter.

The code is given below.

```c
// Variable definition
// Arrays for storing computations
uint16_t grain[GRAIN_LEN_SAMPLE];
uint16_t input_concat[GRAIN_LEN_SAMPLE];

// samples to store between frames
uint16_t x_overlap[OVERLAP_LEN];
uint16_t y_overlap[OVERLAP_LEN];

// Granular synthesis initialization
win_taper_init(&g_win, GRAIN_LEN_SAMPLE, GRAIN_OVERLAP_RATIO);
build_linear_interp_table(&g_interp_val, GRAIN_LEN_SAMPLE, SHIFT_FACTOR);
	
// The process function
void inline process(int16_t *bufferInStereo, int16_t *bufferOutStereo,
		uint16_t size) {

#define GAIN 8

	int32_t static x_1 = 0;
	int32_t x[FRAME_PER_BUFFER];
	int32_t x_filtered[FRAME_PER_BUFFER];
	int32_t y[FRAME_PER_BUFFER];

	// Take signal from left side
	for (uint16_t i = 0; i < size; i += 2) {
		x[i / 2] = bufferInStereo[i];
	}

	// High pass filtering initialization
	x_filtered[0] = x[0] - x_1;
	// Gain initialization
	x_filtered[0] *= GAIN;

	for (uint16_t i = 1; i < FRAME_PER_BUFFER; i++) {
		// High pass filtering
		x_filtered[i] = x[i] - x[i - 1];
		// Gain
		x_filtered[i] *= GAIN;
	}

	// Append samples from previous buffer
	for (int n = 0; n < GRAIN_LEN_SAMPLE; n++) {
		if (n < OVERLAP_LEN) {
			input_concat[n] = x_overlap[n];
		} else {
			input_concat[n] = x_filtered[n - OVERLAP_LEN];
		}
	}

	// Resample
	for (int n = 0; n < GRAIN_LEN_SAMPLE; n++) {
		grain[n] = (g_interp_val.amplitude[n] / MAX_INT
				* input_concat[g_interp_val.sample[n]]
				+ (1 - g_interp_val.amplitude[n] / MAX_INT)
						* input_concat[g_interp_val.sample[n] + 1]);
	}

	// Apply window
	for (int n = 0; n < GRAIN_LEN_SAMPLE; n++) {
		grain[n] = (g_win.win[n] * (int16_t) grain[n]) / MAX_INT;
	}

	// Write to output
	for (int n = 0; n < GRAIN_LEN_SAMPLE; n++) {
		// deal with overlapping part
		if (n < OVERLAP_LEN) {
			y[n] = grain[n] + y_overlap[n];
		} else if (n < g_win.stride) {
			y[n] = grain[n];
		} else {
			// update state variables
			x_overlap[n - g_win.stride] = x_filtered[n - OVERLAP_LEN];
			y_overlap[n - g_win.stride] = grain[n];
		}
	}

	// Backup last sample for next buffer
	x_1 = x[FRAME_PER_BUFFER - 1];

	// Interleaved left and right
	for (uint16_t i = 0; i < size; i += 2) {
		bufferOutStereo[i] = (int16_t) y[i / 2];
		// Put signal on both side
		bufferOutStereo[i + 1] = (int16_t) y[i / 2];
	}
}

```
{% endtab %}
{% endtabs %}

