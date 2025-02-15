---
layout: post
title:  "When does abs() return a negative value?"
visible: 0
date:  2-2-2025
categories: programming
---

Sorry if this was an the unwelcome surprise.

We have come to expect computers to be ... perhaps not infallible, but at least ... predictable? This abs() function, it has _one job_. And it returns negative numbers? 

Well, if it comforts you it does not happen in all languages. But in several mainstream languages (such as  C, Java and Rust), the library function abs() on a 32/64 bit int has a special case where the outcome is negative. Is this a big deal? Most likely no. But I thought it was a nice nerdy topic for my first blog post. My wife appeared to agree: "Well, I guess you've got to start small!".

Now, some of you may have correctly guessed this has to do with _2's complement_. If so, feel free to skip the next section.   

# 2's complement
Your computer uses [2's complement](https://en.wikipedia.org/wiki/Two's_complement). In 2's complement, a 32 bit number can have the values -2 147 483 648 up to 2 147 483 647.
abs() is simple: it will negate all the negative numbers. So the lowest value does not have a positive counterpart. If you negate it, it should become +2 147 483 648. There is your problem: in 32 bit, there is no such value. Now what happens in stead? 

## What 2's complement means
Any digital counter with a fixed number of position will overflow default. If you have mechanical 3 digit "trip length" counter in your car, it will overflow after 999 km/miles to 000 km/miles. The counter can also go backwards if you drive the car in reverse. Now, 2's complement defines -1 similar to that: start at 0x00000000 and count backward to 0xFFFFFFF (in hex). 
For ease of implementation, any number with the highest bit set is considered negative. 

In 32 bit,  the value -2 147 483 648 is represented as 0x8000 0000. To calculate its absolute value, we must negate it, so we flip all the bits, to get 0x7FFF FFFF. Add 1, that becomes 0x8000 0000. That ... exactly what we started with. For the minimum int value, negating its value returns that same value.  This is especcially surprising because it also means that abs() will return negative values. 

The rule for abs() is: test the number's sign. If negative, negate the number, otherwise return it as-is.  
How does one negate a 2's complement number? You can subtract the number from 0 (obviously). In hardware, you cal also flip all the bits, and then add a 1. If you start with 1, flipping all the bits will produce 0xFE. Adding one produces  0xFF, which is -1, as expected.  

# How do programming languages deal with this?
This is of course not neccesarily what your code does. This is what the hardware does. Not all programming languages use hardware registers directly for variables. For instance, in Javascript, all numbers are floating point, and in that case this problem does not occur. Python and Haskell uses unbounded size integers by default. 
Rust was a surprise. In debug mode it panics if you invoke abs(MIN_INT), but in a release build you get MIN_INT. Java simply defines that abs(MIN_INT) shall return MIN_INT. 

One language consistently throws an exception: C#. That was funny: at some point I was a little worried that I never was aware of this edge case, but the good news is I have been programming in C# for twentysomething years and I never saw this exception. 

We were not done with programming languages. 
Let's turn to C and C++. C and C++ compilers try to reason about your code in order to optimize it. 
Now, what would happen if wrote a program that checked if abs() returned a negative number. 
I would expect that that the test would not be optimized away: Although it is not obvious that abs() can return negative values, compiler writers are really smart.

So I godbolted  the following C++ code:

    void Foo(int i)
    {
          if (abs(i) < 0)
          {
               cout << "abs(" << i <<") produced a negative number!";
          }
    }

When you compile that with optimizations, the function was implemented as 

    Foo(int):
            ret

I was a little perplexed at first. Is this a bug in the compiler? 
But after a few minutes I realized... 

Undefined Behavior.

This may be enough explanation for some, but for the rest: The C and C++ standards have "Undefined Behavior" in situations where computer hardware could be different for different architectures. At the time C was designed, 1's complement was also in use, and 1's complement overflow produces different results. 
As you may have noticed abs(MIN_INT) causes overflow. Now overflow is undefined, and the compiler is allowed to assume that it does not happen. As a result the compiler has license to assume that abs() will always be 0 or bigger, so my code may be deleted.
But you might argue, didn't C++ 17 standardize 2's complement? 
Well yes they have, except for abs(MIN_INT). That is still undefined behavior. Don't ask me why. 

Writing this article I realized there is a better way: let abs() return an unsigned int. Of course the programmer can cast it to a signed int. But it may make some of them curious why the return value is unsinged, and it might trigger them to Read The Manual. comparison with constant.
