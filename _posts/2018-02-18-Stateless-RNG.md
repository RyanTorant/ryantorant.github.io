---
layout: post
title: The search for a (kinda) stateless RNG
---

# Stateless (Pseudo) Random Number Generation. Wait, what? Stateless?
While that headline might seem like gibberish, bare with me for a moment. It'll make sense.
I'm going to assume here that you know what I'm talking about when I say PRNG or LCG, because explaining all that would be a post on its own :D (though [Wikipedia](https://en.wikipedia.org/wiki/Pseudorandom_number_generator) it's a good place to start)

Now, most PRNGs are basically a recurrent sequence that applies a more or less complex transformation to the previous value (or values) to output the new value, therefore it needs to store state, and it also needs a starting value. Thing is, that can be problematic if you are working on a multi-threaded environment. There are problems with how to update the state, and how to find seeds (the starting values) that are different enough between threads so that the generated numbers are of good quality.  
For example, LCGs don't behave well when going wide, that is, using a lot of consecutive seeds. Below it's 10.000 numbers generated with a 32 bits LCG (the coefficients were taken from [here](https://en.wikipedia.org/wiki/Linear_congruential_generator#Parameters_in_common_use)) 

![lcg_sequence]({{site.url}}{{site.baseurl}}/_posts/rng/lcg_10000.png) 

Looks random enough right? Well, let's try to generate a 200x200 image, where the columns use consecutive seeds 

![lcg_mat]({{site.url}}{{site.baseurl}}/_posts/rng/lcg_matrix.png)

We clearly have a problem now. If we could somehow get rid of the seed and state, then we would have no problems with concurrency, the problem is how. Enter the **rdtsc** instruction (from now on, this post is x86 / x86_64 exclusive, but the same ideas can apply to other architectures).

## RDTSC, our savior?
The [**rdtsc**](http://www.felixcloutier.com/x86/RDTSC.html) instruction is an instruction on the x86 and x86_64 architecture that reads the processors time-stamp. This time-stamp is incremented each clock cycle, and it's outside of the application, OS, or anyone's control. It'll increase every time. 
And that is exactly what we are going to exploit here, by using the time-stamp as our PRNG state. Now, we can't feed it as-is to for example an LCG, because it'll just linearly cycle between the values, looking like this 

![rdtsc_no_state]({{site.url}}{{site.baseurl}}/_posts/rng/lcg_no_state.png) 

or will it? Let's see how it actually looks like if we try to just grab the value of the instruction, multiply it by some value and then add it some other value (where those "some" values are taken from the same table as before) 

![rdtsc_direct]({{site.url}}{{site.baseurl}}/_posts/rng/rdtsc_direct.png) 

Interesting, what happened here? Sure, there are clear correlations between the values, but there is also some randomness. Remember I said that the time-stamp is incremented every time a clock hits? Well, that's the reason. You don't run a program in isolation nowadays, even if it's your only program running, the OS is full of tasks, so the number of cycles between two instructions of your program is not just higher than 1, it also has a truly unpredictable component, specially if external devices (like hard drives or GPUs) are involved. 
Great! So, now we know that **rdtsc** can give us some level of randomness, but how do we amplify it? Easy, just call it more than once! After you have your multiple results, there are plenty of ways to combine them, some of them better than others, but will get to that in a minute. For now, let's just grab 3 calls and multiply them. To further increase the chance of the OS getting in the way and adding variance, instead of doing 
```cpp
uint64_t x = rdtsc();
uint64_t y = rdtsc();
uint64_t z = rdtsc();

return x * y * z;
```
let's do 
```cpp
return rdtsc() * rdtsc() * rdtsc();
```
And just like that, our awful 200x200 image becomes this nicely random picture

![rdtsc_matrix]({{site.url}}{{site.baseurl}}/_posts/rng/rdtsc_matrix.png)

Also, using threads to create the different columns increases context switch, therefore forcing the OS to get more in the way, and ultimately adding more randomness! 

## But is it any good?
Now that's the real question. It depends on how do you define good. Is it visually random? Yeah, at least to me it looks visually indistinguishable from a bunch of true random numbers. Is it cryptographically secure? Probably not, plus it completely depends on the usage of the system, something that most likely makes it an easy target for attacks. 
But we can at least measure it up against other algorithms, like a standard LCG. 
For that we are going to use [*Dieharder*](https://webhome.phy.duke.edu/~rgb/General/dieharder.php), an updated version of the original battery of tests by Marsaglia, designed to strictly tests the randomness of a generator. 
For comparison, let's use a standard 32 bit LCG, and then different combinations of **rdtsc**. It's important to note that I'm using here a 64 bits system, so **rdtsc** returns a 64 bits number. Keeping only half the bits it's a common way to improve the results, and in any case, *Dieharder* assumes the input to be a stream of 32 bits unsigned integers. In the following table, *r* represents a call to rdtsc. If there are more than one on the same line, those are multiple calls (remember what we talked about earlier?). For the LCG and MT, the seed used is *chrono::high_resolution_clock::now().time_since_epoch().count()*

| Generator                  | Tests Percentage |
| -------------------------- | :--------------: |
| LCG                        |        49%       |
| r\*r                       |      23%         |
| r\*r\*r                    |      36%         |
| r*a + b                    |      16%         |
| r\*r (high bits)           |     32%        |
| r\*r\*r  (high bits)       |      68%         |
| r*a + b  (high bits)       |      24%         |
| r*r*(r*a + b)  (high bits)) |      74%         | 
| [MT19937](http://www.cplusplus.com/reference/random/mt19937/)|      98%        | 

Clearly, you shouldn't be using this for cryptographic applications, even the best case only passes 74% of the tests, but it does tells us that the r\*r\*r version is about as good as an LCG, and taking the high bits (just a right shift) makes it much better. While this talks about quality, there's another important factor missing and that's performance. Let's check that now by measuring how many millions of numbers per second we can generate 

| Generator                  |  MNumbers/s
| -------------------------- | :--------------: |
| LCG                        |        612       |
| r\*r                       |      51         |
| r\*r\*r                    |      34       |
| r*a + b                    |      103         |
| r\*r (high bits)           |      51      |
| r\*r\*r  (high bits)       |      34       |
| r\*a + b  (high bits)       |      103        |
| r\*r\*(r\*a + b)  (high bits)) |      34    | 
| [MT19937](http://www.cplusplus.com/reference/random/mt19937/)|    163       | 

## Wrapping it up
Now, with all the results on the table, we can see that if you are doing anything that requires high quality random numbers, or just want the fastest possible generator, the **rdtsc** thing it's not the best option, but it does solve the concurrency problems, and the *r\*r\*r* version taking the high bits it's usually fast enough, while providing even better quality than an LCG. Overall, I think it's an interesting trick to keep in mind, specially as it solves all those pesky concurrency problems. 
If you have anything to say, feel free to leave a comment below, I'll try to answer it as soon as possible