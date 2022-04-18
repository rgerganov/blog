+++
title = "Reversing 2.4GHz remote control"
date = "2022-04-18T15:26:28+03:00"
tags = []
topics = []
description = ""
+++

I have an old project on Github called [rf-car](https://github.com/rgerganov/rf-car) for controlling a radio car with HackRF.
A few months ago, my daughter received a new RC car made by [Dickie Toys](https://www.amazon.de/-/en/Dickie-Flippy-Control-Rotation-Function/dp/B084PY44PN):

{{% fluid_img class="pure-u-1-1" src="/images/dickie-car.jpg" caption="fsk-car" %}}

This car was faster than the previous one and it was more fun to play with.
I thought it'd be great if I can add support for it in the [rf-car](https://github.com/rgerganov/rf-car) project and the fun began.

FCC docs
---
The car works on 2.4GHz which is clearly stated in the product description.
The remote control has FCC ID [NLB24054TX](https://fccid.io/NLB24054TX) and from the FCC documents I found that it uses [GFSK](https://en.wikipedia.org/wiki/Frequency-shift_keying#Gaussian_frequency-shift_keying) modulation.
From the internal photos of the remote control we can see it pretty much consists of four buttons connected to unlabeled IC:

{{% fluid_img class="pure-u-1-1" src="/images/remote-control-ic.png" caption="remote-control-ic" %}}

So the only useful information I got from the FCC docs is that the car uses GFSK modulation.

Signal analysis
---
The go-to tool for signal analysis is, of course, [inspectrum](https://github.com/miek/inspectrum).
I have successfully reverse engineered the ASK/OOK signal of my old car with it but I didn't have any experience with FSK signals.

This is how the recorded signal looks like when a single button on the remote is pressed:

[{{% fluid_img class="pure-u-1-1" src="/images/rc-fsk-signal.png" %}}](/images/rc-fsk-signal.png)

The remote is sending 9 batches of 16 packets (on the screenshot above only 3 of the 9 batches are shown).
The signal is GFSK, so we add a derived frequency plot (right click, add derived plot, add frequency plot):

[{{% fluid_img class="pure-u-1-1" src="/images/rc-freq-plot.png" %}}](/images/rc-freq-plot.png)

To analyse the frequency plot, we need to decrease the FFT size and increase the zoom.
This is how a single packet looks like:

[{{% fluid_img class="pure-u-1-1" src="/images/rc-freq-plot-zoomed.png" %}}](/images/rc-freq-plot-zoomed.png)

With the symbol markers we can easily find the symbol period which is 1us:

[{{% fluid_img class="pure-u-1-1" src="/images/rc-freq-plot-symbols.png" %}}](/images/rc-freq-plot-symbols.png)

Finally, we add a derived threshold plot from which we can extract the symbols as 0s and 1s (right click, extract symbols):

[{{% fluid_img class="pure-u-1-1" src="/images/rc-threshold-plot.png" %}}](/images/rc-threshold-plot.png)

There are 146 symbols (bits) in each packet. The packets in each batch are the same.

Signal generation
---
Now when we know the symbol rate (1M bits/sec), the modulation and the data being transferred, we can generate the signal ourselves.
The following code modulates the input in `data` using FSK and produces an IQ signal in the `out` vector:

```c++
float phase = 0;
int samples_per_bit = SAMPLE_RATE / SYMBOL_RATE;
for (auto b : data) {
    float freq = b ? FREQ1 : FREQ2;
    float phase_change_per_sample = (2*M_PI * freq) / SAMPLE_RATE;
    for (int i = 0; i < samples_per_bit; i++) {
        out.push_back(cos(phase));
        out.push_back(sin(phase));
        phase += phase_change_per_sample;
        if (phase > 2*M_PI) {
            phase -= 2*M_PI;
        }
    }
}
```

Two-way communication
---
One interesting detail is that the communication between the car and the remote control is actually two-way!
When the remote is switched on, it starts sending designated packets until a response from the car is received.
When response from the car is received, the remote sends a final packet to the car and then the car starts accepting commands from the remote.
It is very similar to the TCP handshake but much simpler because the packets are always the same.
In my implementation I opted to simply transmit the sync packets from the remote with some delay between them on program start.
This works fine as long as the car is already switched on and receives those packets.

Result
---
Here is a short demo of how it works:

{{< youtube mqSv-Nycy_4 >}}

