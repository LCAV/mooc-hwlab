# Alien Voice

In this section we will implement the "alien voice" effect on the microcontroller. In the process, we will start to encounter some of the limitations that arise when trying to design a real-time DSP system.

As shown in the [Jupyter notebook](../introduction-vt.md), the alien voice effect is achieved simply by performing sinusoidal modulation to shift the voice spectrum up in frequency. Given a modulation frequency $$f_{c}$$ \(in Hz\) and an input sample $$x[n]$$ we can compute each output sample $$y[n]$$ instantaneously as:

$$
y[n] = x[n] \, \cos\left(2\pi \frac{f_{c}}{F_s} n \right) = x[n] \, \cos(\omega_c n),
$$

where $$F_s$$ is the input sampling frequency \(in Hz\). The modulation frequency must be kept small in order to preserve intelligibility; still, the resulting signal will be affected by _aliasing_ and other artifacts that we cannot really control.

As mentioned in the Jupyter notebook, this voice transformer is great for real-time applications as it requires only a single multiplication per sample. This means that, compared to the passthrough project, we will not have to write too much new code. But the devil, as they say, is in the details!

As in the previous chapter, text contained in highlighted boxes, as shown below, will require _**you**_ to determine the appropriate solution and implementation.

{% hint style="info" %}
TASK: This is a task for you!
{% endhint %}

```text
        .     .       .  .   . .   .   . .    +  .
          .     .  :     .    .. :. .___---------___.
               .  .   .    .  :.:. _".^ .^ ^.  '.. :"-_. .
            .  :       .  .  .:../:            . .^  :.:\.
                .   . :: +. :.:/: .   .    .        . . .:\
         .  :    .     . _ :::/:               .  ^ .  . .:\
          .. . .   . - : :.:./.                        .  .:\
          .      .     . :..|:                    .  .  ^. .:|
            .       . : : ..||        .                . . !:|
          .     . . . ::. ::\(                           . :)/
         .   .     : . : .:.|. ######              .#######::|
          :.. .  :-  : .:  ::|.#######           ..########:|
         .  .  .  ..  .  .. :\ ########          :######## :/
          .        .+ :: : -.:\ ########       . ########.:/
            .  .+   . . . . :.:\. #######       #######..:/
              :: . . . . ::.:..:.\           .   .   ..:/
           .   .   .  .. :  -::::.\.       | |     . .:/
              .  :  .  .  .-:.":.::.\             ..:/
         .      -.   . . . .: .:::.:.\.           .:/
        .   .   .  :      : ....::_:..:\   ___.  :/
           .   .  .   .:. .. .  .: :.:.:\       :/
             +   .   .   : . ::. :.:. .:.|\  .:/|
             .         +   .  .  ...:: ..|  --.:|
        .      . . .   .  .  . ... :..:.."(  ..)"
         .   .       .      :  .   .: ::/  .  .::\
```

[Source](http://www.asciiworld.com/-Aliens,128-.html).
