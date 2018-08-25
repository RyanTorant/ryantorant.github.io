---
layout: post
title: A short update on RDTSC-based RNGs
---

# Just a quick update : RDTSC as seed
In [this]{{ site.baseurl }}{% post_url 2018-03-18-Stateless-RNG %} post I talked about using the **rdtsc** instruction, that reads the processor timestamp, as a way to generate (pseudo) random numbers, but I just wanted to make a quick remark.

While using it directly to generate numbers might not be the best quality/performance idea, there's a point I forgot to talk about, and it's using **rdtsc** to seed whatever *PRNG* you like the most.  
The usual way to seed a *PRNG* consists on reading some high resolution clock, and if you think about it, the processor timestamp is **the** most accurate clock you can get! By using **rdtsc** you are guaranteed (unless you somehow take enough time between calls to make the 64 timestamp register roll back...) that two consecutive calls will get different values, so if for example you are creating different threads with unique seeds, you can be sure that the threads will have different seeds, something that may not happen if you are using the clock and launch the threads really close to each other.

So, the only question that remains is : *What's faster*. Luckily that's easy to test. On my system, a Xeon E5-2683v3, with 32GB of RAM and running Windows 10, compiling with Visual Studio 2017 with all optimizations enabled and targeting x64, I get the following results, in million of numbers per second.

| Generator | M / s |
|-----------|:-----------:|
| Clock(std::chrono)     | 1.80652e+07 numbers/s
RDSTC : 1.03936e+08 numbers/s
| RDTSC     | 