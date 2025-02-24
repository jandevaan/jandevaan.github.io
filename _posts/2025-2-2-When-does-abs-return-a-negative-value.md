---
layout: post
title:  "When does abs() return a negative value?"
visible: 0
date:  2-2-2025
categories: programming
---

Sorry if this was an the unwelcome surprise. After all, we expect computers to be _predictable_? This abs() function, it has _one job_. And it can return a negative number? 

Well, if it comforts you it does not happen in all languages. But if you compile (optimized) code with C, Java, or even Rust, the library function abs() has exactly one special case where the outcome is negative. Is this a big deal? Most likely no. But I thought it was a nice nerdy topic for my first blog post. My wife appeared to agree: "Well, I guess you've got to start small!". 

Now, some of you may have correctly guessed this has to do with _[2's complement](https://en.wikipedia.org/wiki/Two's_complement)_ representation of signed integers. If you know how that works, free to skip the next section.   

# 2's complement

Two's complement integers work by allowing overflow, so I want to explain that briefly. Digital counters with a fixed number of position can overflow. A mechanical distance counter in a car will overflow after 99999.9 kilometers to 00000.0 kilometers. If we assume that the counter goes backwards if you drive the car in reverse, you could start with a counter of 0, back up a while, and watch the mechanical trip counter roll back to 99999.9. In processor registers, the number of digits is also fixed, and it behaves the same. If you count backward from  0x0000 0000 ([in hex](https://simple.wikipedia.org/wiki/Hexadecimal)) you will get 0xFFFF FFFF, the highest representable number. 

In 2's complement math 0xFFFF FFFF is the bit pattern used to represent -1. To get the other negative numbers, you can subtract any positive number from 0, and what you get due to the overflow is how you represent it in 2's complement. One important advantage is is that at the hardware level, there is little to no distinction between math with unsigned numbers and 2's complement. It saves instructions and transistors. 

For ease of implementation, **any** value with the highest bit set is considered negative. This means half of all the numbers you can represent are negative. The remaining half are all postive numbers **and zero**. Therefore in 2's complement there are more _negative_ numbers than there are positive. The lowest negative number is -2 147 483 648 (hex 0x8000 0000) has no positive counterpart.

# Negating -2 147 483 648 in 2's complement
As said, if we want to negate the 32 bit number -2 147 483 648, it appears we can't: +2 147 483 648 does not exist as a signed 32 bit int. A number that **does** exist is +2 147 483 647, represented as 0x7FFF FFFF. A sensible candiate for +2 147 483 648 would be one higher. That would be 0x8000 0000. However, incrementing 0x7FFF FFFF overflow into the region reserved for negative numbers. So 0x8000 0000 is actually a negative number, and as it happens it is the SAME VALUE as is used for -2 147 483 648. And that my friend, is the answer: abs(-2 147 483 648) returns -2 147 483 648. 

## The why and the how
Why do we end up there? Perhaps the simplest explanation is 0x8000 0000 is both 2 147 483 648 steps away from zero if you count forward, and it is also 2 147 483 648 steps away if you count backward (with overflow). Your processor does something more efficient than counting, but to me the "counting" argument makes intuitive sense.
But what does your processor do when you ask it to negate a number? The "trick" is take the bits (ones and zeroes), invert all of them, and then add 1. It is easy to see that if you apply this to 0, you get 0 back. This works because "inverting all the bits" is the same as subtracting your number from 0xFFFF FFFF. Because 0xFFFF FFFF is -1, we only have to add 1 to get our final result.

Now take -2 147 483 648, represented by 0x8000 0000. Flip all the bits to get 0x7FFF FFFF. Add 1, and you get 0x8000 0000 again.

## A somewhat mathematical argument
If this explanation annoys you, there is one more reason why there is no other way for this to work. 
A property of negation is "double negate is a no-op": negating a number twice produces again that number. With that property in mind, let's review all our numbers. 0 negates to itself.The numbers +1 to +2 147 483 647 that negate to -1 to -2 147 483 647 **and vice versa**. So they also have that property.

Lastly, we have -2 147 483 648. If we negate it, what can it become?
 - Certainly not 0. Then it it would not negate back to itself.
 - Nor any of the other "normal" nonzero numbers, because they negate to their counterparts.
 - Then the only option is that negating it will produce itself. Any other outcome would invalidate "double negate is a no-op". 

# How do programming languages deal with this?
So we agree that 2's complement hardware has this edge case that causes abs(MIN_INT) to return MIN_INT. A programming language can choose to grudgingly accept this, or it can try to somehow fix this. But fixing it will mean adding costly checks to a very simple operation. How doe they deal with it?

Secondly, I was wondering how this affects optimization. Compilers for C and C++ will likely not add checks, but on top of that they will reason about your code, and simplify it based on what they know about your variables. So I was curious. Would they assume abs(x) is always positive? I would expect that won't be fooled. 

But let's try.  I tried Compiler Explorer with the following C++ code:

    void Foo(int i)
    {
          if (abs(i) < 0)
          {
               cout << "abs(" << i <<") produced a negative number!";
          }
    }

When you compile that **with optimizations**, the function was implemented as 

    Foo(int):
            ret

Just do nothing and return. It has optimized away my print statement! A bug in the compiler? It took me a minute to realise. 

I was causing ... signed integer overflow ... that is **Undefined Behavior**.

For those not in the know: "undefined behavior" (UB) means that a long time ago, the C/C++ languages recognized that not all hardware neccessarily uses 2's complement. In particular there is a nearly extinct alternative to 2's complement that behaves different. C and C++ allow the compiler to assume UB does not happen. And in this program the compiler concludes that abs() can only happen due to signed integer overflow. That is undefined behavior. So it is not allowed to happen, and the compiler assumes that it does not. It follows from that that abs() can't return negative numbers, so the compiler can remove the if branch.

Lots of people are angry about UB. 

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

  
A more lighthearted explanation: Causing "Undefined behavior" is like randomly activating the [Improbabilty drive](https://hitchhikers.fandom.com/wiki/Infinite_Improbability_Drive). "Without proper programming **anything** could happen!" - Zaphod Beeblebrox. 
 




