# Granular Synthesis

We will now implement the second voice transformation method described in the Jupyter notebook, namely granular synthesis. We recommend you study the relevant section on the notebook in detail before proceeding, since the algorithmic details are goint to be a bit trickier than what we have seen so far; please make sure you understand the theory and that you are comfortable with the offline implementation before tackling the real-time version of the transformer. 

We will proceed in incremental steps, in line with the following observations.

#### Darth Vader is easier than the Chipmunks

The algorithm that lowers the pitch of the voice \(the "Darth Vader" voice transformer\) is simpler to implement  than the one that raises the pitch \(the "Chipmunk" voice transformer\). This is because lowering the pitch involves _oversampling_ and this operation is causal,  requiring only past data values. 

We can look at it this way: in our granuar synthesis approach we are refilling the grains with an interpolated version of their content. When we oversample, only a fraction of the grain's data will be used to regenerate its content; if a grain's lenght is, say, 100, and we are lowering the frequency by 2/3, we will only need 2/3 of the grain's original data to build the new grain. 

On the other hand, if we raise the pitch, we are undersampling the original data and we will need to somehow "look ahead" and borrow data from the next grain to refill the current one. This is clear when we look at the illustration in the notebook that shows the input sample index as a function of the output sample index:

![input index vs output index for a\) the passthrough, b\) the Chipmun, c\) Darth Vade](../../.gitbook/assets/granular.jpg)

So the chipmunk voice will require some sort of buffering to accomodate its noncausal nature.

#### Overlapping windows 

* _increasing_ the number of samples so a resampled grain will only require samples that are local to the grain. 



