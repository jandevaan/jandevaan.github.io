---
layout: post
title:  "When does abs() return a negative value?"
visible: 0
date:  2-2-2025
categories: programming
---

What do you mean, you might ask... This `abs()` function, it has _one job_: making numbers positive. And it can return a negative number?

Well yes. But only for fixed size integers, and it does not happen in all languages. But if you compile (optimized) code with C, Java, or even Rust, the library function `abs()` (for integers) has _one special case_ where the outcome is negative. Is this a big deal? Most likely no. But I thought it was a nice topic for my first blog post. I tried to explain to my wife what I'd write about. She appeared to agree: "Well, I guess you've got to start small!". 

Now, some of you may have correctly guessed this has to do with _[2's complement](https://en.wikipedia.org/wiki/Two's_complement)_ representation of signed integers. If you know how that works, free to skip the next section. And to those same people: please don't hate on me for oversimplifying. I am trying to cater to a wide audience here.

# 2's complement

![a mechanical odometer at 99999.9](../images/Odometer_rollover.jpg)

Digital counters with a fixed number of position can overflow. The mechanical distance counter shown above overflows after 99999.9 kilometers to 00000.0 kilometers. Most mechanical counters go backwards if you drive the car in reverse: you  start with a counter of 0.0, back up a while, and watch the mechanical trip counter roll back to 99999.9. Integers in a processor register also have a fixed number of digits, and they have the same overflow behavior.

Suppose I have such a processor with unsigned integer registers. What happens if we calculate `1 - 0`? It will wrap around to the highest unsigned value. Now suppose we decide that the resulting value means "-1" from now on. We can similarly define -2,-3, ... etc. You might be puzzled why, but it turns out that this is a definition that not only defines negative numbers, but if we use them in integer math, they behave the same way as actual negative integers.


The common choice is to reserve half of the range for negative values. If the highest bit is set, we consider the number is negative. That means for a byte that the range goes from -128 up to 127 (inclusive). This isn't symmetrical: there are more negative numbers than there are positives! This is because in the positive half we also have reserved a space for the zero. 

This system can also be applied to 32 bits. For numbers of that size, binary values are difficult to read. So I use [hexadecimal](https://simple.wikipedia.org/wiki/Hexadecimal) to represent the bit pattern. [^16]

See the following table how that works out for the smallest and largest possible values:

| value | 32 bit representation (in hex)|
|:-----:|:--------------:|
|  -2 147 483 648 |  8000 0000 | 
|  -2 147 483 647 |  8000 0001 | 
| ... | ... |
| -2  | FFFF FFFE|
| -1  | FFFF FFFF|
| 0   | 0000 0000|
| +1  | 0000 0001|
| ... | ... |
|  +2 147 483 647 |  7FFF FFFF |
 

# Negating -2 147 483 648 in 2's complement
The title of the blog is about the abs() function. For a positive input abs() returns it's input. For a negative value it will return the input negated. So we must consider _negation_. 

Consider what happens if we negate -2 147 483 648. In the table above, we don't have +2 147 483 648. What would be the logical outcome? I think you'll agree the result would be next value after +2 147 483 647 (hex 7FFF FFFF). But that will wrap around to the negate values. And even worse, it would wrap around to 8000 0000. That represents -2 147 483 648, the value we started with. 

And that is our answer: **abs(-2 147 483 648) returns -2 147 483 648**. 

Negating is equivalent to "find the number equally far from zero, but in the opposite direction". Now recall that -2 147 483 648 is represented by hex 8000 0000. That is halfway the range of available 32 bit numbers, so it is equally far from zero if you walk forward or backward. That's why you get the same number when negating. 

Now, what your computer actually does to negate a number is "invert all bits and add 1". That is a smart algorithm, and also efficient to implement in hardware. I don't use here because it did not help to explain our edge case. But it is equivalent.    

# How do programming languages deal with this?

So, by now, I hope we agree that 2's complement hardware has this edge case. A programming language can choose to accept this, or it can try to fix this. Another question is how this edge case affects optimizations in performance focused languages like C and C++. 

Let's try.  I tried [Compiler Explorer (aka "godbolt")](https://abs.godbolt.org/z/YTETW4rY8) with the C++ code directly below. I've written it such that, if you assume abs() is always > 0, it can remove code. I expected that it would not do this. Compiler writers would not make naive assumptions right? This is the C++ code:

```cpp
void Foo(int i)
{
      if (abs(i) < 0)
      {
           cout << "abs(" << i <<") produced a negative number!";
      }
}
```
When you compile that **with optimizations**, the function compiles to:

```assembly
Foo(int):
        ret
```

 **It has optimized away my print statement!** A bug in the compiler? 
It took me a minute to realise that because the abs() function _overflows_ and I just rediscovered **Undefined Behavior**.

For those unfamiliar: "undefined behavior" (UB) in this case means that a long time ago, the C/C++ languages recognized that not all hardware neccessarily uses 2's complement. In particular there is a [nearly extinct alternative](https://en.wikipedia.org/wiki/Ones'_complement) that behaves different. C/C++ dit not want to exclude such platforms, so C/C++ was careful to identify that where the two implementations would differ, the result is undefined.  

