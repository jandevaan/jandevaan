---
layout: post
title:  "When does abs() return a negative value?"
visible: 0
date:  2-2-2025
categories: programming
---


Yes, it happens that in most language, the function abs() on a 32/64 bit int has a special case where the outcome is negative. There is no need for alarm: the software you wrote yesterday will work today just as well. Some of you may have a hunch when this happens, you can skim over the first half: The second half is about how languages deal with it.  

the function abs(), for integers, is normally implemented as x < 0 ? -x : x.
This works most of the time: On today's computers, a 32 bit can have the values -2 147 483 648 up to 2 147 483 647. Notice that the lowest value does not have a positive counterpart. If you negate it, it should become +2 147 483 648. However, there is no signed 32 bit integer with that value. 

It turns out that it becomes -2 147 483 648. 
 
Let's consider what two's complement will do by default. 

For those not familar with [2's complement](https://en.wikipedia.org/wiki/Two's_complement), I'll explain the idea.  
If you have the (unsigned, 8 bit) value 0x00 and subtract 1 from it, it underflows to the value 0xFF. This is what 2's complement defines as -1. If you subtract 1 once again, you get 0xFE which is -2. It is really that simple: binary processors all simply inderflow if you subtract something from 0, and whatever you get is considered "-something". It turns out that, for add/subtract, signed and unsigned values can be treated identically. That is called 2's complement. All values with the highest bit set will be considered negative. Exactly half of all represented values will be negative values. The other half are positive values including 0. Because 0 is lumped in with the positive values, we have one fewer positive values (as I mentioned earlier).  


In 32 bit,  the value -2 147 483 648 is represented as 0x8000 0000. To calculate its absolute value, we must negate it, so we flip all the bits, to get 0x7FFF FFFF. Add 1, that becomes 0x8000 0000. That ... exactly what we started with. For the minimum int value, negating its value returns that same value.  This is especcially surprising because it also means that abs() will return negative values. 

The rule for abs() is: test the number's sign. If negative, negate the number, otherwise return it as-is.  
How does one negate a 2's complement number? You can subtract the number from 0 (obviously). In hardware, you cal also flip all the bits, and then add a 1. If you start with 1, flipping all the bits will produce 0xFE. Adding one produces  0xFF, which is -1, as expected.  

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
