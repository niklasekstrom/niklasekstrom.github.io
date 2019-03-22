---
layout: post
title:  "Local Hardware Clocks"
tag: [time]
date:   2019-03-22 10:00:00 +0200
---
In this post I'll write about how we model, and what we can assume, about local hardware clocks.

As described in the previous post, a clock is a periodic process (an oscillator) and a way to count the number of periods that have elapsed of that process (a counter).

In a normal PC, there are typically multiple clocks. For any of those clocks, the oscillator is usually a quartz crystal oscillator, and the oscillator is connected to a discreet counter register. When the computer is powered on the counter is reset to zero, and is after that incremented with one for each period of the signal output by the oscillator.

![Hardware clock](/assets/hardware_clock.png)

### Oscillator

The oscillator can be described using two parameters: the nominal frequency, fnom, and the frequency stability, rho. The frequency of the signal output from the oscillator, f, is bounded by:

fnom \* (1 - rho) ≤ f ≤ fnom \* (1 + rho) &nbsp; &nbsp; &nbsp; &nbsp; (1)

Note that the frequency relates the rate of the quartz oscillator to the rate of an ideal real time clock.

The frequency of an oscillator can change over time, both in the short term, due to temperature variations etc, and in the long term, due to aging. The rho parameter must be chosen so that despite any such changes in frequency, the bounds expressed by (1) must always hold.

### Counter

The counter used in the clock in a PC is discreet and only count integer periods of output from the oscillator. This can be a problem, as on average the count is a half period off the actual count.

There are a few ways to deal with this.

1. One way is to ignore the half period error, as there are other errors that are larger that has to be accounted for.
2. The following algorithm can be used:
   1. read the current value from the counter -> h
   2. read the counter repeatedly until the counter has a value greater than h
   3. we now know that at some point during the reading, the counter incremented from h to h+1, and hence h+1 is a correct value to return
3. Similarly to option 2, if we know that performing a read of the counter takes at least as long as the period time of the oscillator, then we know that at some point during the reading of the counter, a corresponding non-discreet counter would have the same value as the value returned.

Either of these options allow us to reason about the counter as if it had been real valued.

If we use the [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter) (TSC) found in PCs then we can use option 3, as the frequency of the TSC is that of the CPU execution frequency, and reading the TSC takes at least one cycle.
If another clock with a low frequency is used, you may want to use option 2, and if a clock with a medium to high frequency is used you may use option 1. Note that option 1 and 3 are essentially the same (ignore the error) but for different reasons; in option 3 there isn't any error.

### Normalized Count

After reading the counter of a clock, we often wish to normalize the count read. This is done simply by dividing the count by the oscillator's nominal frequency. When doing the divide we may want to consider rounding errors. One way to deal with that is to encode the resulting time as a pair (n, m) that denotes the rational number n/m, where n is the count read and m is the nominal frequency.

Now, assume that the hardware clock is read two times, first at real time t1 and then at real time t2, and the read normalized values are h1 and h2. Assume we used option 2 or 3 for dealing with the fact that the counter is discreet, so there is no error in the readings. The unit is of the normalized count is almost seconds, but not quite; remember that the time unit second refers to a certain number of periods of frequency emitted by a Caesium atom. The normalized count may advance up to 1 ± rho too fast or slow.

By looking at equation (1) and assuming that the frequency of the hardware clock is as fast and as slow as possible, we find that during the real time Δt = t2 - t1 the hardware clock's normalized count advances by Δh = h2 - h1:

Δt \* (1 - rho) ≤ Δh ≤ Δt \* (1 + rho) &nbsp; &nbsp; &nbsp; &nbsp; (2)

If we, after reading the hardware clock twice, want to know how much real time has passed between those two readings, we have Δh / (1 + rho) ≤ Δt ≤ Δh / (1 - rho). As 1 / (1 - x) ≈ (1 + x) for small values of x, and rho is typically in the order of 1e-4, we will in the following assume that:

Δh \* (1 - rho) ≤ Δt ≤ Δh \* (1 + rho) &nbsp; &nbsp; &nbsp; &nbsp; (3)

### Uncertainty

We now have formulas to compute back and forth between real time and local time (time as counted by the local hardware clock).
However, as we can't know the frequency of a clock exactly there will be some uncertainty in how much real time has passed between two events, when measured on a local hardware clock. This uncertainty is expressed by writing that t ± e seconds have passed, where e = Δh \* rho. All calculations that are based on such uncertainty should maintain the uncertainty bounds throughout the calculation.