Signed overflow is exactly when you get different results, so signed overflow in C and C++ is UB[^1]. And in this program the compiler sees that abs() can only return a negative value due to such overflow. So it is not allowed to happen, and the compiler assumes that it does not. It follows from that that abs() can't return negative numbers, so the compiler can remove the if branch.

Lots of people are angry about how today's compilers handle UB. However by treating "signed overflow" as undefined, compilers can use 64 bit registers as loop counters even when you defined the loop counter as 32 bit. I learned that from [this writeup](https://gist.github.com/rygorous/e0f055bfb74e3d5f0af20690759de5a7#file-gistfile1-txt) by "ryg" (Fabian Giesen). 

Now C/C++ are performance oriented languages, and this notion of UB. Here are some examples:

| Language | default integer | What does abs(MIN_INT) do? <BR>(optimized) |
|:-------:|:-------:|:-------|
| C/C++   | 32 bit   | [UB] returns MIN_INT (UB, [even in C++20](https://stackoverflow.com/a/57363573) |
| JavaScript | --  |  N/A: all numbers are fp doubles |
| Rust    | 32 bit   | [return MIN_INT](https://doc.rust-lang.org/stable/std/primitive.i32.html#method.abs)  (panics in debug mode) |
| Java    | 32 bit   | return MIN_INT  ([see the specification](https://docs.oracle.com/javase/8/docs/api/java/lang/Math.html#abs-int-)) |
| Ruby    | variable length   | it just works |
| Python    | 32/64 bit | [works (by converting to larger type)](https://peps.python.org/pep-0237/) |
| C#    | 32 bit   | [throws OverflowException](https://learn.microsoft.com/en-us/dotnet/api/system.math.abs?view=netstandard-2.1#system-math-abs(system-int32))   |
| Zig   | 32 bit   | [return the (correct) value as unsigned](https://ziglang.org/documentation/master/#abs)  |
 
With regards to specific languages: 
Script languages tend to avoid this problem: there is usually a weak typesystem and no need for fast solutions. As far as I know, 
For both Rust and Java, negating MIN_INT returns MIN_INT, but unlike C/C++, the compiler will not assume the value is always positive.
The C++20 standard has done away with a lot of UB, but -MIN_INT is still undefined behavior, which is why the compiler is still allowed to ignore that it can happen. 
C# does an exception! I do a lot of C#, and I never knew this. So I guess it does not happen that often. 

I really really like Zig's solution: _just make the return type unsigned_. That always fits. I claim (without evidence) that I also thought of this possibility while writing the article. It's also easy to argue against: almost any integer operation can overflow, so why have such an ad-hoc fix for just abs(). But still: it would be truthful. It would be fast, and hopefully it will trigger some programmers to read the manual to figure out why.

## Appendix: negating in hardware

But what does your processor do when you ask it to negate a number? The "trick" is take the bits (ones and zeroes), invert all of them, and then add 1. It is easy to see that if you apply this to 0, you get 0 back. This works because "inverting all the bits" is the same as subtracting your number from 0xFFFF FFFF. Because 0xFFFF FFFF is -1, we only have to add 1 to get our final result.

 [^1]: I know that most (other) UB is now removed from C++, but it would be distracting to go into that here.
 [^16]: A hex numbers 0-15 represent 4 bits, and are written as 0123456789ABCDEF. The value -1 in 32 bit is written as hex FFFF FFFF. The first number with the highest bit set is hex 8000 0000, and that is the first negative number.
 



(I use [hex](https://simple.wikipedia.org/wiki/Hexadecimal) to represent the bit pattern, each digit corresponds to 4 bits)
If you count backward from  0x0000 0000 you will get 0xFFFF FFFF, the highest representable number. 

In 2's complement math 0xFFFF FFFF is the bit pattern used to represent -1. To get the other negative numbers, you can subtract any positive number from 0, and what you get due to the overflow is how you represent it's negative in 2's complement. An advantage of 2's complement is is that at the hardware level, there is little to no distinction between math with unsigned numbers and 2's complement signed integers. It saves instructions and transistors. 

For ease of implementation, **any** value with the highest bit set is considered negative. This means half of all the numbers you can represent are negative. The remaining half are all postive numbers **and zero**. Therefore in 2's complement there are more _negative_ numbers than there are positive. The lowest negative number, -2 147 483 648 (hex 0x8000 0000) has no positive counterpart.


Now take -2 147 483 648, represented by hex 8000 0000. Flip all the bits to get hex 7FFF FFFF. Add 1, and you get hex 8000 0000 again.
