+++
draft = true
date = "2017-02-18T15:39:51-05:00"
title = "Writing a Gameboy emulator in D: Part 1 - Registers"

+++

# About the Gameboy CPU

The Gameboy uses a custom 8-bit Sharp LR35902 CPU (henceforth referred to as the "Gameboy CPU"), which uses a modified version of the [Zilog Z80](https://en.wikipedia.org/wiki/Zilog_Z80) instruction set, which itself is based on the [Intel 8080](https://en.wikipedia.org/wiki/Intel_8080).

The registers of the Gameboy CPU are identical to those of the 8080. Each main register is 8 bits wide, though in many instructions the regsiters can be accessed as groups of two adjacent registers, and are then treated as 16-bit; for example, the register _AF_ is a combination of _A_ and _F_ with the bits of _A_ used as the most significant and _F_ as least significant. A table of the registers is seen below.

$$
\begin{array}{r l}
0..7 & 8..15 \\\\\
\\hline
\mathrm{A} & \mathrm{F} \\\\\
\mathrm{B} & \mathrm{C} \\\\\
\mathrm{D} & \mathrm{E} \\\\\
\mathrm{H} & \mathrm{L} \\\\\
\rlap{\text{SP}} \\\\\
\rlap{\text{PC}}
\end{array}
$$

The reason that the register to the right of _A_ is called _F_ and not _B_ is that _F_ is used for flags set as side effects of certain operations. Because of this, _F_ isn't typically accessable in the same way that the other registers are.

The observant will notice that for _SP_ and _PC_, the letters are right next to each other while the others are spaced out; 
this is not a typo, as these registers are special in that they are always treated as 16 bits wide. _SP_ indicates the stack pointer and _PC_ indicates the program counter.
This registers typically cannot be modified the same as others can, only being accessable using special operations.

Another "special" register is _HL_ -- while this register is treated the same as the normal registers in that the 8-bit halves are registers on their own, _HL_ is often used in operations as a pointer to memory.

# Representing the registers in D
Due to the way that the registers can be accessed individually or grouped by two, there are quite a few ways that differnt Gameboy emulators choose to represent them.

Many emulators written in Java and C++ encapsulate each register pair within an object, holding either two 8-bit integers or one 16-bit integer, with helper methods to set the halves or the whole, which usually will use bit manipulation tricks as [type-punning](https://en.wikipedia.org/wiki/Type_punning) isn't possible or is undefined.

When I started this emulator project, I used C++, and part of the reason that I decided to switch to using D was how hacky my workaround for storing registers without needing bit manipulation was.
Because using unions for type punning is undefined behavior in C++ (it's actually allowed in C), I had to resort to trickier methods. In order to work around this (using a method that is probably worse than using the undefined behavior), I used some pointer and casting magic: I allocated an array of 8-bit integers large enough to fit all of the registers, and created pointers to 8-bit and 16-bit integers to refer to the individual registers and register groups respectively. This is very hacky, and, without looking too far into the C++ specification, I would assume that this trick would result in different values when reading the 16-bit register groups on host machines with different endianness.

So, can D do this better? Despite not being very familiar with the D "way" of doing things (this is my first project in D), I have come up with a way that is at least better than what I was doing in C++.

### Type punning using unions

I mentioned type punning using unions already, when talking about how it wasn't possible in C++. As far as I can tell from skimming the web, punning with unions is defined in D, and it is the best way that I can think of to represent the registers. I'm not sure if this will cause issues with different endianness (it probably will), but I have no easy way to check, and I'm only worried about x86 for now. If endianness matters, I'll update this blog entry.

If you are unfamiliar with unions, they essentially allow you to use a single variable as multiple types. They are similar to structs, except instead of storing multiple values, they store a single value in multiple ways. Unions are the size of the largest member. Hopefully an example will say more than my poor attempt at explanation.

To explain type punning using unions, I will use the following example:

{{< highlight D >}}
union PunnedRegister {
    short ab; // Grouped 8-bit values

    struct { // In order of least significant to most significant
        byte b; // Right value
        byte a; // Left value
    }
}

void main() {
    PunnedRegister r;

    r.a = 0xA;
    r.b = 0xB;
    writefln("%04X", r.ab);
}
{{< /highlight >}}

This code outputs `0A0B` as expected.

The way that this works is that _PunnedRegister_ can either be accessed as single a 16-bit integer or as a struct of two 8-bit integers (which is anonymous, so the integers can be accessed transparently).
When accessed as one of the two 8-bit integers, either the left or right half of the 16-bit value is accessed -- when accessed as a 16-bit integer, the combination of the two 8-bit integers is accessed.

# The final representation

Using mixins to reduce the amount of repetitive code, this is how the registers will be represented in the emulator:

{{< highlight D >}}
alias reg16 = short;
alias reg8 = byte;

template Register(string firstHalf, string secondHalf) {
    const char[] Register = 
    "
    union {
        reg16 " ~ firstHalf ~ secondHalf ~ ";

        struct {
            reg8 " ~ secondHalf ~ ";
            reg8 " ~ firstHalf ~";
        }
    }
    ";
}

struct Registers {
    mixin(Register!("a", "f"));
    mixin(Register!("b", "c"));
    mixin(Register!("d", "e"));
    mixin(Register!("h", "l"));

    reg16 sp;
    reg16 pc;
}
{{< /highlight >}}

I also aliased short to reg16 and byte to reg8 in case I find it important to make them unsigned in the future or alter how they are stored in some other way. Unlike C/C++, it isn't necessary to change the alias on different platforms since type size is [guarenteed](https://dlang.org/spec/type.html).

Because the _SP_ and _PC_ have letters that are shared with individual registers, they can't use the same method of representation as the other registers.

The only downside that I've noticed so far is that my editor doesn't like to autocomplete the registers.

<script type="text/javascript"
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>
