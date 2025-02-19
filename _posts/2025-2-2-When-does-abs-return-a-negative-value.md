---
layout: post
title:  "When does abs() return a negative value?"
visible: 0
date:  2-2-2025
categories: programming
---

Sorry if this was an the unwelcome surprise. After all, we expect computers to be _predictable_? This abs() function, it has _one job_. And it can return a negative number? 

Well, if it comforts you it does not happen in all languages. But if you compile code with C, Java, or even Rust, the library function abs() has exactly one special case where the outcome is negative. Is this a big deal? Most likely no. But I thought it was a nice nerdy topic for my first blog post. My wife appeared to agree: "Well, I guess you've got to start small!".

Now, some of you may have correctly guessed this has to do with _[2's complement](https://en.wikipedia.org/wiki/Two's_complement)_ representation of signed integers. If you know how that works, free to skip the next section.   

# 2's complement

Two's complement integers work by allowing overflow, so I want to explain that briefly. Digital counters with a fixed number of position can overflow. A mechanical distance counter in a car will overflow after 99999.9 kilometers to 00000.0 kilometers. If we assume that the counter goes backwards if you drive the car in reverse, you could start with a counter of 0, back up a while, and watch the mechanical trip counter roll back to 99999.9. In processor registers, the number of digits is also fixed, and it behaves the same. If you count backward from  0x0000 0000 ([in hex](https://simple.wikipedia.org/wiki/Hexadecimal)) you will get 0xFFFF FFFF, the highest representable number. 

In 2's complement math 0xFFFF FFFF is the bit pattern used to represent -1. To get the other negative numbers, you can subtract any positive number from 0, and what you get due to the overflow is how you represent it in 2's complement. One important advantage is is that at the hardware level, there is little to no distinction between math with unsigned numbers and 2's complement. It saves instructions and transistors. 

For ease of implementation, any value with the highest bit set is considered negative. This means half of all the numbers you can represent are negative. The remaining half are all postive numbers AND zero. Therefore in 2's complement there are more _negative_ numbers than there are positive. This means that the lowest negative number is -2 147 483 648 (hex 0x8000 0000) has no positive counterpart.

# Negating -2 147 483 648 in 2's complement
If we want to negate the 32 bit number -2 147 483 648, we immediately run into a problem: +2 147 483 648 does not exist as a signed 32 bit int. A number that does exist is +2 147 483 647, represented as 0x7FFF FFFF. A sensible candiate for +2 147 483 648 would then be 0x8000 0000. That is the SAME VALUE as is used for -2 147 483 648. And that my friend, is the answer. abs(-2 147 483 648) returns -2 147 483 648. 

## The why and the how
Whi is this? Perhaps the simplest explanation is 0x8000 0000 is both 2 147 483 648 steps away from zero if you count forward, and it is also 2 147 483 648 steps away if you count backward (with overflow). Of course your processor does something more efficient than counting, but to me the "counting" argument makes intuitive sense.
But how does your processor do when  you ask it to negate a number? The "trick" is take the bits (ones and zeroes), invert all of them, and then add 1. It is easy to see that if you apply this to 0, you get 0 back. This works because "inverting all the bits" is the same as subtracting your number from 0xFFFF FFFF, which we know is -1. 

Now take -2 147 483 648, represented by 0x8000 0000. Flip all the bits to get 0x7FFF FFFF. Add 1, and you get 0x8000 0000 again.

## A somewhat mathematical argument
If this explanation annoys you, there is one more reason why there is no other way for this to work. 
A property of negation is "double negate is a no-op": negating a number twice produces again that number. With that property in mind, let's review all our numbers. 0 negates to itself.The numbers +1 to +2 147 483 647 that negate to -1 to -2 147 483 647 **and vice versa**. So they also have that property.

Lastly, we have -2 147 483 648. If we negate it, what can it become?
 - Certainly not 0. Then it it would not negate back to itself.
 - Nor any of the other "normal" nonzero numbers, because they negate to their counterparts.
 - Then the only option is that negating it will produce itself. Any other outcome would invalidate "double negate is a no-op". 

# How do programming languages deal with this?
So we agree 2's complement hardware has this edge case that causes abs(MIN_INT) to return MIN_INT. 

Now I got a bit curious. I never gave this much thought before, but now that I have, I was worried about compilers. I know compilers try to reason about your code. If you write y = abs(x), I might falsely assume that y is now always positive. 

But the compiler should not make that assumption, right?

The best thing is to check. I tried Compiler Explorer with the following C++ code:

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

I was a little perplexed at first. It has optimized away my print statement! A bug in the compiler?

Oh wait... signed integer overflow ... that is Undefined Behavior.

For those not in the know: "undefined behavior" means that long ago, the C/C++ language recognized that not all hardware neccessarily does the the same always. In particular there is a nearly extinct alternative to 2's complement that would behave very different. It turns out that the compiler is free to assume that signed overflow does not happen. And in this program it can optimize away the if branch completely, so that's what it does. 

Now C/C++ are performance oriented languages. Other languages might prioritize correctness. Several languages have variable length integers. They won't have this problem. 

| Language | default integer | What does abs(MIN_INT) do (in optimized builds |
|:--------|:-------:|:-------|
| C/C++   | 32 bit   |  returns MIN_INT (UB, [even in C++20](https://stackoverflow.com/a/57363573) |
| JavaScript | --  |  N/A: all numbers are fp doubles |
| Rust    | 32 bit   | [return MIN_INT](https://doc.rust-lang.org/stable/std/primitive.i32.html#method.abs)   |
| Java    | 32 bit   | return MIN_INT  ([see the specification](https://docs.oracle.com/javase/8/docs/api/java/lang/Math.html#abs-int-)) |
| Ruby    | variable length   | it just works |
| Python    | 32/64 bit | [runtime conversion to larger type](https://peps.python.org/pep-0237/) |
| C#    | 32 bit   | [throws OverflowException](https://learn.microsoft.com/en-us/dotnet/api/system.math.abs?view=netstandard-2.1#system-math-abs(system-int32))   |
 
With regards to specific languages: the C++20 standard has done away with a lot of UB, but -MIN_INT is still undefined behavior, which is why the compiler ignores that it can happen. I was surprised that Rust will let abs() return MIN_INT. It's a kind of silent failure. Rust does have unsigned_abs(), which returns an unsigned number, and therefore it will always work. IMO that should have been the default. 

I have done lots and lots of C#. It throws exceptions. I did not know about it until I researched this article, so I guess it doesn't happen that often. I kind of hate it. Exceptions are slow, and, with the exception of divide by zero, I don't expect exceptions from math code.   



# Bonus chatter
As I mentioned you could of course subtract its value from 0. Alternatively you can: 
- Subtract 1 
- Flip all the bits
It does not really matter which way you do for the result. But you might be able to imagine that the above is less complicated to do in hardware. 




