# Granular Synthesis

We will now implement the second voice transformation method described in the Jupyter notebook, namely pitch shifting via _granular synthesis_. 

While the alien voice transformer simply alters the quality of the voice, with a pitch shifter we will be able to move the perceived pitch of the speaker either down \(to create a "Darth Vader" sound\) or up \(to create a "Chipmunks" voice\).

We recommend you study the relevant section on the notebook carefully before proceeding, since the algorithmic details are going to be a bit trickier than what we have seen so far; please make sure you understand the theory and that you are comfortable with the offline implementation before tackling the real-time version of the transformer. 

May the Force be with you!

```text
                                   _.-'~~~~~~~~~~~~`-._
                                  /         ||         \
                                 /          ||          \
                                |           ||          |
                                | __________||__________|
                                |/ -------- \/ ------- \|
                               /     (     )  (     )    \
                              / \     ----- () -----    / \
                             /   \         /||\        /   \
                            /     \       /||||\      /     \
                           /       \     /||||||\    /       \
                          /_        \o=============o/        _\
                            `--...__|`-._       _.-'|__...--'
                                    |    `-----'    |
```

_Figure: Modified from_ [here](http://www.ascii-art.de/ascii/s/starwars.txt).

