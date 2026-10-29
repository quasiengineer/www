---
title: "Calculating Pi digits on first Intel microprocessor (intel 4004)"
date: 2022-04-06 09:16:23
slug: calculating-pi-digits-on-first-intel-microprocessor-intel-4004
layout: post
image: images/2022/04/testing_cropped.jpg
description: "One day I thought about the performance gap between the first Intel processor and modern machines, and decided to research the subject in practice."
---

# Introduction

One day I thought about the performance gap between the first Intel processor and modern machines. Of course, we can try to do some estimations empirically - we know clock rate and how the pipeline is organized and what features intel 4004 CPU has (but it would not be standard FLOPS, because there was no embedded support for float numbers yet). But there are few details: architecture bit width (only 4 bits in comparison with modern 64 bits!), very limited instruction set (it's missing even basic logical operators like AND or XOR) and peripheral limitations (ROM/RAM accesses). So I decided to research the subject in practice. After some thinking, I chose π number calculation as a benchmark. After all, even ENIAC did that (in 1949) and achieved a new record for the amount of calculated digits[^1].

# Hardware limitations

Usually, we chose hardware, based on our goals. But in that case, we need to choose an algorithm, based on restrictions that come with intel 4004. So what do we have?

CPU is very basic and its instruction set has very few ALU operations - addition/subtraction of 4-bit operands, inversion (NOT operator), rotation left/right. And ... that's all, folks. No multiplications, division or any other logical operators.

Program counter register is 12-bit wide, so we can address 2^12 bytes (4 KiB). Most instructions are 1-byte instructions, but some of them can take 2 bytes of ROM. It means that our program should not be too long. That's a pity because we would have to implement a lot of missing arithmetical stuff by ourselves (there is no π calculation algorithm that involves only additions and subtractions).

Each MCS-4 RAM chip (4002) has 4 registers with 20 characters (4-bit). Additionally, there were 4 variants of the chip (actually just two metal options with special pin usage to extend it to 4) and the chip is activated when a data bus contains the corresponding chip number [^2]. CPU can control up to 8 banks (with a simple 3-to-8 demultiplexer, without it - just 4 banks). So the amount of memory that we can use is 8 banks *4 chips* 4 registers *20 characters* 4 bits = 10240 bits.

Ugh, not that much...

# Algorithm

There are plenty of formulas to calculate π. But based on our limitations (mostly RAM) I chose the spigot algorithm of Stan Wagon and Stanley Rabinowitz [^3]. It's simple enough for implementation - only integer division and we don't need bignum arithmetic (as for Machin-like formulas). So what is the idea behind that algorithm in simple words (if you don't want to read their wonderful paper)?

We would use [Horner's method](https://en.wikipedia.org/wiki/Horner%27s_method) (4) to evaluate [Leibniz π's formula](https://en.wikipedia.org/wiki/Leibniz_formula_for_%CF%80) (1) after [Euler transform](https://en.wikipedia.org/wiki/Binomial_transform#Euler_transform) (2) and another simple series transformation (3):

$$ \frac{\pi}{4}=\sum_{n=0}^\infty\frac{(-1)^{n}}{2n+1}\quad(1) $$

$$ \frac{\pi}4=\sum_{n=0}^\infty\frac{(n!)^22^{n-1}}{(2n+1)!}\quad(2) $$

$$ {\pi}=2\sum_{n=0}^\infty\frac{n!}{f(2n+1)},\ f(x)=1 \cdot 3 \cdot 5 \cdot \cdot \cdot x\quad(3) $$

$$ \frac{\pi}2=1+\frac{1}{3}+\frac{1\cdot2}{3\cdot5}+\frac{1\cdot2\cdot3}{3\cdot5\cdot7}+...=1+\frac{1}{3}\left(1+\frac{2}{5}\left(1+\frac{3}{7}(1+...)\right)\right)\quad(4) $$

What is Horner's scheme in particular? Let's apply that to regular decimal numbers (1.234, for example):

$$ 1+\frac{1}{10}\left(2+\frac{1}{10}\left(3+\frac{1}{10}(4)\right)\right) $$

If you compare that example with the π representation then you can notice a similarity. And can say that in that form we know what π is - `2.(2)` .

That's it! We got the π, just need to convert it into decimal form... But the caveat is that it's written in mixed-radix with base:

$$ c=(\frac{1}{3},\frac{2}{5},\frac{3}{7},...,\frac{n}{2n+1}) $$

We want to evaluate π digit-by-digit, and it's trivial for float numbers - print integer part of a number, multiply fractional part by 10, print integer part of the result, drop integer part (set it to 0), multiply fractional part again, and repeat it until we reach necessary precision. That steps could be applied for numbers in mixed-radix as well:

$$ x = {(a_0; a_1, ...,a_n)}_c\\ 10\cdot{x}={(10\cdot{a_0}; 10\cdot{a_1}, ...,10\cdot{a_n})}_c $$

Each digit of c-based number (**a<sub>i</sub>** ) should belongs to alphabet **{ 0, 1, ..., (2i + 1) - 1 } = { 0, 1, ..., 2i }** . It's non-obvious, but you can see that alphabets for digits for base **p/q** and **q/p** are equal and are **{ 0, 1, ..., max(p-1, q-1) }** [^4]. So after multiplication we need to normalize number to be sure that it's in valid form (authors of algorithm calls it *regular representation* ).

We are interested in numbers in the form (0; a<sub>1</sub> , a<sub>2</sub> , a<sub>3</sub> ,... ) with zero as the integer part. For the decimal base that number would be in the range `[0, 1)` . But for our mixed radix, this range is extended to `[0, 2)` . The maximum number in base **c** with zero integer part would be (0; 2, 4, 6, ...) and equals 2 (it is also proved by the original paper).

All that means that when we got (a<sub>0</sub> ; a<sub>1</sub> , a<sub>2</sub> , a<sub>3</sub> ,...) we can't just print **a<sub>0</sub>** in decimal form, because the fractional part can be more than 1, and can increase decimal representation by 1. Let's say we have **d<sub>k</sub> = 5** , but on the next step (after multiplication by 10) we can have **d<sub>k+1</sub> = 12** , so we need to perform decimal carry and print 6 instead of 5.

Moreover, it's not enough to have just a single buffered digit (to print it on the next step when we can say if the next digit is more than 1 and do we need to increase the buffered digit by one or not), because we can meet cascade carry and at some point would have to transform `...29999` to `...30000` because another digit would be more than 10.

# HLL implementation

The original article contains Pascal's listing, but it's not a great choice for modern coding, so I have used JavaScript to implement some reference solution, that would be the basis for the intel 4004 version.

Trivial solution:

```javascript
const N = 1000;
const LEN = Math.floor((10 * N) / 3) + 1;

const a = Array(LEN);
let previousDigit = 2;
let nineCount = 0;

for (let i = 0; i < LEN; i++) {
  a[i] = 2;
}

let printed = 0;
while (printed < N) {
  // multiply each digit by 10
  for (let i = 1; i < LEN; i++) {
    a[i] = a[i] * 10;
  }

  // normalize representation
  //   each digit should be in range 0..2i, so carry extra rank to higher digit
  let carry = 0;
  for (let i = LEN - 1; i > 0; i--) {
    const denominator = i, numerator = 2 * i + 1;
    const x = a[i] + carry;
    a[i] = x % numerator
    carry = Math.floor(x / numerator) * denominator;
  }

  // latest carry would be integer part of current number (and sequental digit of Pi)
  const digitFromCarry = Math.floor(carry / 10);
  const nextDigit = carry % 10;

  // if current digit is 9, then we can't decide if we would have cascade carry
  if (nextDigit === 9) {
    nineCount++;
    continue;
  }

  // print previous digit (because now we knows if current digit is more than 10 or not)
  const currentDigit = previousDigit + digitFromCarry;
  process.stdout.write(currentDigit.toString());
  printed++;

  // if previous digit is followed by 9s, then print them (or 0s, if we have cascade carry)
  for (let i = 0; i < nineCount; i++) {
    process.stdout.write((digitFromCarry === 0 ? 9 : 0).toString());
    printed++;
  }
  nineCount = 0;

  previousDigit = nextDigit;
}
```

Printing could be left as-is, but we can improve the main normalization loop performance (it's always good to improve loop performance!):

1. Unite multiple and normalization steps into a single loop.
2. Instead of numerator calculation on each loop iteration we can just decrease it by 2 from the initial value.
3. Javascript could not perform integer division and modulo operations as a single operator, but of course, we can calculate quotient and reminder in our division implementation.

Taking those optimizations into account, we can update the normalization loop in such a way:

```javascript
  for (let i = LEN - 1, numerator = (2 * LEN - 1); i > 0; i--, numerator -= 2) {
    const x = a[i] * 10 + carry;
    a[i] = x % numerator
    carry = Math.floor(x / numerator) * i;
  }
```

# How many digits can we calculate?

As you can notice, we have to keep the c-based number in memory and step-by-step extract decimal digits from it. But memory is limited. To simplify calculations we are going to work with numbers, which are based on 4-bit RAM words. In theory, we can store, for instance, 14-bit numbers, but it would be pretty inefficient, especially because of missing logical instructions from 4004 ISA. Another option to increase density is to have different bit widths for digits, first digits belong to a much smaller alphabet than the last digits. But again it adds too much complexity to the code.

In the original paper, there is the proof that if we want **N** decimal digits of π, then we need to work with number in base **c** that have at least `(10 * N / 3) + 1` digits. So based on the amount of memory that we have (10240 bits), we can calculate the achievable maximum amount of π's digits.

$$ \left(\left\lfloor\frac{10N}{3}\right\rfloor+1\right)\cdot\left\lceil\log_2{\left(2\left\lfloor\frac{10N}{3}\right\rfloor\right)}\right\rceil\leq10240 $$

Solving that inequation, we got **N = 279** and bit-width as 11 per element. As I mentioned above, I want to have bit width divisible by 4, 8-bit option gives us only 38 digits, but 12 bit allows us to calculate **255** digits of π. And we would have an extra 40 bits in memory for utility needs.

# Intel 4004 assembly implementation

It would be pretty hard to debug code on real hardware, so I decided to use an emulator first. There are few [decent](http://www.4004.com/) [projects](http://e4004.szyc.org/) . But I am used to more powerful IDEs with code highlighting, breakpoints and hotkeys. And I was going to write a not-that-small program. So at first i have developed [intel 4004 assembly](https://github.com/markablov/i40xx-asm) to compile source code into machine code and [browser-based emulator/debugger](https://github.com/markablov/i40xx-emu) .

![emulator](../../../images/2022/04/emulator-1024x473.png)

The instruction set for intel 4004 contains 45 instructions, more than a third of which are related to RAM and input/output. I am not going to describe all nuances of MCS-4 systems, there is a good users manual, even with code samples [^5], but I just want to mention a few interesting details about the architecture and how it affected the process of writing my program:

- CPU has 16 general-purpose 4-bit registers, that could be accessed as a single register or 8-bit register pair (just for a few instructions, all arithmetics is still 4-bit only) and a single 4-bit accumulator. So register value is limited by 0xF (15), but almost all our variables in the program would be much bigger than that, and because of that, we need to be extra careful with register allocation for variables. Especially because we have only 40 bits in memory that would not be occupied by the main array.
- There is no instruction to set the value of the single 4-bit register. You either can set the register pair (which affects the neighbour register) or load the value into the accumulator and then exchange the content of the accumulator and the desired register.
- Intel 4004 supports subroutines, but the call stack is only 3-level deep. That's not a critical restriction, because you have not enough ROM space to keep large programs, but a couple of times I have to re-organize code to fit that limitation.
- There is a carry flag, which is set to 1 if a result of addition is more than 15. But it is the very non-intuitive fact that this flag would be set to 0, if a borrow is necessary after subtraction instruction, and set to 1 otherwise (sic!). Carry/borrow is generated only due operations with the accumulator. An increment of the regular register would not set carry to 1, even if there would be an overflow.
- Some jump instructions (conditional and indirect jumps) are short jumps and can address only the 8-bit range, which means that the high 4-bit part of the 12-bit address stays as-is and you are doing a jump inside the same ROM page. You need to plan the layout of your program carefully and try to fit subroutine inside the same ROM page. For example, you can't conditionally jump from `0x123` to `0x245`, and only can perform jumps to `0x1XX` addresses.
- Jump instructions are even more fun! There are few exceptions when short jump instruction is located at the last bytes of the ROM page (`0xXFF` address). Jump increases high part of program control (even if jump supposed to be inside same ROM page). Very interesting to debug...
- To access a specific address at RAM you need to perform 3 operations - to select RAM bank, then select RAM chip/register/character combo and only after that you can execute the desired action - read, write, access IO port, ...
- You have only complement and rotate logical instructions.

A simple example of the intel 4004 program, which iterates through the whole RAM and writes data to each bank/chip/register set:

```nasm
  FIM r0, 0x80
loop_bank:
  // select bank
  LD rr1
  DCL
  // iterate through reg number
  FIM r1, 0x00
loop_reg:
  LDM 0
  XCH rr3
  SRC r1
  LD rr1
  WRM // write rr1 to [#rr1, #rr2, M0]
  WR0 // write rr1 to [#rr1, #rr2, S0]
  INC rr3
  SRC r1
  LD rr2
  WRM // write rr2 to [#rr1, #rr2, M1]
  WR1 // write rr2 to [#rr1, #rr2, S1]
  ISZ rr2, loop_reg
  // end of loop_reg
  INC rr1
  ISZ rr0, loop_bank
```

Of course, I would not post all 1500 lines of assembly code here, but you can check it [there](https://github.com/markablov/stm32-mcs4-memory-emulator/blob/master/loader/sample/pi.i4004) . Just want to give a brief description of subroutines.

To have some foundation, we need to implement missing single-word (4-bit) arithmetic: multiplication, division, shift operations. Then we have to support multi-word arithmetic (up to 20-bit wide) because the value of our variables would be more than 15. Multi-word division occupies a decent amount of source code. It is based on well-known[^6] algorithms[^7], so not worth describing it in detail.

I told you about optimization by having the denominator and the numerator as separate loop variables, but unfortunately, there was no room in registers/memory to keep extra 3 words, so I had to recalculate the denominator based on the numerator.

Another big chunk of source code belongs to operations with memory. We need to convert the element index in the array into a linear address first (that's simple `address = (denominator - 1) * 3` ) and then map the linear address into the physical layout of the RAM:

```javascript
bank index      = address / 320
register index  = (address % 320) / 20
character index = ((address % 320) % 20)
```

Also, you have to use different instructions to access the first 16 characters from the RAM index (`RDM` / `WRM` ) or to access 4 extra characters (`RDx` / `WRx` , where x is in range 0..3).

Other than that, the code should be straightforward (if you are familiar enough with the intel 4004 assembly).

# Run that program on hardware

Simulation is good, but my goal was to receive actual numbers about performance. So I need to build a real system, based on the Intel 4004 CPU. I don't want to build an authentic MCS-4 system, I just wanted to research the performance of the CPU, so I decided to have a hardware simulation of the 4002 RAM and the 4001 ROM chips. Even mid-end stm32 chips are powerful enough to do that job. Frank Buß did an excellent job with designing [schema](https://frank-buss.de/4004/index.html/) , based on the PIC16 microcontroller. So I used it with some small modifications:![schematic](../../../images/2022/04/schematic-1024x699.png) [Firmware](https://github.com/markablov/stm32-mcs4-memory-emulator) is pretty simple, it generates a 2-phase clock signal (by using two stm32 timers), listens to UART commands: to load ROM image into memory and to start CPU (via RESET signal) or to stop CPU and print statistics. Only one challenge (in firmware) was to be sure that any logic that should be executed due to a specific stage of the instruction cycle would fit into the 1000ns window.

Each instruction cycle takes 8 clock cycles and on each stage CPU either reads or writes the data bus, I spent a lot of time debugging different issues with signal timings, but in the end, I was able to run any programs on the board.

![final device](../../../images/2022/04/final_edited-1024x912.jpg)

# Results

The program had run successfully and output the correct digits of π. Great success. How much time did it take?

3 hours 31 minutes and 13 seconds. I also tracked amount of instruction cycles that were executed, and numbers are pretty similar that i got externally (from control software at PC): `0x3AFAB080 = 989507712 cycles * 8 * 1600ns` = around 12665s.

Just for comparison: on my old PC (one of the first Xeon generation) computation of 25 million digits of Pi took about a second. I don't even want to try to match that numbers. It was a pretty interesting experience by writing some program on the very first Intel processor. All code and schemas are open-source.

# References

[^1]: [Brian J. Shelburne, "The ENIAC's 1949 Determination of π"](https://ieeexplore.ieee.org/document/5999630)
[^2]: [Intel MCS-4 Data Sheet](https://en.wikichip.org/w/images/9/9e/Intel_MCS-4_Data_Sheet.pdf)
[^3]: [Stanley Rabinowitz and Stan Wagon, "A Spigot Algorithm for the Digits of π"](https://www.maa.org/sites/default/files/pdf/pubs/amm_supplements/Monthly_Reference_12.pdf)
[^4]: [Shigeki Akiyama, Christiane Frougny, Jacques Sakarovitch, "On the Representation of Numbers in a Rational Base"](<https://www.irif.fr/~cf/publications/AFSwords05.pdf>)
[^5]: [Intel MCS-4 Users Manual](https://en.wikichip.org/w/images/9/9b/MCS-4_Manual.pdf)
[^6]: [Donald E. Knuth, "Art of Computer Programming, Volume 2: Seminumerical Algorithms", 4.3. Multiple Precision Arithmetic](https://www.amazon.com/Art-Computer-Programming-Seminumerical-Algorithms/dp/0201896842)
[^7]: [Henry S. Warren, "Hacker's Delight"](https://www.amazon.com/Hackers-Delight-2nd-Henry-Warren/dp/0321842685)
