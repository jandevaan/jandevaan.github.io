---
layout: post
title:  "When does abs() return a negative value?"
visible: 0
date:  2-2-2025
categories: programming
---

Sorry if this was an the unwelcome surprise. We don't expect computers to be infallible, but at least _predictable_? This abs() function, it has _one job_. And it can return a negative number? 

Well, if it comforts you it does not happen in all languages. But if you compile code with C, Java, or even Rust, the library function abs() has exactly one special case where the outcome is negative. Is this a big deal? Most likely no. But I thought it was a nice nerdy topic for my first blog post. My wife appeared to agree: "Well, I guess you've got to start small!".

Now, some of you may have correctly guessed this has to do with _[2's complement](https://en.wikipedia.org/wiki/Two's_complement)_ representation of signed integers. Correct. If you know how that works, free to skip the next section.   

# 2's complement

Let's first talk about overflow. Digital counters with a fixed number of position can overflow. A mechanical distance counter in a car will overflow after 99999.9 kilometers to 00000.0 kilometers. If we assume that the counter goes backwards if you reverse, you could start with a counter of 0, back up a while, and watch the mechanical trip counter roll back to 99999.9. In binary arithmetic with fixed lenghts, it works the same. If you count backward from  0x0000 0000 ([in hex](https://simple.wikipedia.org/wiki/Hexadecimal)) you will get 0xFFFF FFFF, the highest representable number. 

In 2's complement math 0xFFFF FFFF is the bit pattern used to represent -1. To get the other negative numbers, you can subtract any positive number from 0, and what you get due to the overflow is how you represent it in 2's complement. For ease of implementation, any value with the highest bit set is considered negative. This means half of all the numbers you can represent are negative. The remaining half are all postive numbers AND zero. Therefore in 2's complement there are more _negative_ numbers than there are positive. This means that the lowest negative number is -2 147 483 648 (hex 0x8000 0000) has no positive counterpart. 

# Negate in 2's complement
If we want to negate -2 147 483 648, we immediately run into a problem: +2 147 483 648 does not exist as a signed 32 bit int. 
However for unsigned integers +2 147 483 648 **does** exist. It is at least plausible that that is the answer we'll get. And +2 147 483 648 is represented as ... 0x8000 0000. That is the SAME VALUE as is used for -2 147 483 648!

How can this be? Well the simplest explanation is 0x8000 0000 is both 2 147 483 648 steps away from zero if you count forward, and it is also 2 147 483 648 steps away if you count backward (with overflow). 

There is another way to negate numbers in 2's complement: Flip all the bits and add 1. 


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



# Bonus chatter
As I mentioned you could of course subtract its value from 0. Alternatively you can: 
- Subtract 1 
- Flip all the bits
It does not really matter which way you do for the result. But you might be able to imagine that the above is less complicated to do in hardware. 




